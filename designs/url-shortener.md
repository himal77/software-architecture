# System Design: URL Shortener
**Designed:** Day 1 | **Scale target:** 1B users

---

## Requirements

### Functional
- User submits a long URL → gets a short code (e.g. `bit.ly/xK9mP2`)
- User clicks short URL → redirected to original URL
- Short codes expire after 30 days, recycled after grace period
- Click analytics tracked per link

### Non-Functional
- Redirect latency: <10ms
- High availability: 99.99%
- Analytics: near real-time (5-min lag acceptable)
- Scale: 1B users, millions of redirects/day

---

## The 4 Questions

| Question | Answer |
|---|---|
| Who uses it? | Consumer-facing, anonymous clickers + registered creators |
| Scale? | 1B users, ~10,000:1 read/write ratio |
| What breaks? | Redirect down = bad UX. Analytics lag = acceptable. Data loss on analytics = tolerable. |
| Constraints? | Low latency redirect is non-negotiable |

---

## Architecture

### 100 users/day
```
Client → Spring Boot App → Postgres
```

### 1M users/day
```
Client → Load Balancer → Spring Boot (3 instances) → Redis Cache → Postgres
```

### 1B users/day
```
                     ┌─── CDN (cache popular redirects at edge)
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
                │               │           Kafka Streams
                └───────────────┘           (aggregate per
                                │            5-min window)
                                │                 │
                          Postgres             Postgres
                       (sharded by           (analytics,
                        short_code)           aggregated)
```

---

## Key Design Decisions

### Sharding key: short_code (not userID)
Primary query is `WHERE short_code = ?` — no userID present. Sharding by userID would require scatter-gather across all shards.

### Short code generation: pre-generated pool
- Offline job generates unique codes in bulk
- App picks from pool at write time — no uniqueness check on critical path
- Expired codes return to pool after grace period + traffic check

### Analytics: write-behind via Kafka
- Click → Kafka topic → Kafka Streams aggregates per 5-min window → Postgres
- Redirect path never blocked by analytics writes
- Kafka retains raw events — replayable if new metrics needed

### CDN caching
- Popular links cached at CDN edge — redirect served without hitting origin
- Cache TTL aligned with code expiry

---

## Failure Modes

| Failure | Impact | Mitigation |
|---|---|---|
| Redis down | Cache miss → DB hit spike | Redis Cluster, fallback to DB |
| DB shard down | Writes/reads to that shard fail | Replication, automatic failover |
| Kafka down | Analytics events lost | Kafka replication factor ≥ 3 |
| App instance down | LB routes around it | Health checks, auto-scaling |
| Code pool empty | Cannot create new links | Alert + background job keeps pool at 80%+ |
