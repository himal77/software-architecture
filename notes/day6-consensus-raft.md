# Day 6 — Consensus & Leader Election
**Chapter 2 | Phase 1: Foundations**
**Date:** 2026-05-22

---

## Key Concepts

### The Consensus Problem
> Get a group of unreliable nodes to agree on a single value, despite failures.

---

### Why Consensus Is Hard

**1. The network lies**
Failure looks identical to slowness. You can't tell if a node crashed or is just slow.

**2. Nodes crash and recover**
A node votes "yes" then crashes. Did it persist its vote? Will it remember after recovery?

**3. FLP Impossibility (1985)**
> In an asynchronous network with even one faulty node, no consensus algorithm can guarantee both safety and liveness.
- **Safety** = never produce a wrong answer
- **Liveness** = always eventually produce an answer
- You can have one always, not both in the worst case
- Real systems sacrifice liveness during partitions to preserve safety

---

### Where Consensus Is Used

| Problem | Why consensus needed |
|---|---|
| Leader election | Multiple nodes — must agree on one leader |
| Distributed locking | Only one client can hold a lock |
| Configuration changes | All nodes must agree simultaneously |
| Atomic broadcast | Same messages, same order, all nodes |
| Distributed transactions | All commit or none |

**Used by:**
- Kafka — ZooKeeper / KRaft for controller election
- Kubernetes — etcd (Raft) for cluster state
- CockroachDB — Raft per data range
- Postgres replication — leader election
- Redis Sentinel — leader election

---

### Raft Algorithm

Designed to be understandable. Used by etcd, Consul, CockroachDB, KRaft.

**3 Roles:**
- **Follower** — passive, responds to leader
- **Candidate** — trying to become leader (during election)
- **Leader** — handles all writes, replicates to followers

Only one leader at a time. All writes go through leader.

---

### Leader Election

**Step 1 — Heartbeat timeout (150–300ms randomized)**
```
Follower hasn't heard from leader
→ Becomes Candidate
→ Increments term number
→ Votes for itself
→ Asks others: "Vote for me as leader of term N"
```

**Step 2 — Voting**
```
Voters say YES if:
  - Haven't voted in this term yet
  - Candidate's log is at least as up-to-date as theirs
Otherwise NO
```

**Step 3 — Outcome**
```
A) Majority votes → Become leader → Send heartbeats
B) Another node won → Step down to follower
C) Split vote → New election with new randomized timeout
```

**Randomized timeout** prevents repeated split votes.

---

### Log Replication

```
Client sends write to Leader
  → Leader appends to log (uncommitted)
  → Leader sends AppendEntries to followers
  → Followers append, ACK
  → Leader waits for MAJORITY (quorum) ACK
  → Leader commits, applies to state machine
  → Leader notifies followers of commit index
  → Leader returns success to client
```

Write committed only after **majority** has it. Same quorum math as Day 5, applied to a log.

---

### Why Odd Numbers (3, 5, 7)

| Nodes | Tolerates | Reason |
|---|---|---|
| 3 | 1 failure | 2/3 = majority |
| 5 | 2 failures | 3/5 = majority |
| 7 | 3 failures | 4/7 = majority |

**Even numbers split exactly in half during partition → neither side has majority → cluster halts.**

5 nodes = production sweet spot — survives 2 failures, allows maintenance + unexpected failure.

---

### Failure Behavior

**Leader crashes:**
```
Followers stop receiving heartbeats
→ Timeout → election → new leader
→ Total downtime: ~300–600ms
→ Writes during window rejected (CP behavior)
```

**Network partition:**
```
Side with majority: elects leader, continues
Side with minority: cannot get majority → no writes
When partition heals: minority syncs from majority
```

---

## Distributed Lock Service Design

### Requirements
- 100 application servers coordinating
- Only one server holds a lock at a time
- Lock auto-releases on crash
- Survives node failures

### Key Decisions

**CP — consistency over availability**
Two clients holding same lock = disaster. Brief unavailability is acceptable.

**Raft consensus**
Leader serializes acquire/release through replicated log. All nodes see same sequence of operations.

**TTL / Lease pattern**
```
Acquire with TTL = 30 sec
Holder renews every 10 sec (heartbeat)
If holder crashes → no renewal → TTL expires → auto-release
```

Critical: TTL must be > processing time, but short enough for quick recovery.

**5 nodes, not 2**
2 nodes = split brain risk + no majority possible during partition. Always 3 or 5.

### Architecture

```
Application Servers (100)
       ↓
   Lock Service Cluster (Raft, 5 nodes)
   ┌──────────────────────────────┐
   │  Leader  + 4 Followers       │
   └──────────────────────────────┘

Acquire flow:
  Client → Leader → log entry → replicate to majority → commit → respond

Heartbeat flow:
  Client renews lock every 10s → leader updates expires_at

Release flow:
  Explicit release OR TTL expiration
```

### 3 Servers Race for Same Lock

```
X, Y, Z all send acquire to leader simultaneously
→ Leader's log serializes them:
   Entry 1: X acquires "payment-123" → SUCCESS
   Entry 2: Y acquires "payment-123" → REJECTED (held by X)
   Entry 3: Z acquires "payment-123" → REJECTED (held by X)
→ Replicated to majority → committed
→ Each client gets clear answer
```

**The serialization happens at the Raft leader's log.** Even microsecond-apart requests get a definitive order.

---

### Real Lock Services

| Service | Notes |
|---|---|
| **etcd** | Raft, used by Kubernetes, robust distributed locks |
| **Consul** | Raft, lock service + service discovery |
| **ZooKeeper** | Zab protocol (Raft-like), used by Kafka pre-KRaft |
| **Redis Redlock** | Multiple Redis instances, controversial — not safe under all failures |

**Rule:** Use etcd/Consul if correctness is critical. Redlock has known correctness issues.

---

## Kubernetes etcd Case Study

**Problem:** Store cluster state for Kubernetes. Multiple control plane nodes. Must be consistent.

**Architecture:**
- 3 or 5 node etcd cluster running Raft
- All API server reads/writes go through etcd
- Leader handles all writes
- Linearizable reads route through leader

**Real production failure modes:**
- Slow disk on leader → followers timeout → re-election storm
- Network partition isolates minority → minority becomes read-only
- Leader CPU saturated → can't process heartbeats → step down → re-election
- 2-node "HA" → any failure → no majority → entire cluster down

**Lesson:** etcd is the heart of Kubernetes. If etcd loses quorum, entire cluster becomes read-only. Always run 3 or 5 nodes on dedicated fast SSD with low-latency networking.

**Diagnostic rule:** Weird Kubernetes behavior (stuck pods, broken controllers) → check etcd health first. Almost always a Raft issue underneath.

---

## Concepts Internalized

| Concept | Status |
|---|---|
| FLP Impossibility — safety vs liveness tradeoff | ✅ |
| Raft 3 roles: Follower, Candidate, Leader | ✅ |
| Leader election: heartbeat timeout, terms, votes, randomization | ✅ |
| Log replication with quorum commit | ✅ |
| Why odd numbers — 3, 5, 7 nodes | ✅ |
| Lock service with TTL/lease pattern | ✅ |
| Split brain in 2-node setups | ✅ |
| Real systems: etcd, Consul, ZooKeeper, Redlock | ✅ |

---

## Next
**Day 7 — Chapter 2 continued:** Replication strategies — single-leader, multi-leader, leaderless. When each fits.
