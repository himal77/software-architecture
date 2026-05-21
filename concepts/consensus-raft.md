# Consensus & Raft
**Introduced:** Day 6

---

## What Consensus Is
Getting a group of unreliable nodes to agree on a single value despite failures (crashes, network partitions, slow nodes).

---

## Why It's Hard — FLP Impossibility (1985)
> In an asynchronous network with even one faulty node, no consensus algorithm can guarantee both safety and liveness.

- **Safety** = never produce a wrong answer
- **Liveness** = always eventually produce an answer

Real systems sacrifice liveness during partitions to preserve safety. Better to be unavailable than wrong.

---

## Raft Algorithm

Designed to be understandable. Used by etcd, Consul, CockroachDB, Kafka KRaft.

### 3 Roles
- **Follower** — passive, responds to leader
- **Candidate** — trying to become leader
- **Leader** — handles all writes, replicates to followers

### Leader Election
1. Follower hasn't heard heartbeat (150–300ms randomized timeout)
2. Becomes Candidate, increments term, votes for self
3. Asks others for votes
4. Majority votes → becomes Leader, sends heartbeats
5. Split vote → new election with new randomized timeout

### Log Replication (Write Path)
```
Client → Leader appends to log (uncommitted)
       → Replicates to followers (AppendEntries)
       → Waits for majority ACK (quorum)
       → Commits, applies to state machine
       → Returns success
```

Write committed only after **majority** has it.

---

## Why Odd Numbers (3, 5, 7)

| Nodes | Tolerates | Production use |
|---|---|---|
| 3 | 1 failure | Minimum |
| 5 | 2 failures | Sweet spot |
| 7 | 3 failures | Large clusters |

**Even numbers can split exactly in half → neither side has majority → cluster halts.**

---

## Real Systems Using Raft / Consensus

| System | Algorithm | Use case |
|---|---|---|
| etcd | Raft | Kubernetes cluster state |
| Consul | Raft | Service discovery, KV store, locks |
| CockroachDB | Raft per range | Global ACID database |
| Kafka KRaft | Raft | Replaces ZooKeeper |
| ZooKeeper | Zab (Raft-like) | Older systems, pre-KRaft Kafka |

---

## The Anchor
You depend on Raft daily without realizing:
- `kubectl apply` → Raft consensus in etcd
- Kafka leader election → KRaft consensus
- Postgres failover → consensus underneath
- Redis Sentinel failover → consensus underneath
