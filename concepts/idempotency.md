# Idempotency
**Introduced:** Day 9 (referenced earlier in Day 3)

---

## Definition
An operation is **idempotent** if calling it multiple times has the same effect as calling it once.

```
Idempotent:
  PUT /users/42 { name: "Alice" }
  → call it 5 times → result is identical (Alice's name set)

Non-idempotent:
  POST /transfers { from: A, to: B, amount: 100 }
  → call it 5 times → 5 transfers happen, total $500 moved
```

---

## Why It Matters

Distributed systems retry. Network blips, timeouts, service crashes — clients and saga orchestrators retry operations regularly. Without idempotency:
- Duplicate trades (user charged 5 times for one BTC purchase)
- Duplicate notifications (5 emails saying "your order shipped")
- Duplicate audit entries (compliance reports wrong)

---

## Idempotency Key Pattern

```
Client generates UUID before sending request:
  POST /trades
  Idempotency-Key: trade-abc123
  { "asset": "BTC", "amount": 1, "price": 60000 }

Server logic:
  Check: have I seen idempotency_key=trade-abc123 before?
    → Yes: return stored result, don't re-execute
    → No: execute, store key + result, return result
```

Storage of keys: usually Redis with 24h TTL, or a dedicated DB table.

---

## Where to Use Idempotency Keys

| Use case | Why |
|---|---|
| Public APIs | Clients retry on network failure |
| Webhooks (incoming) | Provider may send same webhook twice |
| External API calls (outgoing) | Stripe natively supports `Idempotency-Key` |
| Saga step calls | Orchestrator retries on timeout |
| Kafka consumers | At-least-once delivery means duplicates |

---

## Implementation Strategies

### 1. Idempotency-Key header (HTTP APIs)
```
Client passes UUID in header
Server stores key + response
Duplicate → returns cached response
```

### 2. Natural keys (database constraints)
```
Use the business identifier as primary key
INSERT ... ON CONFLICT DO NOTHING
Or unique constraint that detects duplicates
```

### 3. Conditional updates
```
UPDATE accounts SET balance = balance - 100 
WHERE id = 'A' AND status = 'pending' AND version = 5

Only succeeds once — version increments. Retries are no-ops.
```

### 4. Event ID deduplication (Kafka consumers)
```
Track processed event IDs in a set
On consume: check if event_id already processed
  → Yes: skip
  → No: process, record event_id
```

---

## Stripe's Idempotency-Key Pattern

Stripe accepts `Idempotency-Key` header on every API call. Recorded for 24 hours. Duplicate requests with same key return original response without charging the card again.

This is the gold standard. Adopt the same pattern in your fintech APIs.

---

## Rule for Fintech (and any distributed system)

**Every external API call from your services must include an idempotency key.**

If the external API doesn't support it natively, build your own deduplication layer. Without idempotency, distributed systems eventually double-charge customers, send duplicate notifications, or create ghost records. Always.
