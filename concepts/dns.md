# DNS — Domain Name System
**Introduced:** Day 11

---

## What It Does

Translates service names (payment-service) into IP addresses (10.0.0.8).

## Resolution Chain

```
1. Local cache (JVM / OS)
2. Cluster DNS — CoreDNS in Kubernetes
3. Recursive resolver → root → TLD → authoritative nameserver
4. Returns IP + TTL
```

## Kubernetes

```
payment-service → payment-service.default.svc.cluster.local
```

CoreDNS runs as a pod, handles all internal service discovery automatically.

## Architect Concern

**JVM caches DNS forever by default.** In Kubernetes where pod IPs change constantly, this causes stale connections after pod restarts.

Fix: `networkaddress.cache.ttl=30` in JVM options.

TTL tradeoff: short = faster failover, more queries. Long = fewer queries, slower failover.
