# PENDING Status Pattern (Cross-System Write Safety)
**Introduced:** Day 2 | Related: Outbox Pattern (Chapter 10)

---

## The Problem
You cannot wrap writes to two different storage systems (e.g. Postgres + S3) in a single transaction. If one succeeds and the other fails, data is inconsistent.

```
Postgres write: success
S3 write: FAILS
→ Metadata exists pointing to S3 object that doesn't exist
→ User gets broken paste on GET
```

---

## The Solution: Status Flag
```
1. Write metadata to Postgres (status = PENDING)
2. Upload content to S3
3. Update metadata (status = ACTIVE)
4. Only show ACTIVE records to users
5. Background job: retry or clean up PENDING records older than X minutes
```

---

## Why It Works
- User never sees a broken paste — only ACTIVE records are served
- If S3 fails: status stays PENDING → background job retries S3 upload
- If Postgres update (step 3) fails: background job detects PENDING + S3 exists → retries update
- Fully recoverable without distributed transactions

---

## Pastebin POST Flow
```
App receives POST /pastes
  1. INSERT paste (status=PENDING) → Postgres
  2. PUT content → S3
  3. UPDATE paste SET status=ACTIVE → Postgres
  4. SET cache → Redis
  5. Return paste_id to client
```

---

## Related Pattern
This is a simplified version of the **Outbox Pattern** — covered in depth in Chapter 10 (Days 51–55).
The outbox pattern generalizes this to event-driven systems using a message queue.

---

## Rule
When writing to multiple storage systems, always write to the primary DB first with a PENDING flag. Make content visible only after all writes succeed.
