# Two-Phase Commit (2PC)
**Introduced:** Day 9

---

## What It Is
Classical distributed transaction protocol. A coordinator and multiple participants (DBs, services) agree on commit/abort in two phases.

---

## How It Works

```
Phase 1 — PREPARE:
  Coordinator → Participant 1: "Can you commit?"
  Participant 1: does work, locks resources, doesn't commit → "YES, ready"
  
  Coordinator → Participant 2: "Can you commit?"
  Participant 2: does work, locks resources → "YES, ready"
  
  Coordinator → Participant 3: "Can you commit?"
  Participant 3 → "YES, ready"

Phase 2 — COMMIT:
  All YES → Coordinator: "COMMIT"
  All commit. Done.

If any NO:
  Coordinator: "ABORT"
  All rollback.
```

---

## Why 2PC Failed in Practice

| Problem | Effect |
|---|---|
| Blocking protocol | Participants hold locks during prepare. Coordinator crash → stuck locks indefinitely |
| Synchronous = slow | Slow participant blocks all. Network round trips × N participants |
| Coordinator SPOF | Coordinator dies → no one knows whether to commit or abort |
| Doesn't work cross-org | Stripe doesn't expose 2PC. Most modern APIs don't. |
| DB support fading | Postgres XA exists but rare. Cassandra doesn't support it |

---

## When 2PC Is Still Used

- Legacy banking systems with XA-compatible databases
- Some enterprise message queues coordinating with DB writes
- Tightly-coupled internal systems with all participants supporting XA

For modern systems, **Saga is the standard.** You will likely never implement 2PC.

---

## Why Saga Beats 2PC

Saga accepts that intermediate state is briefly visible. Instead of locking everything until everyone agrees, each step commits locally and uses **compensating actions** to undo if a later step fails.

```
2PC:   Lock everything → ask everyone → commit/abort → release
Saga:  Commit step 1 → commit step 2 → if fail, run compensations
```

Saga is non-blocking, works across organizations, and survives coordinator failure. That's why the industry moved on.

---

## Rule
Recognize 2PC when you see XA, JTA, "global transactions" in legacy code. For new systems, design with Saga + outbox + idempotency keys instead.
