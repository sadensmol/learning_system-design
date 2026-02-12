# WhatsApp System Design - Requirements

## Purpose
Learning exercise for system design - designing a WhatsApp-scale global messaging platform.

---

## Business Goals
1. Enable real-time messaging between 1B+ users globally
2. Support 1:1 chats, group chats, voice/video calls, status updates, and media sharing
3. Guarantee exactly-once message delivery
4. Provide end-to-end encryption for all communications
5. Support offline message storage indefinitely until user comes online

---

## Actors

### Primary Actors
| Actor | Description |
|-------|-------------|
| End User | Person using WhatsApp on mobile/web to send messages, make calls, share media |
| Group Admin | User with elevated permissions to manage group membership and settings |

### System Actors
| Actor | Description |
|-------|-------------|
| Message Router | Routes messages between users across global infrastructure |
| Media Server | Handles upload, storage, and delivery of images, videos, documents |
| Notification Service | Sends push notifications to offline users |
| Encryption Service | Manages key exchange and encryption protocols |

---

## Functional Requirements

### F1: User Management
- User registration with phone number verification
- Profile management (name, photo, status)
- Contact sync and discovery
- Block/unblock users
- Last seen and online status

### F2: 1:1 Messaging
- Send/receive text messages in real-time
- Message delivery receipts (sent, delivered, read)
- Typing indicators
- Reply to specific messages
- Forward messages
- Delete messages (for me / for everyone)

### F3: Group Messaging
- Create groups with up to 1024 members
- Add/remove members (admin function)
- Group info and settings
- Mention specific users (@mentions)
- Admin-only messaging option

### F4: Media Sharing
- Share images, videos, audio, documents
- Image/video compression
- Media preview thumbnails
- Media auto-download settings

### F5: Voice/Video Calls
- 1:1 voice calls
- 1:1 video calls
- Group voice calls (up to 32 participants)
- Group video calls (up to 8 participants)
- Call quality adaptation based on network

### F6: Status Updates
- Post text/image/video status
- 24-hour auto-expiry
- View count tracking
- Privacy controls (who can see)

### F7: End-to-End Encryption
- All messages encrypted client-to-client
- Signal Protocol implementation
- Key verification between users
- Server cannot read message content

---

## Non-Functional Requirements

### Scale
| Metric | Target |
|--------|--------|
| Total users | 1B+ |
| Daily active users | 500M+ |
| Messages per day | 100B+ |
| Concurrent connections | 100M+ |
| Media uploads per day | 1B+ |

### Performance
| Metric | Target |
|--------|--------|
| Message delivery latency (online users) | < 100ms p99 |
| Message delivery latency (cross-region) | < 300ms p99 |
| Media upload time (1MB) | < 2s p95 |
| App startup time | < 2s |
| Connection establishment | < 500ms |

### Availability
| Metric | Target |
|--------|--------|
| Uptime | 99.99% (52 min downtime/year) |
| Message delivery guarantee | Exactly-once |
| Data durability | 99.999999999% (11 nines) |

### Geographic Distribution
- Global edge presence with data centers on all continents
- User data residency compliance (GDPR, etc.)
- Edge nodes for connection termination
- Regional message routing optimization

---

## Constraints

### Technical Constraints
- Mobile-first (battery and bandwidth efficiency critical)
- Must work on 2G/3G networks in emerging markets
- Client-side storage limitations
- Push notification platform dependencies (APNs, FCM)

### Business Constraints
- Privacy-focused (minimal metadata collection)
- No message content accessible to servers (E2E encryption)
- Regulatory compliance (data localization requirements)

---

## Hotspots (Areas Requiring Deep Design)

1. **Message ordering and exactly-once delivery** - Complex distributed systems problem
2. **Global presence system** - Tracking online status at scale
3. **E2E encryption key management** - Signal Protocol at WhatsApp scale
4. **Media storage and CDN** - Efficient global media distribution
5. **Group messaging fan-out** - Delivering to 1000+ members efficiently
6. **Offline message queue** - Indefinite storage with efficient retrieval
7. **Real-time connection management** - 100M+ concurrent WebSocket connections
8. **Cross-region message routing** - Optimizing for latency and reliability

---

## Success Criteria
1. Messages delivered within latency targets for 99.9% of cases
2. Zero message loss under any failure scenario
3. Seamless experience across network conditions
4. Encryption that is cryptographically secure and verifiable
5. Horizontal scalability to handle 10x growth
