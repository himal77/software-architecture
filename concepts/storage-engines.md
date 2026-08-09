# Storage Engines — B-tree vs LSM Tree
**Introduced:** Day 21

---

## The Root Cause: Random vs Sequential I/O

```
Sequential read (SSD):  ~500 MB/s
Random 4KB reads (SSD): ~50 MB/s     ← 10× worse (100× on HDD)
```
Every storage-engine design is an answer to "how do I avoid random I/O?"

## B-tree / Heap (Postgres, MySQL, Oracle)

Rows live in a heap; updates modify pages in place. Indexes are B-trees pointing into it.
```
Write: find page → read → modify → write back   → RANDOM I/O, slower
Read:  index lookup → jump to page              → FAST
```
**Fast reads, slower writes.** OLTP, many indexed columns, ACID transactions.

## LSM Tree (Cassandra, RocksDB, LevelDB, ScyllaDB)

Never modifies in place. Only appends.
```
1. Write → in-memory sorted memtable          (instant)
2. Memtable full → flush as immutable SSTable (SEQUENTIAL write)
3. Background compaction merges SSTables
```
```
Write: RAM + occasional sequential flush   → EXTREMELY FAST
Read:  memtable, then SSTable 1, 2, 3...   → SLOWER
```
**Fast writes, slower reads.** Time-series, logs, events, audit trails.

## Tombstones — why LSM deletes are odd

Immutable files can't be edited, so a DELETE is itself a write: append a tombstone.
Reads see the tombstone and treat the row as absent; space is reclaimed only at compaction.

Consequence: **deleting a million rows makes Cassandra temporarily slower and larger.**
Prefer TTL-based expiry over explicit deletes.

## Choosing

| | B-tree | LSM |
|---|---|---|
| Writes | Random, slower | Sequential, very fast |
| Reads | One index lookup | May check several SSTables |
| Deletes | Mark dead, vacuum later | Tombstone + compaction |
| Use for | OLTP, reads dominate | Write-heavy, append-only |

```
accounts, ledger, orders → Postgres  (B-tree)
audit log, events        → Cassandra (LSM)
```
This is physics, not preference — it's the reason behind the Day 7 polyglot persistence choice.
