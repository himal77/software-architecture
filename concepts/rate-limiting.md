# Rate Limiting
**Introduced:** Day 14

---

## What It Does

Controls how many requests a client can make in a given time window. Protects system from abuse, enforces fair usage tiers.

## Four Algorithms

| Algorithm | How | Best For |
|---|---|---|
| Fixed Window Counter | Counter resets at window boundary | Simple use cases |
| Sliding Window Log | Timestamp log, remove old entries | Maximum accuracy |
| Sliding Window Counter | Weighted estimate: current + previous × fraction | Balance of accuracy + efficiency (Cloudflare) |
| Token Bucket | Tokens refill at fixed rate, burst allowed | APIs with burst tolerance (AWS, Stripe) |

## Implementation

Lives at **API Gateway**. Counter stored in **Redis** (shared across all gateway pods).

```
Key: rate_limit:{userId}:{window}
TTL: 2 minutes
Response on breach: HTTP 429 + Retry-After header
```

## Tiers

```
Free:     100 req/min
Premium:  1,000 req/min
Partner:  10,000 req/min
Internal: unlimited
```

Tier read from JWT claim. No separate lookup needed.

## vs Back-pressure

Rate limiting = external clients → your system (protect from outside).
Back-pressure = internal producer → internal consumer (protect downstream).
