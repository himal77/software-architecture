# Case Study: bit.ly / TinyURL
**Covered:** Day 1 | **Topic:** URL Shortener at scale

---

## What They Got Right Early

### Stateless app tier from day 1
- Scaling was never the app problem
- Could add instances behind a load balancer without any re-architecture

### Aggressive caching
- 90%+ of traffic never touched the database
- Redirect lookups served entirely from cache
- This bought them enormous headroom before DB scaling was needed

---

## What They Had to Re-Architect

### Started with single MySQL
- Worked fine up to ~10M links
- Added read replicas to handle read pressure
- Eventually moved hot redirect data to **Cassandra**

### Why Cassandra for redirects?
- Cassandra is optimized for key-based lookups at massive scale
- `short_code → original_url` is exactly this pattern
- Linear horizontal scaling — add nodes, capacity grows predictably
- No single master = no write bottleneck

### Kept MySQL/Postgres for
- User accounts, link metadata, billing
- Transactional data where ACID matters

---

## The One Decision That Saved Millions

**Completely separating the redirect path from the analytics path.**

- Redirect: latency-critical (<10ms). Lean stack: CDN → Cache → DB lookup.
- Analytics: not latency-critical. Can lag minutes. Separate infrastructure.

By separating them, they could:
- Scale redirect infrastructure independently (cheap, optimized for speed)
- Scale analytics infrastructure independently (optimized for throughput)
- A spike in analytics processing never degraded redirect performance

---

## The Lesson
The principle you applied instinctively — "put counts in Redis, flush later" — is the same principle bit.ly built their production system on.

**Separate your critical path from your non-critical path.**
