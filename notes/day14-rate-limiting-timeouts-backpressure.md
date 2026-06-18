# Day 14 — Rate Limiting, Timeouts & Back-pressure
**Chapter 3 | Phase 1: Foundations**
**Date:** 2026-06-17

---

## What Was Covered

Rate limiting (4 algorithms), timeouts (3 types, cascading timeout problem), back-pressure (Kafka buffer, gRPC flow control, reactive), Bitpanda full request flow, market data API design.

---

## Concept 1: Rate Limiting

Controls how many requests a client can make in a given time window. Protects your system from misbehaving clients and enforces fair usage tiers.

### Four Algorithms

**Fixed Window Counter — simplest**
```
Window: 1 minute, Limit: 100
Counter resets at window boundary
Problem: boundary burst — 100 at :59, 100 at :01 = 200 in 2 seconds
```

**Sliding Window Log — most accurate**
```
Store timestamp of every request
On each request: remove timestamps > 1 minute old, count remaining
If count < limit → allow
Problem: high memory usage at scale
```

**Sliding Window Counter — best balance (Cloudflare)**
```
Weighted estimate using current + previous window:
count = current + previous × (1 - elapsed/window_size)
Accurate, memory efficient
```

**Token Bucket — most flexible (AWS, Stripe)**
```
Bucket holds tokens (max = burst capacity)
Tokens refill at fixed rate
Each request costs 1 token
No tokens → 429

Allows short bursts (bucket full) while enforcing average rate
```

### Where Rate Limiting Lives

**API Gateway** — single place for all services. Redis stores counters per user per window.

```java
public boolean isAllowed(String userId, int limitPerMinute) {
    String key = "rate_limit:" + userId + ":" + getCurrentMinute();
    Long count = redisTemplate.opsForValue().increment(key);
    if (count == 1) redisTemplate.expire(key, 2, TimeUnit.MINUTES);
    return count <= limitPerMinute;
}
```

### Tiers and Response

```
Free tier:      100 requests/minute
Premium tier:   1,000 requests/minute
Partner API:    10,000 requests/minute
Internal:       unlimited

Response when exceeded:
HTTP 429 Too Many Requests
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1717027200
Retry-After: 45
```

### Bot Attack Defense

```
Layer 1: Rate limit per IP (even unauthenticated: 10 req/min per IP)
Layer 2: Global endpoint rate limit (total across all IPs)
Layer 3: WAF (Cloudflare/AWS) — detects patterns, blocks at network level
Layer 4: Require auth — rate limit per account, CAPTCHA for registration
Best fix: WebSocket instead of polling — eliminates polling abuse entirely
```

---

## Concept 2: Timeouts

**Rule: always set timeouts. Never wait forever.**

### Three Types

**Connection timeout** — how long to wait to establish connection
```
Pod crashed → connection never comes
Timeout: 200ms → fail fast
```

**Read timeout** — how long to wait for response after connecting
```
Connected but DB query slow
Timeout: 500ms → TimeoutException
```

**Circuit breaker timeout** — how long OPEN state lasts (Day 13: waitDurationInOpenState)

### Values in Practice

```
Internal gRPC call:    200-500ms
External API:          2-5 seconds
Database query:        1-3 seconds
Redis cache:           50-100ms

Rule: timeout = p99 latency × 2-3
```

### Cascading Timeout Problem

```
Wrong (user waits 5s for error):
  Gateway: 5s, Trade: 4s, Wallet: 3s
  Wallet times out at 3s → Trade at 4s → Gateway at 5s

Right (user gets error in 1s):
  Gateway: 3s, Trade: 2s, Wallet: 1s
  Each layer's timeout < upstream's timeout
  Error surfaces quickly
```

### Spring Boot Config

```yaml
grpc:
  client:
    wallet-service:
      deadline: 500ms
```

---

## Concept 3: Back-pressure

Consumer signals producer to slow down when it can't keep up.

