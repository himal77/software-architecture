# MVCC — Multi-Version Concurrency Control
**Introduced:** Day 21

---

## The Core Idea

Postgres **never overwrites a row on UPDATE**. It writes a new version and marks the old dead.

```
UPDATE accounts SET balance = 90 WHERE id = 1;

Row v1: balance=100, xmin=100, xmax=101   ← dead, still on disk
Row v2: balance=90,  xmin=101, xmax=null  ← live
```

Hidden system columns:
- `xmin` — transaction that created this version
- `xmax` — transaction that deleted/superseded it

Each transaction holds a snapshot and sees only versions valid for it.

## The Payoff

```
Txn A (started before the update) → sees 100
Txn B (started after)             → sees 90
Neither blocks the other.
```
**Readers never block writers; writers never block readers.**

## The Costs

1. **Bloat** — update every row 10× → table ~10× larger on disk
2. **VACUUM required** — reclaims dead tuples no live transaction can still see
3. **DELETE frees nothing immediately** — it only sets `xmax`
4. **Long transactions are dangerous** — VACUUM cannot remove versions an open transaction
   might need. An idle-in-transaction connection open for hours blocks vacuum for hours.
   A top cause of real Postgres incidents → keep transactions short.

## VACUUM vs VACUUM FULL vs pg_repack

```
VACUUM      → marks dead space reusable BY THAT TABLE. File size UNCHANGED.
VACUUM FULL → rewrites into a new file, returns space to OS. ACCESS EXCLUSIVE lock.
pg_repack   → rebuilds concurrently, only a brief lock. The production answer.
```
"We deleted 10M rows but disk didn't shrink" → expected. Normal vacuum makes space reusable,
not returned. Usually that's what you want: the table refills without growing.

## Relation to Isolation Levels (Day 5)

The isolation level is really "which snapshot rules apply":
```
READ COMMITTED  → fresh snapshot per statement → non-repeatable reads possible
REPEATABLE READ → one snapshot for whole transaction
SERIALIZABLE    → snapshot + read/write dependency tracking, abort on conflict
```
