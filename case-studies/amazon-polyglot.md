# Case Study: Amazon Polyglot Persistence
**Covered:** Day 7 | **Topic:** Multiple databases per system

---

## The Myth
Amazon uses one giant database.

## The Reality
Amazon uses **dozens of different databases** across the platform. Each chosen for the access pattern.

---

## Which Database for What

| Use case | Database | Why |
|---|---|---|
| Shopping cart | DynamoDB | Leaderless, cross-device, conflict merge |
| Catalog | DynamoDB + ElasticSearch | KV for IDs, search index for queries |
| Order history | DynamoDB partitioned by user | Append-only, query by user+time |
| Payments | Custom internal ACID system | Money, regulated, global |
| Recommendations | Neptune (graph) | Relationship traversal |
| Reviews | DynamoDB + S3 | Metadata + blob (images) |
| Inventory | DynamoDB strong consistency | Real-time stock counts |
| User sessions | ElastiCache (Redis) | Fast in-memory TTL |
| Logs / events | Kinesis + S3 | Streaming pipeline |

---

## The Werner Vogels Quote

> "There is no one-size-fits-all database."
> — Werner Vogels, Amazon CTO

This idea was so foundational that Amazon released the **Dynamo paper (2007)** describing how they built the cart database — which became the blueprint for:
- Cassandra
- Riak
- DynamoDB itself
- Voldemort

You'll read a summary of this paper in Chapter 5.

---

## The Lesson
Designing a serious system means using **2–4 different databases**, not one. Each component picked for:
- Access pattern (read/write/balanced)
- Query type (key, range, complex, search, graph)
- Volume (KB to PB)
- Consistency need
- Geographic distribution

**Rule:** When tempted to force everything into one database, ask: *what's the access pattern of this specific component?* Different patterns → different databases.

---

## Tradeoff
Operational complexity scales with database count. Each DB needs:
- Monitoring
- Backup strategy
- Schema management
- Team expertise

**Rule:** Add a new database only when the access pattern justifies it. Don't add Cassandra "in case we need it" — wait until Postgres genuinely cannot handle the workload.
