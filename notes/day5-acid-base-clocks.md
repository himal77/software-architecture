# Day 5 — ACID vs BASE in Practice, Clocks & Consensus
**Chapter 2 | Phase 1: Foundations**
**Date:** 2026-05-21

---

## Key Concepts

### Where ACID Breaks Down at Scale

**Atomicity across services (distributed transaction problem):**
```
Order Service:   deduct inventory  ✅
Payment Service: charge card       ✅
Shipping Service: create shipment  ❌ FAILS
→ Cannot rollback across separate services/DBs with a DB transaction
→ Solution: Saga pattern (Chapter 10)
```

**Isolation at scale — lock contention:**
```
1,000 concurrent writes → row locking → queue builds up → latency spikes
Solution: choose the right isolation level
```

**Isolation Levels (weakest → strongest):**

| Level | Prevents | Performance | Use when |
|---|---|---|---|
| Read Uncommitted | Nothing | Fastest | Never in production |
| Read Committed | Dirty reads | Fast | Postgres default, most apps |
| Repeatable Read | Dirty + non-repeatable reads | Medium | Reports, analytics |
| Serializable | All anomalies | Slowest | Financial transactions |

**Rule:** ACID fails at scale not because of consistency requirements — because of **lock contention and inability to scale horizontally across regions.**

---

### BASE in Practice — Conflict Resolution Patterns

**Pattern 1: Last Write Wins (LWW)**
```
Two nodes accept writes for same key:
Node 1: X=5 at T=10:00:01.500
Node 2: X=3 at T=10:00:01.600
→ Latest timestamp wins → X=3
```
Simple but dangerous — wall clock drift makes "latest" unreliable. Cassandra default.

**Pattern 2: Read Repair**
```
Client reads from 3 replicas:
Replica 1: X=5 (stale)
Replica 2: X=3 (latest)
Replica 3: X=3 (latest)
→ Returns X=3 to client
→ Background: repairs Replica 1 to X=3
```
Repairs inconsistency lazily during reads. Used by Cassandra, DynamoDB.

**Pattern 3: Quorum Reads/Writes**
```
N=3 replicas, W=2 (write quorum), R=2 (read quorum)
W + R > N → 2+2 > 3 → guaranteed to read at least one node with latest write
→ Strong consistency without full ACID
```
Cassandra tunable consistency. Same concept as Kafka `min.insync.replicas`.

---

### The 3 Clock Problems

**1. Clock Skew** — two servers show different times simultaneously
```
Server A: 10:00:01.500
Server B: 10:00:01.350  ← 150ms behind
→ NTP corrects to ~100ms — not good enough for ms-level ordering
```

**2. Clock Drift** — server clock runs fast/slow, diverges over time
```
After 24 hours: 200ms ahead of real time
After 1 week: 1.4 seconds ahead
NTP corrects periodically but drift accumulates between corrections
```

**3. Time Goes Backwards** — NTP correction can move clock backward
```
Event 1 timestamp: 10:00:01.500
NTP correction: clock jumps back 200ms
Event 2 timestamp: 10:00:01.350 ← appears before Event 1
→ Ordering logic breaks
```

**Fix in code:** Use monotonic clocks for durations (`System.nanoTime()` in Java), wall clocks only for display (`System.currentTimeMillis()`).

**Rule: Never use wall clock timestamps to determine event ordering in distributed systems.**

---

### Lamport Timestamps (Logical Clocks)

Key insight (Leslie Lamport, 1978):
> You don't need to know WHEN something happened. You only need to know WHAT HAPPENED BEFORE WHAT.

```
Node A: event → counter=1 → sends message with counter=1
Node B: receives → max(0, 1) + 1 = counter=2
→ Node A's event (1) provably happened before Node B's response (2)
No clocks. No time. Just causality.
```

**Kafka uses this:** Offsets are logical clocks per partition. Offset 1042 happened before 1043. No timestamps needed. Consumer lag = how many offsets behind.

---

### Vector Clocks

Extends Lamport to detect **concurrent events** (neither happened before the other).

