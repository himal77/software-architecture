# Day 21 — Database Internals: Storage Engines
**Chapter 5: Database Internals | Date: 2026-08-09**
**Phase 2 begins — also Day 1 of the trading-platform parallel build**

---

## Revision Quiz Results — 5.75/10

| Q | Topic | Score |
|---|---|---|
| 1 | Resilience stack (4 patterns, ordered) | 4/10 ⚠️ 3rd miss |
| 2 | Why the token is the Redis key | 3/10 |
| 3 | Database vs schema | 8/10 |
| 4 | Identical 401s (user enumeration) | 10/10 |
| 5 | Register 409 tension | 5/10 |
| 6 | TIMESTAMPTZ | 7/10 |
| 7 | bcrypt 72-byte limit | 6/10 |
| 8 | Atomic getAndDelete | 5/10 |
| 9 | Optimistic vs pessimistic locking | 6/10 |
| 10 | Double-entry ledger | 8/10 |
| 11 | KRaft / Zookeeper | 0/10 |
| 12 | DECIMAL vs DOUBLE | 7/10 |

Drop from 9.2 → 5.75 because this was the first quiz on IMPLEMENTATION detail rather than
concepts — much of it code I wrote rather than the user. Hence the switch to one-method-at-a-time.

### Corrections to drill
- **BCRT**: Bulkhead → Circuit breaker → Retry → [call] → Timeout (missed 3×)
- Refresh token carries **nothing** — opaque random bytes. Key is what you HAVE, value is what you WANT.
- Non-atomic get+delete breaks under **concurrency** (two requests in the gap), not ordering
- Optimistic (version column, detect + retry) vs Pessimistic (SELECT FOR UPDATE, lock + wait)
- TIMESTAMPTZ does NOT store the zone — it converts to UTC and stores an absolute instant
- bcrypt: input capped at 72 bytes, output always exactly 60 chars
- KRaft = Kafka runs its own Raft consensus, replacing Zookeeper (same Raft as Day 6 / etcd)
- DOUBLE cannot represent 0.1 at all — not a display issue, the stored value is wrong

---

## The Foundation: Disks Are Slow

```
CPU register       ~1 ns
RAM                ~100 ns          100× slower
SSD random read    ~100,000 ns      1,000× slower than RAM
HDD random seek    ~10,000,000 ns   100,000× slower than RAM
```
If RAM = 1 second, SSD read = 17 minutes, HDD seek = 4 months.

```
Sequential read (SSD):  ~500 MB/s
Random 4KB reads (SSD): ~50 MB/s     ← 10× worse (100× on HDD)
```

**Every design decision below descends from: minimise disk touches, prefer sequential over random.**

---

## The Page — Unit of All I/O

Databases never read "a row." They read an **8 KB page**.

```
┌────────────────────────────────────────┐
│ Page header (24 bytes)                 │  checksum, free space pointers
├────────────────────────────────────────┤
│ Item pointers → → →                    │  array of (offset, length)
├────────────────────────────────────────┤
│              free space                │
├────────────────────────────────────────┤
│ ← ← ← Row 3 │ Row 2 │ Row 1            │  tuples fill from the end
└────────────────────────────────────────┘
```

The **indirection matters**: an index points to a pointer SLOT, not a byte offset. So a row can
move within its page during cleanup without invalidating every index.

### Consequences
**A row cannot exceed one page** → **TOAST** (The Oversized-Attribute Storage Technique):
large values compressed, then chunked into a side table, main row keeps a pointer.
```sql
SELECT id, email FROM users;      -- fast, never touches TOAST
SELECT id, huge_blob FROM users;  -- slow, extra lookups to reassemble
```
This is why `SELECT *` is genuinely harmful, not just untidy.

**Row order on disk is arbitrary** — rows sit wherever there's free space. Without ORDER BY,
result order is undefined and can change after an update relocates a row.

---

## Two Ways to Organise a Table

### B-tree / heap (Postgres, MySQL, Oracle)
```
Write: find page → read from disk → modify → write back   → RANDOM I/O, slower
Read:  index lookup → jump to page → done                 → FAST, predictable
```
Fast reads, slower writes. OLTP, indexed lookups on many columns, ACID.

