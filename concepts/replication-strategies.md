# Replication Strategies
**Introduced:** Day 7

---

## Why Replicate
- **Availability** — survive node failures
- **Latency** — serve users near them
- **Throughput** — spread load across nodes

Specify which goal — drives the strategy choice.

---

## Single-Leader (Master-Slave)

```
Writes → Leader → Followers (replicate)
Reads → Leader (fresh) or Followers (might lag)
```

**Modes:**
- Synchronous — wait for all followers (no data loss, slow)
- Asynchronous — return immediately (fast, data loss risk)
- **Semi-synchronous** — wait for at least one follower (production standard)

**Same pattern as Kafka:** `acks=all` + `min.insync.replicas=2`

**Read-after-write trap:** User writes, immediately reads from follower → sees stale data. Fix: route the writer's reads to leader for ~1 minute after write.

**Used by:** Postgres, MySQL, MongoDB, Redis, MariaDB.

---

## Multi-Leader (Master-Master)

```
Region A Leader  ←sync→  Region B Leader
   ↓                        ↓
Followers                Followers
```

**Pros:** Multi-region writes with low local latency.
**Killer problem:** Write conflicts when same record updated in two regions. Need conflict resolution (LWW, merge, ask user).

**Justified for:**
- Multi-region writes <50ms latency requirement
- Offline-first apps (CouchDB)
- Specific data residency needs

**Used by:** CouchDB, Postgres BDR, MySQL circular replication.

**For 95% of systems:** single-leader is right. Multi-leader is specialized.

---

## Leaderless (Quorum-based)

```
Client → writes to W replicas
Client → reads from R replicas
W + R > N → consistency guarantee
```

**Mechanisms:**
- **Read repair** — fix stale replicas during reads
- **Hinted handoff** — store writes for down nodes, replay on recovery

**Tradeoffs:**
- Massive write throughput (no leader bottleneck)
- No election downtime
- Tunable consistency per query
- No multi-row transactions, no joins → denormalize, query-driven design

**Used by:** Cassandra, DynamoDB, Riak.

---

## Decision Tree

```
Multi-region writes with low latency? → Multi-leader OR NewSQL
Horizontal write scale beyond one node? → Leaderless OR sharded single-leader
Otherwise → Single-leader + read replicas
```

**Default answer: single-leader.**

---

## Quick Reference

| Strategy | Best for | Example |
|---|---|---|
| Single-leader | Most apps, transactional data | Postgres, MySQL |
| Multi-leader | Multi-region writes, offline-first | CouchDB |
| Leaderless | High write throughput, tunable consistency | Cassandra, DynamoDB |
| NewSQL | Global ACID | Spanner, CockroachDB |
