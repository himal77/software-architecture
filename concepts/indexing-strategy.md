# Indexing Strategy
**Introduced:** Day 22

---

## Composite Index — Left-Prefix Rule

```sql
CREATE INDEX ON trades (user_id, asset, created_at);
```
Sorted by user_id, then asset within that, then created_at within that.
**Usable left-to-right only, no gaps.**

```sql
WHERE user_id = ?                        ✓
WHERE user_id = ? AND asset = ?          ✓
WHERE asset = ?                          ✗ skipped the leading column → seq scan
WHERE user_id = ? AND created_at > ?     ~ partial: seeks on user_id, filters created_at
```
Analogy: a phone book sorted by surname then first name can't find "all people named James."

## Column Order: Equality First, Range Last

```sql
-- Query: WHERE user_id = ? AND created_at > ? ORDER BY created_at
INDEX (user_id, created_at)   ✓ seek to user, scan a sorted range
INDEX (created_at, user_id)   ✗ scan a huge date range, filter every row
```
Once you hit a range column, everything after it is unsorted relative to your scan.

## Covering Index → Index Only Scan

```sql
CREATE INDEX ON trades (user_id, created_at) INCLUDE (amount, price);
```
If the index contains every column the query needs, the heap is never touched.
- **Key columns** → what you filter or sort by
- **INCLUDE columns** → what you only return (stored in leaves, not part of the sort key)

## Partial Index — index only the working set

```sql
CREATE INDEX ON outbox (created_at) WHERE published_at IS NULL;
```
50M-row table, ~200 unpublished → index has ~200 entries, not 50M.
- Reads: tiny index, fully in memory
- Writes: publishing removes the row from the index permanently; the other 50M never participate
- Key on what you ORDER BY, since the filtered column is constant within the index

**Rule: when a query always filters on the same narrow condition, make it a partial index.**

## Six Reasons an Index Goes Unused

1. **Function/arithmetic on the column** — `LOWER(email)`, `created_at::date`, `amount * 100`
2. **Type mismatch** — integer literal against a VARCHAR column
3. **Leading wildcard** — `LIKE '%foo'` ✗ / `LIKE 'foo%'` ✓
4. **Low selectivity** — boolean where half the rows match; seq scan is genuinely cheaper
5. **Small table** — fits in a page or two
6. **Stale statistics** — run `ANALYZE table;`

## The Cost of Every Index

```
INSERT with 5 indexes = 6 writes (heap + 5 B-trees), plus possible node splits
```
Indexes buy read speed with write throughput and disk. **An unused index is pure cost.**
```sql
SELECT relname, indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;
```
