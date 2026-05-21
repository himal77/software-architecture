# Case Study: Kubernetes etcd
**Covered:** Day 6 | **Topic:** Raft consensus in production

---

## The Problem
Kubernetes stores all cluster state — pods, services, deployments, configmaps. Multiple control plane nodes need consistent view. If a pod is scheduled on Node A, that decision must be visible to all schedulers immediately.

---

## Architecture
- 3 or 5 node etcd cluster running Raft
- All API server reads/writes go through etcd
- Leader handles all writes
- Linearizable reads route through leader
- Followers serve reads with eventual consistency

---

## Real Production Failure Modes

| Failure | Symptom | Cause |
|---|---|---|
| Slow disk on leader | Re-election storm, cluster instability | Followers timeout waiting for log sync |
| Network partition (3 vs 2) | Minority side becomes read-only | Cannot achieve majority |
| Leader CPU saturation | Frequent leader changes | Can't process heartbeats in time |
| 2-node "HA" setup | Any single failure = entire cluster down | No majority possible |

---

## Production Best Practices
- Always 3 or 5 nodes — never 2 or 4
- Dedicated fast SSD storage (NVMe preferred)
- Low-latency networking between etcd nodes
- Run on separate hardware from application workloads
- Monitor: leader changes/sec, commit latency, disk fsync time

---

## The Hard-Won Lesson
**etcd is the heart of a Kubernetes cluster.** If etcd loses quorum, the entire cluster becomes read-only — no new pods, no scaling, no recovery actions.

---

## Diagnostic Rule
**Weird Kubernetes behavior** (pods stuck pending, controllers not reconciling, kubectl timeouts) → check etcd health first. Almost always a Raft issue underneath.

```
etcdctl endpoint health
etcdctl endpoint status --cluster -w table
```

These commands tell you: is there a leader? Are nodes in sync? Is commit latency acceptable?
