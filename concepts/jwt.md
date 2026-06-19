# JWT (JSON Web Token)
**Introduced:** Day 17

---

## Structure

```
header.payload.signature

Header:    {"alg":"HS256","typ":"JWT"}  — base64url encoded
Payload:   {"sub":"user-123","tier":"premium","exp":1718812800}  — base64url encoded
Signature: HMAC-SHA256(header + "." + payload, secret)
```

Signature prevents tampering. If payload is modified, signature no longer matches.

## Access + Refresh Token Pattern

```
Access token:  15 min — sent in every request (Authorization: Bearer header)
Refresh token: 7 days — stored in HttpOnly cookie, used only to get new access token
```

Short access token = limited blast radius if stolen. Attacker gets at most 15 minutes.

## Validation Flow (API Gateway)

```
Client request → API Gateway extracts JWT → validates signature → injects X-User-Id header
Downstream services read X-User-Id from header ONLY — never from URL or request body
```

## Key Rule

Never trust userId from URL path or request body. Trust only the JWT claim injected by the gateway.
