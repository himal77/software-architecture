# Day 8 — Partitioning & Sharding
**Chapter 2 | Phase 1: Foundations**
**Date:** 2026-05-24

---

## Key Concepts

### Partitioning vs Replication
```
Replication  → multiple copies of the SAME data (availability, latency, throughput)
Partitioning → SPLIT data across machines (each holds different data)

Most large systems do BOTH:
  Partition into shards → replicate each shard
```

**When to partition:**
- Data exceeds single-machine storage (>1TB typically)
- Write throughput exceeds single-leader limit (>10K writes/sec)
- Read replicas alone can't serve query load

---

### The 3 Partitioning Strategies

```
1. Range-based   → contiguous ranges per shard
2. Hash-based    → hash(key) determines shard
3. Directory-based → lookup table maps key → shard
```

---

## Strategy 1: Range-Based

```
Shard 1: keys A–F
Shard 2: keys G–M
Shard 3: keys N–S
Shard 4: keys T–Z
```

**Pros:** Range queries cheap (only hits 1-2 shards), sequential reads efficient.
**Cons:** Hot spots — uneven data distribution (celebrities, today's date).

**Used by:** HBase, BigTable, MongoDB sharded.

**Hot spot example — Twitter sharded alphabetically:**
- @AdamSandler on Shard 1 (A-F) gets 100M reads
- @JoeAverage on Shard 5 idle
- One shard crushed, others underused

---

## Strategy 2: Hash-Based

```
shard_id = hash(key) % num_shards

"alice"  → hash → 47291 → shard 3
"bob"    → hash → 92481 → shard 1
```

**Pros:** Even distribution by design, no celebrity problem.
**Cons:** Range queries scatter across all shards.

**Used by:** Cassandra, DynamoDB, most modern distributed DBs.

### The Resharding Problem
```
shard = hash(key) % 4 → was the rule
shard = hash(key) % 5 → new rule (added a node)
→ Almost every key maps to different shard
→ Must move 80% of data to rebalance
→ Massive operation
```

**Fix: Consistent Hashing.**

---

### Consistent Hashing

**Mental model: a circle with values 0 to 2^32 - 1.**

```
Place each NODE on circle by hashing name:
  Node A → 100
  Node B → 1000
  Node C → 5000

Place each KEY on circle by hashing it:
  "alice" → 250

Owner = first node walking clockwise from key:
  "alice" at 250 → walks → hits Node B at 1000
  Node B owns "alice"
```

**Adding a node moves only ~1/N of keys** (not 80% like simple modulo):
```
Add Node D at 500
Before: keys 100-1000 → Node B
After: keys 100-500 → Node D
       keys 500-1000 → Node B
Only keys 100-500 move. Other nodes untouched.
```

### Virtual Nodes (vnodes)
```
Each physical node owns 256 virtual positions on circle
→ Even distribution
→ When node fails, 256 ranges spread across many neighbors
   (not all dumped on one)
```

Cassandra uses 256 vnodes per physical node by default.

---

## Strategy 3: Directory-Based

```
Lookup table maps keys → shards:

| Key prefix    | Shard |
|---------------|-------|
| user 1-1M     | Shard 1 |
| user 1M-5M    | Shard 2 |
| EU users      | Shard 3 |
| ENTERPRISE    | Shard 5 |
```

**Pros:** Maximum flexibility — move customers between shards, isolate noisy tenants, data residency.
**Cons:** Directory is a bottleneck and SPOF — must be replicated via consensus (etcd, ZooKeeper).

**Used by:** Vitess (YouTube/Slack), MongoDB config servers, HDFS NameNode.

---

## Hot Spots — 4 Patterns

| Pattern | Cause |
|---|---|
| Celebrity | One key gets disproportionate traffic |
| Time-based | Today's shard takes all writes, yesterday's idle |
| Tenant isolation failure | One enterprise customer crushes their shard |
| Sequential ID | Auto-incrementing IDs always on highest shard |

---

## Hot Spot Fixes

| Fix | How | Tradeoff |
|---|---|---|
| Add randomness | `random_prefix + key` → spreads writes | Reads must query multiple prefixes |
| Two-level partitioning | Pre-fan-out (Twitter pattern) — write to followers' shards | More writes per event |
| Cache hot keys | Heavy CDN/Redis caching | Doesn't help writes |
| Dedicated whale shard | Move biggest tenants to own shard | Requires directory-based |

---

## Composite Keys (Cassandra Pattern)

Combines hash + range — best of both worlds within a partition:

```
Cassandra: PARTITION KEY (chat_id) + CLUSTERING KEY (timestamp)

→ Different chats hash to different shards (even distribution)
→ Within a chat, messages ordered by timestamp (range queries cheap)
→ Query: SELECT WHERE chat_id=X ORDER BY timestamp LIMIT 50
   → Hits single shard, sequential read
```

---

## Messaging System Design (WhatsApp-Scale)

### Scale
```
2B users, 100B messages/day, 100 bytes avg
Messages/sec avg:  100B / 86,400 = ~1.16M/sec
Peak (×3):         ~3.5M messages/sec
Storage/day:       100B × 100B = 10TB/day
Storage/year:      ~3.6PB/year
With replication ×3: ~11PB/year
```

**Architecture implication:**
- Way beyond single-leader (10K writes/sec ceiling)
- Cassandra: 100K writes/sec/node → 50+ nodes minimum
- 3.6PB/year → no single Postgres survives → Cassandra mandatory

### Decisions

**Shard key: chat_id (NOT location)**
- Primary access pattern: "messages in chat X by time"
- Query is `WHERE chat_id = ?` → shard by chat_id
- Co-locates all messages for a conversation
- Works identically for 1-on-1 and group chats

**Partitioning: hash-based, NOT directory-based**
- Every chat treated equally → no need for directory flexibility
- Hash gives even distribution naturally
- Directory adds bottleneck without benefit here

**Composite key: (chat_id, timestamp)**
- Hash by chat_id → even shard distribution
- Range by timestamp → ordered messages within chat
- Pagination cheap (LIMIT 50, sequential scan)

**Hot group fixes:**
1. Cache recent messages in Redis (most reads served here)
2. For huge groups: sub-partition by `(chat_id, time_bucket)` to spread writes
3. Dedicated cluster for largest 0.01% of broadcast groups

**Database: Cassandra + Redis + S3**
- Cassandra: messages (append-only at scale)
- Redis: hot chat caches, presence info
- S3 + CDN: media attachments

### Architecture
```
Client
   ↓
WebSocket (live message delivery)
   ↓
Message Service (stateless)
   ├──→ Cassandra cluster (hash by chat_id)
   ├──→ Redis cache (hot chats)
   └──→ Kafka → push notifications
                (recipient WebSockets)
Media: S3 + CDN
```

### Read Flow
```
Open chat X:
  → Redis (last 50 messages)
  → Cache miss → Cassandra: WHERE chat_id=X ORDER BY timestamp LIMIT 50
  → Cache result
```

### Write Flow
```
Send message in chat X:
  → INSERT into Cassandra (chat_id, timestamp, message)
  → Update Redis cache
  → Kafka → push to recipient WebSockets
  → Recipient sees in <100ms
```

---

## WhatsApp Case Study

**Scale:** ~2B users, ~100B messages/day. Built on Erlang originally.

**Architecture choices:**
- Sharded by chat_id from day one
- Erlang BEAM VM: millions of WebSocket connections per server
- Acquired by Facebook, ran 900M users on **just 50 engineers**
- Messages as append-only logs (SSD-optimized)
- Originally Mnesia (Erlang DB), later MySQL clusters for some data
- Media deduplication: 1000 users send same meme → stored once
- Phone number as user identity → simpler routing

**Lesson:** Right partitioning + right concurrency model + discipline beats raw infrastructure. WhatsApp ran 900M users with 50 engineers.

**Universal pattern:** Slack, Discord, Telegram, Signal all shard by chat_id. Differences are in delivery guarantees and metadata.

---

## Concepts Internalized

| Concept | Status |
|---|---|
| 3 partitioning strategies | ✅ |
| Hot spot patterns + fixes | ✅ |
| Consistent hashing + vnodes | ✅ |
| Resharding problem | ✅ |
| Composite keys (hash + range together) | ✅ |
| Shard messaging by chat_id | ✅ |
| When directory-based fits (tenant isolation, not equal data) | ✅ |

---

## Next
**Day 9 — Chapter 2 continued:** Distributed transactions — 2PC, the saga pattern, why distributed ACID is hard.
