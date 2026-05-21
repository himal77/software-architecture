# System Design: Live Sports Scoreboard
**Designed:** Day 4 | **Scale target:** 500M users, 10,000 concurrent matches

---

## Requirements

### Functional
- Admin updates match scores in real time
- Users see live scores with <5 second lag
- 10,000 concurrent matches during peak events

### Non-Functional
| NFR | Value |
|---|---|
| CAP choice | AP — availability over consistency |
| Consistency model | Eventual — scores are independent events |
| Lag SLA | <5 seconds globally |
| Scale | 500M concurrent users |
| Incorrect score briefly | Tolerable |
| Incorrect score permanently | Not tolerable — Postgres is source of truth |

---

## Scale Estimation
```
10,000 matches × 1 update/30sec = ~333 score writes/sec (trivial)
500M users reading: push model → ~333 fan-out events/sec (trivial)
500M users polling: 500M / 5sec = 100M req/sec (impossible)
→ Must use push model (WebSockets)
```

---

## Architecture

```
Admin Panel
    ↓
Score Update Service
    ├──→ Postgres (source of truth, durable)
    ├──→ Redis (origin hot cache)
    └──→ Kafka "score-updates"
                ↓
        Fan-out Service
        ┌───────┼───────┐
        ↓       ↓       ↓
    CDN     CDN     CDN
  Americas Europe  Asia
        ↓       ↓       ↓
    WS      WS      WS
  servers servers servers
        ↓       ↓       ↓
             500M users
```

---

## Key Flows

**Write:**
```
Admin → Score Update Service
→ Postgres (durable write)
→ Redis (cache update)
→ Kafka event
→ Fan-out → regional WS servers → push to clients
Total lag: ~2–4 seconds
```

**New user connects:**
```
→ CDN/Redis snapshot (current score, <50ms)
→ Subscribe to WebSocket for live updates
```

---

## Why This Works

| Problem | Solution |
|---|---|
| 500M global readers | CDN edges per region |
| <5 sec lag | Kafka fan-out to regional WS servers |
| No polling at scale | WebSocket push model |
| Durable scores | Postgres source of truth |
| Stale score tolerable | Eventual consistency — no coordination needed |

---

## CAP Analysis
```
Network partition between regions:
AP choice → regional WS servers serve last known score
Users in isolated region see score that's slightly stale
Once partition heals → score updates resume
No permanent data loss — Postgres always holds truth
```
