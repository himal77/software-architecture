# Day 7 — Replication Strategies
**Chapter 2 | Phase 1: Foundations**
**Date:** 2026-05-23

---

## Key Concepts

### Why Replicate? — 3 Reasons
| Reason | Goal |
|---|---|
| **Availability** | Survive node failures |
| **Latency** | Serve users from near them |
| **Throughput** | Spread load across nodes |

Always specify *which* reason you're replicating for — it determines the strategy.

---

### The 3 Replication Strategies

```
1. Single-Leader (Master-Slave)
2. Multi-Leader (Master-Master)
3. Leaderless (Quorum-based)
```

---

## Strategy 1: Single-Leader

```
        Writes
          ↓
       LEADER
       /  |  \
   Follower Follower Follower
   (read)  (read)   (read)
```

- All writes → leader
- Leader replicates to followers
- Reads from leader (fresh) or followers (might lag)

**Used by:** Postgres, MySQL, MongoDB, Redis, MariaDB

---

### Synchronous vs Asynchronous Replication

| Mode | How | Pros | Cons |
|---|---|---|---|
| Synchronous | Leader waits for ALL followers ACK before returning | No data loss | High latency, blocked on slow followers |
| Asynchronous | Leader returns immediately, replicates in background | Low latency | Data loss if leader crashes pre-replication |
| **Semi-synchronous** | Leader waits for AT LEAST ONE follower ACK | Balanced | Most production setups |

**Anchor:** This is exactly Kafka's `acks=all` + `min.insync.replicas=2` — semi-sync replication applied to a message log.

---

### Read-After-Write Trap

```
Client writes X=5 to leader
Leader replicates to follower (200ms lag)
Client immediately reads from follower → returns X=3 (stale)
→ User confused: "I just updated my profile, why is it old?"
```

**Solutions:**
1. **Read your writes** — flag user's session, route their reads to leader for ~1 minute after write
2. **Sticky session** — same user → same replica
3. **Wait for replication** — small delay (rare)

Option 1 is the production standard.

---

## Strategy 2: Multi-Leader

```
   Region A         Region B
   Leader  ←sync→   Leader
     ↓                ↓
  Followers       Followers
```

- Multiple nodes accept writes (typically one per region)
- Leaders sync with each other
- Each region serves local users with low latency

### Killer Problem: Write Conflicts

```
Singapore writes email = "alice@new.com" at 10:00:00.500
Frankfurt writes email = "alice@old.com" at 10:00:00.600
Both succeed locally.
Replication happens. Now what?
→ Conflict resolution (LWW / merge / ask user — Day 6)
```

**When justified:**
- Multi-region writes with sub-50ms latency requirements
- Offline-first apps (CouchDB, mobile sync)
- Specific data residency / cross-DC use cases

**For 95% of systems:** single-leader is right. Multi-leader is a specialized tool.

**Used by:** CouchDB, Postgres BDR, MySQL circular replication, some DynamoDB configs.

---

## Strategy 3: Leaderless

```
            Client
              │
       ┌──────┼──────┐
       ↓      ↓      ↓
   Replica1 Replica2 Replica3
   (writes go to W replicas)
   (reads from R replicas)
   (no leader, all equal)
```

- Client writes to W replicas in parallel
- Client reads from R replicas in parallel
- W + R > N → consistency guarantee (Day 6 quorum math)
- Conflict resolution at read time using timestamps / vector clocks

**Used by:** Cassandra, DynamoDB, Riak

### Why Leaderless?

| Benefit | Detail |
|---|---|
| Massive write throughput | No leader bottleneck — 1M+ writes/sec across cluster |
| No election downtime | Node crashes → other replicas keep serving |
| Tunable consistency | Configure W and R per query |

### Mechanisms

**Read Repair:** Inconsistencies fixed lazily during reads.
```
Read from 3 replicas: X=5, X=3, X=3
Latest timestamp wins → X=5
Background: update stale replicas
```

**Hinted Handoff:** Writes to temporarily-down nodes saved as hints, delivered when node returns.

