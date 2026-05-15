# Case Study: Uber Eats Notification System
**Covered:** Day 3 | **Topic:** Notifications at scale

---

## The Problem
Uber Eats sends order confirmations, driver-found, pickup, and delivery notifications.
At peak: millions of rides simultaneously, each generating 4–5 notifications.

---

## What They Got Right

### Separate Kafka topics per notification type
- Independent scaling and failure isolation
- Order confirmation topic can scale without affecting marketing topic

### Pre-computed user segments
- Same pattern as flash sale design — nightly job computes segments
- Avoids full table scan at send time

### Provider abstraction layer
- One internal interface for all providers
- Swap SendGrid for SES without changing any worker code

---

## Problems They Had to Solve

### Notification Deduplication
Network failures caused retries. Users received same "Your driver has arrived" SMS 3 times.

**Fix: Idempotency key per notification**
```
notification_id sent as idempotency key to provider
Provider checks: was this key already sent?
  → Yes: acknowledge, don't send again
  → No: send and record the key
```

### Priority Lanes
Order confirmations (time-critical) and marketing emails (not critical) were in same queue.
Marketing burst → delayed order confirmations → bad user experience.

**Fix: Separate topics by priority**
```
"notifications-critical"  → order confirmations, OTPs
"notifications-marketing" → flash sales, promotions
Dedicated workers per priority — critical never starved by marketing
```

---

## The Lessons
1. Separate Kafka topics by **priority**, not just by channel
2. Idempotency keys are non-negotiable when retries are involved
3. Provider abstraction = swap vendors without code changes
