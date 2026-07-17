# Day 20 — Chapter 4 Wrap-Up + Full API Design Problem
**Chapter 4: API Design Patterns | Date: 2026-07-17**
**Milestone: completes Chapter 4 and Phase 1 (Foundations, Days 1–20)**

---

## Revision Quiz Results

| Q | Topic | Score |
|---|---|---|
| 1 | JWT three parts | 10/10 ⬆️ **GAP CLOSED** (header, payload, signature) |
| 2 | Webhook HMAC | 9/10 ⬆️ **GAP CLOSED** (shared secret, unique per request, origin+integrity) |
| 3 | CDC — who writes contract | 10/10 |
| 4 | Async provider/consumer roles | 7/10 — producer=provider, reader=consumer |
| 5 | Spec-first catch points | 8/10 — missed compile-time catch |
| 6 | BOLA | 10/10 ⬆️ **GAP CLOSED** (object-ownership specific) |
| 7 | API key compromise | 8/10 — reasoning fixed, reorder to revoke-first |
| 8 | Event-carried state transfer | 10/10 |
| 9 | Expand-contract | 10/10 |
| 10 | Circuit breaker OPEN→HALF-OPEN | 10/10 |

**Average: 9.2/10 — best quiz yet. All three persistent gaps (JWT signature, webhook HMAC, BOLA) closed.**

Refinements remaining: async role terminology, "revoke first" ordering on key compromise.

---

## Chapter 4 Synthesis — The API Layer Decision Map

Five decisions, walked in order, for any system's API layer:

```
1. PROTOCOL   → REST | gRPC | GraphQL | WebSocket | Webhook | Kafka event
2. SECURITY   → JWT | OAuth 2.0 | API key | mTLS  + BOLA/authz checks
3. EVOLUTION  → versioning (REST) | expand-contract (events)
4. RESILIENCE → rate limiting, timeouts, circuit breaker, retry (Ch.3 carries in)
5. CONTRACT   → CDC/Pact | Schema Registry | OpenAPI/AsyncAPI spec
```

### Protocol selection
| Need | Protocol |
|---|---|
| Public / partner API | REST |
| Internal service-to-service | gRPC |
| Mobile app, flexible queries | GraphQL |
| Live server push to client device | **WebSocket** (never Kafka to a device) |
| Notify a partner's system | Webhook |
| Internal async fan-out | Kafka event |

Key insight: real systems use several at once, one per boundary. A request can be
REST at the edge → gRPC internally → Kafka event after → webhook to a partner.

### Security per client
```
End users (mobile/web) → OAuth 2.0 → JWT (15 min access + refresh)
Partners (server-side) → API key (long-lived, bcrypt, revocable)
Internal services      → mTLS (zero-trust BY DEFAULT, not optional)
Every endpoint         → BOLA check: JWT identity vs resource owner
```

### Contract by boundary
```
Internal REST/gRPC (both sides controlled) → Pact (consumer-driven)
Kafka event streams (many consumers)       → Avro + Schema Registry
Public partner API (can't run their tests) → publish OpenAPI/AsyncAPI spec
Expand-contract = the universal safe-evolution technique on top of all
```

---

## Design Problem — Food Delivery Platform API Layer

New domain (not Bitpanda) to force applying the framework. Scale: 5M customers, 50K
restaurants, 100K couriers, 10K orders/min peak.

### Q1 — Protocol per boundary (6.5/10)
| Boundary | Answer | Note |
|---|---|---|
| a) Customer app → backend (place order) | REST/HTTPS | ✓ |
| b) Live delivery tracking → customer map | **WebSocket** | Kafka ingests GPS internally, but the DEVICE gets a WebSocket push |
| c) order-service → payment-service | gRPC | ✓ fast, binary, Protobuf |
| d) Notify restaurant tablet of new order | **WebSocket** | Kafka internal → websocket-service → push to tablet |
| e) Completed order → analytics partner | Webhook | ✓ (or subscribable event stream) |
| f) Hotel chain embedded ordering | REST + API key | partner integration |

**Lesson:** a client device needing live updates terminates at WebSocket/SSE, never Kafka.
Chain is: source → Kafka (internal fan-out) → websocket-service → push to device.

### Q2 — Security (9/10)
Customer app → JWT ✓. Hotel partner → API key ✓. Internal → mTLS ✓.
Correction: internal security is zero-trust BY DEFAULT, not "skip unless you care."

### Q3 — BOLA (9/10)
```
customer-456 logged in, calls GET /orders/order-789 (belongs to customer-123)
→ returns another customer's order (address, phone, items)
Fix: fetch order, check order.getCustomerId().equals(authUserId) → else 403
```
Sharper than the wallet case: ownership isn't in the URL, must fetch object then check owner field.
Other surfaces: courier viewing another's earnings, restaurant reading another's orders.

### Q4 — Resilience stack (5/10)
Full stack, outer → inner:
```
① BULKHEAD        → isolated thread pool for Stripe calls
② CIRCUIT BREAKER → trip OPEN on persistent failure, fail fast
③ RETRY           → transient blip: exponential backoff + JITTER (ms, not 30s)
④ Stripe call
⑤ TIMEOUT         → cap wait (e.g. 3s)
```
Mnemonic: Bulkhead → Circuit Breaker → Retry → [call] → Timeout.
- Slow Stripe → timeout + bulkhead
- Down Stripe → circuit breaker
- Blip → retry with backoff
Payments MUST carry an idempotency key on every retry → never double-charge.
(Retry is ms with backoff, NOT 30s — 30s is circuit-breaker territory.)

### Q5 — Contract testing internal vs external (7/10)
```
Internal (notification-service, courier-service) → contract TESTING
  → Pact (consumer-driven) OR Avro + Schema Registry (better for multi-consumer Kafka)
External analytics partner → can't run their tests
  → publish AsyncAPI spec + enforce schema-registry BACKWARD compatibility
  → expand-contract evolution + migration window
Expand-contract = universal safe-change technique, used for BOTH.
```
Framing: testing tool differs by who you control; expand-contract layered on top everywhere.

### Q6 — 202 async order flow (8/10)
```
POST /orders → 202 Accepted { orderId, status: "PENDING", statusUrl: "/orders/order-789" }
Returns in ~100ms; saga (payment → notify restaurant → assign courier) runs async.
Result delivered via: WebSocket push (best, app already has live tracking) / polling / push notif.
```
Request/async-response pattern (Day 18). Caveat: 202 = "received, processing" ≠ "confirmed."
Order is PENDING until payment clears + restaurant accepts; either can flip it to FAILED/REJECTED.

### Design Problem Total: 44.5/60 ≈ 7.4/10
Strong: security, BOLA, async reasoning.
Drill (both Chapter 3 carryovers): (1) client push = WebSocket not Kafka; (2) full resilience stack + ordering + idempotency.

---

## Chapter 4 Complete — Concepts Mastered
Days 15–19: REST design, versioning, GraphQL, webhooks, BFF, API composition, JWT,
OAuth 2.0, API keys, BOLA, event-driven patterns, CloudEvents, AsyncAPI, contract testing
(Pact, CDC, Schema Registry, spec-based).

## Phase 1 (Foundations, Days 1–20) COMPLETE
Chapters 1–4 done. Next: Chapter 5 (Database Internals) + Bitpanda Phase 1 build begins Day 21.
