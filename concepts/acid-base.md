# ACID vs BASE
**Introduced:** Day 5

---

## ACID
Guarantees for relational database transactions.

| Property | Guarantee |
|---|---|
| **Atomicity** | All or nothing — full commit or full rollback |
| **Consistency** | Transaction brings DB from one valid state to another |
| **Isolation** | Concurrent transactions execute as if sequential |
| **Durability** | Committed data survives crashes — written to disk |

**Isolation levels (weakest → strongest):**

| Level | Prevents | Use when |
|---|---|---|
| Read Uncommitted | Nothing | Never |
| Read Committed | Dirty reads | Default (most apps) |
| Repeatable Read | Dirty + non-repeatable | Reports |
| Serializable | All anomalies | Financial transactions |

**ACID fails at scale because:** lock contention under high concurrency + inability to scale horizontally across regions. Not a consistency problem — a throughput problem.

---

## BASE
Properties of AP distributed systems (Cassandra, DynamoDB).

| Property | Meaning | ACID contrast |
|---|---|---|
| **Basically Available** | Always responds, even with stale data | vs ACID Consistency — refuses if can't guarantee correctness |
| **Soft state** | Replicas converging, state changes without new input | vs ACID Isolation — freezes state during transactions |
| **Eventually consistent** | Converges to consistent state given no new updates | vs ACID Consistency — always consistent, immediately |

---

## BASE Conflict Patterns

**Last Write Wins (LWW):** latest timestamp wins. Simple, dangerous (clock drift).
**Read Repair:** return latest, repair stale replicas in background.
**Quorum:** W + R > N guarantees reading at least one fresh replica.

---

## When to Use Each

| Use ACID | Use BASE |
|---|---|
| Money, inventory, healthcare | Social features, preferences, carts |
| Correctness critical | Availability critical |
| Small-medium scale | Massive global scale |
| Postgres, MySQL | Cassandra, DynamoDB |
| Need global ACID | → NewSQL: Spanner, CockroachDB |
