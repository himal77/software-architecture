# System Design: Stock Exchange Order Matching Engine
**Designed:** Day 10 | **Scope:** Order matching at exchange scale

---

## Requirements

- 50M registered traders
- Peak: 500,000 orders per second
- P99 order acknowledgement: < 10ms
- Exactly-once order matching (no double-fills)
- Full audit trail, 7-year regulatory retention
- Survive node failures without losing orders

---

## Key Design Decisions

### 1. Consistency Model: Linearizability

Two traders submitting competing orders at the same millisecond must see a single consistent order book. Once an order is matched, that state must be immediately visible to all nodes. Weaker models risk double-fills — which is fraud.

### 2. Partition Key: Instrument Symbol

```
Partition key = instrument_symbol

BTC/EUR → partition 3  (all BTC/EUR orders here)
ETH/USD → partition 7
AAPL    → partition 1

Matching engine per instrument sees complete order book.
No scatter-gather. Sub-millisecond matching.
```

Within partition: Redis ZSET sorted by price + timestamp (FIFO within price levels).

### 3. Prevent Double-Match: Idempotency + Optimistic Lock

```
Each order gets UUID at submission: order-abc123

Optimistic lock via status field:
UPDATE orders 
SET status = 'MATCHING' 
WHERE order_id = 'abc123' AND status = 'OPEN'

0 rows updated → another engine grabbed it → abort
1 row updated → you own it → match proceeds

Idempotency key: stores result for 24h → duplicate submissions return cached result
```

### 4. Audit Log: Cassandra Composite Partition Key

```
Partition key:  (instrument, date)    → no time-based hot spot
Clustering key: (timestamp, order_id) → FIFO within partition

TTL = 7 * 365 * 86400                → auto-expiry after 7 years

Replay query:
SELECT * FROM audit_log 
WHERE instrument = 'BTC/EUR' AND date = '2026-05-28'
ORDER BY timestamp ASC
```

### 5. Crash Recovery: Kafka + Outbox + Idempotency

```
Orders published to Kafka before matching → durable source of truth
Match result committed via outbox (atomic with Postgres commit)
On crash → restart, replay from last committed Kafka offset
Idempotency key prevents double-match on replay
```

---

## Architecture

```
                    Order Submission
                         │
                 Idempotency-Key: order-abc123
                         ↓
                   API Gateway
                         │
                         ↓
                      Kafka
                  (orders topic)       ← source of truth
                         │
                         ↓
              Order Matching Engine
              (one instance per instrument partition)
                         │
              Optimistic lock: OPEN → MATCHING
                         │
                    ┌────┴────┐
                    ↓         ↓
               Order Book   Postgres
               (Redis ZSET)  + outbox
               price+time    MATCHING→FILLED
               sorted                │
                                     ↓
                              Outbox Publisher
                                     │
                                     ↓
                                   Kafka
                              (matches topic)
                                     │
                    ┌────────────────┼──────────────┐
                    ↓                ↓               ↓
              Wallet Service    Audit Service    Notification
              (debit/credit)    (Cassandra)      (email/push)
              + outbox          (instrument,date)
                                partition
```

---

## Failure Scenarios

| Failure | Outcome | Recovery |
|---|---|---|
| Engine crashes mid-match | Outbox not written, status stays OPEN | Restart, replay Kafka, idempotency prevents double-match |
| Kafka partition leader fails | Orders queue, no loss | Kafka replication (ISR), resume after leader election |
| Wallet service down | Match result queued in Kafka | Wallet replays on restart, outbox ensures delivery |
| Audit service down | Trade succeeded, audit retries forever | Idempotent write, Kafka retention |
| Duplicate order submission | Idempotency key hit → cached result returned | No re-execution |

---

## Concepts Applied

| Concept | Where Used |
|---|---|
| Linearizability | Order book consistency — no weaker model is safe |
| Hash partitioning | Partition by instrument, all orders co-located |
| Redis ZSET | Order book sorted by price + timestamp |
| Optimistic locking | Prevent race condition on order status |
| Idempotency keys | Exactly-once semantics for orders |
| Outbox pattern | Reliable match event to Kafka |
| Kafka | Durable order log, crash recovery replay |
| Cassandra composite key | Audit log — spread load, fast replay |
| Distributed snapshots (Flink) | Analytics pipeline on match events |
| Polyglot persistence | Redis + Postgres + Cassandra + Kafka |
