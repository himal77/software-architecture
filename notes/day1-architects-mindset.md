# Day 1 — The Architect's Mindset
**Chapter 1 | Phase 1: Foundations**
**Date:** 2026-05-11

---

## Key Concepts

### Engineer vs Architect
- Engineer asks: "How do I build this?"
- Architect asks: "Should we build it this way, and what happens when it breaks at scale?"

### The Trade-off Triangle
Optimize for two, never all three:
- Performance + Reliability = expensive
- Performance + Cost = unreliable
- Reliability + Cost = slower

### The 4 Questions (ask before drawing anything)
1. **Who uses it, and how?** — internal vs consumer, read-heavy vs write-heavy, sync vs async
2. **What are the scale numbers?** — DAU, requests/sec, data volume. Never design without numbers.
3. **What breaks, and what's the cost?** — defines RPO (data loss tolerance) and RTO (recovery time)
4. **What are the constraints?** — budget, timeline, compliance, existing stack

---

## Scale Inflection Points

| Inflection | Problem | Solution |
|---|---|---|
| 1 server → multiple | Stateful sessions break | Externalize state (Redis), design stateless from day 1 |
| App scales, DB doesn't | DB becomes bottleneck | Read replicas, caching, eventually sharding |
| Single region → multi-region | Distributed systems problems | CAP tradeoffs, clock sync, network partitions |

**Rule:** Design stateless from day 1, even with one server.

---

## URL Shortener — Design Evolution

### 100 users/day
```
Client → Spring Boot App → Postgres
```
No cache, no sharding. Premature optimization is waste.

### 1M users/day
```
Client → Load Balancer → Spring Boot (3 instances) → Redis Cache → Postgres
```
Cache handles ~95% of redirect traffic. DB handles writes + cache misses only.

### 1B users/day
```
Client → CDN (cache redirects at edge)
       → Load Balancer
       → Spring Boot (N instances, stateless)
       → Redis Cache
       → Postgres (sharded by short_code hash)
```

---

## Key Design Decisions & Lessons

### Shard by query pattern, not by logical ownership
- Wrong: shard by userID (redirect query has no userID)
- Correct: shard by short_code (that's what you query on)
- **Rule:** Always ask "what query hits this shard?"

### Read/write ratio shapes your first optimization
- URL shorteners: 1 write → ~10,000 reads
- Cache first, shard second for read-heavy systems
- Cache buys you 100x before sharding is needed

### Short Code Generation — 3 Approaches

| Approach | How | Pro | Con |
|---|---|---|---|
| Hash-based | `base62(MD5(url)).first(6)` | No coordination, scales infinitely | Collision possible, same URL = same code |
| Centralized counter | `Redis INCR` → base62 encode | Simple, atomic, no collision | Redis = single point of failure |
| Pre-generated pool | Offline job fills pool, app picks from it | No uniqueness check at write time, works with expiry/recycle | Needs background job |

**Best fit with expiry/recycling:** Pre-generated pool — verify uniqueness offline, not at request time.

### Code Expiry + Recycling
- Expire after 30 days, recycle to pool
- Add grace period (e.g. 7 days) before recycling
- Never recycle high-traffic codes — security risk (attacker waits for popular code to expire)

### Analytics — Write-Behind Caching
- Every click → `Redis INCR counter` (atomic, no race conditions)
- Flush to DB every 5 minutes (not once per day — too coarse for real-time dashboards)
- 288x fewer DB writes than per-click writes

### Analytics at 1B scale — Kafka
```
Click → Kafka topic "click-events"
              ↓
        Kafka Streams (aggregate per 5-min window, by country, device)
              ↓
        DB write (aggregated)
```
- Replayable: add new metrics retroactively
- Zero data loss: Kafka holds events if DB is down
- Redis flush: simpler, good to 1M users. Kafka: production choice at scale.

---

## Final Architecture (1B scale)

```
                     ┌─── CDN (cache popular redirects)
Client ──────────────┤
                     └─── Load Balancer
                                │
                  ┌─────────────┴─────────────┐
             App Instance 1           App Instance N   (stateless)
                  └─────────────┬─────────────┘
                                │
                ┌───────────────┼────────────────┐
                │               │                │
           Redis Cache    Pre-gen Code Pool    Kafka
           (redirect       (uniqueness         (click events)
            lookups)        guaranteed)             │
                │               │           Stream Processor
                └───────────────┘                 │
                                │                 │
                          Postgres             Postgres
                       (sharded by           (analytics,
                        short_code)           aggregated)
```

---

## The Principle That Ties It All Together

**Separate your critical path from your non-critical path.**

- Redirect = latency-critical (<10ms). Lean, fast, cache-first.
- Analytics = not latency-critical. Can lag minutes. Different infrastructure.

This is why write-behind caching for click counts works — you instinctively separated the two paths.

---

## bit.ly Case Study

- **Got right early:** Stateless app tier from day 1. Aggressive caching — 90%+ traffic never hit DB.
- **Had to re-architect:** Started with single MySQL → read replicas → moved hot data to Cassandra for redirect path (optimized for key lookups at scale).
- **The key decision:** Completely separated redirect path from analytics path. Scale each independently.

---

## Concepts Internalized

| Concept | Status |
|---|---|
| Architect's 4 questions | ✅ |
| Scale inflection points | ✅ |
| Shard by query pattern, not ownership | ✅ |
| Read-heavy → cache first | ✅ |
| Offline uniqueness beats online check | ✅ |
| Write-behind for non-critical writes | ✅ |
| Separate critical path from non-critical | ✅ |

---

## Next
**Day 2 — Chapter 1 continued:** How to frame any problem you've never seen before — the structured approach architects use before drawing a single box.