### LSM tree (Cassandra, RocksDB, LevelDB, ScyllaDB)
**Never modify in place. Only append.**
```
1. Write → in-memory sorted memtable            (RAM, instant)
2. Memtable full → flush as immutable SSTable   (SEQUENTIAL write)
3. Background compaction merges SSTables, discards old versions
```
```
Write: append to RAM + occasional sequential flush   → EXTREMELY FAST
Read:  check memtable, then SSTable 1, 2, 3...       → may touch MANY files, SLOWER
```
Fast writes, slower reads. Time-series, logs, event streams, audit trails.

### Comparison
| | B-tree (Postgres) | LSM (Cassandra) |
|---|---|---|
| Writes | Random I/O, slower | Sequential append, very fast |
| Reads | One index lookup | May check several SSTables |
| Space cost | Fragmentation from updates | Write amplification from compaction |
| Deletes | Mark dead, reclaim later | Write a **tombstone** |
| Best for | OLTP, reads dominate | Write-heavy, append-only |

### Why LSM deletes are strange
Immutable files can't be modified, so a delete IS a write:
```
DELETE user-123 → append TOMBSTONE
Reads see the tombstone, treat row as absent
Space reclaimed only at compaction
```
So deleting a million rows makes Cassandra temporarily **slower and larger**. This is why
Cassandra prefers TTL expiry over explicit deletes — and why the Day 8 audit log used TTL.

### The polyglot decision, now with the underlying reason
```
accounts, ledger, orders → Postgres   (B-tree, ACID, indexed reads)
audit log, event history → Cassandra  (LSM, huge write volume, append-only, TTL)
```
Not preference. Storage-engine physics.

---

## Inside a B-tree Index

```
                    ┌─────────────┐
                    │  m  │  s    │           root (always cached)
                    └──┬──┴───┬───┘
        ┌──────────────┘      └──────────┐
  ┌─────▼─────┐                   ┌──────▼────┐
  │ d │ h │ k │                   │ u │ x     │   internal nodes
  └──┬────────┘                   └───┬───────┘
┌────▼────────┐                  ┌────▼───────┐
│alice→(5,2)  │ ←──── linked ───→│victor→...  │   leaves: key → (page, slot)
│bob  →(9,1)  │      sideways    │zoe   →...  │
└─────────────┘                  └────────────┘
```

**Depth ~3–4 even for billions of rows** — huge fan-out (hundreds of keys per node):
```
1M rows → depth 3 → 3 page reads
1B rows → depth 4 → 4 page reads     ← 1000× the data, ONE extra read
```

**Always balanced** — every leaf at the same depth → predictable lookup cost.

**Leaves linked sideways** → range scans walk without returning to root. This is why B-trees
serve `>`, `<`, `BETWEEN`, `ORDER BY`, prefix `LIKE 'abc%'` — the data is SORTED.
Hash indexes can do none of that.

### What an index costs
```
INSERT → write the row + update EVERY index
5 indexes = 6 writes for one logical insert
```
Indexes trade write throughput and disk for read speed. **Unused indexes are pure cost.**

### Expression index gotcha
```sql
CREATE UNIQUE INDEX users_email_lower_idx ON users (LOWER(email));

WHERE LOWER(email) = 'a@b.com'   → uses index ✓
WHERE email = 'a@b.com'          → CANNOT use it ✗ (different expression)
```

---

## MVCC — Multi-Version Concurrency Control

**Postgres never overwrites a row on UPDATE. It writes a new version, marks the old one dead.**

```
UPDATE accounts SET balance = 90 WHERE id = 1;

Row v1: balance=100, xmin=100, xmax=101   ← dead, still on disk
Row v2: balance=90,  xmin=101, xmax=null  ← live
```
Hidden columns: `xmin` = creating transaction, `xmax` = deleting/superseding transaction.
Each transaction has a snapshot and sees only versions valid for it.

```
Txn A (started before update) → sees 100
Txn B (started after)         → sees 90
Neither blocks the other.
```
**Readers never block writers; writers never block readers.**

### The costs
1. **Table bloat** — update every row 10× → table ~10× larger on disk
2. **VACUUM is mandatory** — reclaims dead tuples no live transaction can still see
3. **DELETE frees nothing immediately** — just sets `xmax`
4. **Long transactions are dangerous** — vacuum cannot remove versions an open transaction
   might need. An idle-in-transaction connection open 6 hours blocks vacuum for 6 hours.
   Top cause of real Postgres incidents → keep transactions short, use `readOnly = true`.