**Tradeoff:** No multi-row transactions, no joins, no foreign keys.
→ Denormalize aggressively, embrace duplicates, query-driven design.

---

## Decision Tree

```
Need writes from multiple regions with low latency?
│
├── YES: Multi-leader OR NewSQL (Spanner)
│
└── NO: Single-region or single-leader fine
    │
    Need horizontal write scale beyond one node?
    │
    ├── YES: Leaderless (Cassandra) OR sharded single-leader (Citus)
    │
    └── NO: Single-leader + read replicas (Postgres / MySQL)
```

**Default answer: single-leader.** Most systems never outgrow it.

---

## E-commerce Polyglot Design

| Component | Database | Replication | Why |
|---|---|---|---|
| **Payments** | Spanner / CockroachDB | NewSQL global ACID | Money + global + consistency |
| **Product catalog** | Postgres + CDN | Single-leader + read replicas | Read-heavy, cacheable |
| **Shopping carts** | DynamoDB / Cassandra | Leaderless + vector clocks | Cross-device merge |
| **User profiles** | Postgres | Single-leader + read replicas | Standard read-heavy |
| **Order history** | Cassandra | Leaderless | Append-only, query by user+time |

### Key Insights From Each Choice

**Payments — global ACID needed**
- Single-leader Postgres works for single-region only
- Global payments need NewSQL: Spanner uses TrueTime (atomic clocks), CockroachDB uses hybrid logical clocks

**Product catalog — single-leader, NOT leaderless**
- Read-heavy with RARE writes
- Leaderless is for write throughput, not read throughput
- Heavy CDN absorbs 99% of reads
- Single-leader + read replicas + CDN

**Shopping carts — leaderless with vector clocks**
- Cross-device editing → concurrent writes from phone + laptop
- Vector clocks detect concurrent updates → merge both
- This is the Amazon Dynamo paper (2007) use case

**User profiles — single-leader + read-after-write**
- Standard pattern for user-managed data
- Route the user's own reads to leader after write

**Order history — Cassandra, NOT Postgres**
- Append-only, partition by user_id, clustering by timestamp
- 500M+ rows → Postgres degrades, Cassandra handles natively
- Same lesson as Day 5 leaderboard

---

## Polyglot Persistence

**A real system uses multiple databases**, each chosen for its access pattern.

This is the norm at any company past medium scale. Don't try to fit everything into one database.

```
Werner Vogels (Amazon CTO):
"There is no one-size-fits-all database."
```

---

## Replication Lag — Universal Problem

| Strategy | Lag Source |
|---|---|
| Single-leader | Leader → follower replication delay |
| Multi-leader | Leader ↔ leader sync delay |
| Leaderless | Replica convergence at read time |

**Always monitor:**
- Replication lag (Postgres: `pg_stat_replication`)
- Failover time
- In-sync replica count
- Conflict count (multi-leader / leaderless)

Chapter 12 (Observability) goes deep on this.

---

## Amazon Polyglot Persistence Case Study

**The myth:** Amazon uses one giant database.
**The reality:** Dozens of different databases.

| Use case | Database |
|---|---|
| Shopping cart | DynamoDB |
| Catalog | DynamoDB + ElasticSearch |
| Order history | DynamoDB partitioned by user |
| Payments | Custom internal ACID system |
| Recommendations | Neptune (graph DB) |
| Reviews | DynamoDB + S3 |
| Inventory | DynamoDB strong consistency |

**Output:** The Amazon Dynamo paper (2007) → blueprint for Cassandra, Riak, DynamoDB.

---

## Concepts Internalized

| Concept | Status |
|---|---|
| 3 reasons to replicate | ✅ |
| Single-leader + sync/async/semi-sync | ✅ |
| Read-after-write trap and fixes | ✅ |
| Multi-leader and conflict problem | ✅ |
| Leaderless + quorum + read repair + hinted handoff | ✅ |
| Replication strategy decision tree | ✅ |
| Polyglot persistence — multiple DBs per system | ✅ |

---

## Next
**Day 8 — Chapter 2 continued:** Sharding/partitioning strategies — range, hash, directory-based — and the hotspot problem.
