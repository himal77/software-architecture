# Case Study: Stripe Idempotency
**Covered:** Day 9 | **Topic:** Idempotency at fintech scale

---

## The Pattern
Every Stripe API call accepts an `Idempotency-Key` header.

```
POST /charges
Idempotency-Key: trade-abc123-charge
Authorization: Bearer sk_live_...
{
  "amount": 6000000,
  "currency": "eur",
  "source": "tok_visa"
}
```

---

## What Stripe Does

```
1. Receive request with Idempotency-Key
2. Check internal store: have I seen this key before?
   → YES: return original response (don't charge again)
   → NO: process charge, store key + response
3. Records key for 24 hours
4. After 24h, key expires → can be reused
```

---

## Why It Matters

Without idempotency, every retry risks double-charging:

```
Client → POST /charges (no idempotency key)
Network blip → client never sees response
Client retries → POST /charges
Stripe processes BOTH → customer charged twice
```

With idempotency:

```
Client → POST /charges with Idempotency-Key: xyz
Network blip → client never sees response
Client retries → POST /charges with same Idempotency-Key: xyz
Stripe sees key already processed → returns original response
Customer charged exactly once
```

---

## What Stripe's Pattern Tells Us

This is the gold standard. Adopt the same pattern in your fintech APIs:

1. **Accept `Idempotency-Key` header** on all mutation endpoints
2. **Store keys for 24 hours** (Redis with TTL is perfect)
3. **Return original response** on duplicate keys
4. **Document it loudly** so clients know to use it

---

## Implementation Sketch

```java
@PostMapping("/trades")
public TradeResult createTrade(
    @RequestHeader("Idempotency-Key") String idempotencyKey,
    @RequestBody TradeRequest req) {
  
  // Check Redis for existing key
  TradeResult cached = redis.get("idempotency:" + idempotencyKey);
  if (cached != null) return cached;
  
  // Execute trade
  TradeResult result = tradeOrchestrator.execute(req);
  
  // Cache result for 24h
  redis.set("idempotency:" + idempotencyKey, result, 24, HOURS);
  
  return result;
}
```

Production version handles race conditions (two simultaneous requests with same key) by acquiring a distributed lock briefly.

---

## The Lesson

**Every external API call from your services must include an idempotency key.**

If the API doesn't support it natively (most modern APIs do — Stripe, Square, AWS), build your own deduplication layer using a key store and request fingerprinting.

Without idempotency, distributed systems eventually:
- Double-charge customers
- Send duplicate notifications
- Create ghost records
- Cause regulatory issues

There is no "we'll add idempotency later." Add it on day one.

---

## Universal Adoption

Every modern payment API supports idempotency keys:
- Stripe ✅
- Square ✅
- PayPal ✅
- Adyen ✅
- AWS API Gateway ✅
- Bitpanda's API (assumed, as a serious fintech) ✅

Convergent evolution — when an industry universally adopts a pattern, that's the pattern. Use it.
