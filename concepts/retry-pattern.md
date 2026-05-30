# Retry Pattern
**Introduced:** Day 13

---

## What It Does

Handles transient failures — brief blips that self-resolve (network hiccup, connection pool spike).

## Exponential Backoff + Jitter

```
Retry 1: 100ms + random(0-50ms)
Retry 2: 200ms + random(0-50ms)
Retry 3: 400ms + random(0-50ms)
```

Jitter prevents retry storm — without it, all clients retry simultaneously and overwhelm the recovering service.

## Rules

- Only retry **idempotent operations** (GET, PUT, DELETE)
- For non-idempotent (POST): add idempotency key first, then safe to retry
- Never retry business logic failures (insufficient funds, validation errors)
- Always set a max attempt limit

## vs Circuit Breaker

| Retry | Circuit Breaker |
|---|---|
| Transient failures (milliseconds) | Persistent failures (minutes/hours) |
| Service will self-recover | Service needs time to recover |
| Postgres connection pool spike | Wallet Service completely down |

## Spring Boot

```yaml
resilience4j.retry.instances.walletService:
  maxAttempts: 3
  waitDuration: 100ms
  enableExponentialBackoff: true
  exponentialBackoffMultiplier: 2
  retryExceptions:
    - java.net.ConnectException
  ignoreExceptions:
    - com.bitpanda.InsufficientFundsException
```
