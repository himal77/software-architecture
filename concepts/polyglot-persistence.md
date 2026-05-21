# Polyglot Persistence
**Introduced:** Day 7

---

## What It Is
Using **multiple different databases** in a single system, each chosen for the specific access pattern of one component.

> "There is no one-size-fits-all database." — Werner Vogels, Amazon CTO

---

## Why
A relational DB is great for transactions, terrible for storing 500M rows of append-only data. A graph DB is great for relationships, terrible for documents. A KV store is fast but can't run aggregations. **No database does everything well.**

---

## E-commerce Example

| Component | Database | Why |
|---|---|---|
| Payments | NewSQL (Spanner / CockroachDB) | Global ACID needed |
| Product catalog | Postgres + CDN | Read-heavy, cacheable |
| Shopping carts | DynamoDB / Cassandra | Cross-device, leaderless, vector clocks |
| User profiles | Postgres | Standard read-heavy app data |
| Order history | Cassandra | Append-only, partition by user |
| Search | Elasticsearch | Full-text, faceted search |
| Recommendations | Neo4j / Neptune | Graph traversals |
| Sessions | Redis | In-memory, fast TTL |
| Files / images | S3 | Blob storage, cheap, scalable |

---

## How to Decide

For each component, ask:
1. **Access pattern** — read-heavy, write-heavy, balanced?
2. **Query type** — single key, range, complex, full-text, graph?
3. **Volume** — KB, MB, GB, TB, PB?
4. **Consistency need** — strict, eventual, tunable?
5. **Geographic distribution** — single region or global?

The answers tell you which DB fits.

---

## Tradeoff
More databases = more operational complexity. You need expertise in each, monitoring for each, backups for each, schemas for each.

**Rule:** Add a new database only when the access pattern justifies it. Don't introduce Cassandra "in case we need it" — wait until Postgres genuinely can't handle the workload.

---

## Common Mistakes

**Mistake 1:** Using one DB for everything.
→ Postgres trying to handle 500M append-only rows → degrades.

**Mistake 2:** Using too many DBs.
→ 10 different databases for a small system → operational nightmare.

**Mistake 3:** Choosing DB by familiarity.
→ "I know Postgres so let's use it for everything" — wrong tool for graph traversals or 1B-row append-only logs.

---

## Rule
Match the database to the **access pattern + scale + consistency need.** Be willing to use 2–4 different databases in any non-trivial system.
