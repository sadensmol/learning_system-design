# Rate Limiting: Complete Guide

> A beginner-friendly guide to rate limiting strategies and algorithms.

---

## Table of Contents

1. [What is Rate Limiting?](#1-what-is-rate-limiting)
2. [Rate Limiting Algorithms](#2-rate-limiting-algorithms)
   - [Token Bucket](#21-token-bucket)
   - [Leaky Bucket](#22-leaky-bucket)
   - [Fixed Window Counter](#23-fixed-window-counter)
   - [Sliding Window Log](#24-sliding-window-log)
   - [Sliding Window Counter](#25-sliding-window-counter)
3. [Algorithm Comparison](#3-algorithm-comparison)
4. [Implementation Examples](#4-implementation-examples)
5. [Distributed Rate Limiting](#5-distributed-rate-limiting)
6. [Rate Limiting in Practice](#6-rate-limiting-in-practice)
7. [Interview Discussion Points](#7-interview-discussion-points)

---

## 1. What is Rate Limiting?

**Rate limiting** is like a bouncer at a club - it controls how many people (requests) can enter within a certain time.

### Why Do We Need It?

Imagine your API as a restaurant kitchen:
- Kitchen can handle 100 orders per hour
- Suddenly 1000 orders come in at once
- Kitchen overwhelmed, all orders delayed or ruined

Rate limiting protects your service from:

| Threat | Real-World Analogy |
|--------|-------------------|
| **DoS attacks** | Someone calling your phone 1000 times per second |
| **Brute force** | Trying every possible password combination |
| **Scraping** | Copying your entire website automatically |
| **Buggy clients** | A broken app that sends requests in an infinite loop |
| **Unfair usage** | One user hogging all resources |

### The Three Questions of Rate Limiting

Every rate limiter needs to answer:

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  1. WHO is making the request?                              │
│     • IP address (123.45.67.89)                             │
│     • API key (sk_live_abc123)                              │
│     • User ID (user_456)                                    │
│                                                              │
│  2. HOW MANY requests are allowed?                          │
│     • 100 requests                                          │
│     • 1000 requests                                         │
│                                                              │
│  3. IN WHAT TIME PERIOD?                                    │
│     • Per second                                            │
│     • Per minute                                            │
│     • Per day                                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### HTTP Headers

When an API rate limits you, it tells you via HTTP headers:

```http
X-RateLimit-Limit: 100          # "You're allowed 100 requests"
X-RateLimit-Remaining: 45       # "You have 45 left"
X-RateLimit-Reset: 1706792400   # "Counter resets at this time"

# When you've exceeded the limit:
HTTP/1.1 429 Too Many Requests
Retry-After: 30                 # "Wait 30 seconds before trying again"
```

---

## 2. Rate Limiting Algorithms

### 2.1 Token Bucket

**The Arcade Token Analogy**

Imagine you're at an arcade:
- You receive tokens at a steady rate (10 tokens per minute)
- You can save up tokens in your pocket (up to 100 max)
- Each game costs 1 token
- No tokens? You wait until more arrive.

```
┌────────────────────────────────────────────────────────────┐
│                     TOKEN BUCKET                            │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   Tokens drip in constantly                                 │
│   (like a faucet filling a bucket)                         │
│              │                                              │
│              │  10 tokens added every second                │
│              ▼                                              │
│         ┌────────┐                                          │
│         │ ● ● ●  │                                          │
│         │ ● ● ●  │  ← Your bucket (holds max 100 tokens)   │
│         │ ● ●    │    Currently: 47 tokens                 │
│         └───┬────┘                                          │
│             │                                               │
│             ▼                                               │
│     Each request takes                                      │
│     1 token from bucket                                     │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**How It Works Step by Step:**

| Time | Tokens | Event | Result |
|------|--------|-------|--------|
| 0:00 | 100 | Start with full bucket | Ready to go |
| 0:00 | 0 | User makes 100 requests at once | All allowed (burst!) |
| 0:01 | 10 | 1 second passes, 10 tokens refill | |
| 0:01 | 5 | User makes 5 requests | Allowed |
| 0:02 | 15 | 1 second passes, 10 more tokens | |
| 0:10 | 100 | Bucket is full again | Capped at maximum |

**Key Properties:**

| Setting | Meaning | Example |
|---------|---------|---------|
| **Bucket size** | Maximum burst allowed | 100 = can do 100 requests instantly |
| **Refill rate** | Sustained rate over time | 10/sec = 10 requests per second long-term |

**Why Use Token Bucket?**

✅ Allows bursts (good for real usage patterns)
✅ Simple to understand and implement
✅ Memory efficient (just store 2 numbers: token count, last refill time)

❌ Burst at start might overwhelm backend if bucket is large

**Real-World Example:**

Twitter API: "You can make 300 requests per 15 minutes"
- Bucket size: 300 tokens
- Refill rate: 300 tokens / 15 minutes = 0.33 tokens/second
- You could tweet 300 times instantly, then wait 15 minutes

---

### 2.2 Leaky Bucket

**The Leaky Bucket Analogy**

Imagine a bucket with a small hole at the bottom:
- Requests pour in from the top (at any rate)
- Requests "leak" out the bottom at a constant rate
- If bucket overflows, excess requests are rejected

```
┌────────────────────────────────────────────────────────────┐
│                      LEAKY BUCKET                           │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   Requests arrive (sometimes fast, sometimes slow)          │
│        │ │    │     │ │ │ │  │                             │
│        ▼ ▼    ▼     ▼ ▼ ▼ ▼  ▼                             │
│      ┌─────────────────────────┐                           │
│      │  ◆  ◆  ◆  ◆  ◆  ◆  ◆   │                           │
│      │  ◆  ◆  ◆  ◆            │  ← Queue (bucket)         │
│      └──────────┬─────────────┘    Max size: 10 requests  │
│                 │                                           │
│                 │  Leak: exactly 5 requests/second         │
│                 ▼                                           │
│                 ◆ ◆ ◆ ◆ ◆                                  │
│                 │                                           │
│                 ▼                                           │
│           Processed at                                      │
│           constant rate                                     │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**The Key Difference from Token Bucket:**

| Token Bucket | Leaky Bucket |
|--------------|--------------|
| Controls **when** you can request | Controls **when** requests are processed |
| Allows bursts (immediate processing) | No bursts (constant output rate) |
| Request either allowed or rejected | Request either queued or rejected |

**How It Works:**

1. Request arrives → goes into queue
2. Queue full? → Request rejected
3. Requests processed from queue at fixed rate (like water leaking)

**Why Use Leaky Bucket?**

✅ Perfectly smooth output rate (no spikes to backend)
✅ Good when you're calling rate-limited external APIs
✅ Fair queuing (first in, first out)

❌ Adds latency (requests wait in queue)
❌ A burst of new requests waits behind old ones
❌ More complex (need to manage a queue)

**Real-World Example:**

Video streaming buffer:
- Video frames arrive in bursts from network
- Leaky bucket smooths them to constant 30 fps playback
- Buffer (bucket) absorbs network variability

---

### 2.3 Fixed Window Counter

**The Simplest Approach**

Divide time into fixed chunks (windows). Count requests in each window. Reset counter when window ends.

```
┌────────────────────────────────────────────────────────────┐
│                   FIXED WINDOW COUNTER                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Time divided into fixed 1-minute windows:                  │
│                                                             │
│  |-------- 12:00-12:01 --------|-------- 12:01-12:02 ------│
│  |                             |                            │
│  |  Limit: 100 requests        |  Limit: 100 requests      │
│  |                             |                            │
│  |  Counter: 78                |  Counter: 12              │
│  |  ████████░░                 |  ██░░░░░░░░               │
│  |                             |                            │
│  |_____________________________|____________________________|
│                                                             │
│  At 12:01:00, counter resets to 0                          │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**The Problem: Window Boundary Burst**

What if requests cluster around the window boundary?

```
┌────────────────────────────────────────────────────────────┐
│                  THE BOUNDARY PROBLEM                       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  |-------- 12:00-12:01 --------|-------- 12:01-12:02 ------│
│  |                             |                            │
│  |                 ████████████|████████████               │
│  |                 100 requests|100 requests               │
│  |                     ↑       |    ↑                      │
│  |                  12:00:59   | 12:01:01                  │
│  |_____________________________|____________________________|
│                                                             │
│  Result: 200 requests in 2 seconds!                        │
│  But each window thinks it's under limit.                  │
│                                                             │
│  ⚠️ User effectively gets DOUBLE the intended rate         │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**Why Use Fixed Window?**

✅ Dead simple to implement
✅ Very memory efficient (just 1 counter)
✅ Easy to understand

❌ 2x burst possible at window boundaries
❌ Not suitable when strict limits are required

**Real-World Example:**

"Free tier: 1000 API calls per day"
- Window: 24 hours (midnight to midnight)
- Simple to explain to users
- Boundary issue less problematic at large time scales

---

### 2.4 Sliding Window Log

**The Most Accurate Approach**

Keep a log of every request timestamp. Count how many fall within the sliding window.

```
┌────────────────────────────────────────────────────────────┐
│                   SLIDING WINDOW LOG                        │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Current time: 12:01:30                                     │
│  Window size: 60 seconds                                    │
│  Limit: 5 requests                                          │
│                                                             │
│  We store timestamp of EVERY request:                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 12:00:25  12:00:45  12:01:02  12:01:15  12:01:28   │   │
│  └─────────────────────────────────────────────────────┘   │
│       ↑         ↑                                           │
│       │         └── This one is also expired (> 60s ago)   │
│       └── Expired! (more than 60 seconds ago)              │
│                                                             │
│  The "window" slides with current time:                     │
│                                                             │
│     12:00:30 ◄──────── 60 seconds ────────► 12:01:30       │
│              [          ACTIVE WINDOW          ]            │
│                                                             │
│  Requests in window: 3 (12:01:02, 12:01:15, 12:01:28)      │
│  3 < 5 limit → New request ALLOWED                         │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**How It Works:**

1. New request arrives
2. Remove all timestamps older than (now - window_size)
3. Count remaining timestamps
4. If count < limit, allow and add current timestamp

**Why Use Sliding Window Log?**

✅ Most accurate - no boundary issues at all
✅ Exact count within any 60-second period

❌ Memory hungry (stores every timestamp)
❌ Cleanup is O(n) - slow with many requests
❌ Not practical for high-volume APIs

**Real-World Example:**

Banking fraud detection:
- "Flag if more than 5 transactions in any 10-minute period"
- Accuracy critical, volume relatively low
- Worth the memory cost

---

### 2.5 Sliding Window Counter

**The Best of Both Worlds**

Combines fixed window efficiency with sliding window accuracy. Uses weighted average of current and previous window.

```
┌────────────────────────────────────────────────────────────┐
│                 SLIDING WINDOW COUNTER                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Current time: 12:01:20 (20 seconds into current window)   │
│  Window size: 60 seconds                                    │
│  Limit: 100 requests                                        │
│                                                             │
│  |-------- Previous --------|-------- Current --------|    │
│  |      12:00 - 12:01       |      12:01 - 12:02      |    │
│  |                          |                          |    │
│  |  Count: 84 requests      |  Count: 36 requests     |    │
│  |                          |          ↑              |    │
│  |__________________________|__________|______________|    │
│                                        │                    │
│                                   We are here               │
│                                   (20 sec in)               │
│                                                             │
│  The clever part - WEIGHTED ESTIMATE:                       │
│                                                             │
│  Time into current window: 20 seconds                       │
│  Time remaining that overlaps with "prev": 40 seconds       │
│  Overlap percentage: 40/60 = 67%                            │
│                                                             │
│  Estimated requests = (prev × overlap%) + current           │
│                     = (84 × 0.67) + 36                      │
│                     = 56 + 36                               │
│                     = 92 requests                           │
│                                                             │
│  92 < 100 limit → Request ALLOWED                          │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**The Intuition:**

Think of it as estimating: "Based on the rate of the previous window, how many requests would have happened in the part of that window that overlaps with our sliding view?"

**Visual Explanation:**

```
         Previous Window          Current Window
    |─────────────────────────|─────────────────────────|
                         ◄── Our 60-second sliding view ──►
                         |=========================|
                              ↑
                         We weight the previous
                         window by how much of it
                         falls in our view
```

**Why Use Sliding Window Counter?**

✅ Memory efficient (just 2 counters like fixed window)
✅ Smooths out boundary issues (unlike fixed window)
✅ Good enough accuracy for most use cases
✅ Best balance of simplicity and accuracy

❌ Approximation, not 100% precise
❌ Slightly more complex than fixed window

**Real-World Example:**

Most production APIs (GitHub, Stripe, etc.) use this approach:
- Simple to implement
- Low memory
- Good enough accuracy
- No major boundary issues

---

## 3. Algorithm Comparison

### Quick Comparison Table

| Algorithm | Memory | Accuracy | Allows Bursts? | Best For |
|-----------|--------|----------|----------------|----------|
| **Token Bucket** | Low (2 values) | High | Yes (controlled) | Most APIs |
| **Leaky Bucket** | Medium (queue) | High | No (smoothed) | Constant rate output |
| **Fixed Window** | Very Low (1 counter) | Low | Yes (at boundaries) | Simple/internal use |
| **Sliding Log** | High (all timestamps) | Perfect | No | Low-volume, high-accuracy |
| **Sliding Counter** | Low (2 counters) | Good | Smoothed | Production APIs |

### How They Handle a Burst

Imagine 100 requests arrive at once, with a limit of 50 per minute:

```
Token Bucket (bucket=50, refill=50/min):
  └── First 50 allowed, rest rejected
      Next 50 requests allowed 1 minute later

Leaky Bucket (queue=50, rate=50/min):
  └── First 50 queued, rest rejected
      Queued requests processed over 1 minute

Fixed Window:
  └── First 50 allowed, rest rejected
      Counter resets at window boundary

Sliding Log:
  └── First 50 allowed, rest rejected
      Must wait until old timestamps expire

Sliding Counter:
  └── Similar to sliding log but approximate
```

### Decision Flowchart

```
Need rate limiting?
         │
         ▼
Is smooth output rate required?
(calling external API, video streaming)
         │
    ┌────┴────┐
   YES        NO
    │         │
    ▼         ▼
 LEAKY     Do you need to allow bursts?
 BUCKET         │
           ┌────┴────┐
          YES        NO
           │         │
           ▼         ▼
        TOKEN    Is perfect accuracy required?
        BUCKET        │
                 ┌────┴────┐
                YES        NO
                 │         │
                 ▼         ▼
              SLIDING   SLIDING WINDOW
              WINDOW    COUNTER
              LOG       (recommended default)
```

---

## 4. Implementation Examples

### 4.1 Token Bucket with Redis

**Redis Data Structure:**

```
Key: "rate:user:123"
Value: Hash {
    tokens: 47        # Current tokens available
    last_refill: 1706792400.5  # Timestamp of last refill
}
```

**The Logic (in words):**

1. Get current tokens and last refill time from Redis
2. Calculate how much time passed since last refill
3. Add tokens based on time passed (but cap at maximum)
4. If tokens >= 1, consume one and allow request
5. Save updated state back to Redis

**Go Implementation:**

```go
// TokenBucket represents a rate limiter using the token bucket algorithm
type TokenBucket struct {
    redis      *redis.Client
    capacity   int64    // Maximum tokens (burst size)
    refillRate float64  // Tokens added per second
}

// Allow checks if a request should be allowed
func (tb *TokenBucket) Allow(ctx context.Context, key string) (bool, error) {
    // This Lua script runs atomically in Redis
    // It handles refill calculation and token consumption in one operation
    script := `
        local tokens = tonumber(redis.call('HGET', KEYS[1], 'tokens')) or ARGV[1]
        local last = tonumber(redis.call('HGET', KEYS[1], 'last')) or ARGV[3]

        -- Refill based on time elapsed
        local elapsed = ARGV[3] - last
        tokens = math.min(ARGV[1], tokens + elapsed * ARGV[2])

        -- Try to consume
        if tokens >= 1 then
            tokens = tokens - 1
            redis.call('HSET', KEYS[1], 'tokens', tokens, 'last', ARGV[3])
            return 1
        end
        return 0
    `
    // Execute script with: capacity, refill_rate, current_timestamp
    result, err := tb.redis.Eval(ctx, script, []string{key},
        tb.capacity, tb.refillRate, time.Now().Unix()).Int()

    return result == 1, err
}
```

### 4.2 Sliding Window Counter with Redis

**Redis Data Structure:**

```
Key: "rate:user:123:1706792400"  # Current window (timestamp in key)
Value: 36  # Request count

Key: "rate:user:123:1706792340"  # Previous window
Value: 84  # Request count
```

**Go Implementation:**

```go
// SlidingWindowCounter implements sliding window rate limiting
type SlidingWindowCounter struct {
    redis      *redis.Client
    limit      int64  // Max requests per window
    windowSecs int64  // Window size in seconds
}

// Allow checks if a request should be allowed
func (sw *SlidingWindowCounter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now().Unix()
    currentWindow := now / sw.windowSecs
    timeIntoWindow := now % sw.windowSecs

    // Build keys for current and previous window
    currKey := fmt.Sprintf("%s:%d", key, currentWindow)
    prevKey := fmt.Sprintf("%s:%d", key, currentWindow-1)

    // Get counts from both windows
    pipe := sw.redis.Pipeline()
    prevCmd := pipe.Get(ctx, prevKey)
    currCmd := pipe.Get(ctx, currKey)
    pipe.Exec(ctx)

    prevCount, _ := prevCmd.Int64()  // Defaults to 0 if not exists
    currCount, _ := currCmd.Int64()

    // Calculate weighted estimate
    overlap := float64(sw.windowSecs-timeIntoWindow) / float64(sw.windowSecs)
    weighted := int64(float64(prevCount)*overlap) + currCount

    if weighted < sw.limit {
        // Allow and increment current window
        sw.redis.Incr(ctx, currKey)
        sw.redis.Expire(ctx, currKey, time.Duration(sw.windowSecs*2)*time.Second)
        return true, nil
    }

    return false, nil
}
```

### 4.3 HTTP Middleware

```go
func RateLimitMiddleware(limiter *TokenBucket) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Identify the client (API key or IP address)
            clientID := r.Header.Get("X-API-Key")
            if clientID == "" {
                clientID = r.RemoteAddr
            }

            // Check rate limit
            allowed, err := limiter.Allow(r.Context(), "rate:"+clientID)

            // If Redis is down, fail open (allow request)
            if err != nil {
                log.Printf("Rate limiter error: %v", err)
                next.ServeHTTP(w, r)
                return
            }

            if !allowed {
                w.Header().Set("Retry-After", "60")
                w.WriteHeader(http.StatusTooManyRequests)
                json.NewEncoder(w).Encode(map[string]string{
                    "error": "Rate limit exceeded. Please wait and retry.",
                })
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

---

## 5. Distributed Rate Limiting

### The Problem: Multiple Servers

When you have multiple servers behind a load balancer, local rate limiting doesn't work:

```
┌────────────────────────────────────────────────────────────┐
│                      THE PROBLEM                            │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   User sends 300 requests, limit is 100                    │
│                   │                                         │
│                   ▼                                         │
│           ┌──────────────┐                                 │
│           │Load Balancer │                                 │
│           └──────┬───────┘                                 │
│        ┌─────────┼─────────┐                               │
│        ▼         ▼         ▼                               │
│   ┌────────┐┌────────┐┌────────┐                          │
│   │Server 1││Server 2││Server 3│                          │
│   │Local:  ││Local:  ││Local:  │                          │
│   │100 req ││100 req ││100 req │                          │
│   └────────┘└────────┘└────────┘                          │
│        ↑         ↑         ↑                               │
│        └─────────┴─────────┘                               │
│        Each thinks it's under limit!                       │
│                                                             │
│   Result: 300 requests allowed instead of 100              │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Solution 1: Centralized Counter (Redis)

All servers check a single shared counter:

```
┌────────────────────────────────────────────────────────────┐
│                 CENTRALIZED SOLUTION                        │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌────────┐ ┌────────┐ ┌────────┐                        │
│   │Server 1│ │Server 2│ │Server 3│                        │
│   └───┬────┘ └───┬────┘ └───┬────┘                        │
│       │          │          │                              │
│       └──────────┼──────────┘                              │
│                  │                                          │
│                  ▼                                          │
│          ┌─────────────┐                                   │
│          │    Redis    │  ← Single source of truth        │
│          │  count: 87  │                                   │
│          └─────────────┘                                   │
│                                                             │
│   ✅ Accurate global limit                                 │
│   ❌ Redis becomes critical dependency                     │
│   ❌ Network latency on every request                      │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Solution 2: Local Cache + Periodic Sync

Fast local checks, periodic synchronization with central store:

```
┌────────────────────────────────────────────────────────────┐
│                    HYBRID SOLUTION                          │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────┐ ┌─────────────────┐                  │
│   │    Server 1     │ │    Server 2     │                  │
│   │ ┌─────────────┐ │ │ ┌─────────────┐ │                  │
│   │ │Local Cache  │ │ │ │Local Cache  │ │                  │
│   │ │ count: 28   │ │ │ │ count: 31   │ │                  │
│   │ └──────┬──────┘ │ │ └──────┬──────┘ │                  │
│   └────────┼────────┘ └────────┼────────┘                  │
│            │                   │                            │
│            │   Sync every      │                            │
│            │   1-5 seconds     │                            │
│            └─────────┬─────────┘                            │
│                      ▼                                      │
│              ┌─────────────┐                                │
│              │    Redis    │                                │
│              │  count: 59  │                                │
│              └─────────────┘                                │
│                                                             │
│   ✅ Fast (local check first)                              │
│   ✅ Works if Redis temporarily down                       │
│   ❌ May slightly exceed limit between syncs               │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Solution 3: Token Pre-fetching

Each server grabs a batch of tokens in advance:

```
┌────────────────────────────────────────────────────────────┐
│                  TOKEN PRE-FETCHING                         │
├────────────────────────────────────────────────────────────┤
│                                                             │
│   Redis: Token Pool (1000 tokens total)                    │
│                                                             │
│   Server 1         Server 2         Server 3               │
│   "I'll take 100"  "I'll take 100"  "I'll take 100"       │
│        │                │                │                  │
│        ▼                ▼                ▼                  │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐           │
│   │ Local   │      │ Local   │      │ Local   │           │
│   │ tokens: │      │ tokens: │      │ tokens: │           │
│   │   100   │      │   100   │      │   100   │           │
│   └─────────┘      └─────────┘      └─────────┘           │
│                                                             │
│   • Each server serves requests from local pool            │
│   • When pool runs low, fetch more from Redis              │
│   • Very fast (no network call per request)                │
│                                                             │
│   ✅ Extremely fast                                        │
│   ✅ Accurate within batch size                            │
│   ❌ Wasted tokens if server crashes with tokens           │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### What If Redis Goes Down?

You have three choices:

| Strategy | Behavior | Use When |
|----------|----------|----------|
| **Fail Open** | Allow all requests | Availability > Protection |
| **Fail Closed** | Reject all requests | Protection > Availability |
| **Fallback** | Use local/memory cache | Best of both |

Most production systems use **Fail Open** with alerting - it's better to serve requests than to have a complete outage because your rate limiter is down.

---

## 6. Rate Limiting in Practice

### 6.1 Tiered Rate Limits

Different users get different limits:

```
┌────────────────────────────────────────────────────────────┐
│                    USER TIERS                               │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Tier         │ Requests/min │ Burst  │ Price              │
│  ─────────────┼──────────────┼────────┼─────────           │
│  Anonymous    │      10      │   20   │ Free               │
│  Free         │     100      │  200   │ Free (with signup) │
│  Pro          │   1,000      │ 2,000  │ $29/month          │
│  Enterprise   │  10,000      │20,000  │ Custom             │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 6.2 Per-Endpoint Limits

Some endpoints are more expensive than others:

```
┌────────────────────────────────────────────────────────────┐
│                ENDPOINT-SPECIFIC LIMITS                     │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Endpoint            │ Limit/min │ Reason                  │
│  ────────────────────┼───────────┼──────────────────       │
│  GET  /users         │    100    │ Light read              │
│  POST /users         │     10    │ Creates DB records      │
│  GET  /search        │     30    │ Heavy computation       │
│  POST /export        │      5    │ Very heavy, async       │
│  GET  /health        │ Unlimited │ For monitoring          │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 6.3 Good 429 Response

When you reject a request, be helpful:

```json
{
    "error": {
        "code": "rate_limit_exceeded",
        "message": "You've exceeded 100 requests per minute",
        "details": {
            "limit": 100,
            "current": 105,
            "window": "1 minute",
            "resets_at": "2024-02-01T12:02:00Z"
        },
        "retry_after_seconds": 45,
        "help_url": "https://api.example.com/docs/rate-limits",
        "upgrade_url": "https://example.com/pricing"
    }
}
```

**Always include:**
- Clear error message
- When they can retry
- Link to documentation
- How to get higher limits

---

## 7. Interview Discussion Points

### Common Questions

**Q: Which algorithm would you use for a new API?**

> Start with **Sliding Window Counter** - it's the best balance of simplicity, accuracy, and memory efficiency. If you need burst handling, consider **Token Bucket**.

**Q: How do you handle rate limiting across multiple data centers?**

> Three options:
> 1. **Global Redis** - Simple but adds latency
> 2. **Per-region limits** - Each region gets portion of limit (US: 40%, EU: 40%, Asia: 20%)
> 3. **Eventual consistency** - Local counters with periodic sync

**Q: What if your rate limiter (Redis) goes down?**

> Implement graceful degradation:
> 1. Have a fallback (local memory cache)
> 2. Fail open if no fallback (allow requests)
> 3. Alert on-call engineer
> 4. Never let the rate limiter cause a complete outage

**Q: How do you prevent users from creating multiple accounts to bypass limits?**

> Defense in depth:
> 1. Rate limit by IP address too (not just API key)
> 2. Require phone/credit card verification
> 3. Monitor for suspicious patterns
> 4. Use device fingerprinting

**Q: Token Bucket vs Leaky Bucket?**

> - **Token Bucket**: Allows bursts. Good for user-facing APIs where traffic is naturally bursty.
> - **Leaky Bucket**: Smooths output. Good when calling external rate-limited APIs or when you need constant processing rate.

### Anti-Patterns to Avoid

| Don't Do This | Why It's Bad | Do This Instead |
|---------------|--------------|-----------------|
| Store counters in local memory only | Doesn't work with multiple servers | Use Redis or similar |
| Same limits for all endpoints | Heavy endpoints overwhelm system | Per-endpoint limits |
| No headers in response | Users can't debug issues | Always include rate limit headers |
| Silent failures when Redis down | Hidden problems | Log, alert, fail gracefully |
| Rate limit by only one identifier | Easy to bypass | Combine API key + IP + user ID |

---

## Quick Reference Card

### Which Algorithm to Use?

| Scenario | Recommended Algorithm |
|----------|----------------------|
| General purpose API | Sliding Window Counter |
| Need burst handling | Token Bucket |
| Calling external APIs | Leaky Bucket |
| Simple internal service | Fixed Window |
| High-accuracy, low-volume | Sliding Window Log |

### Redis Commands Cheat Sheet

```redis
# Fixed Window
INCR rate:user:123:minute:1234567
EXPIRE rate:user:123:minute:1234567 60

# Sliding Window Counter
GET rate:user:123:current_window
GET rate:user:123:previous_window

# Token Bucket
HSET rate:user:123 tokens 95 last_refill 1706792400
EXPIRE rate:user:123 3600
```

### HTTP Headers

```http
# Always include in responses
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1706792400

# On 429 responses
Retry-After: 30
```

---

*Guide created for learning and interview preparation.*
