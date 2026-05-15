# Critical vs Non-Critical Path Separation
**Introduced:** Day 1

---

## The Principle
**Separate your critical path from your non-critical path. Scale each independently.**

---

## Definition

| | Critical Path | Non-Critical Path |
|---|---|---|
| Definition | Operations the user is waiting on | Operations that can happen after the response |
| Latency requirement | Must be fast (<10–100ms) | Can lag seconds, minutes |
| Failure impact | User-facing error | Background degradation |
| Examples | Redirect, checkout, login | Analytics, notifications, audit logs |

---

## URL Shortener Example

**Critical path:** `short_code → original_url → redirect`
- Must complete in <10ms
- Served from Redis cache
- Never blocked by analytics writes

**Non-critical path:** Click count analytics
- User doesn't wait for this
- Written to Redis async, flushed to DB every 5 min
- Can use Kafka for durability at scale

---

## How to Identify Your Critical Path
Ask: *"Is the user waiting for this?"*
- Yes → critical path. Make it fast, keep it lean, protect it from non-critical load.
- No → non-critical path. Defer it, batch it, async it.

---

## At Scale: Kafka for Non-Critical Path
```
Critical:     Click → Redis lookup → redirect (sync, <10ms)
Non-critical: Click → Kafka event → stream processor → DB (async, minutes ok)
```
Kafka holds events if DB is down. Non-critical path never blocks critical path.
