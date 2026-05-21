# Partitioning / Sharding Strategies
**Introduced:** Day 8

---

## When You Need to Partition
- Data exceeds single-machine storage (>1TB)
- Write throughput exceeds single-leader limit (>10K writes/sec)
- Read replicas alone can't serve query load

---

## The 3 Strategies

### Range-Based
```
Shard 1: A–F  |  Shard 2: G–M  |  Shard 3: N–Z
```
- ✅ Range queries cheap (only 1-2 shards)
- ❌ Hot spots from uneven data (celebrities, today's date)
- Used by: HBase, BigTable, MongoDB sharded

### Hash-Based
```
shard = hash(key) % num_shards
```
- ✅ Even distribution by design
- ❌ Range queries scatter across all shards
- ❌ Naive `% N` requires moving 80% of data when adding a node
- Used by: Cassandra, DynamoDB, most modern distributed DBs
- **Production fix: consistent hashing**

### Directory-Based
```
Lookup table: key → shard
```
- ✅ Maximum flexibility (move tenants, isolate whales, data residency)
- ❌ Directory is bottleneck + SPOF
- Used by: Vitess (YouTube/Slack), MongoDB config servers

---

## Consistent Hashing
Used by Cassandra/DynamoDB. Adding a node moves only ~1/N of keys.

**Mental model:** circle (0 to 2^32-1). Hash nodes onto positions. Hash keys onto positions. Owner = first node walking clockwise from key.

**Virtual nodes:** Each physical node owns 256 virtual positions → even distribution + smooth rebalancing on failure.

---

## Composite Keys (Cassandra Pattern)
Combines hash + range:
```
PARTITION KEY (chat_id) → hash for even distribution
CLUSTERING KEY (timestamp) → range for ordered scans within partition
```

Best of both worlds within a single chat/partition.

---

## Shard Key Rules

1. **Shard by the key you query on** (not by logical ownership)
2. **Match access pattern to strategy:**
   - Range queries → range-based or composite
   - Even distribution → hash-based
   - Tenant isolation → directory-based
3. **Avoid sequential keys** as shard keys (creates hot spot on highest shard)

---

## Hot Spot Patterns

| Pattern | Cause |
|---|---|
| Celebrity | One key disproportionately popular |
| Time-based | Today's shard takes all writes |
| Tenant failure | One customer crushes their shard |
| Sequential ID | Auto-increment all on highest shard |

---

## Hot Spot Fixes

| Fix | Approach |
|---|---|
| Add randomness | Random prefix on key spreads writes |
| Two-level partitioning | Pre-fan-out (Twitter celebrity pattern) |
| Cache hot keys | CDN/Redis absorb reads |
| Dedicated shard for whales | Largest tenants get own shard |
