# Snapshot + Delta Pattern
**Introduced:** Day 12

---

## What It Is

Pattern for efficiently streaming real-time state to large numbers of clients.

**Snapshot** = complete current state, sent once on connect.
**Delta** = only what changed, sent continuously after.

## Why Both Are Needed

```
Only deltas: new user has no base state to apply them to → wrong/empty data
Only snapshots: full state every 100ms × 500K users → enormous bandwidth

Solution:
  On connect  → send full snapshot (from Redis)
  After that  → stream deltas every 100ms
  Client merges deltas onto snapshot → always consistent
```

## Real-World Usage

- **Binance/Coinbase order book:** REST snapshot on connect, WebSocket deltas every 100ms
- **Google Docs:** full document on open, operational transforms (deltas) as edits happen
- **Live dashboards:** full metrics snapshot on load, incremental updates streamed

## Throttling

Don't send every individual change. Aggregate into batches:
```
10,000 order book changes/sec → batch every 100ms → 10 updates/sec per user
Binance standard: 100ms depth update intervals
```
