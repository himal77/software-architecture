# System Design: Bitpanda Trade Saga
**Designed:** Day 9 | **Scope:** "User buys 1 BTC for €60,000"

---

## Why Saga

```
Multiple resources updated in this trade:
  - Order book (Redis)
  - Buyer's fiat wallet (Postgres)
  - Seller's fiat wallet (Postgres)
  - Buyer's crypto wallet (Postgres)
  - Seller's crypto wallet (Postgres)
  - Audit log (Cassandra)
  - Notification system (Kafka)
  
@Transactional cannot span all of these.
2PC fails (Stripe doesn't support it, complex coordinator).
→ Saga with compensating actions is the only viable approach.
```

---

## Saga Steps (Forward Path)

| Step | Action | Storage | Idempotency Key |
|---|---|---|---|
| 1 | Match buy order with seller order | Redis ZSET | `trade-abc123:step1-match` |
| 2 | Deduct €60K from buyer's fiat wallet | Postgres (`@Transactional` + outbox) | `trade-abc123:step2-deduct-buyer` |
| 3 | Credit €60K to seller's fiat wallet | Postgres + outbox | `trade-abc123:step3-credit-seller` |
| 4 | Debit 1 BTC from seller's crypto wallet | Postgres + outbox | `trade-abc123:step4-debit-seller-crypto` |
| 5 | Credit 1 BTC to buyer's crypto wallet | Postgres + outbox | `trade-abc123:step5-credit-buyer-crypto` |
| 6 | Mark order book entries filled | Redis | `trade-abc123:step6-fill-orders` |
| 7 | Append trade to audit log | Cassandra | `trade-abc123:step7-audit` |
| 8 | Publish "TradeCompleted" event | Kafka (via outbox) | `trade-abc123:step8-notify` |

---

## Compensations

| If step N fails | Compensations (reverse order) |
|---|---|
| Step 5 fails | C4: credit BTC back to seller. C3: debit fiat from seller. C2: credit fiat back to buyer. C1: cancel order book reservation |
| Step 4 fails | C3: debit fiat from seller. C2: credit fiat back to buyer. C1: cancel reservation |
| Step 3 fails | C2: credit fiat back to buyer. C1: cancel reservation |
| Step 2 fails | C1: cancel reservation |
| Step 1 fails | None — no state changed |
| Step 6/7/8 fails | Trade succeeded — retry indefinitely on these (idempotent), alert on persistent failure |

**Compensations must always succeed.** Make them idempotent, retriable forever, simple SQL UPDATEs.

---

## Pattern: Orchestration with Temporal

```
TradeOrchestrator (Temporal workflow):
  - Persists state at every step
  - On crash, resumes from last completed step
  - Calls each service via gRPC/REST
  - Triggers compensations automatically on failure
  - Single place to debug the entire flow
```

Why Temporal over choreography:
- 8 ordered steps with mandatory compensation order
- Auditable: one log shows the entire trade flow
- Recovery built-in (workflow state in DB)
- Industry standard for fintech orchestration

---

## Outbox Pattern

Every wallet operation does:
```
@Transactional
public void deductFiat(...) {
  walletRepo.deduct(...);
  outboxRepo.save(new Event("FiatDeducted", payload));
}
```

Background outbox publisher:
- Polls outbox table every few seconds (or CDC via Debezium)
- Publishes to Kafka
- Marks rows as published

**Result:** Every DB commit eventually produces a Kafka message. No lost events even with crashes.

---

## Idempotency Keys

```
Trade-level (frontend → backend):
  Frontend generates UUID before submit
  Sent in HTTP header: Idempotency-Key: trade-abc123
  Backend stores key + result for 24h
  Duplicate submit → returns stored result without re-executing

Step-level (orchestrator → services):
  Each step uses derived key: trade-abc123:step3-credit-seller
  Service stores key + result
  Saga retry → service sees existing key → returns success without re-crediting

External API (Stripe):
  Stripe natively supports Idempotency-Key header
  Saga passes trade-derived key on every charge/refund call
```

---

## Architecture

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

## What Could Go Wrong (And How It's Handled)

| Failure | What happens | Recovery |
|---|---|---|
| Buyer wallet service crashes mid-deduct | DB rollback, no outbox event | Orchestrator retries step 2 |
| Seller wallet credit fails | Saga aborts, runs C2 (refund buyer), C1 (cancel order) | Trade fully rolled back |
| BTC transfer fails after fiat moved | Compensations run for fiat (C2, C3), reservations cancelled | Trade rolled back |
| Notification fails after trade succeeded | Trade is COMPLETE — retries indefinitely on notification | Retry queue |
| Orchestrator crashes mid-saga | Temporal persists state, resumes from last step on restart | Workflow continues |
| Network partition isolates wallet service | Saga step times out → retries with backoff | Eventually consistent |

---

## Connection to Curriculum

| Curriculum Day | Concept Used |
|---|---|
| Day 1 | Critical path (trade) vs non-critical (notifications) |
| Day 2 | PENDING/draft pattern within order book |
| Day 5 | Redis ZSET for order book |
| Day 6 | Temporal uses consensus underneath |
| Day 7 | Polyglot persistence — Postgres + Redis + Cassandra + Kafka |
| Day 8 | Audit log partitioned by user_id in Cassandra |
| Day 9 | Saga + outbox + idempotency (this design) |

This is exactly what Bitpanda's actual trade engine looks like in production. The patterns are universal across fintech.
