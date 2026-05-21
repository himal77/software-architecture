# System Design: Global Messaging System (WhatsApp-Scale)
**Designed:** Day 8 | **Scale target:** 2B users, 100B messages/day

---

## Requirements

### Functional
- 1-on-1 chats and group chats (up to 1000 members)
- Send and receive messages with low latency
- Show message history paginated by time
- Media attachments (images, videos)

### Non-Functional
| NFR | Value |
|---|---|
| Send latency | <100ms |
| Chat history load | <200ms |
| Scale | 2B users, 100B msg/day |
| Avg message size | 100 bytes |

---

## Scale Math
```
100B/day total messages
Avg/sec:        100B / 86,400 = ~1.16M/sec
Peak (×3):      ~3.5M/sec
Storage/day:    100B × 100 bytes = 10TB/day
Storage/year:   3.6PB/year
With ×3 replication: ~11PB/year
```

**Architecture implications:**
- 3.5M writes/sec → far beyond single-leader (10K/sec ceiling)
- Cassandra: 100K writes/sec/node → 50+ nodes minimum
- 11PB total → no Postgres/MySQL alone survives

---

## Key Design Decisions

### Shard Key: chat_id
- Primary query: `WHERE chat_id = ? ORDER BY timestamp`
- Co-locates all messages for a conversation
- Same model for 1-on-1 and group chats
- Rule: shard by the key you query on (Day 1 lesson)

### Strategy: Hash-Based Partitioning
- Even distribution across cluster
- No directory bottleneck
- All chats treated equally (no tenant isolation needed)

### Composite Key
```
PARTITION KEY: chat_id        → hash for even distribution
CLUSTERING KEY: timestamp     → range scan within chat
```
Best of both worlds: even shards + ordered messages within a chat.

### Database: Cassandra + Redis + S3
- **Cassandra** — message storage (append-only at scale)
- **Redis** — hot chat caches, presence info
- **S3 + CDN** — media attachments

---

## Architecture

```
Client (mobile/web)
   ↓
WebSocket (live message delivery)
   ↓
Message Service (stateless, scaled)
   │
   ├──→ Cassandra cluster (50+ nodes, hash by chat_id)
   ├──→ Redis cache (hot chats)
   └──→ Kafka → push to recipient WebSocket servers

Media path:
   → Upload to S3
   → Serve via CDN
```

---

## Read Flow
```
User opens chat X:
  → Redis: last 50 messages cached?
     YES → return immediately (<10ms)
     NO  → Cassandra: SELECT WHERE chat_id=X ORDER BY timestamp DESC LIMIT 50
         → cache result in Redis (TTL 5 min)
         → return
```

## Write Flow
```
User sends message in chat X:
  → INSERT into Cassandra (chat_id, timestamp, message)
  → Update Redis cache for chat X
  → Publish to Kafka topic
  → Recipient's WebSocket server consumes → pushes to client
  → Recipient sees message in <100ms
```

---

## Hot Group Mitigations

**Problem:** Most chats are 2 people. Some groups have 1000 members and 100K messages/day. One chat_id saturates one shard.

| Fix | Detail |
|---|---|
| Cache | Redis serves recent messages → most reads avoid Cassandra |
| Sub-partition | For huge groups: shard by `(chat_id, time_bucket)` to spread writes |
| Dedicated cluster | Top 0.01% of broadcast groups → separate infra tuned for write rate |

---

## Bottleneck Analysis

| Bottleneck | Trigger | Fix |
|---|---|---|
| Single chat write spike | Group with 100K msg/day | Sub-partition by time |
| Hot chat reads | Viral group | Redis cache + WebSocket fan-out |
| Storage growth | 11PB/year | Cassandra horizontal scaling, archival to S3 |
| Cross-region latency | User in EU messaging US user | Regional WebSocket clusters, async cross-region replication |

---

## Lessons
- **Partition by access pattern, not by user attribute** — chat_id, not country
- **Hash + range composite** = ideal for messaging
- **Cassandra + WebSocket + Redis** = the core stack for messaging at scale
- Same fundamental design used by WhatsApp, Slack, Discord, Telegram, Signal
