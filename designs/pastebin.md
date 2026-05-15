# System Design: Pastebin
**Designed:** Day 2 | **Scale target:** 10M DAU

---

## Requirements

### Functional
- User pastes text (up to 10MB) → gets a short URL
- Anyone with the URL can view the paste
- Pastes expire after 30 days
- Users can delete their paste before expiry
- Anonymous pastes supported

### Non-Functional
| NFR | Value |
|---|---|
| Availability | 99.9% uptime |
| Latency | Retrieve paste < 100ms |
| Durability | RPO = 0 (no data loss after creation) |
| Consistency | Paste visible immediately after POST |
| Scale | 10M DAU, 5:1 read/write ratio |
| Storage | 10MB max per paste, 10KB average |

---

## Scale Estimation
```
Writes: 10M / 86,400 = ~116/sec → peak ~350/sec
Reads:  116 × 5      = ~580/sec → peak ~1,750/sec
Total peak: ~2,100 req/sec → LB + multiple instances + cache tier

Avg paste size: 10KB
Storage/day: 10M × 10KB = 100GB
Active storage (30-day window): 3TB × 1.5 buffer = ~4.5TB
```

---

## Data Model

**Paste** (Postgres — metadata only)
| Field | Type | Notes |
|---|---|---|
| `paste_id` | string | Short code, primary key |
| `user_id` | string | Nullable for anonymous |
| `title` | string | Optional |
| `s3_key` | string | Pointer to content |
| `visibility` | enum | PUBLIC / PRIVATE |
| `status` | enum | PENDING / ACTIVE |
| `created_at` | datetime | |
| `expires_at` | datetime | +30 days |

**Paste Content** (S3)
- Key: `s3_key`
- Value: raw text up to 10MB

**User** (Postgres)
- `user_id`, `email`, `created_at`

---

## API
```
POST   /pastes              → create, returns paste_id
GET    /pastes/{paste_id}   → retrieve
DELETE /pastes/{paste_id}   → delete (owner only)
```

---

## Architecture

**GET /pastes/{paste_id}**
```
Client
  ↓
Load Balancer
  ↓
App Instance
  ↓
Redis ── hit ──→ s3_key → CDN/S3 → content → return
  │
  miss
  ↓
Postgres (metadata + s3_key)
  ↓
Redis (cache metadata)
  ↓
CDN/S3 (fetch content)
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
Return paste_id
```

---

## Key Design Decisions

### Metadata + Blob separation
Postgres stores only metadata (~200 bytes). Content lives in S3 (up to 10MB). Never store large blobs in Postgres.

### PENDING status for cross-system write safety
No distributed transactions across Postgres + S3. Use status flag — only ACTIVE pastes visible. Background job retries PENDING records.

### Cache metadata only, not content
Redis caches paste metadata (~200 bytes). Content fetched directly from S3/CDN. Caching 10MB objects in Redis would exhaust memory.

### CDN in front of S3
Public paste content cached at CDN edge. Latency: ~50ms (S3) → ~5ms (CDN). S3 only hit on first request per region.

---

## Bottlenecks & Solutions

| Bottleneck | Trigger | Solution |
|---|---|---|
| Thundering herd | Viral paste, cold cache | Cache mutex |
| S3 latency | High read traffic | CDN in front of S3 |
| DB write pressure | Traffic spike on creates | Kafka write buffer |
| App saturation | 10× concurrent S3 waits | Auto-scaling + async I/O |
