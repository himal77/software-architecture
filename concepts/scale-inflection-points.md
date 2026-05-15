# Scale Inflection Points
**Introduced:** Day 1

Scale doesn't change systems linearly — it creates sudden cliff edges.

---

## Inflection 1: Single machine → need to think about state

**Problem:** You add a second app instance. Users randomly lose sessions because they hit different servers.

**Solution:** Externalize all state. Redis for sessions. No local in-memory state.

**Rule:** Design stateless from day 1, even with one server.

---

## Inflection 2: App scales, DB doesn't

**Problem:** App instances scale horizontally easily. Postgres does not. Writes queue up, read replicas lag, table scans lock everything.

**Solution:** Read replicas → connection pooling → caching → eventually sharding.

**Rule:** The database is almost always the first real bottleneck. Plan for it.

---

## Inflection 3: Single region → distributed system problems

**Problem:** Multi-region means clocks are not synchronized, network partitions happen, you cannot have strong consistency AND high availability simultaneously.

**Solution:** Understand CAP theorem (Chapter 2). Design for the failure modes of your specific system.

**Rule:** Every distributed systems problem exists because of this inflection point.

---

## Summary

| Scale | Architecture | First bottleneck |
|---|---|---|
| 100 users | 1 app + 1 DB | Nothing yet |
| 1M users | LB + N apps + cache + DB | DB reads |
| 1B users | CDN + LB + N apps + cache cluster + sharded DB | DB writes, hot partitions |
