# WAL — Write-Ahead Log
**Introduced:** Day 21

---

## The Problem

You commit. Postgres says OK. Power fails 1ms later. Why isn't the data lost?

## The Mechanism

Before touching any data page, append the *intent* to a sequential log and flush that.

```
COMMIT sequence:
  1. Append change records to WAL
  2. fsync() WAL to physical disk        ← THE durability point
  3. Report success to the client
  4. Modify actual data pages... later, lazily, batched
```

## Why It's Fast

```
WAL:        sequential append      ~500 MB/s
Data pages: random scattered I/O   ~50 MB/s
```
The durability-critical write is **sequential**, so commits are fast. The slow random writes to
data pages happen afterwards, off the critical path.

## Crash Recovery

On restart, replay WAL from the last checkpoint: committed-but-unwritten changes are applied,
uncommitted ones discarded. This is the **D** in ACID, mechanically.

## The Same Log Powers Three More Features

```
Streaming replication  → ship WAL to a replica and replay it   (Day 7 replication)
Point-in-time recovery → replay WAL up to a chosen timestamp
Logical decoding (CDC) → decode WAL into row-level events → Debezium → Kafka
```

**The Day 9 outbox pattern can be implemented by reading the WAL** rather than polling an outbox
table. Debezium tails Postgres's WAL and publishes committed changes to Kafka — at-least-once
delivery of every commit, with no polling and no dual-write problem.

## The Tuning Knob

```
synchronous_commit = on   → fsync WAL before ack. Safe, slower. (default)
synchronous_commit = off  → ack immediately. Faster; a crash loses ~200ms of commits.
```
Trading platform: **on, always.** Losing an acknowledged trade is unacceptable.
Click-analytics table: `off` is a reasonable trade.