```
Without back-pressure:
  Producer: 50,000/sec, Consumer: 5,000/sec
  Queue fills → OOM crash OR stale data

With back-pressure:
  Consumer signals → producer slows to 5,000/sec
  Queue stays manageable
```

### Kafka Back-pressure

```
Consumer gets slow → stops polling Kafka
Messages accumulate in Kafka partition (7-day retention)
Consumer catches up when it recovers
No data loss — Kafka is the durable buffer

Monitor: consumer lag = how far behind consumer is
High lag = consumer overwhelmed = investigate
```

### gRPC Back-pressure

Built into HTTP/2 flow control. Consumer sends WINDOW_UPDATE=0 → producer pauses. Automatic — you don't implement it.

### Reactive (WebFlux)

```java
tradeStream
    .onBackpressureBuffer(1000)
    .subscribe(trade -> processTrade(trade));
```

### Back-pressure vs Rate Limiting

```
Rate Limiting:   external clients → your system (protect from outside)
Back-pressure:   internal producer → internal consumer (protect downstream)
```

---

## Concept 4: Slow WebSocket Client

User on bad mobile network — message queue keeps growing.

**For market data (time-sensitive):**
```
Option 1 — Drop old messages:
  Queue > N messages → drop oldest, keep newest
  User gets latest price when connection improves
  Stale price = useless anyway

Option 2 — Disconnect and reconnect:
  Queue exceeds threshold → close connection
  Client reconnects → fetches fresh Redis snapshot
  Clean state, no stale data
```

**For critical data (trade confirmations, wallet updates):**
```
Never drop. Persist in Kafka, retry until acknowledged.
Delivery guarantee is non-negotiable.
```

---

## Full Bitpanda Request Flow

```
Mobile App
    │ POST /trades  Idempotency-Key: trade-abc123
    ▼
API Gateway
    ├── Rate limit check (Redis counter)       → 429 if exceeded
    ├── JWT validation                         → 401 if invalid
    └── Route to Trade Service
    ▼
Trade Service (timeout: 2s)
    ├── Bulkhead check                         → reject if pool full
    ├── Circuit breaker check                  → fail fast if wallet down
    ├── gRPC → Wallet Service (timeout: 500ms) → check + deduct balance
    ├── gRPC → Order Book (timeout: 200ms)     → match order
    ├── Outbox → Kafka (async)                 → Notification + Audit
    └── Return TradeResult
    ▼
Mobile App
    ├── REST response (sync result)
    └── WebSocket push (real-time notification)
```

---

## Design: Market Data Rate Limiting + Back-pressure

**Scale:** 50,000 price updates/sec from exchanges, 2M WebSocket subscribers

### Back-pressure Solution

```
Exchanges → Kafka (price-updates topic) ← durable buffer
         → WebSocket broadcaster consumes from Kafka
         → Batches every 100ms → broadcasts to subscribers

If broadcaster slow:
  Stop polling Kafka → messages buffer in partition
  Catch up when recovered
  Monitor consumer lag metric

Batching effect: 50,000/sec → 10 broadcasts/sec per user
```

### Rate Limiting Strategy

```
Tiers: Free=100/min, Premium=1,000/min, Partner=10,000/min
Algorithm: Sliding window counter (Cloudflare approach)
Location: API Gateway
Storage: Redis (shared across all gateway pods)
Response: 429 + Retry-After header
```

### Slow Client Handling

```
Market data: drop old messages (stale prices = useless)
Trade confirmations: never drop, Kafka persistence + retry
```

---

## Quiz Gaps Carried Forward

| Gap | Correct Answer |
|---|---|
| ACID anomaly mapping | Read Committed→dirty reads, Repeatable Read→non-repeatable, Serializable→phantoms |
| gRPC 4 patterns | Unary, Server streaming, Client streaming, Bidirectional |
| Rate limiting algorithms | Fixed window, Sliding log, Sliding counter, Token bucket |
| Cascading timeout | Each layer timeout < upstream timeout. Surfaces errors fast. |
| Back-pressure mechanism | Kafka buffer + consumer lag. gRPC HTTP/2 flow control. |
