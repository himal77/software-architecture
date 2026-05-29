# Day 11 — Networking & Protocols
**Chapter 3 | Phase 1: Foundations**
**Date:** 2026-05-29

---

## What Was Covered

DNS resolution chain, TCP vs UDP, TLS handshake, HTTP/1.1 vs HTTP/2 vs HTTP/3, WebSockets vs SSE vs long polling. Bitpanda API layer design.

---

## Concept 1: DNS

Translates service names to IP addresses.

**Resolution chain:**
```
1. Local cache (JVM / OS)
2. Cluster DNS (CoreDNS in Kubernetes)
3. Recursive resolver → root → TLD → authoritative nameserver
4. Returns IP + TTL
```

**In Kubernetes:**
```
payment-service → payment-service.default.svc.cluster.local
CoreDNS handles all internal service discovery automatically
```

**Architect concern — DNS TTL:**
- Short TTL = faster failover, more queries
- Long TTL = fewer queries, slower failover
- JVM caches DNS forever by default → set `networkaddress.cache.ttl=30` in K8s environments

---

## Concept 2: TCP vs UDP

**TCP:**
- Guarantees delivery, order, no duplicates
- Requires 3-way handshake before data flows
- Retransmits lost packets → latency spikes
- Head-of-line blocking: one lost packet blocks all subsequent packets
- Used by: HTTP, gRPC, Kafka, Redis, Postgres

**UDP:**
- No delivery guarantee, no ordering
- No handshake → send immediately
- No retransmission → lower latency
- Used by: DNS, video streaming, gaming, QUIC (HTTP/3)

**TCP 3-way handshake:**
```
Client → SYN      "I want to connect"
       ← SYN-ACK  "OK, ready"
Client → ACK      "Connected"
One full round trip before any data flows.
```

---

## Concept 3: TLS

Encrypts data and verifies identity.

**TLS 1.3 handshake:**
```
Client → ClientHello    (supported cipher suites)
       ← ServerHello    (chosen cipher + certificate)
Client verifies certificate against trusted CA
Client → Key exchange   (using server's public key)
       ← Finished
Both sides derive same symmetric key → all traffic encrypted
```

**TLS 1.2 vs 1.3:**
- TLS 1.2: 2 round trips before data
- TLS 1.3: 1 round trip before data
- TLS 1.3 supports 0-RTT resumption for known servers

**In Kubernetes:**
- TLS terminated at ingress controller (nginx, Istio)
- Services receive plain HTTP inside cluster
- One certificate to manage, not one per service

**mTLS (mutual TLS):**
- Both client and server present certificates
- Used in service meshes (Istio) for zero-trust
- Every service-to-service call authenticated + encrypted
- Sidecar proxies handle it — services don't manage certificates

---

## Concept 4: HTTP/1.1 vs HTTP/2 vs HTTP/3

**HTTP/1.1:**
- One request at a time per connection
- Multiple connections workaround (browsers open 6 per domain)
- Head-of-line blocking at application level
- Text protocol

**HTTP/2:**
- Multiplexing: many requests over one connection, in parallel
- Header compression (HPACK) — big win for microservices
- Binary protocol — faster parsing
- TCP head-of-line blocking still exists (transport layer)
- Used by gRPC

**HTTP/3:**
- Built on QUIC (UDP-based)
- One QUIC connection with independent streams
- Lost packet in stream 1 does NOT block stream 2
- 0-1 round trips before data (vs 2 for HTTP/2)
- No TCP head-of-line blocking

**Round trip comparison:**
```
HTTP/1.1 + TLS 1.2: 3 round trips before data
HTTP/2   + TLS 1.3: 2 round trips before data
HTTP/3   + QUIC:    1 round trip (0-RTT for known servers)
```

---

## Concept 5: WebSockets vs SSE vs Long Polling

For server-push scenarios where HTTP request-response is insufficient.

**WebSockets:**
- Persistent bidirectional connection
- Server pushes any time without client request
- Sub-100ms latency typical
- Best for: trade notifications, chat, live collaboration

**Server-Sent Events (SSE):**
- Persistent connection, server → client only (one-way)
- Simpler than WebSockets
- Works over HTTP/2
- Best for: live scores, notifications, read-only streams

**Long Polling:**
- Client sends request, server holds it until event occurs
- On event: server responds, client immediately re-polls
- Works everywhere, no special protocol
- Higher latency and overhead than WebSockets
- Best for: fallback when WebSockets not available

---

## Design: Bitpanda API Layer

**Scale:** 50,000 requests/sec peak, 12 microservices, mobile + web clients

```
Mobile/Web Clients
       │
       │ HTTPS (TLS 1.3 / HTTP/3 where supported)
       ▼
Kubernetes Ingress (nginx)
  - TLS terminated HERE (one certificate)
  - Rate limiting
  - JWT authentication
       │
       │ HTTP/2 plain internally
       ▼
API Gateway Service
  ├── REST/HTTP2 → public-facing endpoints
  └── WebSocket → persistent connections for trade notifications
       │
       │ gRPC (HTTP/2 + Protobuf)
       ▼
Internal Microservices (12 services)
  ├── Trade Service
  ├── Wallet Service
  ├── Order Book Service (Redis)
  └── Notification Service
            │
            │ pushes via WebSocket back through API Gateway
            ▼
         End User receives: "TradeCompleted: +1 BTC"
```

### Key Decisions

| Decision | Choice | Reason |
|---|---|---|
| Client → API Gateway | HTTPS / HTTP3 | No head-of-line blocking, QUIC, 0-RTT |
| Internal service calls | gRPC (HTTP/2 + Protobuf) | Binary, 5-10x smaller than JSON, streaming |
| Real-time notifications | WebSockets | Bidirectional, persistent, sub-100ms |
| TLS termination | Ingress only | 1 certificate to manage, not 12 |
| Internal encryption | Istio mTLS | Zero-trust without per-service cert management |

### Design Gaps from Session

- **WebSockets for push:** HTTP request-response cannot push to clients — server has no open channel. WebSockets establish a persistent connection the server can write to at any time.
- **TLS at ingress, not per service:** 12 certificates = 12 expiry dates, 12 rotation events, 12 failure points. One ingress certificate is operationally simple. Use Istio mTLS for internal encryption if zero-trust is required.
- **gRPC vs REST internally:** REST+JSON is fine but Protobuf is 5-10x smaller and strongly typed. At 50K req/sec the CPU and bandwidth savings are significant.

---

## How This Connects to Your Stack

```
Browser/Mobile
    │ HTTPS TLS 1.3
    ▼
K8s Ingress (TLS terminates here)
    │ HTTP/2
    ▼
Spring Boot Service A
    │ gRPC (HTTP/2 + Protobuf) for internal calls
    │ JDBC over TCP for DB
    ▼
Postgres / Redis / Kafka
```

DNS resolves every service name. CoreDNS handles internal resolution. JVM DNS cache TTL must be set explicitly in K8s.

---

## Quiz Gaps Carried Forward

| Gap | Correct Answer |
|---|---|
| ACID isolation levels | Must know without notes: Read Uncommitted → Read Committed → Repeatable Read → Serializable |
| Lamport clocks | Logical counter for causal ordering. Cannot detect concurrency (need vector clocks) |
| Read-after-write fixes | Route own writes to leader / track LSN / short window on leader |
| WebSockets vs gRPC | gRPC = internal service calls. WebSockets = server push to end users |
| TLS termination | Always at ingress. mTLS via Istio for internal zero-trust |
