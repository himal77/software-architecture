# System Design: Bitpanda API Layer
**Designed:** Day 11 | **Scope:** Client-facing API + internal communication

---

## Requirements

- 50,000 requests/second peak
- Mobile + web clients
- 12 internal microservices
- Real-time trade confirmations within 500ms
- Zero-trust internal security

---

## Architecture

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
         End User: "TradeCompleted: +1 BTC"
```

---

## Key Decisions

| Layer | Protocol | Reason |
|---|---|---|
| Client → Ingress | HTTPS / HTTP/3 | No head-of-line blocking, 0-RTT, QUIC on mobile |
| Internal service calls | gRPC (HTTP/2 + Protobuf) | Binary, 5-10x smaller than JSON, strongly typed |
| Real-time notifications | WebSockets | Bidirectional, persistent, sub-100ms push |
| TLS termination | Ingress only | 1 certificate, not 12 |
| Internal encryption | Istio mTLS | Zero-trust without per-service cert management |

---

## Why Not gRPC for Client Notifications

gRPC is for internal service-to-service calls. Browsers and mobile apps cannot initiate gRPC natively. For server-push to end users, WebSockets is the correct choice — persistent bidirectional connection the server can write to at any time.

---

## Why TLS at Ingress, Not Per Service

12 microservices × 1 certificate each = 12 expiry dates, 12 rotation events, 12 failure points.
Ingress handles 1 certificate. Services get plain HTTP inside the trusted cluster network.
For internal encryption, Istio mTLS sidecar proxies handle it transparently — no service code changes required.
