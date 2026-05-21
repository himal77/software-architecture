# Redis Sorted Sets (ZSET)
**Introduced:** Day 5

---

## What It Is
A Redis data structure that stores members with a numeric score, automatically kept in sorted order. All operations are O(log N).

---

## Key Commands

```
ZADD leaderboard 1500 "player_alice"      # add/update member with score
ZINCRBY leaderboard 50 "player_alice"     # increment score, auto re-sorts
ZREVRANGE leaderboard 0 99 WITHSCORES     # top-100 in descending order
ZREVRANK leaderboard "player_alice"       # player's rank (0-indexed)
ZCARD leaderboard                         # total number of members
```

---

## Performance
- All operations: O(log N)
- 100M members: O(log 100M) ≈ 27 operations → microseconds
- 1,000 updates/sec → trivial

---

## Leaderboard Pattern
```
Write path:
  Player earns points
  → ZINCRBY leaderboard {points} {player_id}  (Redis, instant)
  → Async write to Cassandra (durability)

Read path — top-100 (shared):
  Every 30 sec → ZREVRANGE → push to CDN
  Players read from CDN edge (<10ms)

Read path — own rank (personalized):
  ZREVRANK {player_id} → microseconds
  App caches per-player for 5 sec
  Never goes through CDN (personalized data)
```

---

## When to Use Redis Sorted Sets
- Leaderboards (games, social, search ranking)
- Rate limiting (sliding window counters)
- Priority queues
- Time-series data with score = timestamp
- Any "top-N" query that needs to be fast at scale
