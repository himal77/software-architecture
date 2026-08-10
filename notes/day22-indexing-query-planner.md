# Day 22 — Indexing Strategy & the Query Planner
**Chapter 5: Database Internals | Date: 2026-08-10**

---

## Revision Quiz Results — 4.9/10

| Q | Topic | Score |
|---|---|---|
| 1 | Resilience stack BCRT | 10/10 ⬆️ **GAP CLOSED** (4th attempt) |
| 2 | 8 KB page read | 8/10 |
| 3 | 5 indexes → insert cost | 7/10 — it's 6 writes, not 2 |
| 4 | MVCC xmin/xmax | 8/10 — missed column names; dead space IS reclaimed by VACUUM |
| 5 | Why MVCC beats locking | 8/10 — readers see last COMMITTED (stale), never dirty |
| 6 | Cassandra tombstones | 0/10 |
| 7 | B-tree depth / fan-out | 5/10 — ~4 reads for 1B rows, due to ~500-key fan-out |
| 8 | Expression index not used | 0/10 |
| 9 | synchronous_commit | 3/10 — conflated durability with concurrency |
| 10 | Other WAL uses | 0/10 |

### The one conceptual fix that matters most
```
MVCC → CONCURRENCY (who sees what, simultaneously)
WAL  → DURABILITY  (surviving a crash)
```
Two independent systems that both run during an UPDATE. Merged them in Q4 and Q9.

### Found a real bug in the project via Q8
`UserRepository.findByEmail` generates `WHERE email = ?`, which **cannot use** the
`LOWER(email)` unique index → sequential scan on every login. Correct results, wrong
performance. Needs an explicit `@Query` using `LOWER(email)`.

---

## EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'trader@example.com';
```
- `EXPLAIN` → intended plan + estimates
- `EXPLAIN ANALYZE` → actually runs it, reports REAL times and row counts. Always use this.

```
Seq Scan on users  (cost=0.00..1.05 rows=1 width=180)
                   (actual time=0.015..0.018 rows=1 loops=1)
  Filter: ((email)::text = 'trader@example.com'::text)
  Rows Removed by Filter: 4
```
```
Seq Scan               → no index used, reading every row
cost=0.00..1.05        → estimated startup..total, arbitrary planner units
rows=1                 → ESTIMATED
actual ... rows=1      → REAL
Rows Removed by Filter → rows read then discarded = waste
```

**Most useful habit: compare estimated `rows` to actual `rows`.** Big divergence means bad
statistics, which means the planner will keep choosing bad plans.

---

## Scan Types (cheapest → most expensive)

```
Index Only Scan   → answered entirely from the index, heap never touched. Fastest.
Index Scan        → index gives locations → fetch heap pages
Bitmap Heap Scan  → many matches: collect page numbers, sort, read in physical order
Seq Scan          → read the whole table
```

**Bitmap scan** is the middle ground: converts many random heap lookups into sequential reads
by sorting page numbers first.

**A Seq Scan is not always a bug.** Small table, or a query returning ~40% of rows → scanning
beats thousands of random index lookups. The planner is **cost-based, not rule-based** — adding
an index does not force its use.

---

## Composite Indexes — the Left-Prefix Rule

```sql
CREATE INDEX ON trades (user_id, asset, created_at);
```
Sorted by user_id, then asset within that, then created_at within that. Like a phone book
sorted by surname, then first name.

**Usable left-to-right only, no gaps:**
```sql
WHERE user_id = ?                                    ✓ full use
WHERE user_id = ? AND asset = ?                      ✓ full use
WHERE user_id = ? AND asset = ? AND created_at > ?   ✓ full use
WHERE asset = ?                                      ✗ skipped user_id → seq scan
WHERE asset = ? AND created_at > ?                   ✗ skipped user_id
WHERE user_id = ? AND created_at > ?                 ~ PARTIAL: seeks on user_id,
                                                       then filters created_at
```
Why: finding `asset='BTC'` would require looking inside every user_id group — the sort order
gives no help. Same as finding everyone named "James" in a surname-sorted phone book.

### Column order rule
```
1. Equality columns first (=)
2. Range columns last (>, <, BETWEEN)
```
```sql
-- Query: WHERE user_id = ? AND created_at > ? ORDER BY created_at
INDEX (user_id, created_at)   ✓ jump to user, scan a sorted range
INDEX (created_at, user_id)   ✗ scan a huge date range, filter every row
```

**One composite index often replaces several single-column indexes** → fewer indexes, faster writes.

---

## Covering Indexes → Index Only Scan

```sql
CREATE INDEX ON trades (user_id, created_at) INCLUDE (amount, price);

