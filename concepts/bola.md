# BOLA — Broken Object Level Authorization
**Introduced:** Day 17

---

## What It Is

#1 vulnerability in OWASP API Security Top 10.

Server checks "is the user authenticated?" but not "does this user OWN this resource?"

## The Attack

```
Attacker authenticates as user-456.
Calls: GET /api/v1/wallets/user-123
Server fetches user-123's wallet and returns it.

Attacker can enumerate any userId and read any wallet.
```

## The Fix

```java
@GetMapping("/wallets/{userId}")
public Wallet getWallet(@PathVariable String userId, HttpServletRequest request) {
    String authenticatedUserId = (String) request.getAttribute("X-User-Id"); // JWT, trusted
    if (!authenticatedUserId.equals(userId)) {
        throw new ForbiddenException("Access denied");  // 403
    }
    return walletService.getWallet(userId);
}
```

**Rule:** Path param = untrusted input. JWT claim = trusted identity. Always compare.

## Other API Vulnerabilities

| Vulnerability | Description |
|---|---|
| BFLA | Broken Function Level Authorization — regular user calls admin endpoint |
| Mass Assignment | POST body `{"isAdmin":true}` → server blindly binds to model |
| Excessive Data Exposure | Returns full object when only 2 fields needed |
| Injection | SQL/NoSQL/command injection via unsanitised API input |
