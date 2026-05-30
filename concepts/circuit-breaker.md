# Circuit Breaker Pattern
**Introduced:** Day 13

---

## What It Does

Stops calling a failing downstream service. Returns fallback immediately. Gives downstream time to recover. Prevents cascading failures.

## Three States

```
CLOSED   → normal operation, requests pass through
OPEN     → failure threshold exceeded, fail fast, return fallback
HALF-OPEN → after timeout, probe requests to test recovery
```

Transitions: CLOSED → OPEN (on threshold breach) → HALF-OPEN (after timeout) → CLOSED (on success) or OPEN (on failure).

## When to Use

**Persistent failures only** — service is down for minutes/hours.

Do NOT use for transient failures (brief spikes) — use Retry instead. Circuit breaker opening for a 200ms blip blocks the critical path for 30 seconds.

## Spring Boot

```java
@CircuitBreaker(name = "walletService", fallbackMethod = "walletFallback")
public WalletBalance getBalance(String userId) {
    return walletServiceClient.getBalance(userId);
}
```

```yaml
resilience4j.circuitbreaker.instances.walletService:
  slidingWindowSize: 10
  failureRateThreshold: 50
  waitDurationInOpenState: 30s
```
