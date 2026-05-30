# Day 12 — Load Balancing, API Gateways & gRPC
**Chapter 3 | Phase 1: Foundations**
**Date:** 2026-05-29

---

## What Was Covered

Load balancing algorithms, Layer 4 vs Layer 7, health checks, API Gateway responsibilities, gRPC deep dive (Protobuf, 4 communication patterns), live order book design (WebSockets at scale, snapshot + delta pattern).

---

## Concept 1: Load Balancing

Distributes incoming requests across multiple backend instances.

### Algorithms

**Round Robin:** requests cycle through servers 1→2→3→1. Simple but ignores server capacity and request weight.

**Weighted Round Robin:** servers get requests proportional to their weight (capacity). Server with 8 cores gets 4x more traffic than server with 2 cores.

**Least Connections:** next request goes to server with fewest active connections. Best for variable-length requests.

**IP Hash (Sticky Sessions):** same client IP always routes to same server. Used for server-side session state. Problem: server failure loses all sessions for that IP. Better: externalize sessions to Redis.

**Least Response Time:** routes to server with lowest combination of active connections + average response time. Most intelligent, used by nginx/HAProxy in production.

### Layer 4 vs Layer 7

**Layer 4 (Transport):**
```
Sees: IP address + port only
Cannot see: HTTP headers, URLs, cookies
Fast — forwards raw TCP packets
Used for: raw throughput, non-HTTP protocols, DB connections
```

**Layer 7 (Application):**
```
Sees: full HTTP request (URL, headers, body, cookies)
Routes by: URL path, auth headers, content type
/api/trades   → Trade Service
/api/wallets  → Wallet Service
Slower than L4 but far more flexible
Used for: microservices routing, A/B testing, canary deployments
```

In Kubernetes:
- **Ingress controller (nginx)** = Layer 7 load balancer
- **Service resource** = Layer 4 load balancer (distributes TCP across pods)

### Health Checks

```
Every 10 seconds: GET /actuator/health → each server
503 or no response → remove from rotation
Passes again → re-add automatically
```

**Liveness probe:** is the app alive? Fail → restart pod.
**Readiness probe:** is the app ready for traffic? Fail → remove from LB rotation, don't restart.

A warming-up pod should fail readiness but pass liveness.

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

---

## Concept 2: API Gateway

Single entry point for all client requests. Handles cross-cutting concerns so microservices don't have to.

### Responsibilities

- **Authentication:** validates JWT once at gateway, forwards user context to services
- **Rate limiting:** per-user/per-tier limits via Redis counter, returns 429 on breach
- **Routing:** URL path → correct microservice
- **SSL termination:** HTTPS → plain HTTP internally
- **Request/response transformation:** format conversion between external and internal
- **Observability:** single place to log all traffic, latency, errors

### API Gateway vs Load Balancer

| | Load Balancer | API Gateway |
|---|---|---|
| Layer | L4 or L7 | L7 only |
| Auth | No | Yes |
| Rate limiting | No | Yes |
| Routing | IP/port/path | Path + headers + auth |
| Transforms | No | Yes |
| Examples | nginx, HAProxy | Kong, AWS API Gateway, Spring Cloud Gateway |

In Kubernetes: Ingress = load balancer. Kong or Spring Cloud Gateway = API gateway (sits behind ingress).

---

## Concept 3: gRPC

Remote procedure call framework. Service A calls Service B like a local method. Built on HTTP/2 + Protocol Buffers.

### Protocol Buffers (Protobuf)

Define service contract in `.proto` file. Compiler generates client + server code.

```protobuf
syntax = "proto3";

service WalletService {
  rpc GetBalance (BalanceRequest) returns (BalanceResponse);
  rpc StreamTransactions (TransactionRequest) returns (stream Transaction);
}

message BalanceRequest {
  string user_id = 1;
  string currency = 2;
}

message BalanceResponse {
  string user_id = 1;
  double balance = 2;
  string currency = 3;
}
```

### Protobuf vs JSON

```
JSON:     {"userId":"abc123","balance":60000.00,"currency":"EUR"}
          52 bytes, string parsing, no type safety

Protobuf: binary encoded
          ~15 bytes, direct memory deserialization, strongly typed
          3-5x smaller, 5-10x faster to serialize/deserialize
```

### Four Communication Patterns

