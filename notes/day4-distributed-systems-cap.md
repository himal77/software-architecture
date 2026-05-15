# Day 4 — Distributed Systems Theory: CAP Theorem
**Chapter 2 | Phase 1: Foundations**
**Date:** 2026-05-14

---

## Key Concepts

### CAP Theorem
A distributed system can only guarantee **two of three** simultaneously:
- **C — Consistency:** every read returns the most recent write
- **A — Availability:** every request receives a response (not an error)
- **P — Partition Tolerance:** system continues when nodes can't communicate

**The trap:** CAP is NOT a free choice of two. Network partitions are not optional — they will happen. The real choice is always:

```
CP — reject requests during partition (consistency over availability)
AP — serve stale data during partition (availability over consistency)
```

CA only exists in single-node systems — not distributed systems.

---

### Real CAP Choice — Bank Account Example

Network partition between Frankfurt and Singapore nodes:

```
CP (banks): User tries to withdraw → system rejects request
            "Service temporarily unavailable"
            → User frustrated, but no money lost ✅

AP (social media): User reads like count → gets stale number
                   "1,000 likes" instead of "1,003 likes"
                   → Harmless for a few seconds ✅
```

---

### Real Systems CAP Choices

| System | Choice | Reason |
|---|---|---|
| Zookeeper, etcd | CP | Distributed coordination must be consistent |
| Cassandra, DynamoDB | AP | Availability prioritised, eventual consistency ok |
| HBase | CP | Strong consistency for analytics |
| Kafka (default) | AP | Message delivery prioritised |
| Kafka (acks=all) | → CP | Trades write latency for consistency |
| Postgres (single node) | CA → CP in distributed | ACID requires consistency |
| DNS | AP | Availability critical, stale records tolerable |

**Kafka insight:** `acks=all` + `min.insync.replicas=2` = CP configuration. Sacrifices write throughput for consistency guarantee.

---

### Consistency Models (strongest → weakest)

**1. Linearizability (Strict Consistency)**
- Every read anywhere returns the most recent write immediately
- Most expensive — requires coordination between all nodes
- Used by: etcd, Zookeeper, Google Spanner, single-node Postgres
- Example: bank balance must always be exact

**2. Sequential Consistency**
- All nodes agree on operation order, but not necessarily real-time
- Cheaper than linearizable
- Example: distributed lock managers

**3. Causal Consistency**
- Causally related operations appear in correct order
- Independent operations can appear in any order
- Used by: MongoDB causal sessions, some Cassandra configs
- Example: must see a post before seeing a reply to that post

**4. Eventual Consistency**
- All replicas converge to same value given no new updates
- No guarantee on when convergence happens
- Cheapest, most available
- Used by: Cassandra, DynamoDB, DNS
- Example: social media likes, shopping cart, user profiles, scoreboards

---

### Choosing the Right Consistency Model

| Data relationship | Model |
|---|---|
| Must be exact right now | Linearizable |
| Cause-effect relationships between writes | Causal |
| Independent updates, "eventually same" is fine | Eventual |

**Rule:** Match the consistency model to the actual relationship between your data — not to what feels safest.

---

## Scoreboard Design — AP + Eventual Consistency

### Requirements
- 500M global users, 10,000 concurrent matches
- Score updates every ~30 sec
- Acceptable lag: <5 seconds
- Brief incorrect score tolerable, permanently wrong is not

### CAP Choice: AP
- 5 sec lag acceptable → availability over consistency
- Users seeing slightly stale score is tolerable
- System keeps serving during network partitions

### Consistency Model: Eventual (not causal)
- Score updates are independent events — no cause-effect relationship
- Causal consistency adds overhead with no benefit
- Eventual consistency: score propagates to all regions within SLA

### Why Redis Alone Fails for 500M Global Users
```
Redis in Frankfurt
Singapore user → 150ms round trip
500M users × 1 req/30sec = 16.7M req/sec on one cluster → impossible
```
Redis solves the DB problem. CDN solves the global distribution problem.

### Why Polling Fails at Scale
```
500M users × poll every 5 sec = 100M req/sec → impossible
Solution: WebSocket push model
Score changes → push to connected clients
333 score updates/sec total → trivial to handle
```

### Complete Architecture

```
Admin Panel
    ↓
Score Update Service
    ↓
    ├──→ Postgres (durable, source of truth)
    ├──→ Redis (hot cache, origin)
    └──→ Kafka topic "score-updates"
                ↓
        Fan-out Service
                ↓
    ┌───────────┼───────────┐
    ↓           ↓           ↓
CDN edge    CDN edge    CDN edge
(Americas)  (Europe)    (Asia)
    ↓           ↓           ↓
WebSocket   WebSocket   WebSocket
servers     servers     servers
    ↓           ↓           ↓
         500M users
```

### Write Flow
```
Admin updates score
→ Postgres (source of truth)
→ Redis (origin cache)
→ Kafka → fan-out service → regional WebSocket servers → clients
Total lag: ~2–4 seconds ✅
```

### Read Flow (new user)
```
Client connects → nearest WebSocket server
→ fetch current score from regional Redis/CDN (snapshot)
→ subscribe to live WebSocket push updates
→ receives score changes as they happen
```

### Component Responsibilities

| Component | Solves |
|---|---|
| Postgres | Durability — permanent source of truth |
| Redis | Origin cache — protects DB from read load |
| CDN edges | Global distribution — <10ms from anywhere |
| Kafka | Reliable fan-out to all regions |
| WebSockets | Push model — eliminates polling at 500M scale |

---

## ESPN Case Study

**Problem:** 100M+ concurrent viewers (Super Bowl, World Cup). Score updates every few seconds.

**Architecture:**
- Score updates → central service → Kafka
- Kafka consumers fan out to regional edge servers
- Edge servers maintain WebSocket connections to clients
- CDN caches "current state" snapshot for new connections

**Key insight: Separate initial load from live updates**
```
New user: CDN snapshot → <50ms (current score)
         + WebSocket subscription → live updates from that point
Never polls. Zero wasted requests.
```

**The number that drove everything:**
```
100M users × poll every 5 sec = 20M req/sec → impossible
100M users × WebSocket push   = ~1,000 updates/sec → trivial
```

**Lesson:** At consumer scale, always push. Never poll.

---

## Concepts Internalized

| Concept | Status |
|---|---|
| CAP theorem — real choice is CP vs AP | ✅ |
| Network partitions are not optional | ✅ |
| 4 consistency models + when each fits | ✅ |
| Causal vs eventual — when each applies | ✅ |
| Redis alone doesn't solve global distribution | ✅ |
| Push (WebSocket) vs poll — scale difference | ✅ |
| CDN for global low-latency reads | ✅ |
| Fan-out via Kafka to regional edges | ✅ |

---

## Next
**Day 5 — Chapter 2 continued:** ACID vs BASE, clocks in distributed systems (wall clock, logical clock, vector clock).
