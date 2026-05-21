# Case Study: ESPN Live Scoreboard
**Covered:** Day 4 | **Topic:** Real-time data delivery at consumer scale

---

## The Problem
100M+ concurrent viewers during major events (Super Bowl, World Cup).
Score updates every few seconds. Must be global, low latency.

---

## Architecture They Built
- Score updates → central service → Kafka
- Kafka consumers fan out to regional edge servers
- Edge servers maintain WebSocket connections to clients
- CDN caches "current state" snapshot for new connections

---

## The Key Insight: Separate Initial Load from Live Updates
```
New user opens app:
  Step 1: CDN snapshot → current score in <50ms
  Step 2: WebSocket subscription → live updates from that point forward

Never polls. Zero wasted requests after connection.
```

---

## The Number That Drove Everything
```
Polling: 100M users × poll every 5 sec = 20M req/sec → impossible
Push:    100M users × WebSocket        = ~1,000 updates/sec → trivial
```

One architectural choice (push vs poll) reduced infrastructure requirements by 20,000×.

---

## Lessons
1. At consumer scale, always push — never poll
2. Separate initial state load (CDN snapshot) from live updates (WebSocket)
3. Fan-out via Kafka to regional edge servers keeps origin load trivial
4. Eventual consistency is perfect for scoreboards — brief stale data is acceptable
