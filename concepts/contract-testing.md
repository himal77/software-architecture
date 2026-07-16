# API Contract Testing
**Introduced:** Day 19

---

## Problem

Microservices deploy independently. Integration breaks at the boundary, not inside a service.
Unit tests test each service in isolation — the contract between them is never tested.

```
Provider renames { "balance" } → { "availableBalance" }
Consumer still reads "balance" → null → breaks in production
Nobody's unit tests caught it.
```

## Contract Testing

Verify the request/response shape between two services WITHOUT running both together.

- **Consumer** = the service that calls
- **Provider** = the service that responds

Both test against the same contract independently. Provider change that breaks a consumer
fails in the provider's own CI before production.

## Consumer-Driven Contract Testing (CDC)

**The consumer defines the contract**, because the provider exists to serve consumers.

Consumer declares only what it uses → provider sees the exact union of real expectations →
any breaking change fails CI. Provider-first would list everything and hide which changes matter.

## Pact (the tool)

De-facto CDC framework. Consumer test generates a "pact" file → published to Pact Broker →
provider replays it against real endpoints.

```bash
# CI gate before deploy
pact-broker can-i-deploy --pacticipant wallet-service --version 2.1.0 --to-environment production
```
"Will deploying this break any consumer currently in production?" Yes → block. No → deploy.

## When NOT to use CDC

Public/partner APIs — you can't run partners' tests. Publish OpenAPI/AsyncAPI spec as the
contract and validate against the spec. CDC is only for services you control on both sides.

## Async / Kafka (no HTTP API)

Roles flip: the message PRODUCER is the provider, the READER is the consumer.
The contract is the message shape. Pact verifies the serialized payload directly
(ProviderType.ASYNCH) — no Kafka or running service needed, just payload vs expected payload.

## Why a stale provider rename can't sneak through

The provider does NOT verify its own expectations. With `@PactBroker` it pulls the CONSUMERS'
pacts and verifies against those. Rename a field → produced message no longer matches a
consumer's pact → verification fails. `can-i-deploy` gates deploy on the broker's results, not
a local green build. Blind spot: consumers with no published pact.

## Spec-First alternative: AsyncAPI/Avro + Schema Registry

Write the schema first; both services generate models from it. A rename breaks at COMPILE time
(stale code won't compile). For Kafka, Confluent Schema Registry enforces compatibility
(BACKWARD/FORWARD/FULL) at RUNTIME — breaking changes rejected at publish.

- **Pact** = precise (knows who consumes what, can drop unused fields). Best for request/response, both sides controlled.
- **Schema Registry** = conservative (protects unknown consumers). Best for event streams with many consumers.

Bitpanda: Avro + Schema Registry for Kafka events; Protobuf for gRPC; Pact optional for REST BFF; OpenAPI spec for public partner API.

## Test Pyramid Position

Unit (inside a service) < **Contract (boundary)** < Integration (service + deps) < E2E (whole system)
