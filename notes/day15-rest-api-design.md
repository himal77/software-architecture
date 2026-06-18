# Day 15 — REST API Design Principles & Chapter 3 Wrap-up
**Chapter 3 | Phase 1: Foundations**
**Date:** 2026-06-18

---

## What Was Covered

REST constraints, resource design, HTTP methods + status codes, pagination, error format, API versioning (4 strategies), breaking vs non-breaking changes, idempotency in REST, Chapter 3 concept map, Bitpanda public API design.

---

## Concept 1: REST Principles

### Six Constraints

1. **Uniform Interface** — nouns not verbs, standard HTTP methods, self-descriptive messages
2. **Stateless** — every request contains all needed info, no server-side session, enables horizontal scaling
3. **Client-Server** — UI and data concerns separated, evolve independently
4. **Cacheable** — responses declare cacheability via Cache-Control headers
5. **Layered System** — client doesn't know about intermediaries (CDN, gateway, LB)
6. **Code on demand** (optional) — server sends executable code

### Resource Design — Nouns Not Verbs

```
Wrong:                      Right:
POST /createTrade    →      POST   /trades
GET  /getTrades      →      GET    /trades
POST /cancelTrade    →      DELETE /trades/{id}
                            GET    /trades/{id}
                            PATCH  /trades/{id}
```

### HTTP Methods

| Method | Meaning | Idempotent | Safe |
|---|---|---|---|
| GET | Read | Yes | Yes |
| POST | Create | No | No |
| PUT | Replace entirely | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove | Yes | No |

**Safe** = no side effects. **Idempotent** = N calls = 1 call result.

### HTTP Status Codes

```
200 OK              → successful GET, PATCH
201 Created         → successful POST (+ Location header)
204 No Content      → successful DELETE
400 Bad Request     → validation failed, malformed JSON
401 Unauthorized    → missing/invalid JWT
403 Forbidden       → valid JWT, no permission
404 Not Found       → resource doesn't exist
409 Conflict        → duplicate, state conflict
422 Unprocessable   → business logic rejection (insufficient funds)
429 Too Many        → rate limit exceeded
500 Internal Error  → unexpected exception (never expose stack traces)
503 Unavailable     → circuit breaker open
504 Gateway Timeout → upstream timed out
```

**Common mistake:**
```
Wrong: 200 + {"error": "insufficient funds"}
Right: 422 + {"code": "INSUFFICIENT_FUNDS", "message": "..."}
```

### Pagination

**Offset-based:**
```
GET /trades?page=2&size=20
Response: {"data": [...], "pagination": {"page": 2, "total": 847, "next": "..."}}
Problem: page shifts if new data inserted during pagination
```

**Cursor-based (better for real-time data):**
```
GET /trades?after=trade-abc123&size=20
Response includes: "next_cursor": "trade-xyz789"
Stable position — no skipped or duplicate rows
Used by: Twitter, Facebook, Stripe
```

### Error Response Format

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Wallet balance €500 is below required €60,000",
    "field": "amount",
    "request_id": "req-abc123"
  }
}
```

`request_id` — critical for debugging, customer support, log correlation.

---

## Concept 2: API Versioning

### Four Strategies

**URL versioning (recommended):**
```
/v1/trades, /v2/trades
Explicit, easy to route, cacheable. Used by Stripe, Twitter.
```

**Header versioning:**
```
API-Version: 2026-06-17
Clean URLs, harder to test. Used by GitHub, Azure.
```

**Query parameter:**
```
/trades?version=2
Easy to test, caching issues. Rarely used in production.
```

**Content negotiation:**
```
Accept: application/vnd.bitpanda.v2+json
RESTfully correct, poor tooling support. Rarely used.
```

**Bitpanda choice: URL versioning** — `/v1/`, `/v2/`

### Breaking vs Non-Breaking Changes

**Non-breaking (safe, no version bump):**
```
✅ Adding new optional response field
✅ Adding new endpoint
✅ Adding optional query parameter
✅ Relaxing validation
```

**Breaking (requires new version):**
```
❌ Removing a field
❌ Renaming a field (amount → total_eur)
❌ Changing field type
❌ Making optional field required
❌ Changing URL structure
❌ Changing error format
```

**Deprecation timeline:**
```
1. Release v2
2. Keep v1 running + add deprecation warning header
3. Announce sunset date (6 months notice)
4. Shut down v1 after 6 months
```

---

## Concept 3: Idempotency in REST

```
GET, DELETE, PUT → naturally idempotent
POST, PATCH      → NOT idempotent → add Idempotency-Key header

