# Day 9 — Distributed Transactions
**Chapter 2 | Phase 1: Foundations**
**Date:** 2026-05-22

---

## Key Concepts

### Why Distributed Transactions Are Hard

A distributed transaction spans multiple resources (DBs, services, external APIs) that need to commit or roll back as a unit.

```
Local transaction:
  BEGIN
    UPDATE accounts SET balance = balance - 100 WHERE id = 'A'
    UPDATE accounts SET balance = balance + 100 WHERE id = 'B'
  COMMIT
  → Postgres handles atomicity natively (WAL + ACID)

Distributed transaction:
  Service 1: deduct €100 from buyer's wallet (its DB)
  Service 2: charge Stripe (external API)
  Service 3: credit seller's wallet (different DB)
  Service 4: send email
  → No single system can guarantee atomicity across all
  → Step 3 fails → can't roll back the Stripe charge
```

@Transactional only spans a single DB connection. Multiple services + external APIs require Saga.

---

## Pattern 1: Two-Phase Commit (2PC)

Classical approach. Used by traditional banking/telco. Largely abandoned.

### How 2PC Works
```
Phase 1 — PREPARE:
  Coordinator → Participant: "Can you commit?"
  Participant → does work, locks resources, doesn't commit → "YES, ready"

Phase 2 — COMMIT:
  All YES → coordinator says "COMMIT" → all commit
  Any NO  → coordinator says "ABORT" → all rollback
```

### Why 2PC Fails in Practice
| Problem | Effect |
|---|---|
| Blocking protocol | Locks held during prepare phase. Coordinator crash → stuck locks |
| Synchronous = slow | Slow participant blocks all. Network round trips × N |
| Coordinator SPOF | Coordinator dies → no one knows commit/abort |
| Cross-org impossible | Stripe doesn't expose 2PC. Modern APIs don't either |
| DB support fading | Postgres XA exists but rare. Cassandra doesn't support it |

**You will likely never implement 2PC.** Industry moved on. Saga is what real systems use.

---

## Pattern 2: Saga (The Modern Standard)

A sequence of local transactions, each with a defined compensating action.

```
Forward path:
  T1: deduct fiat from buyer wallet
  T2: charge card via Stripe
  T3: credit fiat to seller wallet
  T4: transfer crypto

If T4 fails:
  C3: reverse seller credit (debit it back)
  C2: refund Stripe charge
  C1: credit fiat back to buyer wallet
```

**Key insight:** accept that intermediate state is briefly visible. Don't try to hide it. Ensure every step can be undone.

---

### Choreography vs Orchestration

**Orchestration: CENTRAL coordinator**
```
TradeOrchestrator:
  STEP 1: walletService.deductFiat(...)
  STEP 2: paymentService.chargeCard(...)
  STEP 3: walletService.creditFiat(...)
  STEP 4: cryptoService.transferBTC(...)
  
  On failure → run compensations in reverse order
  Persists state at every step (DB or workflow engine)
  On crash → resume from last persisted step
```

**Choreography: NO central coordinator**
```
Wallet service deducts → publishes "FiatDeducted" event
Payment service consumes → charges Stripe → publishes "PaymentCharged"
...

If any step fails → publishes "Failed" event
Each service reacts to events, knows its compensations
```

| | Orchestration | Choreography |
|---|---|---|
| Coordinator | Central | None — distributed |
| Logic location | One place | Spread across services |
| Tooling | Camunda, Temporal, Step Functions | Just Kafka + consumers |
| Debugging | Easy — one log shows flow | Hard — trace events across services |
| Coupling | Higher | Lower |
| Best for | Complex flows, many ordered steps | Simple event-driven, independent reactions |

**Production preference for complex flows: Orchestration.** Easier to reason about.
**Choreography fits:** notification fan-out, simple event reactions.

---

### Critical Saga Properties

Every step must be:

**1. Idempotent**
```
Calling same step twice = same effect as once
Caller passes Idempotency-Key
Service records key → duplicate returns stored result without re-executing
```

**2. Reversible (has compensation)**
```
Every forward step has an undo
"Deduct €60K" ↔ "Credit €60K"
"Charge card" ↔ "Refund card"
"Send email" ↔ design around it (send "we're sorry" instead)
```

**3. Retriable**
```
Transient failures (network, timeout) → automatic retry
Permanent failures (insufficient funds) → no retry
Distinguish them in error responses
```

**Critical rule: Compensations must always succeed.**
Failed compensation = inconsistent state with no automated recovery.
Compensations must be: simple, retriable forever, idempotent, alert humans on repeated failure.

---

## The Outbox Pattern

Solves: "wrote to DB but failed to publish event."

```
Naive approach (broken):
  @Transactional
  public void deductFiat(...) {
    walletRepo.deduct(...);              // commits to DB
    kafka.publish("FiatDeducted", ...);  // publishes event
  }
  
  DB commit succeeds, Kafka publish fails (network blip)
  → DB shows fiat deducted, no event published
  → Saga never proceeds → orphaned state
```

```
Outbox pattern:
  @Transactional
  public void deductFiat(...) {
    walletRepo.deduct(...);
    outboxRepo.save(new Event("FiatDeducted", payload));
    // Both writes in same DB transaction → atomic
  }

  Background outbox publisher:
    Reads unpublished outbox rows
    Publishes to Kafka
    Marks rows as published
```

