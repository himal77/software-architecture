# Day 10 — Distributed Snapshots & Chapter 2 Wrap-up
**Chapter 2 | Phase 1: Foundations**
**Date:** 2026-05-28

---

## What Was Covered

Final day of Chapter 2. Distributed snapshots (Chandy-Lamport), Chapter 2 concept map, and a comprehensive order matching engine design integrating all Chapter 2 concepts.

---

## Concept: Distributed Snapshots

### The Problem

Taking a consistent backup of a distributed system without pausing it.

**Naive approach:** pause everything, snapshot, resume → impossible in production.

**The real problem:** messages in flight during snapshot create inconsistency.
```
Node A: balance = 0       (just sent €60K, snapshot taken)
Message in flight: €60K
Node B: balance = 0       (hasn't received yet, snapshot taken)

Total in snapshot = €0. Actual total = €60K. Snapshot is garbage.
```

### Chandy-Lamport Algorithm (1985)

Use a special MARKER message to divide time into "before snapshot" and "after snapshot" — without stopping the system.

**Rule 1 — Initiator:**
```
1. Record your own local state
2. Send MARKER on every outgoing channel
3. Start recording messages on each incoming channel
```

**Rule 2 — First time receiving MARKER:**
```
1. Record your own local state immediately
2. Stop recording on the channel you received MARKER from (state = empty)
3. Send MARKER on all outgoing channels
4. Start recording on all other incoming channels
```

**Rule 3 — Receiving MARKER again (different channel):**
```
Stop recording on that channel.
Channel state = all messages recorded since first snapshot.
```

**Result:** every node's state + every in-flight message = consistent global snapshot.

### Why It Works

MARKER acts as a logical timestamp. Messages before MARKER belong to snapshot. Messages after do not.

```
A sends: [€60K payment] → [MARKER] → [€5K payment]
B records [€60K] as in-flight
B receives MARKER → stops recording this channel
B receives [€5K] → post-snapshot, ignored

Snapshot correctly captures: €60K was in flight.
```

No global pause. No clock synchronization needed.

### Production Usage

**Apache Flink (exactly-once stream processing):**
```
Every few seconds: inject BARRIER into Kafka stream (= Chandy-Lamport MARKER)
When all operators processed barrier → consistent checkpoint saved to storage
On failure: restart from checkpoint, replay Kafka from that offset
Result: exactly-once processing guarantees
```

**Kubernetes etcd:** periodic cluster state snapshots for disaster recovery.

---

## Chapter 2 — Full Concept Map

```
DISTRIBUTED SYSTEMS THEORY
│
├── ORDERING & TIME
│   ├── Clock skew/drift → can't trust wall clocks
│   ├── Lamport clocks → causal ordering
│   ├── Vector clocks → detect concurrency
│   └── Distributed snapshots → consistent global state
│
├── CONSISTENCY
│   ├── CAP theorem → CP vs AP choice
│   ├── 4 consistency models → linearizable → eventual
│   ├── ACID → isolation levels → Read Committed (PG default)
│   └── BASE → eventual consistency for scale
│
├── CONSENSUS
│   ├── FLP impossibility → safety vs liveness tradeoff
│   ├── Raft → leader election + log replication + quorum
│   └── Distributed locks → TTL lease pattern
│
├── DATA DISTRIBUTION
│   ├── Replication → single-leader / multi-leader / leaderless
│   ├── Partitioning → range / hash / directory
│   ├── Consistent hashing → minimal data movement on scale
│   └── Polyglot persistence → right DB for right job
│
└── TRANSACTIONS
    ├── 2PC → atomic but blocking, fails with external systems
    ├── Saga → orchestration vs choreography
    ├── Outbox → reliable event publishing
    └── Idempotency → exactly-once semantics
```

---

## Design Problem: Stock Exchange Order Matching Engine

**Scale:** 50M traders, 500K orders/sec peak, P99 < 10ms, 7-year audit retention

### Architecture

```
                    Order Submission
                         │
                 Idempotency-Key: order-abc123
                         ↓
                   API Gateway
                         │
                         ↓
                      Kafka
                  (orders topic)       ← source of truth, crash recovery
                         │
                         ↓
              Order Matching Engine
              (partitioned by instrument)
                         │
              Optimistic lock: OPEN → MATCHING
                         │
                    ┌────┴────┐
                    ↓         ↓
               Order Book   Postgres
               (Redis ZSET)  + outbox
               price+time    (MATCHING→FILLED)
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
              + outbox          instrument+date
                                partition
```

### Key Design Decisions

**1. Consistency model: Linearizability**
Two traders competing on the same order must see a single consistent order book. Linearizability guarantees once an order is matched, that state is immediately visible everywhere. Weaker models risk double-fills.

**2. Partition key: instrument symbol**
```
Partition key = instrument_symbol (BTC/EUR, ETH/USD, AAPL)
All orders for same instrument → same partition
Matching engine sees complete order book for that instrument
No scatter-gather needed
```
Within partition: sorted by price + timestamp (FIFO within price levels) using Redis ZSET.

**3. Prevent double-match: idempotency + optimistic locking**
```
Step 1 — Idempotency key per order (UUID)
Step 2 — Optimistic lock via status field:

UPDATE orders 
SET status = 'MATCHING' 
WHERE order_id = 'abc123' AND status = 'OPEN'

0 rows updated → already grabbed by another engine → abort
1 row updated → you own it → proceed
```

**4. Audit log: Cassandra with composite partition key**
```
Partition key:  (instrument, date)    → spreads load, no time-based hot spot
Clustering key: (timestamp, order_id) → sorted for replay

TTL = 7 * 365 * 86400 seconds → auto-expiry after 7 years

SELECT * FROM audit_log 
WHERE instrument = 'BTC/EUR' AND date = '2026-05-28'
ORDER BY timestamp ASC
→ Full day replay for regulators in one query
```

**5. Crash recovery: Kafka + outbox + idempotency**
```
Orders published to Kafka BEFORE matching (source of truth)
Match result written via outbox (atomic with DB commit)
On crash: restart, replay from last Kafka offset
Idempotency key prevents double-match on replay
```

### Concepts Applied

| Concept | Where Used |
|---|---|
| Linearizability | Order book consistency requirement |
| Hash partitioning | Partition by instrument symbol |
| Redis ZSET | Order book sorted by price+time |
| Optimistic locking | Prevent double-match race condition |
| Idempotency keys | Exactly-once order execution |
| Outbox pattern | Reliable match event publishing |
| Kafka | Durable order log + crash recovery |
| Cassandra composite key | Audit log — no hot spot, fast replay |
| Distributed snapshots | Flink checkpoint for analytics pipeline |

---

## Quiz Gaps to Drill (Day 11 onwards)

| Gap | Correct Answer |
|---|---|
| Raft quorum formula | ⌊N/2⌋ + 1. For 5 nodes = 3 |
| 2PC fatal flaw | Coordinator crash blocks all participants holding locks |
| Hot spot patterns | Time-based, celebrity, sequential ID — and fixes |
| Lamport clocks | Logical counter for causal ordering. Cannot detect concurrency |
| Read-after-write | Read own writes from leader / track LSN / short window on leader |
| Read Committed anomaly | Allows non-repeatable reads |
