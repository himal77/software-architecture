# Day 2 — The Architect's Framework
**Chapter 1 | Phase 1: Foundations**
**Date:** 2026-05-12

---

## Key Concepts

### The 5-Step Framework (use for every design problem)
```
Step 1: Clarify requirements       (2 min)
Step 2: Estimate scale             (3 min)
Step 3: Define the data model      (5 min)
Step 4: Design the high-level      (10 min)
Step 5: Deep dive bottlenecks      (remaining time)
```
**Never skip to Step 4.** Boxes are wrong if Steps 1–3 are wrong.

---

### Step 1: Requirements — Two Buckets

**Functional** — what the system does
- User can create a paste
- User can view a paste via link
- Paste expires after 30 days

**Non-functional** — how the system behaves under pressure
- Availability: 99.9% uptime
- Latency: retrieve paste < 100ms
- Durability: no data loss once paste created (RPO = 0)
- Consistency: paste visible immediately after creation
- Scale: 10M DAU, 5:1 read/write ratio

> Never mix solutions into requirements. "Use Redis" is not a requirement. "Retrieve paste in <100ms" is.

---

### Step 2: Back-of-Envelope Estimation

**Key numbers to memorize:**

| Metric | Value |
|---|---|
| Seconds in a day | 86,400 |
| 1M requests/day | ~12 req/sec |
| 1B requests/day | ~12,000 req/sec |
| Peak = average × | 3 |
| SSD read latency | ~0.1ms |
| RAM / Redis latency | ~0.0001ms |
| Same DC network round trip | ~0.5ms |
| Cross-region round trip | ~100–200ms |

**Pastebin estimation:**
```
10M DAU, 5:1 read/write ratio
Writes: 10M / 86,400 = ~116/sec → peak ~350/sec
Reads:  116 × 5 = ~580/sec → peak ~1,750/sec

Avg paste = 10KB (not 10MB max)
Storage/day: 10M × 10KB = 100GB
Active storage (30-day expiry): 100GB × 30 = 3TB × 1.5 buffer = ~4.5TB
```

**Architecture tier thresholds:**

| Req/sec | Architecture needed |
|---|---|
| < 100 | Single server |
| 100–1,000 | Single server + cache |
| 1,000–10,000 | LB + multiple instances + cache |
| 10,000–100,000 | Sharding starts to matter |
| > 100,000 | Full distributed, multi-region |

---

### Step 3: Data Model

**Entity 1: Paste metadata** (Postgres)

| Field | Type | Notes |
|---|---|---|
| `paste_id` | string | Short code, primary key |
| `user_id` | string | Nullable — anonymous allowed |
| `title` | string | Optional |
| `s3_key` | string | Pointer to content in S3 |
| `visibility` | enum | PUBLIC / PRIVATE |
| `status` | enum | PENDING / ACTIVE |
| `created_at` | datetime | |
| `expires_at` | datetime | created_at + 30 days |

- Read by: `paste_id`
- Write: once at creation
- Size: ~200 bytes — tiny, cache in Redis

**Entity 2: Paste content** (S3 / blob storage)

| Field | Notes |
|---|---|
| `s3_key` | Matches paste_id |
| `content` | Raw text, up to 10MB |

- Read by: `s3_key`
- Write: once at creation
- Size: 10KB avg, 10MB max — never store in Postgres

**Entity 3: User** (Postgres)

| Field | Type |
|---|---|
| `user_id` | string |
| `email` | string |
| `created_at` | datetime |

- Read by: `user_id` or `email`
- Size: ~200 bytes

---

### Step 4: High-Level Architecture

**GET /pastes/{paste_id}**
```
Client
  ↓
Load Balancer
  ↓
App Instance
  ↓
Redis ── hit ──→ get s3_key from cache → S3 → return content
  │
  miss
  ↓
Postgres (fetch metadata + s3_key)
  ↓
Redis (populate cache)
  ↓
S3 (fetch content)
  ↓
Return to client
```

**POST /pastes**
```
Client
  ↓
Load Balancer
  ↓
App Instance
  ↓
  1. Postgres (status = PENDING)
  2. S3 (upload content)
  3. Postgres (status = ACTIVE)
  4. Redis (cache metadata)
  ↓
Return paste_id to client
```

---

### Step 5: Bottlenecks

**Bottleneck 1: Thundering Herd**
- Viral paste, not yet cached → 10,000 users hit simultaneously → all miss cache → 10,000 DB queries for same row
- Solution: Cache mutex — first request acquires lock, fetches DB, populates cache. Others wait and read from cache.

**Bottleneck 2: S3 latency at scale**
- Every request fetches content from S3 (~50ms)
- 10× spike = 17,500 S3 fetches/sec → app instances blocked waiting
- Solution: CDN in front of S3 — popular pastes cached at edge (~5ms)

**Bottleneck 3: DB write pressure on spike**
- 10× spike → 3,500 writes/sec to Postgres
- Solution: Kafka write buffer — absorbs spikes, DB processes at steady rate

---

## Concepts Learned

### Metadata + Blob Storage Pattern
Never store large content (files, images, videos, large text) in the DB.
```
Postgres: metadata + pointer (s3_key)   ← tiny, fast, cacheable
S3:       actual content                ← cheap, scalable, direct fetch
```
Used by: GitHub Gist, Notion, Google Docs, Pastebin.

### Why Transactions Don't Span Storage Systems
Postgres transactions work within Postgres only. S3 has no concept of transactions. You cannot wrap both in a single transaction.

**Solution: PENDING status pattern**
```
1. Write metadata to Postgres (status = PENDING)
2. Upload content to S3
3. Update metadata (status = ACTIVE)
4. Only ACTIVE pastes are visible to users
5. Background job retries or cleans up PENDING records
```
This is the **outbox pattern** — covered in depth in Chapter 10.

### CDN (Content Delivery Network)
Network of geographically distributed servers. Caches static/public content at edge nodes close to users.
```
Without CDN: User in India → US server → ~200ms
With CDN:    User in India → Mumbai edge → ~10ms
```
- CDN = geographic cache for static content
- Good for: public pastes, images, JS/CSS, videos
- Bad for: user-specific dynamic data, private content
- AWS: CloudFront in front of S3 = one config, global distribution
- Covered in depth: Chapter 3 (Days 11–15)

### What NOT to Cache in Redis
Don't cache large content objects in Redis.
```
1,000 pastes × 10MB = 10GB Redis memory → exhausted → evicts everything → cache useless
```
Cache only metadata (200 bytes). Fetch content directly from S3/CDN.

---

## GitHub Gist Case Study
**Same problem as Pastebin — real production solution**

- Content stored in Git objects (blob storage) not in DB — same metadata/content split
- Metadata (gist_id, owner, visibility) in MySQL — tiny, fast, cacheable
- Heavy CDN usage — static content at edge, origin never hit on repeat views
- **Syntax highlighting:** expensive CPU operation — render once, cache the HTML output. Never re-render unless content changes.
- **Forks/revisions:** Git content-addressable storage — identical files share the same blob. Massive storage saving.

**Lesson:** Metadata/blob separation is production-proven. Every major system handling user-generated content uses this split.

---

## Gaps from Revision Quiz (Day 1 concepts to reinforce)
- CAP theorem ≠ trade-off triangle — different tools, different stages
- Shard key must be in the query — scatter-gather defeats sharding
- Hash-based code generation: same URL = same code (business failure for multi-campaign)
- RPO = data loss tolerance | RTO = recovery time target

---

## Next
**Day 3 — Chapter 1 final day:** Putting it all together — how to handle unknown systems under time pressure. Architecture Decision Records (ADRs).