POST /trades
Idempotency-Key: trade-abc123

Server:
  Check Redis for key
  Exists → return cached result, no re-execution
  Missing → execute + store result (24h TTL)

Retry due to network timeout → same key → same result → no duplicate trade
```

Every mutation endpoint in Bitpanda must support idempotency keys.

---

## Chapter 3 — Full Concept Map

```
NETWORKING & PROTOCOLS
│
├── FOUNDATIONS
│   ├── DNS → name to IP, CoreDNS in K8s, JVM TTL=30
│   ├── TCP → reliable, ordered, handshake, HoL blocking
│   ├── UDP → fast, unreliable, QUIC/HTTP3 built on it
│   └── TLS → encrypt + verify, terminate at ingress, mTLS internal
│
├── HTTP EVOLUTION
│   ├── HTTP/1.1 → one request per connection, HoL blocking
│   ├── HTTP/2   → multiplexing, binary, TCP HoL remains
│   └── HTTP/3   → QUIC/UDP, independent streams, 0-RTT
│
├── COMMUNICATION PATTERNS
│   ├── REST     → client-initiated, public APIs, JSON
│   ├── gRPC     → internal services, binary Protobuf, 4 patterns
│   └── WebSocket → persistent, server push, end users
│
├── INFRASTRUCTURE
│   ├── Load balancing    → 5 algorithms, L4 vs L7, health checks
│   ├── API Gateway       → auth, rate limiting, routing, SSL
│   └── Service discovery → client-side (Eureka) vs server-side (K8s)
│
├── RESILIENCE
│   ├── Circuit breaker → persistent failures, 3 states, fallback
│   ├── Retry + backoff → transient failures, jitter, idempotency
│   └── Bulkhead        → thread pool isolation
│
├── FLOW CONTROL
│   ├── Rate limiting  → 4 algorithms, API gateway, Redis counter
│   ├── Timeouts       → 3 types, inner < outer (cascading fix)
│   └── Back-pressure  → Kafka buffer, HTTP/2 flow control
│
└── API DESIGN
    ├── REST principles → stateless, uniform interface, nouns
    ├── Status codes    → 2xx/4xx/5xx correct usage
    ├── Versioning      → URL versioning, breaking vs non-breaking
    └── Idempotency     → Idempotency-Key on all POST mutations
```

---

## Design: Bitpanda Public API

### Core Endpoints

```
POST   /v1/trades              ← submit trade (Idempotency-Key required)
GET    /v1/trades              ← list user's trades (cursor pagination)
GET    /v1/trades/{id}         ← get single trade
DELETE /v1/trades/{id}         ← cancel pending trade

GET    /v1/wallets             ← list user's wallets
GET    /v1/wallets/{currency}  ← get specific wallet balance

GET    /v1/prices/BTC-EUR      ← current price (Cache-Control: max-age=5)
GET    /v1/orderbook/BTC-EUR   ← current order book snapshot

POST   /v1/auth/login          ← get JWT
POST   /v1/auth/refresh        ← refresh JWT
```

### Duplicate Trade Prevention

```
1. Client generates UUID before sending: Idempotency-Key: trade-abc123
2. POST /v1/trades with header
3. Server checks Redis
4. Network timeout → client retries with SAME key
5. Redis hit → cached result returned → no duplicate
```

### Real-time Confirmation

```
Trade completes → Kafka → Notification Service → WebSocket Gateway
→ pushes to user's open connection:
{"type": "TRADE_COMPLETED", "trade_id": "abc123", "btc": 1.0, "eur": 60000}
```

### Breaking Change Handling

```
Adding fee_eur field → non-breaking → deploy directly to v1
Renaming amount → total_eur → breaking → release v2, 6-month deprecation of v1
```

---

## Quiz Gaps Carried Forward (Chapter 4)

| Gap | Correct Answer |
|---|---|
| Circuit breaker 3 states | CLOSED → OPEN (on failure threshold) → HALF-OPEN (after timeout) → CLOSED/OPEN |
| Retry vs circuit breaker | Retry = transient (ms). CB = persistent (minutes). Never swap. |
| Rate limiting algorithms | Fixed window, Sliding log, Sliding counter (Cloudflare), Token bucket (Stripe/AWS) |
| Cascading timeout | Inner timeout < outer timeout. Error surfaces at innermost layer first. |
| Client vs server-side discovery | Client-side: Eureka (app picks instance). Server-side: K8s Service (infra picks). |
