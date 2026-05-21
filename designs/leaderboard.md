# System Design: Distributed Game Leaderboard
**Designed:** Day 5 | **Scale target:** 100M players globally

---

## Requirements

### Functional
- Players earn points during gameplay
- Top-100 leaderboard visible to all players
- Player's own rank visible on profile

### Non-Functional
| NFR | Value |
|---|---|
| Point updates | ~1,000/sec globally |
| Top-100 lag SLA | 30 seconds |
| Own rank lag SLA | 5 seconds |
| Scale | 100M players |
| Consistency | BASE — brief staleness acceptable |

---

## Key Design Decisions

### BASE for point storage
ACID fails at scale: 1,000 writes/sec + global distribution = lock contention.
Cassandra: high write throughput, eventual consistency, horizontally scalable.

### Tunable consistency (not different models)
Both top-100 and own rank use eventual consistency — tuned differently:
```
Top-100:   Cassandra consistency=ONE   → fastest reads, 30 sec lag fine
Own rank:  Cassandra consistency=QUORUM → read majority, 5 sec lag
```

### Redis Sorted Set for computation
```
ZINCRBY leaderboard {points} {player_id}  → O(log N), auto-sorted
ZREVRANGE leaderboard 0 99                → top-100 instantly
ZREVRANK leaderboard {player_id}          → own rank instantly
```

---

## Architecture

```
Gameplay
    ↓
Point Update Service
    ├──→ ZINCRBY → Redis Sorted Set
    └──→ Async write → Cassandra (durability)

Every 30 sec background job:
    ZREVRANGE top-100 from Redis
    → Push to CDN edges globally (TTL=30sec)

Player opens leaderboard:
    → CDN edge → <10ms anywhere ✅

Player checks own rank:
    → App → ZREVRANK Redis → microseconds
    → App caches 5 sec per player ✅
```

---

## CDN vs Direct Redis

| Data | Serving layer | Why |
|---|---|---|
| Top-100 (shared) | CDN | Same for all players, cacheable |
| Own rank (personalized) | Direct Redis | Different per player, can't cache |

---

## Scale Numbers
```
1,000 ZINCRBY/sec → trivial for Redis
ZREVRANGE top-100 → microseconds regardless of 100M members
CDN absorbs 100M reads/sec for top-100
Redis only serves own-rank lookups directly
```
