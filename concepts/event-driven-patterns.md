# Event-Driven API Patterns
**Introduced:** Day 18

---

## Three Core Patterns

### 1. Request / Async Response
Submit work → get job ID → poll for result.
```
POST /trades → 202 Accepted { tradeId, statusUrl }
GET /trades/abc123 → { status: "COMPLETED" }
```
Use when: operation > 500ms, financial transactions.

### 2. Event Notification
Producer publishes event to Kafka. Multiple consumers act independently.
Producer has zero knowledge of consumers.
Use when: one action triggers work in multiple services.

### 3. Event-Carried State Transfer
Events contain enough data that consumers need no follow-up API call.

**Rule:** include what the majority of consumers need. Don't expose internals.

## Expand-Contract Pattern (Schema Evolution)

Safe way to rename or remove a field:
1. **Expand** — add new field alongside old: `{ "amount": 0.01, "quantity": 0.01 }`
2. **Deprecate** — notify consumers, 6-month migration window
3. **Contract** — remove old field after all consumers migrated

Never remove a field without this process — breaking change.
