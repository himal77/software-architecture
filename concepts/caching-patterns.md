# Caching Patterns
**Introduced:** Day 1 | Deepened: Chapter 6 (Days 29–33)

---

## Write-Behind Caching (Write-Behind / Write-Back)
**Used for:** Non-critical, high-volume writes (e.g. click counts, view counts)

```
Write → Cache (fast) → async flush to DB (batched, periodic)
```

**Example — URL shortener click counts:**
- Click happens → `Redis INCR counter` (atomic, instant)
- Every 5 minutes → flush aggregated counts to Postgres
- Result: 288x fewer DB writes than per-click writes

**Tradeoff:**
- ✅ DB write pressure massively reduced
- ✅ Redis INCR is atomic — no race conditions
- ❌ Cache crash = data loss for unflushed window → mitigate with Redis AOF persistence

---

## Why 5 Minutes, Not Once Per Day
- Once/day = dashboard only shows yesterday's data
- 5 min = near real-time analytics, still 288x fewer writes than per-click
- Flush frequency = tradeoff between freshness and DB pressure

---

## Cache Patterns (preview — deepened in Chapter 6)

| Pattern | Flow | Best for |
|---|---|---|
| Cache-aside | App checks cache, on miss reads DB + populates cache | Read-heavy, flexible |
| Write-through | Write to cache AND DB synchronously | Consistency critical |
| Write-behind | Write to cache, async flush to DB | High-volume, non-critical writes |
| Read-through | Cache handles DB reads transparently | Simplifies app code |

---

## Read/Write Ratio Rule
Always identify the read/write ratio before choosing a caching strategy.

- URL shortener: 1 write → ~10,000 reads → cache first, shard second
- Cache can buy you 100x scale before sharding is needed
