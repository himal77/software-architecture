# REST API Design
**Introduced:** Day 15

---

## Core Principles

- **Stateless:** every request self-contained, no server session
- **Uniform interface:** nouns not verbs, standard HTTP methods
- **Cacheable:** Cache-Control headers on every response

## Resource Design

```
POST   /trades          ← create
GET    /trades          ← list (paginated)
GET    /trades/{id}     ← get one
PATCH  /trades/{id}     ← partial update
DELETE /trades/{id}     ← remove
```

## HTTP Methods

| Method | Idempotent | Safe |
|---|---|---|
| GET | Yes | Yes |
| DELETE | Yes | No |
| PUT | Yes | No |
| POST | No | No |
| PATCH | No | No |

## Status Codes to Know

```
201 Created      → POST success (+ Location header)
204 No Content   → DELETE success
400 Bad Request  → validation failed
401 Unauthorized → bad/missing JWT
403 Forbidden    → valid JWT, no permission
422 Unprocessable → business logic rejection
429 Too Many     → rate limit
503 Unavailable  → circuit breaker open
```

## Pagination

Cursor-based preferred for real-time data:
```
GET /trades?after=trade-abc123&size=20
```
Offset-based (`page=2`) shifts when data is inserted — skips/duplicates rows.

## Versioning

URL versioning: `/v1/trades`, `/v2/trades`

Breaking changes require new version:
- Removing/renaming fields
- Changing field types
- Making optional fields required

Non-breaking (no version needed):
- Adding optional fields
- Adding new endpoints

Deprecation: 6-month notice before sunsetting old version.
