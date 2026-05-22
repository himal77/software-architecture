# Saga Pattern
**Introduced:** Day 9

---

## What It Is
A distributed transaction made of a **sequence of local transactions**, each with a defined **compensating action** to undo it if a later step fails.

```
Forward path: T1 → T2 → T3 → T4
If T4 fails:  C3 → C2 → C1   (compensations in reverse)
```

Each compensation is a NEW local transaction. Nothing held uncommitted. No locks across services.

---

## Why Saga Won Over 2PC

| 2PC | Saga |
|---|---|
| Synchronous, blocking | Asynchronous, non-blocking |
| Locks held during prepare | No locks across steps |
| Coordinator SPOF | Tolerant of failures |
| Doesn't work across orgs (Stripe) | Works with any API |
| Modern DBs dropping support | Industry standard |

---

## Two Implementations

### Orchestration (CENTRAL coordinator)
```
Orchestrator calls each service in sequence.
On failure, runs compensations in reverse.
State persisted at every step.
On crash, resumes from last completed step.

Tools: Temporal, Camunda, AWS Step Functions, Spring State Machine.
```

**Best for:** Complex flows with many ordered steps, compensations, recovery needs.

### Choreography (NO coordinator)
```
Services react to events from each other.
Wallet service deducts → publishes event
Crypto service consumes → transfers → publishes event
...

Tools: just Kafka + consumers.
```

**Best for:** Simple event-driven flows, independent reactions.

**Production preference for complex flows: Orchestration.** Easier to debug, single source of business logic.

---

## 3 Properties Every Saga Step Must Have

### 1. Idempotent
Calling same step twice = same effect as once.
Use `Idempotency-Key` header. Service stores key + result. Duplicates return cached result.

### 2. Reversible
Every forward step has a compensation.
- Deduct ↔ Credit
- Charge ↔ Refund
- Send email → can't unsend → design around it

### 3. Retriable
- Transient failure (network) → automatic retry with backoff
- Permanent failure (insufficient funds) → don't retry
- Distinguish via error response

---

## Critical Rule: Compensations Must Always Succeed

A failed compensation = inconsistent state with no automated recovery.

Make compensations:
- Simple (single UPDATE)
- Retriable forever
- Idempotent
- Alert humans on repeated failure

---

## When to Use Saga

```
✅ Multiple services / databases must update together
✅ External API calls with side effects (Stripe, email, SMS)
✅ Long-running workflows (minutes to days)
✅ Need crash recovery and resumability

❌ Single DB → use @Transactional instead
❌ All operations are reads → no atomicity needed
```

---

## Bitpanda Example

Trading "Buy 1 BTC for €60K":
- 8 steps across Postgres + Redis + Cassandra + Kafka
- Each step idempotent with `trade-abc123:step-N` key
- Orchestrator (Temporal) tracks state, runs compensations on failure
- Outbox pattern emits events reliably to Kafka
- Stripe charge uses native Idempotency-Key header

See [bitpanda-project/README.md](../bitpanda-project/README.md) for full saga design.
