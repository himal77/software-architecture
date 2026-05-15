# Message TTL Check (Expiry Validation)
**Introduced:** Day 3

---

## The Problem
Time-sensitive messages (flash sale notifications, OTPs, alerts) can sit in a queue longer than they are valid. Sending an expired notification is worse than sending none — it confuses users.

---

## The Pattern
Include `expires_at` in every time-sensitive event message.
Worker checks before sending:

```
consume message from Kafka
  if now > expires_at:
    → write status=EXPIRED to DB
    → discard, do not send
  else:
    → send notification
```

**Key rule:** Producer sets the expiry rule. Worker enforces it. Never hardcode durations in workers.

---

## Wrong Approach
```
if (now - created_at) > 10 minutes → discard
```
Hardcoded duration breaks when sale duration changes. Logic belongs in the event, not the worker.

---

## Right Approach
```json
{
  "user_id": "abc123",
  "type": "FLASH_SALE",
  "created_at": "2026-05-13T10:00:00Z",
  "expires_at": "2026-05-13T10:30:00Z",
  "product_id": "xyz789"
}
```
Worker checks `now > expires_at` — flexible, producer-controlled.

---

## Always Track Expired Messages
Don't silently discard — write `status=EXPIRED` to DB.
Business can see: "2M users missed flash sale notification due to queue lag."
Ops can see: consumer lag is causing expiry spikes → scale up workers.

---

## Related Concepts
- Consumer lag monitoring (operational bottleneck)
- Token bucket rate limiting (provider throttling)
- Idempotency keys (deduplication on retry)
