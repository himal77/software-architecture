# Outbox Pattern
**Introduced:** Day 9

---

## The Problem
Services often need to commit data to a DB AND publish an event for downstream consumers. These two writes are not atomic — one can succeed while the other fails.

```
Naive (broken):
  @Transactional
  public void deductFiat(...) {
    walletRepo.deduct(...);              // commits to DB
    kafka.publish("FiatDeducted", ...);  // publishes event
  }

Failure mode:
  DB commit succeeds
  Kafka publish fails (network blip)
  → DB shows fiat deducted, no event published
  → Downstream services never notified
  → Saga stuck, audit log missing entry
```

You can't include Kafka in a DB transaction. Writing to two systems atomically = same problem as 2PC.

---

## The Solution

```
@Transactional
public void deductFiat(...) {
  walletRepo.deduct(...);
  outboxRepo.save(new Event("FiatDeducted", payload));
  // Both writes in same DB transaction → ATOMIC
}

Background outbox publisher:
  Reads unpublished outbox rows
  Publishes them to Kafka
  Marks rows as published
```

---

## Outbox Table Schema

```sql
CREATE TABLE outbox (
  event_id UUID PRIMARY KEY,
  event_type VARCHAR NOT NULL,
  payload JSONB NOT NULL,
  published BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT NOW(),
  published_at TIMESTAMP
);

CREATE INDEX idx_outbox_unpublished 
  ON outbox(created_at) 
  WHERE published = false;
```

---

## Why It Works

| Failure | What happens |
|---|---|
| App crash before DB commit | Nothing committed → user retries safely |
| App crash after DB commit, before publish | Outbox publisher will retry (rows still marked unpublished) |
| Kafka temporarily down | Publisher retries → eventually delivers |
| Publisher crashes | Resumes from where it left off (rows still unpublished) |

**Guarantee:** Any DB commit will eventually result in a Kafka message.

---

## Two Implementations

### Polling Outbox (simpler)
```
Background worker:
  Every 5 seconds:
    SELECT * FROM outbox WHERE published = false ORDER BY created_at LIMIT 100
    For each row:
      kafka.publish(row.event_type, row.payload)
      UPDATE outbox SET published = true, published_at = NOW() WHERE event_id = ?
```

Pros: simple, easy to monitor.
Cons: latency = polling interval (~5 sec).

### CDC / Transactional Log Tailing (advanced)
```
Debezium reads Postgres WAL (Write-Ahead Log) directly
→ Detects INSERTs into outbox table in real-time
→ Publishes to Kafka with low latency
→ No app code changes needed for publishing
```

Pros: low latency (<100ms), no polling overhead.
Cons: more infra complexity (Debezium cluster).

---

## Consumer-Side Idempotency

Outbox guarantees AT-LEAST-ONCE delivery (events may be delivered twice if publisher crashes between publish and mark-as-published).

Consumers must be idempotent:
```
Consumer receives "FiatDeducted" event with event_id=uuid-3
Check: have I processed event_id=uuid-3 already?
  → Yes: skip
  → No: process, record event_id as processed
```

This is universal: every event consumer should track processed event IDs to handle duplicates.

---

## Where to Use Outbox

```
ANYWHERE you commit data + emit an event:
  - Saga steps (state change → notify orchestrator)
  - Order created → notify shipping, billing
  - Payment processed → notify accounting
  - User registered → notify email service, analytics
  - Any audit log requirement
```

For Bitpanda: every wallet operation, every trade step, every audit entry.

---

## Rule
If you write to your DB AND need to emit an event, use the outbox pattern. Don't try to publish to Kafka directly from your app code without it — you will lose events at scale.
