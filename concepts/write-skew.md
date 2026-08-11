# Write Skew
**Introduced:** Day 23

---

## What It Is

Two transactions each read a **set** of rows, make a decision, then write **somewhere else**.
No two transactions touch the same row, so no conflict is detectable — yet the combined result
violates an invariant.

## The Classic Example

Rule: at least one doctor must remain on call.

```
Currently on call: Alice, Bob (count = 2)

Txn A (Alice):  SELECT COUNT(*) WHERE on_call = true   → 2, fine, I can leave
Txn B (Bob):    SELECT COUNT(*) WHERE on_call = true   → 2, fine, I can leave
Txn A:          UPDATE doctors SET on_call = false WHERE name = 'Alice'
Txn B:          UPDATE doctors SET on_call = false WHERE name = 'Bob'

Both commit. Nobody is on call. Different rows written → no conflict detected.
```

## Why Isolation Levels Don't Catch It

```
READ COMMITTED   → both reads were of committed data. Legal.
REPEATABLE READ  → catches concurrent updates to the SAME row. These are different rows.
SERIALIZABLE     → catches it: tracks read/write DEPENDENCIES, not row collisions.
```

Postgres REPEATABLE READ raises `could not serialize access due to concurrent update` only when
two transactions write the same row. Write skew slips through because the writes don't collide.

## The Ledger Version

```
A: SELECT SUM(amount) FROM ledger_entries WHERE account_id = X   → 100
B: SELECT SUM(amount) FROM ledger_entries WHERE account_id = X   → 100
A: INSERT ledger_entry (-30)     ← new row
B: INSERT ledger_entry (-50)     ← new row
→ both succeed. Balance decided from a stale aggregate.
```

## Fixes

```
1. SERIALIZABLE isolation      → correct, but every transaction may abort → retry everywhere
2. Materialise the invariant   → keep a balance column with CHECK (balance >= 0), so the
                                 constraint enforces what the aggregate cannot
3. Explicit lock               → SELECT ... FOR UPDATE on a row representing the set
```

Option 2 is why the wallet keeps a cached `accounts.balance` with a `CHECK` rather than deriving
the balance from `SUM(amount)` on every write. The denormalised column turns a multi-row
invariant into a single-row one, which a constraint *can* enforce.

## The Takeaway

**Isolation levels protect against reading wrong data. They do not automatically protect an
invariant spanning multiple rows.** That requires SERIALIZABLE, an explicit lock, or restructuring
the invariant so a constraint can enforce it.
