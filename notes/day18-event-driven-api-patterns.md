# Day 18 — Event-Driven API Patterns
**Chapter 4: API Design Patterns | Date: 2026-07-15**

---

## Revision Quiz Results

| Q | Topic | Score |
|---|---|---|
| 1 | JWT three parts | 6/10 — "signature" not "source" |
| 2 | OAuth 2.0 code exchange reason | 7/10 — code in URL → logged; token must not appear in URL |
| 3 | Compromised API key response | 2/10 — needs full 5-step revocation process |
| 4 | BOLA | 5/10 — concept correct, example missing |
| 5 | IP Hash algorithm | 5/10 — said "sticky service", correct term is IP Hash |
| 6 | Circuit breaker states + trigger | 7/10 — states correct, missed: timer triggers OPEN→HALF-OPEN |
| 7 | Token bucket parameters | 8/10 — correct concept, needed names: bucket size + refill rate |
| 8 | Cascading timeout fix | 10/10 |
| 9 | gRPC 4 patterns | 8/10 — described correctly, need exact names |
| 10 | Webhook HMAC security | 0/10 |
| 11 | BFF problem definition | 5/10 — BFF solves "one API compromises for all clients", not URL routing |
| 12 | Cursor vs offset pagination | 0/10 |

**Average: 5.6/10** — persistent gaps: IP Hash, BOLA example, webhook HMAC, cursor pagination.

---

## Why Event-Driven?

Synchronous APIs block the client and couple failure:

```
Client → POST /trades → waits 3s → response
If downstream is slow → client blocks
If downstream is down → request fails
```

Event-driven decouples:
```
Client → POST /trades → 202 Accepted (immediately)
Trade executes asynchronously
Client gets result via webhook / WebSocket / polling
```

---

## Pattern 1 — Request / Async Response

Submit work, get job ID back, check later.

```
POST /trades
→ 202 Accepted
  { "tradeId": "abc123", "status": "PENDING", "statusUrl": "/trades/abc123" }

GET /trades/abc123
→ { "status": "COMPLETED", "executedAt": "...", "price": 62000 }
```

`statusUrl` in response is clean — client knows exactly where to poll.

**When to use:** any operation > 500ms, financial transactions, file processing.

---

## Pattern 2 — Event Notification

Server publishes event when something happens. Multiple consumers act independently.

```
Trade completes → Kafka: topic=trade-events
  { "eventType": "TRADE_COMPLETED", "tradeId": "abc123", "userId": "user-456" }

Consumers (all parallel):
  notification-service  → send email/push
  audit-service         → write to Cassandra
  analytics-service     → update dashboards
  tax-service           → record for reporting
  webhook-publisher     → POST to partner URL
```

Producer has **zero knowledge** of consumers. Adding tax-service later = zero changes to trade-service.

---

## Pattern 3 — Event-Carried State Transfer

Events contain enough data that consumers don't need follow-up calls.

```
Too thin:
{ "eventType": "TRADE_COMPLETED", "tradeId": "abc123" }
→ notification-service must call trade-service → coupling + extra load

Right:
{
  "eventType": "TRADE_COMPLETED",
  "tradeId":   "abc123",
  "userId":    "user-456",
  "asset":     "BTC",
  "amount":    0.01,
  "price":     62000,
  "currency":  "EUR",
  "timestamp": "2026-07-15T10:00:00Z"
}
→ all common consumers are self-sufficient
```

**Rule:** include enough data that the majority of consumers need no follow-up call. Don't include internal implementation details.

Edge case: if one niche consumer needs very specific data, it's acceptable for *that consumer* to make a follow-up call rather than bloating every event.

---

## CloudEvents Standard

CNCF open standard for event envelope. Used by Google Cloud, Azure Event Grid, Knative.

Solves inconsistent event structures across services.

```json
{
  "specversion": "1.0",
  "id":          "evt-abc123",
  "source":      "/bitpanda/trade-service",
  "type":        "com.bitpanda.trade.completed",
  "time":        "2026-07-15T10:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "tradeId": "abc123",
    "userId":  "user-456",
    "asset":   "BTC",
    "amount":  0.01,
    "price":   62000
  }
}
```

| Field | Purpose |
|---|---|
| `id` | Unique event ID — deduplication |
| `source` | Which service produced it |
| `type` | Reverse-DNS: `com.bitpanda.trade.completed` |
| `time` | ISO 8601 timestamp |
| `data` | Actual payload |

**Bitpanda will use CloudEvents** across all Kafka topics — consistent logging, tracing, deduplication.

---

## AsyncAPI

REST has OpenAPI (Swagger). Event-driven APIs have **AsyncAPI**.

Documents Kafka topics so partners know exactly what schema to expect — just like Swagger for REST.

```yaml
asyncapi: '2.6.0'
channels:
  trade-events:
    subscribe:
      message:
        payload:
          type: object
          properties:
            tradeId: { type: string }
            userId:  { type: string }
            amount:  { type: number }
```

---

## Event Schema Evolution

| Change | Safe? |
|---|---|
| Add optional field | ✅ Safe — old consumers ignore it |
| Remove field | ❌ Breaking |
| Rename field | ❌ Breaking (remove + add) |
| Change field type | ❌ Breaking |
| Add required field | ❌ Breaking — old producers won't send it |

### Expand-Contract Pattern

Safe migration strategy for renaming `amount` → `quantity`:

```
Step 1 — Expand: publish both fields
{ "amount": 0.01, "quantity": 0.01, "fee": 0.001 }
Old consumers read "amount" ✓, new consumers read "quantity" ✓

Step 2 — Deprecate: notify consumers, give 6 months

Step 3 — Contract: remove "amount" after all consumers migrated
{ "quantity": 0.01, "fee": 0.001 }
```

Same principle as REST API versioning — applies identically to event schemas.

---

## Full Bitpanda Event Flow

```
Mobile App
  │ POST /trades
  ▼
API Gateway → Trade Service
  │ 202 Accepted { tradeId, statusUrl }
  │
  Saga executes:
    Wallet reserves funds → Order matched → Trade COMPLETED

Trade Service publishes CloudEvent:
  topic: trade-events
  type:  com.bitpanda.trade.completed

Consumers (parallel, no coordination needed):
  notification-service  → push/email
  audit-service         → Cassandra append
  analytics-service     → dashboard metrics
  tax-service           → annual reporting
  webhook-publisher     → POST to partner URL
```

---

## Concepts Introduced Today
- Request/async response pattern (202 + statusUrl)
- Event notification (fan-out via Kafka)
- Event-carried state transfer (fat events, self-sufficient consumers)
- CloudEvents standard (CNCF envelope)
- AsyncAPI (event stream documentation)
- Expand-contract pattern (safe schema evolution)
