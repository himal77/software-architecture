# The Trade-off Triangle
**Introduced:** Day 1

Every system design decision lives inside this triangle. You can optimize for **two, never all three.**

```
            Performance
               /\
              /  \
             /    \
            /      \
    Cost --/--------\ Reliability
```

| Optimize for | Sacrifice | Example |
|---|---|---|
| Performance + Reliability | Cost | Redundant infra, premium DBs, multi-region |
| Performance + Cost | Reliability | No redundancy, single points of failure |
| Reliability + Cost | Performance | Cheaper hardware, eventual consistency |

## How to use it
When given a system to design, first ask: **which corner matters most to this business?**

- A bank: Reliability > everything
- A startup MVP: Cost > everything
- A gaming leaderboard: Performance > everything