Each node tracks a counter for every other node:
```
3 nodes A, B, C → each maintains [A_count, B_count, C_count]

Node A: [1,0,0] → sends to B
Node B: receives → [1,0,0] → event → [1,1,0] → sends to C
Node C: receives → [1,1,0] → event → [1,1,1]

[1,1,0] vs [0,2,0] → neither dominates → CONCURRENT events → conflict!
```

Used by: Amazon DynamoDB, Riak, CRDTs.

---

### NewSQL — Global ACID

When you need both global scale AND strong consistency (payments, banking):

| Database | Type | Use when |
|---|---|---|
| Postgres | ACID, single region | Standard transactional apps |
| Cassandra | BASE, global | High write throughput, eventual ok |
| **Google Spanner** | ACID + global scale | Global payments, financial systems |
| **CockroachDB** | ACID + global scale | Fintech startups needing global ACID |

Spanner uses atomic clocks + GPS to synchronize time globally within ~7ms — enabling global linearizable transactions.

---

## Leaderboard Design

### Requirements
- 100M players, 1,000 point updates/sec
- Top-100 leaderboard: 30 sec lag SLA
- Own rank: 5 sec lag SLA

### Key Decisions

**BASE for point storage:**
- 1,000 writes/sec globally — ACID lock contention would crush it
- Brief inconsistency (player sees stale score for 1 sec) = harmless
- Cassandra: high write throughput, eventual consistency

**Eventual consistency for both leaderboard and own rank — tuned differently:**
```
Top-100:   consistency=ONE   → fastest, 30 sec lag fine
Own rank:  consistency=QUORUM → read majority of replicas, 5 sec lag
```
Same model, different tuning. Don't change the model — tune the quorum.

**Redis Sorted Set for leaderboard computation:**
```
ZADD leaderboard 1500 "player_alice"
ZINCRBY leaderboard 50 "player_alice"  → O(log N), auto re-sorts
ZREVRANGE leaderboard 0 99             → top-100 instantly
ZREVRANK leaderboard "player_alice"    → own rank instantly, O(log N)
```
100M members in Redis Sorted Set: all operations in microseconds.

### Architecture
```
Gameplay
    ↓
Point Update Service
    ├──→ ZINCRBY → Redis Sorted Set (computation, O(log N))
    └──→ Async → Cassandra (durability)

Every 30 sec:
Background job → ZREVRANGE top-100 from Redis
              → Push to CDN edges globally (TTL=30sec)

Player opens leaderboard:    CDN edge → <10ms ✅
Player checks own rank:      App → ZREVRANK Redis → microseconds ✅
                             App caches per-player 5 sec
```

**CDN for shared public data (top-100). Direct Redis for personalized data (own rank).**

---

## Clash of Clans Case Study

**Problem:** Global clan leaderboards for 100M+ players across dozens of game servers.

**Architecture:**
- Game servers write scores to regional Redis Sorted Sets
- Global leaderboard: top-N per region merged and re-sorted centrally every 5 minutes
- Result pushed to CDN globally
- 99% of queries hit CDN. Redis handles own-rank + regional top lists only.

**The key insight:**
Players care about regional ranking in real-time and global ranking approximately. Split:
- Regional leaderboard: real-time, Redis ZSET per region
- Global leaderboard: 5-min batch, merged + CDN

**Lesson:** When global real-time is too expensive, ask if regional real-time + global approximate is good enough. It almost always is.

---

## Concepts Internalized

| Concept | Status |
|---|---|
| ACID isolation levels | ✅ |
| Why ACID fails at scale (lock contention, not consistency) | ✅ |
| BASE conflict patterns: LWW, read repair, quorum | ✅ |
| 3 clock problems: skew, drift, backwards | ✅ |
| Lamport timestamps — causality without clocks | ✅ |
| Vector clocks — detecting concurrent events | ✅ |
| NewSQL: Spanner/CockroachDB for global ACID | ✅ |
| Redis Sorted Sets — O(log N) leaderboard | ✅ |
| Tunable consistency — same model, different quorum | ✅ |
| CDN for shared data, direct cache for personalized | ✅ |

---

## Next
**Day 6 — Chapter 2 continued:** Consensus algorithms — why distributed agreement is hard, Paxos/Raft simplified, leader election.
