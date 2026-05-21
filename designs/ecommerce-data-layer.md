# System Design: Global E-commerce Data Layer
**Designed:** Day 7 | **Scale target:** 100M users worldwide

---

## Requirements

### Functional
- User profiles
- Product catalog (10M products)
- Shopping carts (cross-device)
- Order history (append-only, 50M/year)
- Payments (100K/day at peak)

### Non-Functional
| NFR | Value |
|---|---|
| Geographic distribution | US, EU, Asia, South America |
| Payment durability | RPO = 0 — never lose a transaction |
| Cart cross-device | Phone + laptop simultaneously |
| Catalog read latency | <50ms globally |
| Order history scale | 500M rows in 10 years |

---

## Database Choices (Polyglot)

### Payments — NewSQL (Spanner / CockroachDB)
```
Why:
- Money cannot be lost (RPO=0) → synchronous replication
- Global users → low local write latency
- Need ACID transactions (no double-charging)
- Linearizable consistency required

Single-leader Postgres = wrong (single-region only)
Cassandra = wrong (eventual consistency)
NewSQL = right (global ACID)
```

### Product Catalog — Postgres + CDN
```
Why:
- Read-heavy, RARE writes (admins update prices)
- Can be cached aggressively
- Need consistency when admin updates (don't show old prices)

Architecture:
  Admin → Postgres leader
        → Read replicas (regional)
        → CDN cache
  User → CDN edge (99% hit rate)
       → Read replica on miss
```

### Shopping Carts — DynamoDB / Cassandra (Leaderless)
```
Why:
- Cross-device editing → concurrent writes
- Vector clocks detect concurrent updates → MERGE both
- Massive write volume (frequent add/remove)
- Brief inconsistency tolerable
- Original Amazon Dynamo paper use case

Conflict scenario:
  User adds item on phone (offline)
  User adds different item on laptop
  Both come online → vector clocks detect concurrent
  → Merge both items into cart
  → Better than picking one and losing the other
```

### User Profiles — Postgres + Read Replicas
```
Why:
- Read-heavy, occasional writes (address updates)
- Standard CRUD pattern
- Read-after-write: route writer's reads to leader for ~1 min
```

### Order History — Cassandra (Leaderless)
```
Why:
- Append-only, query by user_id + timestamp
- 50M/year × 10 years = 500M rows → Cassandra handles natively
- No joins needed
- Eventual consistency fine (orders shown after payment confirmed anyway)

Postgres at 500M+ rows = degrades without manual sharding
Cassandra = native horizontal scaling
```

---

## Architecture Overview

```
                        Client
                          ↓
              ┌───────────┴───────────┐
              ↓                       ↓
         Read paths              Write paths
              ↓                       ↓
     ┌────────┴────────┐    ┌────────┴────────┐
     ↓                 ↓    ↓                 ↓
   CDN          App layer   App layer    App layer
                     ↓           ↓             ↓
              ┌──────┴──────┬───┴────┬─────────┴────┐
              ↓             ↓        ↓              ↓
        Postgres        Spanner   DynamoDB     Cassandra
       (catalog,     (payments)  (carts)      (orders)
        profiles)
```

---

## Polyglot Justification

| Component | Why this DB |
|---|---|
| Payments | Global ACID = NewSQL only category that fits |
| Catalog | Read-heavy + cacheable = single-leader + CDN |
| Carts | Concurrent multi-device writes = leaderless + vector clocks |
| Profiles | Standard read-heavy app data = Postgres |
| Orders | Append-only at scale = Cassandra |

---

## Key Lessons
- **Polyglot persistence is the norm** at scale, not a rarity
- **Leaderless ≠ read-heavy.** Leaderless is for write throughput, not read scale
- **Read-heavy + cacheable** is solved by single-leader + CDN, not leaderless
- **Global + ACID** = NewSQL category (Spanner/CockroachDB)
- **Append-only at scale** = Cassandra, not Postgres
