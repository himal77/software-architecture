# Day 19 — API Contract Testing
**Chapter 4: API Design Patterns | Date: 2026-07-16**

---

## Revision Quiz Results

| Q | Topic | Score |
|---|---|---|
| 1 | IP Hash | 10/10 ⬆️ **GAP CLOSED** (4 quizzes to get here) |
| 2 | BOLA example | 6/10 — described general auth, BOLA is object-ownership specific |
| 3 | Webhook HMAC | 0/10 ⚠️ 2nd consecutive miss |
| 4 | Cursor vs offset | 7/10 — got performance, missed consistency-under-inserts |
| 5 | Event-carried state transfer | 10/10 |
| 6 | Expand-contract | 10/10 |
| 7 | CloudEvents | 8/10 — problem + data/type correct, missed `id` (dedup) |
| 8 | JWT three parts | 0/10 ⚠️ — still missing "signature" |
| 9 | API key compromise | 7/10 — revoke correct, but API keys are long-lived not 30-min |
| 10 | Circuit breaker OPEN→HALF-OPEN | 9/10 — time-based trigger correct, minor ordering |
| 11 | Saga compensating transactions | 8/10 — mechanism right; styles are Orchestration/Choreography |
| 12 | gRPC 4 patterns | 8/10 — described correctly, need proper names |

**Average: 7.6/10** (up from 5.6 yesterday)

### Hard gaps to keep drilling
- **Webhook HMAC** (0/10 x2): signed with shared secret, unique per request, proves origin + integrity
- **JWT parts** (0/10): header, payload, **signature**

---

## The Problem Contract Testing Solves

Microservices deploy independently. Integration breaks at the **boundary**, not inside a service.

```
wallet-service renames: { "balance": 1000 } → { "availableBalance": 1000 }
Their own tests pass (updated).
trade-service still reads "balance" → null → breaks in production.
```

Unit tests test each service in isolation. The **contract between them** is never tested.
This is the #1 failure mode of microservices.

---

## Why Not Just E2E Tests

```
Spin up all services together:
- Slow (minutes to boot)
- Flaky (any service down = all fail)
- Hard to pinpoint which service broke the contract
- Combinatorial explosion at 20+ services
```

E2E has its place (full user journeys) but is the wrong tool for verifying service-to-service contracts.

---

## Contract Testing

Test the agreed request/response shape between two services WITHOUT running both together.

```
Consumer → service that CALLS      (trade-service)
Provider → service that RESPONDS   (wallet-service)
```

Both sides test against the same contract independently:
- Consumer test → "I send this, I can handle this response" → generates contract
- Provider test → "given this contract, do I actually produce this response?" → verifies it

Provider change that breaks a consumer → fails in provider's own CI, before production.

---

## Consumer-Driven Contract Testing (CDC)

**The consumer defines the contract, not the provider.**

Reason: the provider exists to serve consumers. Consumers declare what they need; provider must satisfy every consumer.

```
Provider-first (wrong): provider lists everything it can return → can't tell which
                        changes break a real consumer → silent breakage
Consumer-driven (right): each consumer declares ONLY what it uses → provider sees
                         the exact union of real expectations → breaks caught in CI
```

Flow:
```
1. trade-service (consumer) writes test: "GET /wallets/user-456 → { balance, currency }"
2. Test generates a CONTRACT FILE (a "pact")
3. Pact published to a broker, shared with wallet-service
4. wallet-service (provider) runs the pact against real code
   → pass → safe to deploy   → fail → broke a consumer → block deploy
```

---

## Pact — The Standard Tool

De-facto CDC framework (JVM, JS, Python, Go, .NET). Contract file is called a "pact."

### Consumer side (trade-service)
```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "wallet-service")
class WalletClientPactTest {

    @Pact(consumer = "trade-service")
    RequestResponsePact getWalletPact(PactDslWithProvider builder) {
        return builder
            .given("user-456 has a wallet")
            .uponReceiving("a request for user-456 wallet")
                .path("/wallets/user-456").method("GET")
            .willRespondWith()
                .status(200)
                .body(new PactDslJsonBody()
                    .numberType("balance", 1000)
                    .stringType("currency", "EUR"))
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getWalletPact")
    void testGetWallet(MockServer mockServer) {
        WalletClient client = new WalletClient(mockServer.getUrl());
        Wallet wallet = client.getWallet("user-456");
        assertThat(wallet.getBalance()).isEqualTo(1000);
        // Verifies client works AND generates the pact file
    }
}
```