**Outbox table:**
```
event_id | event_type      | payload | published | created_at
uuid-1   | FiatDeducted    | {...}   | true      | T-5min
uuid-2   | PaymentCharged  | {...}   | true      | T-3min
uuid-3   | SellerCredited  | {...}   | false     | T-1min  ← will be published
```

**Why it works:**
- DB write + outbox write = atomic (same transaction)
- Service crashes after DB commit before publish → outbox publisher retries
- Kafka eventually receives the event
- Consumer-side idempotency handles duplicates

**Variants:**
- **Polling outbox** — worker queries outbox table every few seconds
- **CDC (Debezium)** — reads Postgres WAL → publishes to Kafka. Faster, no app code changes, more infra complexity.

**Rule:** Anywhere you commit data + emit event, you need outbox.

---

## Bitpanda Trade Saga Design

### The Flow: "User buys 1 BTC for €60,000"

```
Step 1: Match order in order book (Redis ZSET)
        Find seller willing to sell 1 BTC at €60K
        Reserve buyer + seller order_ids

Step 2: Deduct €60K from buyer's fiat wallet (Postgres)
        @Transactional: deduct + outbox event

Step 3: Credit €60K to seller's fiat wallet (Postgres)
        @Transactional: credit + outbox event

Step 4: Debit 1 BTC from seller's crypto wallet (Postgres)
        @Transactional: debit + outbox event

Step 5: Credit 1 BTC to buyer's crypto wallet (Postgres)
        @Transactional: credit + outbox event

Step 6: Update order book — mark orders filled (Redis)

Step 7: Write trade to audit log (Cassandra append-only)

Step 8: Publish "TradeCompleted" event (Kafka via outbox)
        Notification service sends email/push
```

### Compensations

| Failed step | Compensations (reverse order) |
|---|---|
| Step 5 | C4: credit BTC back to seller. C3: debit fiat from seller. C2: credit fiat back to buyer. C1: cancel order book reservation |
| Step 4 | C3: debit fiat from seller. C2: credit fiat back to buyer. C1: cancel reservation |
| Step 3 | C2: credit fiat back to buyer. C1: cancel reservation |
| Step 2 | C1: cancel reservation |
| Step 1 | None — no state changed |

### Idempotency Keys

```
Trade-level (from frontend):
  Idempotency-Key: trade-abc123
  Backend stores key + result
  Duplicate submission → returns stored result

Step-level (within saga):
  trade-abc123:step1-match
  trade-abc123:step2-deduct-buyer
  trade-abc123:step3-credit-seller
  ...
  Saga retries step 3 → wallet sees key already processed → returns success

External API:
  Stripe natively supports Idempotency-Key header
```

### Choice: Orchestration

```
Reasons:
1. Complex multi-step business process — clarity matters
2. State must persist at each step (crash recovery)
3. Compensations run in specific order — easier centrally
4. Auditors trace flow easily
5. Tools like Temporal/Camunda specialize in this

Specifically: Use Temporal or Spring State Machine.
Persists workflow state. Resumes from last completed step on crash.
```

### Architecture Diagram

```
                    User clicks "Buy 1 BTC"
                            │
                    Idempotency-Key: trade-abc123
                            ↓
                    TradeOrchestrator (Temporal workflow)
                            │
       ┌────────────┬───────┴───────┬─────────────┬────────────┐
       ↓            ↓               ↓             ↓            ↓
   Order Book    Wallet         Wallet        Crypto       Crypto
   (Redis)      Service        Service       Service      Service
                (buyer)        (seller)      (seller)     (buyer)
                  │              │             │            │
                  ↓              ↓             ↓            ↓
                Postgres       Postgres      Postgres    Postgres
                + outbox       + outbox      + outbox    + outbox
                  │              │             │            │
                  └──────────────┴─────────────┴────────────┘
                            │
                       Outbox Publisher
                            │
                            ↓
                          Kafka
                            │
       ┌────────────────────┼────────────────────┐
       ↓                    ↓                    ↓
   Audit Service     Notification Service   Analytics
   (Cassandra)       (email/SMS/push)      (data warehouse)
```

---

## Stripe Idempotency Case Study

Every Stripe API call accepts `Idempotency-Key` header.

```
POST /charges
Idempotency-Key: trade-abc123-charge
{ "amount": 6000000, "currency": "eur" }
```

Stripe records key for 24h. Duplicate request with same key returns original response — no double charge.

**Production rule for fintech:**
- Every external API call must include idempotency key
- If API doesn't support it natively, build your own deduplication layer
- Without idempotency, distributed systems eventually double-charge customers

---

## Concepts Internalized

| Concept | Status |
|---|---|
| Why @Transactional doesn't span services | ✅ |
| 2PC and why it failed | ✅ |
| Saga pattern — local transactions + compensations | ✅ |
| Choreography vs orchestration | ✅ (was reversed initially) |
| Outbox pattern for reliable event publishing | ✅ |
| Idempotency keys at every saga step | ✅ |
| Compensations must always succeed | ✅ |
| Bitpanda trade saga design | ✅ |

---

## Next
**Day 10 — Chapter 2 final day:** Wrapping up Chapter 2 with consensus integration, distributed snapshots, and a full-system design exercise.
