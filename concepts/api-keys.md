# API Keys
**Introduced:** Day 17

---

## What They Are

Long-lived credentials for server-to-server authentication. No user context involved.

```
sk_live_a1b2c3d4e5f6g7h8...
^prefix  ^32 random bytes
```

## Lifetime — LONG-LIVED

API keys are valid for months to years. They are **not** short-lived.

The 10-minute expiry is for OAuth authorization codes — not API keys.

Rotation happens: manually on schedule, or immediately when compromised.

## Security Rules

| Rule | Why |
|---|---|
| Header only (`Authorization: ApiKey sk_...`) | Never in URL — URLs are logged everywhere |
| Store hashed (bcrypt) | DB breach does not expose raw keys |
| Prefix (`sk_live_`, `sk_test_`) | GitHub secret scanners can auto-detect and alert |
| Never in client-side code | End users can extract it from the browser |

## Compromised Key Response

1. Set `status = REVOKED` in DB immediately
2. API Gateway rejects all requests with that key from this moment
3. Issue a new key via secure channel (vault, encrypted email)
4. Audit all requests made with that key — what data was accessed?
5. Notify security team if sensitive data was exposed

## When to Use

- Server-to-server calls (partner bank calls your API)
- No user context needed (just "is this partner authorised?")
- Stable, long-lived integrations

Not for: end users on mobile/web → use JWT.