### Provider side (wallet-service)
```java
@Provider("wallet-service")
@PactBroker(url = "https://pact-broker.bitpanda.internal")
class WalletServiceProviderTest {

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPacts(PactVerificationContext context) {
        context.verifyInteraction();  // replays every consumer's pact against real endpoints
    }

    @State("user-456 has a wallet")
    void setupUserWallet() {
        walletRepository.save(new Wallet("user-456", 1000, "EUR"));
    }
}
```

Rename `balance` → `availableBalance` → provider test fails (pact still expects `balance`) → deploy blocked.

---

## Pact Broker + can-i-deploy

Shared hub storing pacts + verification results.

```
        ┌───────────── Pact Broker ─────────────┐
        │  pacts + verification results          │
        └────────────────────────────────────────┘
           ▲                          ▲
    publishes pact          verifies & publishes result
           │                          │
     trade-service              wallet-service
     (consumer)                 (provider)
```

Killer CI feature:
```bash
pact-broker can-i-deploy \
  --pacticipant wallet-service --version 2.1.0 \
  --to-environment production
```
Answers: "If I deploy this version, will I break any consumer in production?"
Yes → block deploy. No → green light. This is what makes independent deployment safe.

---

## Test Type Comparison

| Test | Scope | Speed | Catches |
|---|---|---|---|
| Unit | One class | ms | Logic bugs inside a service |
| **Contract** | **Boundary between 2 services** | **fast** | **Broken integration shape** |
| Integration | Service + DB/Kafka | medium | Wiring, real dependencies |
| E2E | Whole system | slow | Full user journeys |

---

## Bitpanda Application

```
trade-service   → consumer of wallet-service, order-service
api-gateway BFF → consumer of every downstream service
Partner API     → partners are consumers of YOUR public API

Every internal gRPC/REST boundary → a pact
Provider CI runs can-i-deploy before every deploy
→ teams deploy independently, no shared integration-test bottleneck
```

**Public partner API:** don't use CDC (can't run partners' tests). Publish OpenAPI/AsyncAPI spec as the contract, test against the spec (schema validation). CDC is for services you control on both sides.

---

## Deep Dive 1: Contract Testing for Async / Kafka (no HTTP API)

Question raised: "if the producer has no API but sends events, how does contract testing work?"

**The roles flip vs HTTP:**
```
HTTP:  caller = consumer, responder = provider
       trade-service CALLS wallet-service → trade-service is CONSUMER

Kafka: reader = consumer, PRODUCER = provider
       trade-service PUBLISHES event, wallet-service READS it
       → trade-service = PROVIDER (produces the message)
       → wallet-service = CONSUMER (reads the message)
```

The contract is about the **message shape**, not a request/response pair. Whoever reads the
message is the consumer and declares what fields they need.

**Pact for messages** — swaps the mock HTTP server for direct payload verification:

Consumer side (wallet-service reads the event):
```java
@PactTestFor(providerName = "trade-service", providerType = ProviderType.ASYNCH)  // async, not HTTP
@Pact(consumer = "wallet-service")
MessagePact tradeCompletedPact(MessagePactBuilder builder) {
    return builder.expectsToReceive("a trade completed event")
        .withContent(new PactDslJsonBody()
            .stringType("tradeId", "abc123")
            .stringType("userId", "user-456")
            .numberType("amount", 0.01))
        .toPact();
}
// Test feeds the pact message into the REAL @KafkaListener handler → proves it can parse this shape
```

Provider side (trade-service produces the event):
```java
@PactVerifyProvider("a trade completed event")
String verifyTradeCompletedMessage() {
    TradeCompletedEvent event = tradeEventFactory.buildCompletedEvent("abc123", "user-456", 0.01);
    return objectMapper.writeValueAsString(event);  // return the ACTUAL message this service emits
}
```
Pact compares the produced payload against the consumer's declared shape. No Kafka, no running
service — just payload vs payload. Rename `amount`→`quantity` in producer → mismatch → CI fails.

---

