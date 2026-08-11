# Locking Strategies
**Introduced:** Day 23

---

## Four Ways to Handle a Write Conflict

### 1. Optimistic — version column
```sql
UPDATE accounts SET balance = 70, version = 6 WHERE id = ? AND version = 5;
-- 0 rows updated → someone else won → throw, retry with fresh data
```
JPA: `@Version private long version;` — Hibernate appends the predicate automatically.
**Best when conflicts are rare.** No lock held, so no contention and no deadlocks.

### 2. Pessimistic — SELECT ... FOR UPDATE
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findByIdForUpdate(UUID id);
```
Other transactions block until commit. **Best when conflicts are frequent** (a hot row).
```
FOR UPDATE        → blocks writers and other readers-for-update
FOR NO KEY UPDATE → weaker, allows FK checks
FOR SHARE         → many readers, writers blocked
```

### 3. Atomic SQL — eliminate the read
```sql
UPDATE accounts SET balance = balance - 30 WHERE id = ? AND balance >= 30;
```
No read-then-write → **no lost update by construction.** Fastest.
Cost: the resulting value isn't available to application code.

### 4. Constraint as backstop
```sql
CHECK (balance >= 0)
```
Doesn't prevent the conflict; makes the corrupt outcome impossible.

## Choosing

```
Rare conflicts, need the resulting value   → optimistic
Frequent conflicts, hot row                → pessimistic
Simple increment, value not needed         → atomic SQL
Any of the above                           → plus a CHECK constraint
```

**This is a per-row-contention decision, not a codebase-wide one.** A system can correctly use
optimistic locking for per-user rows and atomic SQL for a shared treasury row.

## Why Optimistic Fails on a Hot Row

```
1000 trades/sec on ONE treasury row:
  all 1000 read version = 5
  one succeeds, 999 fail → retry → one succeeds, 998 fail...
  → retry storm, throughput collapse, starvation
```
Optimistic locking is fast when you rarely lose the race. On a maximally contended row you
almost always lose — paying for the work, then discarding it.

Fixes: atomic SQL, pessimistic locking, or **shard the hot row** into N sub-rows and aggregate.

## Deadlocks

```
Txn A: locks row-1, wants row-2
Txn B: locks row-2, wants row-1
→ Postgres detects the cycle after deadlock_timeout (1s), kills one
```

**Fix: acquire locks in a deterministic global order.**
```java
Stream.of(eurAccountId, btcAccountId).sorted().forEach(this::lock);
```
If every transaction agrees on the ordering, a waiting cycle cannot form. Discipline, not
configuration.

Also: keep transactions short, and touch tables in a consistent order across the codebase.
