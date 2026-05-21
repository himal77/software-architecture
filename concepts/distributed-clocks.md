# Distributed Clocks
**Introduced:** Day 5

---

## The 3 Clock Problems

**1. Clock Skew** — servers show different times simultaneously
- NTP corrects to ~100ms — not good enough for ms-level ordering

**2. Clock Drift** — clock runs fast/slow, diverges over time
- 200ms after 24h, 1.4sec after 1 week without correction

**3. Time Goes Backwards** — NTP correction moves clock backward
- Events appear to happen in wrong order

**Rule: Never use wall clock timestamps for event ordering in distributed systems.**

In code: `System.nanoTime()` (monotonic, for durations) vs `System.currentTimeMillis()` (wall clock, display only).

---

## Lamport Timestamps (Logical Clocks)

Key insight: you only need to know WHAT HAPPENED BEFORE WHAT, not when.

```
Node A: event → counter=1 → sends message with counter=1
Node B: receives → max(own, received) + 1 = counter=2
→ Event at counter=1 provably happened before counter=2
```

**Kafka uses this:** offsets are logical clocks. Offset 1042 before 1043. No timestamps needed.

---

## Vector Clocks

Extends Lamport to detect concurrent events (no causal relationship).

Each node tracks a counter per node: `[A_count, B_count, C_count]`

```
[1,1,0] vs [0,2,0] → neither vector dominates the other
→ These events are CONCURRENT → conflict detected
```

Used by: DynamoDB, Riak for shopping cart conflict detection.

---

## Comparison

| Clock | Tracks | Problem solved |
|---|---|---|
| Wall clock | Real time | Display only — unreliable for ordering |
| Lamport | Causal order | A happened before B |
| Vector | Causal order per node | Detects concurrent events / conflicts |