## Deep Dive 2: "What stops the producer test passing with a stale rename?"

Question: "if the producer test isn't updated, doesn't the contract just pass?"

**The provider does NOT verify against its own expectations.** With `@PactBroker`, the provider
pulls the CONSUMERS' pacts from the broker and verifies against those:
```
1. trade-service CI fetches ALL consumer pacts from broker (notification, audit, tax...)
2. builds its real message → { quantity }
3. compares against each consumer pact (which still expects { amount })
4. MISMATCH → verification FAILS
```
The developer can't make it pass by editing trade-service — the expectation lives in the
consumers' pacts on the broker, not in trade-service.

**What keeps the CONSUMER's pact honest?**
1. Pact is GENERATED from the consumer's real handler code (calls getAmount() → pact must contain "amount")
2. `can-i-deploy` gates deploy on the broker's verification results, not a local green build
3. Broker webhooks re-trigger provider verification whenever a consumer publishes a new pact

**Honest caveat:** only protects consumers that HAVE a pact in the broker. A silent consumer
(no pact) or one reading an undeclared field is a blind spot. Discipline: every consumer must
publish a pact, broker is source of truth (never local `@PactFolder` for shared contracts).

---

## Deep Dive 3: Spec-First / Schema-Driven Contracts (AsyncAPI + Schema Registry)

Question: "what about a shared AsyncAPI/Avro schema both services generate models from?"

Different philosophy: instead of consumers DISCOVERING the shape (Pact), write the SCHEMA FIRST,
both services generate code from it.

```
        trade-events.yaml (AsyncAPI)  or  trade-events.avsc (Avro)   ← single source of truth
              │                              │
      generate producer POJO         generate consumer POJO
              ▼                              ▼
        trade-service                  wallet-service
```

**Solves the rename fear at COMPILE time, not test time:**
```
Rename amount → quantity in schema → regenerate both services
→ wallet-service code calling getAmount() NO LONGER COMPILES → build fails before any test
```

**For Kafka in practice: Avro/Protobuf + Confluent Schema Registry** enforces at RUNTIME too:
```
trade-service tries to publish schema v2
→ registry checks compatibility (BACKWARD / FORWARD / FULL)
→ breaking change (removed a required field) → REJECTED at publish time
```
Infrastructure refuses the breaking change — not a test, the registry itself.

### Pact vs Spec-First — the trade-off

| | Pact (consumer-driven) | AsyncAPI/Avro + registry (spec-first) |
|---|---|---|
| Source of truth | Consumers' expectations aggregated | One shared schema |
| Catches break via | Verification + can-i-deploy | Compile failure + registry rejection |
| Knows WHO consumes | Yes (each pact explicit) | No |
| Silent consumer | Blind spot | Still gets compatible schema (safer default) |
| "Can I remove this field?" | "No pact uses it → yes" (precise) | "Removing required field = incompatible → no" (conservative) |
| Best for | Both sides controlled, few consumers | Many consumers, event streams, Kafka |

Pact = precise (knows exactly who uses what, can drop unused fields).
Registry = conservative (protects even unknown consumers via compatibility rules).

---

## Bitpanda Decision: Which Boundary Uses Which

```
Kafka event streams (trade-events, price-updates)
  → Avro + Confluent Schema Registry (spec-first)
  → many consumers, compatibility enforcement wins

Internal gRPC (trade-service → wallet-service)
  → Protobuf IS the schema (spec-first by nature)

Internal REST BFF boundaries
  → optional Pact where per-consumer precision is wanted

Public partner API
  → publish OpenAPI/AsyncAPI spec; partners generate clients from it
```
Industry standard for a Kafka-centric fintech system: **Schema Registry + Avro for events**,
Pact as the complement for request/response edges.

---

## Concepts Introduced Today
- Contract testing (verify boundary shape without running both services)
- Consumer-driven contract testing (CDC — consumer defines the contract)
- Pact (the tool) + Pact Broker + can-i-deploy CI gate
- Async/message contract testing (roles flip: producer = provider)
- Why the provider can't pass a stale rename (verifies against consumers' broker pacts)
- Spec-first contracts: AsyncAPI/Avro + Schema Registry (compile-time + runtime enforcement)
- Pact vs spec-first trade-off; Bitpanda per-boundary decision
