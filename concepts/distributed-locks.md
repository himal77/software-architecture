# Distributed Locks
**Introduced:** Day 6

---

## The Problem
Multiple nodes need exclusive access to a shared resource (process a payment, send a unique email, update a shared counter). Only one can hold the lock at a time.

---

## Why It Needs Consensus
All nodes must agree on "who currently holds lock X?" Without consensus, two nodes might both think they hold the lock — disaster.

Solution: Lock service backed by Raft. Leader serializes all acquire/release operations through replicated log. Every node sees the same sequence.

---

## TTL / Lease Pattern

```
Acquire lock with TTL = 30 seconds
Holder renews every ~10 seconds (heartbeat)
If holder crashes → no renewal → TTL expires → auto-release
```

**Critical:** TTL must be longer than expected processing time, but short enough for fast recovery.

**Why not just "release on crash"?** The lock service can't distinguish:
- Server crashed
- Server slow
- Network blip

If you release on no response, you might give the lock to someone else while original holder is still working → two holders.

---

## Race Resolution (Multiple Clients Acquire Same Lock)

```
3 clients X, Y, Z send acquire request simultaneously
→ All requests arrive at Raft leader
→ Leader serializes in its log:
   Entry 1: X acquires → SUCCESS
   Entry 2: Y acquires → REJECTED (held by X)
   Entry 3: Z acquires → REJECTED (held by X)
→ Replicated to majority → committed
→ Each client gets clear answer
```

The Raft log gives serializable ordering even for microsecond-apart requests.

---

## Real Lock Services

| Service | Algorithm | Notes |
|---|---|---|
| **etcd** | Raft | Robust, used by Kubernetes |
| **Consul** | Raft | Lock service + service discovery |
| **ZooKeeper** | Zab | Older but mature |
| **Redis Redlock** | Multiple Redis | Controversial — not safe under all failure modes |

**Rule:** Use etcd or Consul for correctness-critical locks. Redlock has known issues under specific failure scenarios.

---

## When You Need Distributed Locks
- Cron jobs that should run on only one server
- Processing unique events (payment IDs, idempotency)
- Leader election in your own application
- Coordinating writes to a shared resource

## When You Don't Need Them
- If you can use idempotency keys instead — usually preferable
- Database row locks (use Postgres `SELECT FOR UPDATE`)
- Single-node coordination (use language-level locks)
