# Load Balancing
**Introduced:** Day 12

---

## Algorithms

| Algorithm | How | Best For |
|---|---|---|
| Round Robin | Cycle through servers | Equal requests, equal servers |
| Weighted Round Robin | Proportional to server capacity | Servers with different specs |
| Least Connections | Server with fewest active connections | Variable-length requests |
| IP Hash | Same IP → same server | Sticky sessions |
| Least Response Time | Lowest connections + latency | Production default (nginx/HAProxy) |

## Layer 4 vs Layer 7

**L4:** sees IP + port only. Fast. Cannot inspect HTTP content.
**L7:** sees full HTTP request. Routes by URL, headers, cookies. Used for microservice routing.

Kubernetes: Ingress = L7. Service = L4.

## Health Checks

Load balancer polls `/actuator/health` every N seconds.
Fail → remove from rotation. Pass again → re-add.

**Liveness:** app alive? Fail → restart pod.
**Readiness:** app ready for traffic? Fail → remove from LB, don't restart.
