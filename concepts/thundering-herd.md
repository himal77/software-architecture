# Thundering Herd Problem
**Introduced:** Day 2 | Deepened: Chapter 6 — Caching (Days 29–33)

---

## What It Is
When a cache entry expires or is cold, many simultaneous requests all miss the cache and hit the DB at the same time — overwhelming it with duplicate queries for the same data.

---

## Example
```
Viral paste "xK9mP2" — not yet cached
10,000 users click link simultaneously
→ All 10,000 hit Redis → all miss
→ All 10,000 query Postgres for same row
→ DB crushed under 10,000 identical queries
→ DB returns same result 10,000 times
→ All 10,000 try to write to Redis simultaneously
```

---

## Solution: Cache Mutex (Locking)
```
Request 1: cache miss → acquires lock → queries DB → populates cache → releases lock
Requests 2–9,999: see lock → wait briefly → read from cache (already populated)
```
Result: 1 DB query instead of 10,000.

---

## Other Solutions

| Solution | How | Best for |
|---|---|---|
| Cache mutex | Lock on miss, others wait | General purpose |
| Cache warming | Pre-populate cache before content goes live | Predictable viral content |
| Probabilistic early expiry | Randomly refresh before TTL expires | High-traffic keys |
| Request coalescing | Queue duplicate requests, serve one DB response to all | High concurrency |

---

## Related Problems
- **Cache stampede:** Same as thundering herd — triggered by TTL expiry
- **Cache avalanche:** Many different cache keys expire at same time → DB hit on all of them simultaneously. Solution: randomize TTL slightly (e.g. 30min ± 5min random jitter)

---

## Rule
Any cache key that could receive high concurrent traffic needs protection against thundering herd. A mutex or probabilistic refresh is the minimum.
