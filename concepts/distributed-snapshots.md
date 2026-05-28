# Distributed Snapshots
**Introduced:** Day 10

---

## The Problem

Capturing a consistent global state of a distributed system without pausing it.

Messages in flight during a naive snapshot cause inconsistency — money disappears, events are lost, state is corrupted.

---

## Chandy-Lamport Algorithm

Uses a MARKER message to divide time into before/after snapshot — no global pause, no clock synchronization.

**Three rules:**
1. Initiator records own state, sends MARKER on all outgoing channels, starts recording incoming
2. First MARKER received → record own state, send MARKER forward, record other incoming channels
3. Subsequent MARKER on same channel → stop recording that channel (state = messages recorded since step 2)

**Result:** node states + in-flight messages = consistent global snapshot

---

## Production Usage

- **Apache Flink:** checkpoint barriers (= MARKERs) injected into Kafka streams every few seconds → exactly-once processing guarantees
- **Kubernetes etcd:** cluster state snapshots for disaster recovery

---

## Why It Matters

Without consistent snapshots:
- Backup/restore corrupts data
- Stream processors cannot guarantee exactly-once
- Disaster recovery loses in-flight work

With Chandy-Lamport: take snapshots of live systems safely, at any scale.
