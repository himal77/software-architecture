# Consistency Models
**Introduced:** Day 4

Consistency is a spectrum from strongest (most expensive) to weakest (cheapest, most available).

---

## 1. Linearizability (Strict Consistency)
Every operation appears instantaneous and globally ordered. Once a write completes, every read anywhere returns that value.

```
Write X=5 completes at time T
Read at T+1 from ANY node → always returns 5
```

- Cost: highest — requires coordination between all nodes
- Used by: etcd, Zookeeper, Google Spanner, single-node Postgres
- Use when: bank balances, distributed locks, leader election

---

## 2. Sequential Consistency
All nodes agree on the same operation order, but not necessarily matching real time.

- Cost: high
- Used by: some distributed lock managers
- Use when: order matters, but wall-clock precision doesn't

---

## 3. Causal Consistency
Causally related operations appear in correct order. Independent operations can appear in any order.

```
Post created (write A)
Reply to post (write B — causally depends on A)
→ Any node seeing B must also have seen A
→ Unrelated post C can appear before or after A/B
```

- Cost: medium
- Used by: MongoDB causal sessions, some Cassandra configs
- Use when: social feeds, collaborative editing, comment threads

---

## 4. Eventual Consistency
Given no new updates, all replicas eventually converge to the same value. No timing guarantee.

```
Write X=5 to node 1
Read from node 2 immediately → might return X=3 (old value)
Read from node 2 after 200ms → returns X=5 (converged)
```

- Cost: lowest — no coordination needed
- Used by: Cassandra, DynamoDB, DNS, Redis replication
- Use when: social likes, user profiles, scoreboards, shopping cart

---

## How to Choose

| Data relationship | Model |
|---|---|
| Must be exact right now | Linearizable |
| Cause-effect relationships between writes | Causal |
| Independent updates, "eventually same" is fine | Eventual |

**Common mistake:** Using causal consistency when eventual is sufficient. Causal adds tracking overhead — only use it when you have actual causal dependencies between writes.
