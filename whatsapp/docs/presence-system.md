# Presence System

Tracking online status, last seen, and typing indicators at 100M+ concurrent connections.

---

## Presence States

```mermaid
stateDiagram-v2
    [*] --> Offline
    Offline --> Online: Connect
    Online --> Away: Idle 5min
    Away --> Online: Activity
    Online --> Offline: Disconnect
    Away --> Offline: Disconnect

    Online --> Typing: Start typing
    Typing --> Online: Stop/Send
```

| State | Meaning | TTL |
|-------|---------|-----|
| Online | Active connection, recent activity | Heartbeat-based |
| Away | Connected but idle | After 5 min inactivity |
| Offline | No connection | Show last seen timestamp |
| Typing | Actively typing | 5 seconds TTL |

---

## Architecture

```mermaid
graph TB
    subgraph "Edge Layer"
        Edge1[Edge 1<br/>100K conn]
        Edge2[Edge 2<br/>100K conn]
        EdgeN[Edge N<br/>100K conn]
    end

    subgraph "Presence Aggregation"
        PA1[Presence Aggregator 1]
        PA2[Presence Aggregator 2]
        PAN[Presence Aggregator N]
    end

    subgraph "Storage"
        Redis[Redis Cluster<br/>Presence State]
        PubSub[Redis Pub/Sub<br/>Change Events]
    end

    subgraph "Distribution"
        FanOut[Fan-out Service]
    end

    Edge1 --> PA1
    Edge2 --> PA2
    EdgeN --> PAN

    PA1 --> Redis
    PA2 --> Redis
    PAN --> Redis

    Redis --> PubSub
    PubSub --> FanOut
    FanOut --> Edge1
    FanOut --> Edge2
    FanOut --> EdgeN
```

---

## Heartbeat Protocol

### Client Behavior

```
Every 30 seconds:
  - Send heartbeat with current state
  - Include activity flag (user recently active)

On activity:
  - Send immediate heartbeat if state changed

On disconnect:
  - Server detects via TCP keepalive or timeout
```

### Server Handling

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Edge Server
    participant PA as Presence Aggregator
    participant R as Redis

    C->>E: Heartbeat(active=true)
    E->>E: Update local conn state
    E->>PA: Batch update (every 5s)
    PA->>R: SETEX presence:{user_id} TTL=45s

    Note over R: If TTL expires, user is offline

    loop Every 30s
        C->>E: Heartbeat
    end

    Note over C,E: Connection drops

    E->>PA: User disconnected
    PA->>R: DEL presence:{user_id}
    PA->>R: SET last_seen:{user_id} timestamp
```

---

## Presence Data Model

### Redis Schema

```
# Current presence (volatile)
presence:{user_id}
  Type: Hash
  Fields:
    - state: "online" | "away"
    - edge_id: "edge-west-42"
    - connected_at: timestamp
  TTL: 45 seconds (refreshed by heartbeat)

# Last seen (persistent)
last_seen:{user_id}
  Type: String
  Value: Unix timestamp
  TTL: none

# Typing indicator (very volatile)
typing:{conversation_id}:{user_id}
  Type: String
  Value: "1"
  TTL: 5 seconds
```

---

## Subscription Model

### Problem

User A has 500 contacts. Subscribing to all 500 presence updates = massive fan-out.

### Solution: Active Conversation Subscription

```mermaid
graph LR
    subgraph "User A's App"
        ChatList[Chat List View]
        ChatOpen[Open Conversation]
    end

    subgraph "Subscriptions"
        ActiveSub[Active Chat Subscriptions<br/>Max 20]
        RecentSub[Recent Chats<br/>On chat list fetch]
    end

    ChatOpen -->|Subscribe| ActiveSub
    ChatList -->|Batch query| RecentSub
