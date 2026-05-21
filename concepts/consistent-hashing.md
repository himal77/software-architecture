# Consistent Hashing
**Introduced:** Day 8

---

## The Problem It Solves
With naive `hash(key) % N`, adding a single node forces ~80% of keys to move to different shards. Massive rebalancing operation, hours of network traffic.

---

## The Algorithm

**Mental model:** A circle with values 0 to 2^32 - 1.

```
1. Hash each NODE's name → position on circle
   Node A → 100
   Node B → 1000
   Node C → 5000

2. Hash each KEY → position on circle
   "alice" → 250

3. Owner = first node clockwise from key's position
   "alice" at 250 → walks clockwise → hits Node B at 1000
   → Node B owns "alice"
```

---

## Why It's Genius

**Adding a node:**
```
Existing: A=100, B=1000, C=5000
Add Node D at position 500

Before: keys 100-1000 → all owned by Node B
After:  keys 100-500 → Node D
        keys 500-1000 → Node B (unchanged)

Only keys 100-500 move. Other nodes untouched.
```

**Removing a node:** Mirror — only keys owned by removed node move to next clockwise neighbor.

---

## Virtual Nodes (vnodes)

With 3 physical nodes on a circle, distribution is uneven (one node might own 60% of the circle by accident of hashing).

**Solution:** Each physical node owns MANY virtual positions.

```
Node A → 256 positions across the circle
Node B → 256 positions
Node C → 256 positions

→ Statistical distribution is even
→ When Node A fails, its 256 ranges spread to many neighbors
   (not all dumped on the next clockwise node)
```

Cassandra default: 256 vnodes per physical node. Configurable.

---

## Where It's Used

| System | Use |
|---|---|
| Cassandra | Native consistent hashing for partitions |
| DynamoDB | Underlying partition mechanism |
| Memcached client libraries | Distribute keys across cache servers |
| Akamai CDN | Distribute content across edge nodes |
| Discord | Voice server distribution |

---

## Why You Should Know It
This is one of the few algorithms architects actually need to understand at a mechanism level. It comes up in:
- Database sharding decisions
- Cache cluster design
- Load balancer key-based routing
- Service mesh routing
- CDN edge distribution

The principle: **make rebalancing cheap when nodes change.**