SELECT amount, price FROM trades WHERE user_id = ? AND created_at > ?;
→ Index Only Scan — heap never touched
```
`INCLUDE` columns live in leaf pages but are NOT part of the sort key, so they don't bloat the
tree or affect ordering.
- **Key** → columns you filter or sort by
- **INCLUDE** → columns you only return

Order-of-magnitude win on hot read paths, at the cost of a fatter index.

---

## Partial Indexes

```sql
CREATE INDEX ON trades (user_id) WHERE status = 'PENDING';
```
```
Full index:    10,000,000 entries
Partial index:     50,000 entries      ← 200× smaller
```
Smaller index → fits in memory, shallower tree, AND updates to non-matching rows don't touch
the index at all → cheaper writes too.

### The outbox case (Day 9 pattern)
```sql
-- 50M rows, ~200 unpublished at any moment
CREATE INDEX ON outbox (created_at) WHERE published_at IS NULL;
→ ~200 entries instead of 50,000,000 — 250,000× smaller
```
```
READS  → 200-entry index entirely in memory, depth 1
WRITES → publishing REMOVES the row from the index, never re-added.
         The 50M published rows never participate. A plain index would be
         updated for all 50M.
DISK   → kilobytes instead of gigabytes
```
Note the key is `created_at`, not `published_at` — every row in the index is already NULL, so
you index what you **ORDER BY**, making `ORDER BY created_at LIMIT 100` a straight index walk.

**Rule: when a query always filters on the same narrow condition, put that condition in a
partial index and key it on what you sort/range-scan by.**

---

## Six Reasons the Planner Ignores Your Index

**1. Function or arithmetic wrapping the column**
```sql
WHERE LOWER(email) = 'a@b.com'         ✗ can't use index on email
WHERE created_at::date = '2026-08-09'  ✗
WHERE amount * 100 > 5000              ✗
-- Fix: leave the column bare
WHERE created_at >= '2026-08-09' AND created_at < '2026-08-10'   ✓
```
**2. Type mismatch** — `WHERE user_id = 12345` on a VARCHAR column → implicit cast → unusable
**3. Leading wildcard** — `LIKE '%@gmail.com'` ✗, `LIKE 'trader%'` ✓
**4. Low selectivity** — boolean where 50% are true → seq scan is genuinely cheaper
**5. Small table** — 5 rows fit in one page
**6. Stale statistics** — run `ANALYZE table;` (autovacuum usually handles it, but not after a bulk load)

---

## Applied to the Wallet Ledger

Expected queries:
```
① user's transaction history, newest first, cursor-paginated
② all entries for one transaction (verify double-entry sums to zero)
③ current balance for one account
```
```sql
-- ① equality on account, range on time, DESC matches the ORDER BY → no sort step
CREATE INDEX idx_ledger_account_created ON ledger_entries (account_id, created_at DESC);

-- ② verify a transaction balances
CREATE INDEX idx_ledger_transaction ON ledger_entries (transaction_id);
```

### Why cursor pagination is O(1) — the mechanical reason
```sql
SELECT * FROM ledger_entries
WHERE account_id = ? AND created_at < ?    -- ? = cursor
ORDER BY created_at DESC LIMIT 20;
```
Index gives the exact start point → read 20 sorted entries → stop.
`OFFSET 100000` would walk 100,020 index entries to return 20.

---

## Summary

| Concept | Key point |
|---|---|
| EXPLAIN ANALYZE | Always compare estimated vs actual rows |
| Scan types | Index Only < Index < Bitmap < Seq |
| Seq scan | Not always a bug — right for small tables / low selectivity |
| Composite index | Left-prefix only, no gaps |
| Column order | Equality first, range last |
| Covering index | INCLUDE returned columns → Index Only Scan |
| Partial index | WHERE clause → far smaller, cheaper writes too |
| Index ignored | Function-wrapped column, type mismatch, leading %, low selectivity, small table, stale stats |

---

## Design Questions

**Q: `INDEX (user_id, asset, created_at)` — which queries can use it?** 10/10
```
a) WHERE user_id = ? AND created_at > ?   ✓ partial (user_id seek, created_at filtered)
b) WHERE asset = 'BTC'                    ✗ no left prefix
c) WHERE user_id = ? AND asset = 'BTC'    ✓ full (contiguous prefix)
```

**Q: Outbox, 50M rows, ~200 unpublished. What index?** 3/10 — answered `user_id`; the point
was the **partial index** `(created_at) WHERE published_at IS NULL`, 250,000× smaller and
excluded from writes once a row is published.

---

## Concepts Introduced
- EXPLAIN / EXPLAIN ANALYZE, estimated vs actual rows
- Scan types: Index Only, Index, Bitmap Heap, Seq
- Cost-based (not rule-based) planning
- Composite indexes, left-prefix rule, equality-before-range column ordering
- Covering indexes with INCLUDE → index-only scans
- Partial indexes for narrow working sets
- The six reasons an index goes unused
