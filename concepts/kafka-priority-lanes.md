# Kafka Priority Lanes
**Introduced:** Day 3

---

## The Problem
Mixing high-priority and low-priority messages in the same Kafka topic means low-priority bursts delay high-priority messages.

```
Same topic: "notifications"
Flash sale marketing email burst → 10M messages
Order confirmation (time-critical) → queued behind 10M marketing emails
User places order → confirmation arrives 10 minutes late
```

---

## The Solution: Separate Topics by Priority

```
"notifications-critical"   → order confirmations, payment receipts, OTPs
                             dedicated workers, never starved
"notifications-standard"   → shipping updates, reminders
"notifications-marketing"  → flash sales, promotions, newsletters
                             workers can lag without affecting critical
```

---

## Rule
Separate Kafka topics by **priority**, not just by channel (email/SMS/push).

A flash sale push notification and an order confirmation push notification
both go to the push worker — but they should come from different topics
with different SLAs.

---

## Real-World Example: Uber Eats
Mixed order confirmations and marketing emails in same queue.
Marketing burst delayed order confirmations.
Fix: separate high-priority and low-priority topics with dedicated workers.

---

## When to Apply
Any time you have:
- Messages with different latency SLAs in the same system
- Burst traffic from one message type that could starve another
- Different retry/expiry policies for different message types
