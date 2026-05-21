# System Design: Distributed Lock Service
**Designed:** Day 6 | **Scale target:** 100 application servers coordinating

---

## Requirements

### Functional
- Multiple servers coordinate access to shared resources
- Only one server holds a given lock at a time
- Lock auto-releases if holder crashes
- Acquire/release reliably

### Non-Functional
| NFR | Value |
|---|---|
| CAP choice | CP — correctness over availability |
| Lock acquisition latency | <100ms |
| Failover time | <500ms |
| Survives | 2 simultaneous node failures |

---

## Key Design Decisions

### CP — not AP
Two clients holding same lock = disaster. Brief unavailability during failover is acceptable.

### Raft consensus
- Leader serializes acquire/release through replicated log
- All followers see the same sequence
- Quorum (majority) commit before acknowledging

### TTL/Lease pattern for crash recovery
```
Acquire with TTL = 30 sec
Holder renews every 10 sec
No renewal = TTL expires = auto-release
```

### 5 nodes, not 2 or 4
- 2 nodes → split brain risk + no majority possible
- Even numbers split exactly in half during partition
- 5 nodes → tolerates 2 failures, allows maintenance + unexpected failure

---

## Architecture

```
Application Servers (100)
       ↓
   Lock Service Cluster (Raft, 5 nodes on dedicated SSD)
   ┌──────────────────────────────────┐
   │  Leader  + 4 Followers           │
   │  Replicated log of operations    │
   └──────────────────────────────────┘
```

### Acquire Flow
```
Client → Leader
  → Append "acquire lock-X by client-Y, TTL=30s" to log
  → Replicate to followers
  → Quorum (3/5) ACK
  → Commit, apply to state machine
  → Return SUCCESS to client
```

### Heartbeat Flow
```
Every 10 sec: client → Leader
  → Append "renew lock-X by client-Y" to log
  → Update expires_at = now + 30s
  → Replicate, commit
```

### Release Flow
```
Explicit release OR TTL expiration
  → Append "release lock-X" to log
  → Replicate, commit
  → Lock available for next acquirer
```

---

## Race Condition Handling

3 clients race for same lock:
```
X, Y, Z all send acquire simultaneously
→ Requests reach leader
→ Leader's log serializes them in arrival order
→ Entry 1: X acquires → SUCCESS
→ Entry 2: Y acquires → REJECTED (held by X)
→ Entry 3: Z acquires → REJECTED (held by X)
→ Replicated to majority → committed
→ Each client gets definitive answer
```

---

## Failure Scenarios

| Failure | Impact | Recovery |
|---|---|---|
| Leader crashes | Brief unavailability (~300–600ms) | Followers timeout, election, new leader |
| Follower crashes | None — quorum still possible | Sync from leader on restart |
| Network partition (3 vs 2) | Minority side read-only | Heals when partition resolves |
| Client holding lock crashes | Lock held until TTL expires | Auto-release, next acquirer succeeds |
| Slow disk on leader | Replication slows, may trigger re-election | Move to faster disk |

---

## Lessons (from etcd production experience)
- Always 3 or 5 nodes — never 2 or 4
- Dedicated fast SSD storage — slow disk causes re-election storms
- Low-latency network between nodes — latency directly affects commit time
- Run on separate hardware from application — etcd/lock service is critical infrastructure
