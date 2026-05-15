# System Design: Notification System
**Designed:** Day 3 | **Scale target:** 50M users, 10M notifications/day

---

## Requirements

### Functional
- Send order confirmation via email
- Send shipping updates via email + SMS
- Send flash sale alerts via push notification
- User preference management (opt in/out per channel)
- Notification history viewable by user
- Do-not-disturb / quiet hours support
- Retry failed sends up to 3×

### Non-Functional
| NFR | Value |
|---|---|
| Throughput | 350 notifications/sec peak (normal), 833,000/sec (flash sale) |
| Latency | Order confirmation delivered <30 sec |
| Availability | 99.9% |
| Durability | RPO = 0 for notification events |
| Deliverability | Retry failed sends up to 3× |
| Scalability | Handle 10× spike during flash sales |

---

## Scale Estimation
```
50M users, 10M notifications/day
Average: 10M / 86,400 = ~116/sec
Peak (normal): 116 × 3 = ~350/sec

Flash sale spike:
50M users × push in 60 sec = 833,000 push/sec ← dominant constraint

Storage (1 year retention):
50M users × 200 notifications/year × 500 bytes = 5TB/year
With replication (×3) = ~15TB
→ 10B rows/year → Cassandra required
```

---

## Data Model

**Notification** (Cassandra — partition by user_id)
| Field | Type | Notes |
|---|---|---|
| `notification_id` | UUID | |
| `user_id` | UUID | Partition key |
| `type` | enum | ORDER_CONFIRMED, SHIPPING_UPDATE, FLASH_SALE |
| `channel` | enum | EMAIL, SMS, PUSH |
| `status` | enum | PENDING, SENT, DELIVERED, FAILED, EXPIRED |
| `title` | string | |
| `body` | string | |
| `metadata` | JSON | Type-specific data |
| `created_at` | timestamp | Sort key |
| `expires_at` | timestamp | For time-sensitive messages |
| `sent_at` | timestamp | Nullable |

**User Preferences** (Postgres + Redis cache)
| Field | Type |
|---|---|
| `user_id` | UUID |
| `email` | boolean |
| `sms` | boolean |
| `push` | boolean |
| `email_address` | string |
| `phone_number` | string (nullable) |
| `device_token` | string (nullable) |
| `quiet_hours_start` | time |
| `quiet_hours_end` | time |

**Template** (Postgres)
- template_id, type, channel, subject_template, body_template

---

## Architecture

```
Event Sources
  Order Service          Flash Sale Admin
       ↓                       ↓
Kafka: "order-placed"   Kafka: "flash-sale-start"
       ↓                       ↓
       └───────────────────────┘
                  ↓
        Notification Service
        (stateless, scaled)
                  ↓
        Preference Check
        (Redis → Postgres fallback)
                  ↓
    ┌─────────────┼─────────────┐
    ↓             ↓             ↓
Kafka:email   Kafka:sms    Kafka:push
    ↓             ↓        (priority: critical / standard / marketing)
Email Worker  SMS Worker   Push Worker Pool (auto-scales)
    ↓             ↓             ↓
SendGrid/SES  Twilio        FCM/APNS
    │             │             │
    └─────────────┴─────────────┘
                  ↓
              Cassandra
        (notification history + status)
```

---

## Flash Sale Flow
```
Nightly offline job:
  SELECT user_id WHERE push=true → Redis SET "segment:push-enabled"

Flash sale starts:
  Read segment from Redis (instant, ~40M user_ids)
  → Push all IDs to Kafka "notifications-marketing" topic
  → Push Worker Pool auto-scales
  → Rate limiter caps FCM calls at provider limit
  → expires_at check discards stale messages
```

## Worker Logic
```
consume message
  if now > expires_at → Cassandra status=EXPIRED, discard
  else → send via provider
           success → Cassandra status=DELIVERED
           failure → retry 3× exponential backoff
                  → Cassandra status=FAILED, alert
```

---

## Key Design Decisions

### Centralised preference check (see ADR-001 below)
### One Kafka topic per channel — independent scaling
### Priority lanes — critical vs standard vs marketing topics
### Pre-computed segments — avoid full table scan at flash sale start
### Message TTL via expires_at field — producer sets rule, worker enforces

---

## ADR-001: Centralised Preference Check Before Channel Routing

**Status:** Accepted

**Context:**
Notification system routes events to three channel workers: email, SMS, push. User preferences determine which channels each user receives. Two approaches considered: check centrally before routing, or check inside each worker after consuming.

**Decision:**
Check preferences centrally in Notification Service before publishing to channel-specific Kafka topics. Only publish to a channel if user has opted in.

**Alternatives Considered:**
Check inside each worker — rejected. Single event consumed by all 3 workers means 3× preference reads per notification. At 833,000 req/sec during flash sales this triples Redis/Postgres load. Users who opted out still have events sitting in wrong channel topics consuming worker capacity before discard.

**Consequences:**
+ Single preference lookup per notification regardless of channel count
+ Channel topics contain only valid pre-filtered messages
+ Adding 4th channel requires no changes to existing workers
- Notification Service must stay in sync with preference updates
- If user updates preferences after routing, old preference applies to in-flight notification

---

## Bottlenecks & Solutions

| Bottleneck | What breaks | Fix |
|---|---|---|
| External provider rate limits | FCM/Twilio throttles at flash sale scale | Token bucket rate limiter per provider |
| Consumer lag during flash sale | Messages expire before processing | Lag monitoring + auto-scale workers |
| Preference lookup at scale | Redis overwhelmed | Pre-computed segments (offline job) |
| Silent message expiry | No visibility into missed notifications | Write EXPIRED status to Cassandra |
| Notification deduplication | Retries cause duplicate sends | Idempotency key per notification at provider |
