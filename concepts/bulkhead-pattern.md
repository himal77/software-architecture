# Bulkhead Pattern
**Introduced:** Day 13

---

## What It Does

Isolates thread pools per downstream service so one slow service cannot consume all threads and starve others.

Named after ship bulkheads — watertight compartments prevent one hull breach from sinking the ship.

## Without vs With

```
Without: shared pool of 200 threads
  Wallet Service slow → 200 threads consumed → Trade + Notification starved

With:
  Wallet Service:  50 threads
  Trade Service:  100 threads
  Notification:    50 threads
  Wallet slow → only 50 threads affected → others unaffected
```

## When to Use

Use for every external service call in a critical system. Pairs with circuit breaker and retry — bulkhead goes first in the chain.

## Spring Boot

```java
@Bulkhead(name = "walletService", type = Bulkhead.Type.THREADPOOL)
public WalletBalance getBalance(String userId) {
    return walletServiceClient.getBalance(userId);
}
```

## Full Resilience Stack Order

```
Bulkhead → Circuit Breaker → Retry → Downstream Service → Timeout
```
