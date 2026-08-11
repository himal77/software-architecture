# Day 23 — Transactions, Isolation and Locking in Practice
**Chapter 5: Database Internals | Date: 2026-08-11**

---

## Revision Quiz Results — 6.3/10 (up from 4.9)

| Q | Topic | Score |
|---|---|---|
| 1 | Atomic writes vs check-then-act | 2/10 — answered ADR-004 instead |
| 2 | Reconciliation invariant | 7/10 (after clarification) |
| 3 | MVCC xmin/xmax | 6/10 — mechanism yes, column names no |
| 4 | Reader sees stale not dirty | 8/10 |
| 5 | Composite index column order | 4/10 — restated the order, didn't explain it |
| 6 | Partial index | 6/10 — right choice, wrong reason |
| 7 | synchronous_commit | 10/10 ⬆️ |
| 8 | Other WAL uses | 0/10 — 2nd consecutive miss |
| 9 | Immutability via mapping | 10/10 |
| 10 | Single-transaction atomicity | 10/10 |

**Pattern:** conceptual questions (7, 9, 10) all 10/10. Mechanical detail (xmin/xmax names,
index ordering rules, WAL's other uses) is where marks are lost.

### Corrections
- Refresh token carries NO identity — userId comes from the access token's `sub` claim
- `xmin` = born (creating txn), `xmax` = died (superseding txn)
- **Snapshot** is what a transaction holds; **row version** is what's on disk. Different things.
- Index order rule: **equality first, range last, direction matching ORDER BY**
- Partial index on `published_at` isn't *wrong*, it's *huge* — 50M entries to serve 200
- WAL also gives: streaming replication, point-in-time recovery, CDC (logical decoding)
- Our ledger has NO per-transaction balancing invariant (can't add EUR to BTC) — only
  per-account reconciliation. Claiming full double-entry in an interview would be wrong.

---

## The Four Anomalies

```
DIRTY READ           read uncommitted data
NON-REPEATABLE READ  same row read twice, different values
PHANTOM READ         same query twice, different row COUNT
LOST UPDATE          two read-modify-writes, one silently overwrites the other
```

| Level | Dirty | Non-rep | Phantom | Postgres reality |
|---|---|---|---|---|
| READ UNCOMMITTED | ✗ | ✗ | ✗ | **Doesn't exist** — becomes READ COMMITTED |
| READ COMMITTED | ✓ | ✗ | ✗ | **Default.** New snapshot per STATEMENT |
| REPEATABLE READ | ✓ | ✓ | ✓ | One snapshot per TRANSACTION |
| SERIALIZABLE | ✓ | ✓ | ✓ | Snapshot + dependency tracking, aborts on conflict |

Two Postgres-specific facts:
- **READ UNCOMMITTED is unimplementable under MVCC** — the `xmin` visibility check inherently
  excludes uncommitted versions. Setting it silently gives READ COMMITTED.
- **Postgres REPEATABLE READ also prevents phantoms**, which the SQL standard doesn't require,
  because one snapshot covers the whole transaction.

### The READ COMMITTED trap
```java
@Transactional  // READ COMMITTED
public void process(UUID id) {
    var a = accounts.findById(id);   // balance = 100
    // another transaction commits balance = 50
    var b = accounts.findById(id);   // balance = 50  ← DIFFERENT
}
```
Each STATEMENT gets a fresh snapshot. Logic assuming both reads agree is wrong, and only
fails under load.

---

## Lost Update — Isolation Alone Doesn't Save You

```
A: read 100
B: read 100
A: write 100-30 = 70
B: write 100-50 = 50     ← A's deduction gone
```

Postgres REPEATABLE READ *does* catch this when both write the **same row**:
```
ERROR: could not serialize access due to concurrent update
```

But not when the writes go elsewhere — **write skew**:
```
A: SELECT SUM(amount) WHERE account_id = X   → 100
B: SELECT SUM(amount) WHERE account_id = X   → 100
A: INSERT ledger_entry (-30)     ← different row
B: INSERT ledger_entry (-50)     ← different row
→ both succeed, no row collision, invariant violated
```

Each transaction reads a SET of rows, decides, then writes SOMEWHERE ELSE. No two touch the
same row, so no conflict is visible. **Only SERIALIZABLE catches write skew**, because it
tracks read/write dependencies rather than row collisions.

**Lesson: isolation levels protect against reading wrong data. They do not automatically
protect an invariant spanning multiple rows.** That needs explicit locking, SERIALIZABLE, or
a constraint.

---

## Four Ways to Handle a Write Conflict

### 1. Optimistic — `@Version`
```sql
UPDATE accounts SET balance = 70, version = 6 WHERE id = ? AND version = 5;
-- 0 rows → someone else won → retry
```
Detect and retry, no lock held. **Best when conflicts are rare.**

### 2. Pessimistic — `SELECT ... FOR UPDATE`
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findByIdForUpdate(UUID id);
```
Lock and wait. **Best when conflicts are frequent** (hot row).
```
FOR UPDATE        → blocks other writers and readers-for-update
FOR NO KEY UPDATE → weaker, lets FK checks proceed
FOR SHARE         → multiple readers, writers blocked
```

### 3. Atomic SQL — skip the read
```sql
UPDATE accounts SET balance = balance - 30 WHERE id = ? AND balance >= 30;
```
No read-then-write → no lost update **by construction**. Fastest and safest.
Cost: you don't learn the resulting balance (why the wallet doesn't use it — needs `balance_after`).

### 4. Constraint as backstop
```sql
CONSTRAINT accounts_balance_non_negative CHECK (balance >= 0)
```
Doesn't prevent the conflict; makes the corrupt OUTCOME impossible.

### Choosing
```
Rare conflicts, need resulting value        → optimistic (@Version)
Frequent conflicts, hot row                 → pessimistic (FOR UPDATE)
Simple increment, don't need the value      → atomic SQL
All of the above                            → plus a CHECK constraint
```
**Optimistic vs pessimistic is a per-row-contention decision, not a codebase-wide one.**
The same system can correctly use both.

---

## Deadlocks

```
Txn A: locks account-1, wants account-2
Txn B: locks account-2, wants account-1
→ cycle. Postgres detects after deadlock_timeout (1s) and kills one.
```

Directly relevant to trade-service (two accounts per trade):
```
Trade 1 (buy):  lock EUR → then BTC
Trade 2 (sell): lock BTC → then EUR
→ deadlock
```
Also happens with ONE user trading twice concurrently on their own two accounts.

### The fix: deterministic global lock order
```java
List<UUID> ordered = Stream.of(eurAccountId, btcAccountId).sorted().toList();
ordered.forEach(id -> accounts.findByIdForUpdate(id));
```
If every transaction locks in ascending ID order, a cycle **cannot form**. No configuration,
no retry — just discipline. To be implemented in trade-service.

Supporting practices: keep transactions short; touch tables in a consistent order everywhere.

---

## Spring Transaction Pitfalls

### `@Transactional` is a proxy — self-calls silently do nothing
```java
public void outer() {
    this.credit(...);      // ← NO TRANSACTION. Bypasses the proxy.
}
@Transactional
public void credit(...) { }
```
Same for `private` and `final` methods — the proxy can't override them. No error, no transaction.

### Only unchecked exceptions roll back
```java
@Transactional
public void transfer() throws IOException {
    accounts.save(account);
    throw new IOException();      // CHECKED → COMMITS ANYWAY
}
```
Default rollback rule is `RuntimeException` and `Error` only. Override with
`@Transactional(rollbackFor = Exception.class)`.

**This is why every wallet exception extends `RuntimeException`** — not style, it's what makes
rollback happen.

### Long transactions block VACUUM
```java
@Transactional
public void process() {
    var data = repository.findAll();
    externalApi.call(data);        // ← 30s HTTP call holding the transaction open
    repository.saveAll(data);
}
```
Vacuum can't remove versions an open transaction might need → bloat for the whole duration.
**Never hold a transaction across a network call.** Read, commit, call, then write in a new one.

### `readOnly = true` is not just a hint
Hibernate skips dirty-checking, Postgres skips assigning a transaction ID, and the connection
can be routed to a read replica later.

---

## The Cost of SERIALIZABLE

Postgres implements it as **SSI** (Serializable Snapshot Isolation): tracks read/write
dependencies, detects cycles, aborts one transaction.

```
1. ANY transaction may abort → ALL code must handle retry
2. Tracking overhead grows with concurrency
3. Under contention, abort rates can be worse than pessimistic locking
4. Aborts happen at COMMIT — after all the work is done and wasted
```

**When it's right:** a genuine multi-row invariant not expressible as a constraint. Classic
case — "at least one doctor on call": two doctors each see count = 2, both remove themselves,
invariant broken with no row collision. Write skew. Only SERIALIZABLE catches it.

**For the trading platform:** READ COMMITTED + optimistic locking + CHECK constraints. Every
invariant we care about is single-row or expressible as a constraint.

---

## Summary

| Concept | Key point |
|---|---|
| READ UNCOMMITTED | Doesn't exist in Postgres |
| READ COMMITTED | Default. Snapshot per STATEMENT → two reads can differ |
| REPEATABLE READ | Snapshot per TRANSACTION; in Postgres also stops phantoms |
| SERIALIZABLE | Only level stopping write skew; mandatory retry handling |
| Lost update | Not prevented by isolation alone |
| Write skew | Read a set, write elsewhere, no collision, invariant broken |
| Optimistic | Rare conflicts → detect and retry |
| Pessimistic | Frequent conflicts → lock and wait |
| Atomic SQL | No read → no lost update by construction |
| Deadlock | Fix with deterministic global lock ordering |
| Self-call | Bypasses the proxy — no transaction, silently |
| Checked exception | Does NOT roll back without `rollbackFor` |

---

## Design Questions

**Q: trade-service debits EUR and credits BTC. Two simultaneous opposite trades. Risk and fix?**
2/10 — answered "different accounts, no risk". The risk is **deadlock**: the two legs are locked
in opposite order (buy locks EUR→BTC, sell locks BTC→EUR). Happens even for one user trading
twice concurrently on their own two accounts. Fix: sort account IDs and lock in ascending order
so a cycle cannot form.

**Q: You add a treasury account every trade touches. Why does that break ADR-002?**
0/10. It destroys the premise ("accounts are rarely contended"). 1000 trades/sec on one row →
999 optimistic-lock failures → retry storm → throughput collapse and starvation. Fixes in order:
(1) atomic SQL `balance = balance + ?` (no read, no conflict), (2) pessimistic lock (queue, don't
retry), (3) shard the treasury into N sub-accounts and sum for reporting — what high-volume
systems actually do. ADR-002's own Negative section already flags this.

---

## Concepts Introduced
- The four anomalies mapped to isolation levels, with Postgres-specific deviations
- Lost update and write skew; why isolation level alone is insufficient
- The four conflict-handling strategies and how to choose between them
- Deadlocks and deterministic lock ordering
- Spring transaction pitfalls: proxy self-calls, checked-exception rollback, long transactions
- SERIALIZABLE / SSI and its real costs