### VACUUM vs VACUUM FULL (the Q2 correction)
```
VACUUM      → marks dead space reusable BY THAT TABLE. File size UNCHANGED.
              Future inserts fill holes instead of extending the file.
VACUUM FULL → rewrites table into a fresh file, returns space to the OS.
              Takes ACCESS EXCLUSIVE lock — table unusable meanwhile.
pg_repack   → production answer: rebuilds concurrently, only brief lock at swap.
```
Postgres treats disk as its own to manage, not something to hand back.

### Connecting to Day 5 isolation levels
```
READ COMMITTED  → fresh snapshot per statement → non-repeatable reads possible
REPEATABLE READ → one snapshot for whole transaction → consistent view
SERIALIZABLE    → snapshot + read/write dependency tracking, abort on conflict
```
The isolation level is really "which snapshot rules apply."

---

## WAL — Write-Ahead Log

Commit → power fails 1ms later → why isn't data lost?

```
COMMIT sequence:
  1. Append change records to WAL
  2. fsync() WAL to physical disk       ← THE durability point
  3. Report success to client
  4. Modify actual data pages... later, lazily, batched
```

**Why it works: sequential beats random.**
```
WAL:        sequential append      ~500 MB/s
Data pages: random scattered I/O   ~50 MB/s
```
The durability-critical write is sequential → fast commits. Slow random page writes happen
afterwards, off the critical path.

**Crash recovery** replays WAL from the last checkpoint: committed-but-unwritten changes
applied, uncommitted discarded. That's the D in ACID, mechanically.

### Same log, reused for everything
```
Streaming replication  → ship WAL to replica, replay      → Day 7 replication
Point-in-time recovery → replay WAL to a chosen timestamp
Logical decoding (CDC) → decode WAL into row events → Debezium → Kafka
```
**The Day 9 outbox pattern can be implemented by reading the WAL** instead of polling a table.
Debezium tails Postgres WAL → publishes to Kafka → at-least-once delivery of every commit.

### The tuning knob
```
synchronous_commit = on   → fsync WAL before ack. Safe, slower. (default)
synchronous_commit = off  → ack immediately. Faster, crash loses last ~200ms of commits.
```
Trading platform: **on, always.** Losing an acknowledged trade is unacceptable.

---

## Summary

| Concept | The one thing to remember |
|---|---|
| Disk latency | Random I/O 10–100× worse than sequential. Everything follows. |
| Page (8 KB) | The unit of I/O. Never "a row." |
| TOAST | Oversized values move to a side table → `SELECT *` really costs |
| B-tree storage | Fast reads, slower random writes → OLTP → Postgres |
| LSM tree | Sequential appends, fast writes, slower reads → Cassandra |
| B-tree index | Depth 3–4 even for billions; sorted so ranges work; a write per index |
| MVCC | New version per update; readers never block writers; needs VACUUM |
| WAL | Sequential log fsync'd at commit = durability; also replication + CDC |

---

## Design Questions

**Q: Audit log, 50K writes/sec, append-only, rarely queried, 7-year retention. B-tree or LSM?**
LSM/Cassandra (9/10). Writes hit an in-memory memtable then flush as SEQUENTIAL SSTables —
50K/sec sequential appends is easy; B-tree would need 50K random page modifications per second.
"Rarely queried" means LSM's weaker read path costs nothing. TTL handles 7-year expiry without
a delete storm.

**Q: "We deleted 10M rows but disk usage didn't drop."**
Dead tuples remain until VACUUM (8/10) — and even after VACUUM the file won't shrink. VACUUM
makes space reusable by that table; only VACUUM FULL (exclusive lock) or pg_repack (online)
returns space to the OS. Usually you want the default: the table refills without growing.

---

## Concepts Introduced
- Storage hierarchy and the sequential-vs-random I/O gap
- Pages, item pointers, TOAST
- B-tree/heap vs LSM tree storage engines; tombstones
- B-tree index structure, fan-out, leaf linking, index write cost
- MVCC, xmin/xmax, bloat, VACUUM vs VACUUM FULL vs pg_repack
- WAL, fsync durability, crash recovery, logical decoding/CDC
