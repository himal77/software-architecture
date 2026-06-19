# Day 16 — API Design Patterns
**Chapter 4 | Phase 1: Foundations**
**Date:** 2026-06-18

---

## What Was Covered

GraphQL (3 operations, N+1 problem, DataLoader), event-driven APIs (webhooks, streaming, polling), webhook security (HMAC), Backend for Frontend (BFF), API composition pattern, idempotency across microservices, Bitpanda partner API design.

---

## Concept 1: GraphQL

Solves REST's over-fetching and under-fetching problems. Client specifies exactly what data it needs in one request.

### Three Operations

**Query (read):**
```graphql
query {
  user(id: "abc123") {
    name
    wallet(currency: "BTC") { balance }
  }
}
```

**Mutation (write):**
```graphql
mutation {
  createTrade(input: { userId: "abc123", amount: 60000, currency: "BTC", type: BUY }) {
    tradeId
    status
  }
}
```

**Subscription (real-time push — WebSocket under the hood):**
```graphql
subscription {
  priceUpdated(symbol: "BTC/EUR") { price timestamp }
}
```

### GraphQL vs REST

| | REST | GraphQL |
|---|---|---|
| Data shape | Fixed (server defines) | Flexible (client defines) |
| Multiple resources | Multiple requests | One request |
| Caching | Easy (HTTP by URL) | Hard (dynamic POST queries) |
| Mobile apps | OK | Excellent (bandwidth-sensitive) |
| Public APIs | Industry standard | Less common |

**Bitpanda:** External/partner API → REST. Mobile internal → GraphQL worth considering. Service-to-service → gRPC.

### N+1 Problem

```
query { trades { user { name } } }

Naive: 1 query for trades + 1 query per trade for user = N+1 queries
Fix: DataLoader batches all user IDs → 1 query: SELECT * FROM users WHERE id IN (...)
Always use DataLoader for relationship queries in GraphQL.
```

---

## Concept 2: Event-Driven APIs

### Webhooks

Server pushes HTTP POST to client's registered URL when event occurs.

```
Partner registers: POST https://bank.com/webhooks/trades
Trade completes → Bitpanda POSTs to that URL:
{
  "event": "TRADE_COMPLETED",
  "trade_id": "abc123",
  "amount": 60000
}
```

NOT the same as WebSocket — no persistent connection. Bitpanda makes outbound HTTP calls to partner's URL.

Used by: Stripe, GitHub, Twilio.

**Reliability — failed webhook handling:**
```
Partner down → 503 → store in Kafka (webhook-events topic)
Retry with exponential backoff for up to 24 hours
Kafka ensures no webhook lost even if Bitpanda restarts
```

**Security — HMAC signature:**
```
Bitpanda signs body: HMAC-SHA256(body, shared_secret)
Header: X-Bitpanda-Signature: sha256=abc123...

Partner verifies: compute HMAC with their secret copy → compare
Match → genuine. Mismatch → reject.

Why HMAC not token: HMAC is different for every request
Intercepting one request gives attacker nothing reusable.
```

### Event Streaming

```
Exchange → Kafka → SSE stream → partner's system
Continuous stream of events. Client subscribes once, receives all events.
Used for: market data feeds, audit streams.
```

### Polling

```
GET /events?since=2026-06-18T10:00:00Z
Simple, works everywhere, inefficient.
Use when: webhook delivery unreliable, client behind firewall.
```

---

## Concept 3: Backend for Frontend (BFF)

Different clients have different data needs. One API serving all clients badly.

```
Mobile BFF  → minimal response (8 fields, bandwidth-optimised)
Web BFF     → rich response (aggregated dashboard data)
Partner BFF → stable contract, separate versioning, API key auth

Each BFF calls internal microservices via gRPC and shapes response for its client.
```

**Why BFF:**
- Each client type evolves independently
- No single API compromising for all clients
- Different auth per client (mobile JWT, partner API key)

---

## Concept 4: API Composition

Clients need data from multiple services. BFF makes parallel calls internally.

```java
CompletableFuture<TradeHistory> trades = CompletableFuture.supplyAsync(
    () -> tradeClient.getHistory(userId));
CompletableFuture<WalletBalance> wallet = CompletableFuture.supplyAsync(
    () -> walletClient.getBalance(userId));

CompletableFuture.allOf(trades, wallet).join();
// Total time = slowest call, not sum of all calls
```

Client makes 1 request. BFF makes N parallel gRPC calls. Client gets combined response.

---

## Concept 5: Idempotency Across Services

Every service-to-service call in trade saga uses derived keys:
```
trade-abc123:step1-match
trade-abc123:step2-deduct-buyer
trade-abc123:step3-credit-seller
```

Orchestrator retries step → service sees key → returns cached result → no double deduction.

---

## Design: Bitpanda Partner API

### Architecture

```
Partner App
    │ REST + API Key (HTTPS)
    ▼
API Gateway
    ├── API key → identify partner tier
    ├── Rate limit check (Redis, per partner per window)
    ├── JWT/API key validation
    └── Route to Partner BFF
    ▼
Partner BFF
    │ gRPC
    ├── Trade Service
    ├── Wallet Service
    └── Market Data Service
    ▼
Trade completes → Kafka → Webhook Publisher → POST to partner's registered URL
                                               Retry up to 24h on failure (Kafka)
```

### Key Decisions

| Decision | Choice | Reason |
|---|---|---|
| Protocol | REST | Industry standard, partners expect JSON/HTTP |
| Notifications | Webhooks | No persistent connection needed, outbound HTTP POST |
| Webhook reliability | Kafka queue + retry | No webhook lost even if Bitpanda restarts |
| Webhook security | HMAC-SHA256 signature | Different per request, can't be replayed |
| Rate limits | Per API key in Redis | Different limits per partner tier |
| Internal calls | gRPC | Binary, fast, type-safe |

### Per-Partner Rate Limits

```
API key → tier lookup → Redis counter per partner per window

small-bank-key:  1,000 req/min
large-bank-key:  50,000 req/min

Key: rate_limit:{api_key}:{window}
429 if exceeded + Retry-After header
```

### Webhook vs WebSocket — Key Distinction

```
WebSocket:  partner maintains persistent connection, server pushes over it
Webhook:    Bitpanda makes outbound HTTP POST to partner's registered URL
            No persistent connection
            Partner just needs an HTTP endpoint
            Much simpler for partner integration
```

---

## Quiz Gaps Carried Forward

| Gap | Correct Answer |
|---|---|
| Retry vs circuit breaker | Retry = transient (ms). CB = persistent (minutes). |
| Cascading timeout fix | Inner timeout < outer. Wallet < Trade < Gateway. |
| Load balancing: IP Hash | Same IP → same server. Sticky sessions. |
| Load balancing: Least Response Time | Lowest connections + latency. Production default. |
| Cassandra for audit | High write throughput, time-series, TTL. NOT CockroachDB. |
| Webhook vs WebSocket | Webhook = outbound HTTP POST. WebSocket = persistent connection. |
