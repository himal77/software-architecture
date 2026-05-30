# Service Discovery
**Introduced:** Day 13

---

## What It Does

Allows services to find each other dynamically when pod IPs change constantly.

## Two Models

**Client-side:** service asks registry for list of healthy instances, picks one directly. Examples: Eureka, Consul.

**Server-side:** service calls DNS name, load balancer/proxy asks registry internally and forwards. Service never knows pod IPs. Examples: Kubernetes Service + CoreDNS.

## In Kubernetes

```
Service A: http://wallet-service:8080/balance
CoreDNS resolves "wallet-service" → stable ClusterIP
Kubernetes Service routes to healthy pod
```

**Service registry = etcd** (Raft-based). CoreDNS reads from etcd.

Never hardcode pod IPs. Always use service names. Kubernetes Service provides a stable virtual IP that never changes even as pods restart.
