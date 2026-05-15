# ID / Code Generation Strategies
**Introduced:** Day 1

---

## The Problem
At scale, multiple app instances need to generate unique IDs/codes simultaneously without coordination overhead.

---

## 3 Approaches

### 1. Hash-Based
```
short_code = base62( MD5(original_url) ).first(6)
```
| | |
|---|---|
| ✅ | No coordination needed, scales infinitely |
| ✅ | Deterministic — same URL always gives same code |
| ❌ | Collision possible (different URLs → same hash prefix) |
| ❌ | Same URL can't have two separate links (e.g. different campaigns) |

---

### 2. Centralized Counter (Redis INCR)
```
Redis INCR global_counter → get number → base62 encode → short_code
```
| | |
|---|---|
| ✅ | Atomic — no race conditions, no collision |
| ✅ | Simple |
| ❌ | Redis is a single point of failure (need Sentinel/Cluster) |
| ❌ | Reveals business volume (sequential IDs) |

---

### 3. Pre-Generated Pool (best at scale with expiry)
```
Offline job → generates N unique random codes → stores in "available_codes" table
App → picks code from pool → marks "in_use"
After expiry → returns to pool (with grace period check)
```
| | |
|---|---|
| ✅ | Uniqueness verified offline, not at request time |
| ✅ | Works perfectly with expiry + recycling |
| ✅ | No single point of failure |
| ❌ | Requires background job to keep pool filled |

---

## Code Expiry + Recycling Rules
- Expire after 30 days active use
- Grace period: 7 days expired but not yet recycled
- Never recycle high-traffic codes — security risk (attacker waits for popular code to expire and hijacks it)
- Check traffic threshold before returning code to pool

---

## Rule
Move uniqueness verification **off the critical path** — verify offline (pool generation) not online (at every write request).
