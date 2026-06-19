# Day 17 — API Security
**Chapter 4: API Design Patterns | Date: 2026-06-19**

---

## Revision Quiz Results

| Q | Topic | Score |
|---|---|---|
| 1 | ACID isolation levels (Repeatable Read vs Serializable) | ✅ |
| 2 | IP Hash load balancing | ✅ |
| 3 | gRPC streaming patterns | ✅ |
| 4 | Circuit breaker states | ✅ |
| 5 | Cascading timeout fix | ✅ |
| 6 | Webhook vs WebSocket | ✅ |
| 7 | BFF pattern | ✅ |
| 8 | Rate limiting algorithms | ✅ |
| 9 | DataLoader / N+1 fix | ✅ |
| 10 | REST versioning | ✅ |

---

## New Content: API Security

### Authentication vs Authorisation
- **Authentication** — who are you? (JWT, API key, mTLS)
- **Authorisation** — are you allowed? (RBAC, BOLA check)

---

## JWT (JSON Web Token)

### Structure
```
header.payload.signature

Header:    {"alg":"HS256","typ":"JWT"}
Payload:   {"sub":"user-123","tier":"premium","exp":1718812800}
Signature: HMAC-SHA256(base64(header) + "." + base64(payload), secret)
```

### Key properties
- **Stateless** — server needs no DB lookup, just validates signature
- **Self-contained** — payload carries user claims (userId, tier, roles)
- **Short-lived** — access token 15 min, refresh token 7 days

### Access + Refresh Token Flow
```
1. Login → server issues: access token (15 min) + refresh token (7 days, HttpOnly cookie)
2. Every request → send access token in Authorization: Bearer header
3. Access token expires → client sends refresh token → server issues new access token
4. Refresh token expires → full re-login required
```

### Why short access token expiry?
Token stolen → attacker has 15 minutes maximum. Without expiry they'd have forever.

### Spring Boot — Generate Token
```java
String token = Jwts.builder()
    .setSubject(userId)
    .claim("tier", tier)
    .setExpiration(new Date(System.currentTimeMillis() + 15 * 60 * 1000))
    .signWith(SignatureAlgorithm.HS256, secretKey)
    .compact();
```

### Spring Boot — Validate in API Gateway Filter
```java
Claims claims = jwtService.validateToken(token);
// Inject trusted userId into downstream request header
mutatedRequest.header("X-User-Id", claims.getSubject());
```

Downstream services **never** read userId from the request body or URL — only from the trusted `X-User-Id` header set by the gateway.

---

## OAuth 2.0

### What it solves
Third-party app needs access to user's data without the user sharing their password.

### Authorization Code Flow
```
1. User clicks "Connect with Google"
2. App redirects to Google: /authorize?client_id=...&redirect_uri=...&scope=email
3. User logs in to Google, grants consent
4. Google redirects back: /callback?code=abc123  ← short-lived code (10 min, single-use)
5. App backend POSTs to Google: /token with code + client_secret
6. Google returns: access_token + refresh_token
7. App uses access_token to call Google APIs on behalf of user
```

### Why not return the token directly in step 4?
Code is in the URL → visible in browser history, server logs, referrer headers.
Backend-to-backend code exchange uses client_secret → token never touches browser.

### Scopes
Fine-grained permissions: `scope=email profile` → app can only read email and profile, cannot access Drive or Calendar.

### Bitpanda usage
- Mobile users login → OAuth 2.0 with JWT as the resulting access token
- Partners don't use OAuth — they use API keys (server-to-server, no user involved)

---

## API Keys

### What they are
Long-lived credentials for **server-to-server** authentication. No user context.

```
sk_live_a1b2c3d4e5f6...
^prefix  ^random 32 bytes
```

### Lifecycle — long-lived, NOT short-lived
- Valid for months to years
- Manually rotated on schedule or when compromised
- **Not** 10 minutes — that is the OAuth authorization code expiry

### Security rules
| Rule | Why |
|---|---|
| Send in header only (`Authorization: ApiKey sk_...`) | Never in URL — URL logged everywhere |
| Store hashed (bcrypt) | DB breach doesn't expose raw keys |
| Prefix with `sk_live_` / `sk_test_` | Scanners (GitHub) can auto-detect and alert |
| Never in client-side code | Exposed to end users |

### Compromised key response process
1. Partner reports compromise (or your scanner detects it in a public repo)
2. Immediately set `status = REVOKED` in DB
3. API Gateway rejects all requests using that key from this moment
4. Issue a new key via secure channel (encrypted email, vault)
5. Pull audit logs: all requests made with that key — what data was accessed?
6. Notify security team if sensitive data was exposed

---

## BOLA — Broken Object Level Authorization

The #1 API vulnerability (OWASP API Security Top 10).

### The attack
```
Attacker is user-456.
GET /api/v1/wallets/user-123  ← path param says user-123
Server fetches user-123's wallet and returns it.
```
Server only checked "is the user authenticated?" — not "does this user OWN this wallet?"

### The fix
```java
@GetMapping("/wallets/{userId}")
public Wallet getWallet(@PathVariable String userId, HttpServletRequest request) {
    String authenticatedUserId = (String) request.getAttribute("X-User-Id"); // from JWT via gateway
    if (!authenticatedUserId.equals(userId)) {
        throw new ForbiddenException("Access denied");
    }
    return walletService.getWallet(userId);
}
```

Rule: **path param = untrusted input. JWT claim = trusted identity.**

### Other API vulnerabilities (brief)
| Vulnerability | Description |
|---|---|
| BFLA | Broken Function Level Authorization — user accesses admin endpoints |
| Mass Assignment | POST body has `isAdmin: true` → server binds it blindly |
| Excessive Data Exposure | Returning full user object when only name needed |
| Injection | SQL/NoSQL/command injection via API input |

---

## Auth Strategy Summary — Bitpanda

| Client | Mechanism | Why |
|---|---|---|
| Mobile users | JWT (OAuth 2.0 flow) | Stateless, short-lived, carries claims |
| Web users | JWT (same) | Same as mobile |
| Partner banks | API key | Server-to-server, no user involved, long-lived |
| Internal microservices | mTLS | Mutual certificate auth, zero-trust network |

---

## Design Problem: Bitpanda Auth System

**Score: 7.5/10**

| Question | Score | Note |
|---|---|---|
| Auth mechanism per client | 8/10 | Mechanisms correct; API key lifetime wrong |
| Stolen access token mitigation | 10/10 | Short expiry = limited blast radius |
| Compromised API key response | 3/10 | "10 min session" wrong — needs revocation process |
| BOLA vulnerability on GET /wallets/{userId} | 10/10 | Correct 403 + ownership check |

**Key correction:** API keys are long-lived (months/years). The 10-minute expiry is for OAuth authorization codes, not API keys. When an API key is compromised, you revoke it — you don't wait for it to expire.

---

## Concepts Introduced Today
- JWT structure and access/refresh token pattern
- OAuth 2.0 authorization code flow
- API keys — long-lived, bcrypt storage, revocation process
- BOLA and API security vulnerabilities

## Designs Built Today
- Bitpanda auth design (see designs/bitpanda-auth.md)