**1. Unary** — one request, one response
```
Client → GetBalance(request) → Server
       ← BalanceResponse     ←
```

**2. Server streaming** — one request, stream of responses
```
Client → StreamPrices(BTC/EUR) → Server
       ← Price(61,000)         ←
       ← Price(61,050)         ← (continuous)
```

**3. Client streaming** — stream of requests, one response
```
Client → Order(buy 0.1 BTC)  →
       → Order(sell 0.5 ETH) → Server
       ← BatchResult          ←
```

**4. Bidirectional streaming** — both sides stream simultaneously
```
Client → Order(...) →
       ← Fill(...)  ←
Client → Order(...) →
       ← Fill(...)  ← (full duplex)
```

Bidirectional streaming = how a real-time trading terminal works.

### gRPC vs REST

| | gRPC | REST |
|---|---|---|
| Use for | Internal microservices | External/public APIs |
| Payload | Protobuf (binary) | JSON (human-readable) |
| Type safety | Compile-time | Runtime |
| Browser support | No (needs grpc-web proxy) | Yes |
| Streaming | Built-in (4 patterns) | Limited (SSE, chunked) |

**Bitpanda rule:**
- Service → Service (internal): gRPC
- Client → API Gateway (external): REST or WebSocket
- Real-time push to browser: WebSocket + JSON (not Protobuf — browsers can't parse it natively)

### Spring Boot gRPC Example

```java
// Client (calling another service)
@GrpcClient("wallet-service")
private WalletServiceGrpc.WalletServiceBlockingStub walletStub;

BalanceResponse balance = walletStub.getBalance(
    BalanceRequest.newBuilder().setUserId(userId).build()
);
```

---

## Design: Live Order Book

**Scale:** 500K concurrent users, 10K order book updates/second, <200ms push latency

### Key Decisions

**1. Transport: WebSockets**
- Persistent bidirectional connection
- Server pushes updates without client polling
- 500K connections ÷ 50K per server = 10 WebSocket gateway servers

**2. Throttling: Batch updates every 100ms**
```
10,000 updates/sec → aggregate → 1 snapshot per 100ms per instrument
10 updates/sec per user instead of 10,000
Binance and Coinbase use exactly this pattern
```

**3. Snapshot + Delta pattern**
```
Snapshot = complete current order book state (sent on connect)
Delta    = only what changed since last update (sent every 100ms)

New user connects:
  Step 1: REST call → GET full snapshot from Redis
  Step 2: WebSocket stream starts → receive deltas
  Client applies deltas onto snapshot → always consistent

Why: deltas are tiny (1-2 rows). Snapshots are large (100+ rows).
     Sending full snapshot every 100ms × 500K users = too much bandwidth.
     Sending only deltas to new user = no base state to apply them to.
```

**4. Instrument-based routing**
```
Kafka topics (one per instrument):
  orderbook.BTC/EUR → 10 partitions
  orderbook.ETH/USD → 10 partitions

WebSocket Gateway servers subscribe to relevant Kafka partitions:
  Server 1 → BTC/EUR watchers
  Server 2 → ETH/USD watchers

When order book updates → Kafka → correct gateway server
→ pushes to all connected users for that instrument
```

### Architecture

```
Order Matching Engine
        │ order placed/filled
        ↓
     Kafka
  (orderbook.BTC/EUR, orderbook.ETH/USD, ...)
        │
        ↓
WebSocket Gateway Cluster (10 servers)
  Each server:
    - Consumes its Kafka partition
    - Aggregates updates every 100ms
    - Pushes delta to connected users
        │
  Redis (current order book snapshot per instrument)
        │
        ↓
500K Browser/Mobile users
  On connect: GET snapshot from Redis (via REST)
  Then: receive live deltas via WebSocket
```

---

## Quiz Gaps Carried Forward

| Gap | Correct Answer |
|---|---|
| ACID anomaly names | Dirty read / Non-repeatable read / Phantom read — must name without notes |
| WebSockets at scale | Batch updates (100ms), snapshot on reconnect, sticky sessions on LB |
| HTTP/2 multiplexing | Multiple requests over ONE connection in parallel — not per-request handshake |
| gRPC vs WebSocket | gRPC = internal service calls. WebSocket = server push to browsers/mobile |
| Snapshot + delta | Snapshot = full state on connect. Delta = changes streamed after. |
