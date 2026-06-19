# Bitpanda Authentication System Design
**Day 17 | Chapter 4: API Design Patterns**

---

## Requirements

Three client types with different trust levels and access patterns:
- Mobile/web users (end users, untrusted network)
- Partner banks (server-to-server, trusted partners)
- Internal microservices (inside cluster, zero-trust)

---

## Auth Mechanism Per Client

```
Mobile / Web User
  └─ OAuth 2.0 authorization code flow → JWT (access + refresh)
     Access token: 15 min (Authorization: Bearer header)
     Refresh token: 7 days (HttpOnly cookie)

Partner Bank
  └─ API key (sk_live_...)
     Long-lived (months–years), manually rotated
     Header only: Authorization: ApiKey sk_live_...
     Stored bcrypt-hashed in DB

Internal Microservice (service-to-service)
  └─ mTLS (mutual TLS)
     Istio handles certificate issuance + rotation
     Each service has its own certificate identity
     No JWT or API key needed inside the cluster
```

---

## Architecture

```
                        ┌─────────────────────────────────┐
Mobile App ─── JWT ───► │                                 │
Web App ─────── JWT ───►│         API Gateway             │
Partner ─── ApiKey ────►│  • validate JWT signature       │
                        │  • verify ApiKey hash           │
                        │  • inject X-User-Id header      │
                        │  • inject X-Client-Type header  │
                        └──────────────┬──────────────────┘
                                       │ X-User-Id (trusted)
                                       ▼
                        ┌─────────────────────────────────┐
                        │       Microservices (mTLS)      │
                        │  wallet-service                 │
                        │  trade-service                  │
                        │  order-service                  │
                        └─────────────────────────────────┘
```

---

## JWT Validation at Gateway

```java
// Spring Cloud Gateway filter
@Component
public class JwtAuthFilter implements GlobalFilter {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractBearerToken(exchange.getRequest());
        Claims claims = jwtService.validateToken(token);  // throws if invalid/expired
        
        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-User-Id", claims.getSubject())
            .header("X-User-Tier", (String) claims.get("tier"))
            .build();
        
        return chain.filter(exchange.mutate().request(mutated).build());
    }
}
```

---

## BOLA Protection in Microservices

```java
// Every endpoint that accesses user-specific data
@GetMapping("/wallets/{userId}")
public Wallet getWallet(@PathVariable String userId, HttpServletRequest request) {
    String authenticatedUserId = request.getHeader("X-User-Id");  // injected by gateway, trusted
    if (!authenticatedUserId.equals(userId)) {
        throw new ForbiddenException("Access denied");
    }
    return walletService.getWallet(userId);
}
```

---

## Stolen Access Token — Blast Radius

```
Token stolen at T=0
Token expires at T+15min
Attacker window: ≤ 15 minutes

Mitigation layers:
1. Short TTL (15 min) — auto-expiry limits damage
2. Refresh token in HttpOnly cookie — XSS cannot steal refresh token
3. Token rotation — new refresh token on each use (old one invalidated)
4. Anomaly detection — concurrent sessions from different geos → auto-revoke
```

---

## Compromised API Key — Response Runbook

```
T+0   Partner reports key compromise (or GitHub scanner detects it)
T+1   Set status = REVOKED in api_keys table
T+1   API Gateway picks up change within 30s (TTL-based cache)
T+2   Issue new key to partner via encrypted channel
T+5   Pull audit log: SELECT * FROM api_request_log WHERE key_id = ? AND ts > now()-30d
T+10  Review accessed endpoints and data
T+15  Notify security team + legal if PII or financial data accessed
```

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Access token format | JWT | Stateless validation at gateway, no DB lookup per request |
| Refresh token storage | HttpOnly cookie | XSS cannot read it; CSRF mitigated by SameSite=Strict |
| API key storage | bcrypt hash | Raw key shown only at creation; DB breach doesn't expose keys |
| Internal auth | mTLS via Istio | Zero-trust: even internal services must prove identity |
| userId propagation | X-User-Id header (gateway-injected) | Single source of truth; downstream never trusts request body |

---

## Trade-offs Considered

**JWT stateless vs stateful sessions**
- JWT: no DB lookup per request (fast), but can't revoke until expiry
- Stateful: can revoke instantly, but DB hit every request at scale
- Decision: JWT with short TTL (15 min) → acceptable revocation window

**API key hashing: bcrypt vs SHA-256**
- bcrypt: slow by design (prevents brute force), but 100ms per auth check
- SHA-256: fast, but vulnerable to rainbow tables
- Decision: bcrypt + prefix lookup (prefix is stored plain, used to find the record first)
