# Query Planner & EXPLAIN
**Introduced:** Day 22

---

## EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'trader@example.com';
```
- `EXPLAIN` → intended plan + estimates only
- `EXPLAIN ANALYZE` → actually executes, reports real times and row counts

```
Seq Scan on users  (cost=0.00..1.05 rows=1 width=180)
                   (actual time=0.015..0.018 rows=1 loops=1)
  Filter: ((email)::text = 'trader@example.com'::text)
  Rows Removed by Filter: 4
```

| Field | Meaning |
|---|---|
| `cost=start..total` | Planner's estimate in arbitrary units (not ms) |
| `rows=` | ESTIMATED row count |
| `actual ... rows=` | REAL row count |
| `Rows Removed by Filter` | Rows read then discarded — wasted I/O |

**Most useful habit: compare estimated `rows` vs actual `rows`.** A large divergence means the
statistics are stale, so the planner will keep picking bad plans. Fix with `ANALYZE table;`

## Scan Types (cheapest → most expensive)

```
Index Only Scan   → answered entirely from the index, heap never read
Index Scan        → index gives locations → fetch heap pages
Bitmap Heap Scan  → many matches: collect page numbers, sort them, read sequentially
Seq Scan          → read the entire table
```

**Bitmap Heap Scan** turns many random heap lookups into sequential reads by sorting page
numbers before fetching. It's the planner's answer to "many rows match, scattered across pages."

## The Planner Is Cost-Based, Not Rule-Based

It estimates the cost of every viable plan and picks the cheapest. Consequences:
- **Creating an index does not force its use.**
- **A Seq Scan is often correct** — small table, or a query returning a large fraction of rows.
  Thousands of random index lookups can cost more than one sequential pass.

If you believe the planner is wrong, the cause is usually stale statistics or a query written so
the index can't apply (see [[indexing-strategy]]).