```

**Rules:**
1. Subscribe to presence only for open conversations
2. Max 20 concurrent subscriptions per client
3. Batch query presence when rendering chat list (not subscribe)
4. Unsubscribe when leaving conversation

---

## Presence Query vs Subscribe

### Query (Pull)

For chat list rendering:

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Edge
    participant R as Redis

    C->>E: GetPresence([user1, user2, ..., user50])
    E->>R: MGET presence:user1 presence:user2 ...
    R-->>E: [online, null, away, ...]
    E-->>C: {user1: online, user2: offline, ...}
```

### Subscribe (Push)

For active conversations:

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Edge
    participant PS as Pub/Sub

    C->>E: Subscribe(user_id=bob)
    E->>PS: SUBSCRIBE presence:bob

    Note over PS: Bob comes online

    PS->>E: Message(presence:bob, online)
    E->>C: PresenceUpdate(bob, online)

    Note over C: User leaves chat

    C->>E: Unsubscribe(user_id=bob)
    E->>PS: UNSUBSCRIBE presence:bob
```

---

## Typing Indicators

### Flow

```mermaid
sequenceDiagram
    participant A as Alice
    participant EA as Edge A
    participant R as Redis
    participant EB as Edge B
    participant B as Bob

    A->>EA: StartTyping(conv=alice-bob)
    EA->>R: SETEX typing:alice-bob:alice 5s
    R->>EB: PUBLISH typing:alice-bob
    EB->>B: ShowTyping(alice)

    Note over A: Alice stops or sends

    A->>EA: StopTyping(conv=alice-bob)
    EA->>R: DEL typing:alice-bob:alice
    R->>EB: PUBLISH typing:alice-bob
    EB->>B: HideTyping(alice)
```

### Optimizations

- Debounce on client: Only send after 500ms of typing
- Coalesce on server: Batch typing updates
- Auto-expire: 5s TTL handles forgotten cleanup
- Rate limit: Max 1 update per second per user per conversation

---

## Scaling Considerations

### Connection Distribution

| Entity | Count | Strategy |
|--------|-------|----------|
| Users | 100M concurrent | - |
| Edge servers | 1,000 | 100K connections each |
| Presence aggregators | 100 | Consistent hash by user_id |
| Redis cluster | 50 nodes | Sharded by user_id |

### Batching

Edge servers batch presence updates to aggregators:
- Collect updates for 5 seconds
- Send batch to reduce network calls
- Accept up to 5s staleness in presence

### Fan-out Limiting

```mermaid
graph TB
    subgraph "Without Limits"
        U[User with 5000 contacts]
        FO1[5000 presence notifications<br/>per status change]
    end

    subgraph "With Limits"
        U2[User with 5000 contacts]
        FO2[Active subscribers only<br/>~20 notifications]
    end
```

---

## Privacy Controls

### Last Seen Settings

```mermaid
graph LR
    Everyone[Everyone] --> Contacts[My Contacts] --> Nobody[Nobody]
```

### Implementation

```
User settings:
  last_seen_visibility: "everyone" | "contacts" | "nobody"

On presence query:
  if target.last_seen_visibility == "nobody":
    return null
  if target.last_seen_visibility == "contacts":
    if requester not in target.contacts:
      return null
  return last_seen
```

---

## Failure Handling

### Edge Server Crash

1. All connections on that edge drop
2. Other edges see heartbeat timeout
3. Users reconnect to different edge (DNS/LB)
4. New edge updates presence

### Redis Node Failure

1. Redis Cluster handles failover
2. Presence data on failed node lost
3. Clients re-establish presence on reconnect
4. Brief inconsistency acceptable (presence is soft state)

### Network Partition

- Presence may show stale (user appears online when offline)
- Heartbeat timeout eventually corrects
- Design for eventual consistency

---

## Monitoring

| Metric | Alert Threshold |
|--------|-----------------|
| Presence update latency p99 | > 500ms |
| Subscription fan-out count | > 1000 per update |
| Redis cluster memory usage | > 80% |
| Stale presence (missed heartbeat) | > 1% of users |
| Typing indicator delivery time | > 200ms |
