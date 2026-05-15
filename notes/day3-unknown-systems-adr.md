# Day 3 — Handling Unknown Systems + Architecture Decision Records
**Chapter 1 Final Day | Phase 1: Foundations**
**Date:** 2026-05-13

---

## Key Concepts

### The Architect's Opening Move (first 60 seconds)
```
1. Repeat the problem back in your own words     (10 sec)
2. Ask the 4 clarifying questions                (60 sec)
3. State your assumptions explicitly             (30 sec)
4. Begin Step 2 — estimation                     (start working)
```
Never skip step 3. Implicit assumptions become bugs.

---

### The 3 System Archetypes

**Archetype 1: Read-Heavy Storage System**
Examples: URL shortener, Pastebin, Google Drive, CDN
```
Signature: reads >> writes, data stored and retrieved by key
Hard problems: cache strategy, storage cost, read latency
Default: App → Cache → DB → Blob storage
```

**Archetype 2: Write-Heavy Streaming System**
Examples: Twitter feed, logging, analytics, IoT, Kafka consumers
```
Signature: writes >> reads OR writes at massive volume
Hard problems: write throughput, durability, ordering, consumer lag
Default: App → Message queue → Stream processor → DB
```

**Archetype 3: Compute-Heavy Processing System**
Examples: YouTube encoding, search indexing, Uber matching, fraud detection
```
Signature: complex computation on data, not just storage/retrieval
Hard problems: job scheduling, worker scaling, partial failure, idempotency
Default: App → Job queue → Worker pool → Result store
```

Most real systems are combinations. Identify the dominant archetype first.

---

### Reasoning From First Principles
When designing a system you've never built:
1. What does it need to do?
2. What's the access pattern?
3. What's the latency requirement?
4. What breaks first?

Then derive the architecture from answers — don't guess from familiarity.

---

### Architecture Decision Records (ADRs)

**When to write one:**
- You chose technology A over B and someone will ask why
- You made a tradeoff that isn't obvious from the code
- You rejected a "better" solution for a valid but non-obvious reason
- A future engineer might be tempted to "fix" your design

**When NOT to write one:**
- Obvious decisions
- Standard patterns
- Things fully explained by the code

**ADR Format:**
```markdown
# ADR-00X: Title

## Status
Accepted / Rejected / Deprecated

## Context
Why was this decision needed? What constraints existed?

## Decision
What exactly was decided — specific, not vague.

## Alternatives Considered
What else was evaluated and why it was rejected.

## Consequences
Honest pros AND cons.
```

Most important element: **Alternatives Considered** — proves you evaluated options and made a reasoned choice.

---

## Notification System Design

### Scale
```
50M users, 10M notifications/day
Average: ~116 notifications/sec
Peak (normal): ~350/sec
Peak (flash sale): 50M / 60sec = ~833,000 push/sec  ← real challenge
```

### Data Model

**Notification** (Cassandra — partition by user_id)
| Field | Type | Notes |
|---|---|---|
| `notification_id` | UUID | Primary key |
| `user_id` | UUID | Partition key |
| `type` | enum | ORDER_CONFIRMED, SHIPPING_UPDATE, FLASH_SALE |
| `channel` | enum | EMAIL, SMS, PUSH |
| `status` | enum | PENDING, SENT, DELIVERED, FAILED, EXPIRED |
| `title` | string | |
| `body` | string | |
| `metadata` | JSON | Type-specific (order_id, tracking_number, product_id) |
| `created_at` | timestamp | Sort key |
| `expires_at` | timestamp | For time-sensitive notifications |
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
- Without this, adding new notification type requires code deploy

### Architecture
```
Event Sources (Order Service, Flash Sale Admin)
  ↓
Kafka topics: "order-placed", "flash-sale-start"
  ↓
Notification Service (stateless, horizontally scaled)
  ↓
Preference Check (Redis cache → Postgres fallback)
  ↓
Route to channel-specific Kafka topics:
  "email"        "sms"        "push"
     ↓              ↓            ↓
Email Worker   SMS Worker   Push Worker Pool
     ↓              ↓         (auto-scales)
SendGrid/SES   Twilio        FCM/APNS
     ↓              ↓            ↓
         Cassandra (notification history + status)
```

### Flash Sale — Pre-computed Segments
```
Problem: SELECT user_id WHERE push=true on 50M users = minutes
Fix: Nightly offline job pre-computes segments → Redis SET "segment:push-enabled"
Flash sale: read segment from Redis (instant) → push all IDs to Kafka
```

### Message TTL Check (Worker Logic)
```
consume message from Kafka
  if now > expires_at → write status=EXPIRED to Cassandra, discard
  else → send via provider
           success → status=DELIVERED
           failure → retry 3× with exponential backoff
                  → status=FAILED, alert
```

Key: include `expires_at` in the event message — producer sets the rule, worker enforces it.

---

## Key Design Decisions

### One Kafka Topic Per Channel (not one "notifications" topic)
- Email workers lag without blocking SMS workers
- Each channel scales independently
- Adding 4th channel doesn't affect existing topics

### Centralised Preference Check Before Routing
- Single preference lookup per notification regardless of channel count
- Channel topics contain only pre-filtered valid messages — no wasted worker processing
- 3× more efficient than checking inside each worker

### Priority Lanes in Kafka
- Order confirmations (time-critical) and marketing emails (not critical) need separate topics
- Mixing them means marketing burst delays order confirmations
- Rule: separate Kafka topics by **priority**, not just by channel

---

## Bottlenecks

| Bottleneck | What breaks | Fix |
|---|---|---|
| External provider rate limits | FCM/Twilio throttles at 833K req/sec | Token bucket rate limiter per provider |
| Consumer lag during flash sale | Messages expire before processing | Lag monitoring + auto-scale workers |
| Preference lookup at scale | Redis overwhelmed during flash sale | Pre-computed segments (offline job) |
| Silent message expiry | No visibility into missed notifications | Write EXPIRED status to Cassandra |

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
- If user updates preferences after routing, old preference applies to that notification

---

## Uber Eats Case Study

**What they got right:**
- Separate Kafka topics per notification type — independent scaling and failure isolation
- Pre-computed user segments for marketing — same pattern as flash sales above
- Provider abstraction layer — swap SendGrid for SES without changing workers

**Problems they had to solve:**
- **Notification deduplication:** Network failures caused retries → user received same SMS 3 times. Fix: idempotency key per notification — provider checks if key already sent.
- **Priority lanes:** Order confirmations and marketing emails in same queue → marketing burst delayed order confirmations. Fix: separate high-priority and low-priority Kafka topics with dedicated workers.

**Lesson:** Separate Kafka topics by priority, not just by channel.

---

## Concepts Internalized

| Concept | Status |
|---|---|
| 3 system archetypes | ✅ |
| Architect's opening move | ✅ |
| Reasoning from first principles | ✅ |
| Architecture Decision Records | ✅ |
| Message TTL check for expiry | ✅ |
| External provider rate limiting | ✅ |
| Consumer lag as operational bottleneck | ✅ |
| Pre-computed segments for bulk sends | ✅ |
| Priority lanes in Kafka | ✅ |
| Notification deduplication (idempotency key) | ✅ |

---

## Next
**Day 4 — Chapter 2: Distributed Systems Theory** — CAP theorem in depth, consistency models, ACID vs BASE.
