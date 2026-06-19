# OAuth 2.0
**Introduced:** Day 17

---

## What It Solves

Third-party app needs access to user's data without user sharing their password.

## Authorization Code Flow

```
1. User clicks "Connect with Google"
2. App redirects → Google /authorize?client_id=...&redirect_uri=...&scope=email
3. User logs in, grants consent
4. Google redirects back → /callback?code=abc123   ← short-lived (10 min, single-use)
5. App backend POSTs to Google /token with code + client_secret
6. Google returns access_token + refresh_token
7. App calls Google APIs with access_token
```

## Why Code Exchange (Step 5)?

Authorization code is in the URL → visible in browser history, server logs, referrer headers.

Backend-to-backend exchange uses `client_secret` → token never passes through the browser.

## Scopes

Fine-grained permissions declared upfront.

```
scope=email profile  → can read email + profile only
scope=drive.readonly → can read Drive files only
```

User sees exactly what they're consenting to. App cannot exceed declared scope.

## Key Timings to Remember

| Thing | Lifetime |
|---|---|
| Authorization code | 10 minutes, single-use |
| Access token | 15 minutes |
| Refresh token | 7 days |
| API key | Months to years |
