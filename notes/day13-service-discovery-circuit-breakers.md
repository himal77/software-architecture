# Day 13 — Service Discovery, Circuit Breakers & Retry Patterns
**Chapter 3 | Phase 1: Foundations**
**Date:** 2026-05-29

---

## What Was Covered

Service discovery (client-side vs server-side), cascading failures, circuit breaker (3 states), retry with exponential backoff + jitter, bulkhead pattern, full resilience stack order, Bitpanda resilience design.

---

## Concept 1: Service Discovery

How services find each other dynamically when pod IPs change constantly.

### Client-Side Discovery
```
Service A asks registry: "Where is Service B?"
Registry returns list of healthy IPs
Service A picks one and calls directly
Examples: Netflix Eureka, Consul
```

### Server-Side Discovery
```
Service A calls DNS name / load balancer
Load balancer asks registry internally, forwards to healthy instance
Service A never knows pod IPs
Examples: Kubernetes Service + CoreDNS (what you use)
```

### In Kubernetes
```
Service A calls: http://wallet-service:8080/balance
CoreDNS resolves "wallet-service" → ClusterIP (stable, never changes)
Kubernetes Service routes to a healthy pod
```

**Service registry = etcd** (Raft-based, from Day 6). CoreDNS reads from etcd to answer DNS queries. Every pod registration stored in etcd.

Never hardcode pod IPs. Always use service names.

---

## Concept 2: Cascading Failures

One slow downstream service takes down everything upstream.

```
Trade Engine → Notification Service (8s response time)
Trade Engine threads wait 8s each
1000 req/sec × 8s = 8000 threads waiting
Trade Engine out of threads → down

Non-critical service (notifications) killed critical service (trading)
```

**Root cause:** synchronous call to a non-critical service on the critical path.
**Real fix:** move notifications off critical path → Kafka (fire and forget).

---

## Concept 3: Circuit Breaker

Stops calling a failing downstream service. Returns fallback immediately. Gives downstream time to recover.

### Three States

```
CLOSED → normal, requests pass through, failure counter increments
  │
  │ failures >= threshold (e.g. 50% of last 10 calls)
  ↓
OPEN → all requests fail fast, no downstream call, returns fallback
  │
  │ after timeout (e.g. 30s)
  ↓
HALF-OPEN → probe requests allowed
  │           success → CLOSED
  └────────── failure → OPEN again
```

### Spring Boot (Resilience4j)

```java
@CircuitBreaker(name = "walletService", fallbackMethod = "walletFallback")
public WalletBalance getBalance(String userId) {
    return walletServiceClient.getBalance(userId);
}

public WalletBalance walletFallback(String userId, Exception e) {
    return WalletBalance.builder()
        .userId(userId)
        .balance(cachedBalanceService.get(userId))
        .stale(true)
        .build();
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      walletService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
```

### When to Use

Circuit breaker = **persistent failures** (service is down, not coming back soon).
NOT for transient failures (brief blips) — use retry for those.

---

## Concept 4: Retry Pattern

Handles **transient failures** — brief blips that self-resolve.

```
Network hiccup → fail → retry 100ms later → success
Postgres connection pool momentarily exhausted → fail → retry → success
```

### Naive Retry Problems

**Retry storm:** all clients fail at same time → all retry at same time → overwhelm recovering service → it stays down.

**Non-idempotent operations:** POST /trades retried → duplicate trade → customer charged twice.

### Exponential Backoff + Jitter

```
Retry 1: 100ms + random(0-50ms)
Retry 2: 200ms + random(0-50ms)
Retry 3: 400ms + random(0-50ms)
Retry 4: 800ms + random(0-50ms)
```

Jitter spreads retries across time → prevents thundering herd / retry storm.

Only retry **idempotent operations**. For non-idempotent (POST /trades): add idempotency key, then safe to retry.

### Spring Boot (Resilience4j)

```yaml
resilience4j:
  retry:
    instances:
      walletService:
        maxAttempts: 3
        waitDuration: 100ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.bitpanda.exceptions.InsufficientFundsException
```

`ignoreExceptions`: don't retry business logic failures — insufficient funds is not transient.

---

## Concept 5: Bulkhead Pattern

Isolates failures so one slow service can't consume all threads.

```
Without bulkhead:
  Shared pool: 200 threads
  Wallet Service slow → 200 threads consumed → all services starved

With bulkhead:
  Wallet Service:       50 threads
  Trade Service:       100 threads
  Notification:         50 threads

  Wallet slow → only 50 threads affected → Trade + Notification unaffected
```

Named after ship bulkheads — watertight compartments prevent one breach from sinking the ship.

---

## The Full Resilience Stack

```
Incoming request
       │
       ▼
  Bulkhead          ← reject if thread pool full (shed load)
       │
       ▼
  Circuit Breaker   ← fail fast if downstream known bad
       │
       ▼
  Retry + Backoff   ← handle transient failures
       │
       ▼
  Downstream service
       │
  Timeout           ← always set — never wait forever
```

Order matters. Bulkhead first, then circuit breaker, then retry.

---

## Circuit Breaker vs Retry — Key Distinction

| | Circuit Breaker | Retry |
|---|---|---|
| **For** | Persistent failures | Transient failures |
| **When service is** | Down for minutes/hours | Blipping for milliseconds |
| **Action** | Stop calling, return fallback | Call again with backoff |
| **Example** | Wallet Service crashed | Postgres connection pool spike |

**Common mistake:** using circuit breaker for a 200ms Postgres spike → circuit opens → blocks critical path for 30 seconds. Massive overreaction. Use retry instead.

---

## Design: Bitpanda Resilience

### Problem: Notification Service (8s latency) took down Trade Engine

**Wrong architecture:**
```
Trade Engine → [sync call] → Notification Service (8s)
Trade Engine threads block → cascade → Trade Engine down
```

**Fixed architecture:**
```
Trade Engine → Kafka → Notification Service (async)
Trade Engine thread free immediately
Notification slowness invisible to trade flow
```

### Resilience Pattern per Service

| Service | Pattern | Reason |
|---|---|---|
| Wallet Service | Retry + backoff | Transient Postgres spikes |
| Wallet Service | Bulkhead | Isolate thread consumption |
| Order Book (Redis) | Circuit breaker | Redis down = persistent, return cached |
| Audit Service | Fire and forget via Kafka | Non-critical, don't block trade |
| Notification Service | Fire and forget via Kafka | Non-critical, don't block trade |

### Triple-Charge Prevention on Retry

```
Client generates UUID before first attempt: trade-abc123
Every retry sends same Idempotency-Key: trade-abc123

Server checks Redis:
  Key exists → return cached result (no re-execution)
  Key missing → execute trade, store result with 24h TTL

3 retries = 1 trade = 1 charge
```

---

## Quiz Gaps Carried Forward

| Gap | Correct Answer |
|---|---|
| ACID anomaly mapping | Read Committed→dirty reads, Repeatable Read→non-repeatable, Serializable→phantoms |
| gRPC 4 patterns | Unary, Server streaming, Client streaming, Bidirectional streaming |
| Circuit breaker vs retry | CB = persistent failures. Retry = transient failures. Never swap. |
| Load balancing algorithms | Round robin, Weighted RR, Least connections, IP hash, Least response time |
