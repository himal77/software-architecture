# 90-Day System Architecture Curriculum

**Goal:** Transition from software engineer to system architect  
**Schedule:** 5 hours/day × 90 days = 450 hours  
**Start date:** 2026-05-11  
**Target date:** 2026-08-09

---

## Daily Schedule

| Block | Time | Activity |
|---|---|---|
| Morning | 1.5h | Theory — concept + mental model |
| Mid-morning | 1h | Deep dive — tradeoffs, failure cases |
| Afternoon | 1.5h | Design problem — build, then get critiqued |
| Evening | 1h | Real-world case study (Netflix, Uber, etc.) |

---

## Progress Tracker

| Chapter | Topic | Days | Status |
|---|---|---|---|
| 1 | The Architect's Mindset | 1–3 | 🟡 In progress (Day 1 done) |
| 2 | Distributed Systems Theory | 4–10 | ⬜ Not started |
| 3 | Networking & Protocols | 11–15 | ⬜ Not started |
| 4 | API Design Patterns | 16–20 | ⬜ Not started |
| 5 | Database Internals | 21–28 | ⬜ Not started |
| 6 | Caching | 29–33 | ⬜ Not started |
| 7 | Data Pipelines & Streaming | 34–40 | ⬜ Not started |
| 8 | Scaling Strategies | 41–45 | ⬜ Not started |
| 9 | Partitioning & Sharding | 46–50 | ⬜ Not started |
| 10 | Advanced Patterns | 51–55 | ⬜ Not started |
| 11 | Fault Tolerance | 56–62 | ⬜ Not started |
| 12 | Observability | 63–68 | ⬜ Not started |
| 13 | Infrastructure Patterns | 69–73 | ⬜ Not started |
| 14 | Security Architecture | 74–78 | ⬜ Not started |
| 15 | Classic System Designs | 79–85 | ⬜ Not started |
| 16 | Architecture Defense | 86–90 | ⬜ Not started |

---

## Phase 1 — Foundations (Days 1–20)

> **Goal:** Build the mental framework architects use to *think*

---

### Chapter 1: The Architect's Mindset (Days 1–3)

**Topics:**
- What architects actually do vs engineers
- How to frame any problem: scale, consistency, latency, cost
- The 4 questions to ask before any design
  - Who uses it?
  - How much load?
  - What breaks?
  - What's the budget?

**Problem:** Design a system for 100 users. Now 1M. Now 1B. What changes each time?

---

### Chapter 2: Distributed Systems Theory (Days 4–10)

**Topics:**
- CAP theorem — what it actually means in practice (not just the triangle)
- Consistency models: strong, eventual, causal, linearizable
- ACID vs BASE — when to choose each
- Clocks in distributed systems: wall clock, logical clock, vector clock
- Consensus: why it's hard, Paxos/Raft simplified

**Daily Problem:** Pick a real system, identify its consistency model and justify it.

---

### Chapter 3: Networking & Protocols (Days 11–15)

**Topics:**
- TCP vs UDP — when UDP wins
- HTTP/1.1 vs HTTP/2 vs HTTP/3 — head-of-line blocking, multiplexing
- DNS — how it works, TTL tradeoffs, GeoDNS
- Load balancer types: L4 vs L7, algorithms (round robin, least connections, consistent hash)
- CDN internals — edge caches, origin pull vs push

**Problem:** Design the network layer for a video streaming service.

---

### Chapter 4: API Design Patterns (Days 16–20)

**Topics:**
- REST vs gRPC vs GraphQL — tradeoffs table
- Pagination patterns: offset, cursor, keyset — why offset breaks at scale
- Versioning strategies
- Idempotency — why it's non-negotiable in distributed systems
- Rate limiting algorithms: token bucket, leaky bucket, sliding window

**Problem:** Design a public API for a payment system — versioning, rate limiting, auth.

---

## Phase 2 — Data Layer Mastery (Days 21–40)

> **Goal:** Know every storage technology and when each wins

---

### Chapter 5: Database Internals (Days 21–28)

**Topics:**
- How indexes work: B-Tree vs LSM Tree — read vs write tradeoffs
- Query planning — why your query is slow and how to fix it
- Replication: single-leader, multi-leader, leaderless
- Sharding strategies: range, hash, directory-based — hotspot problem
- Read replicas — lag, use cases, pitfalls
- NoSQL landscape:
  - Document → MongoDB
  - Wide-column → Cassandra
  - Key-value → Redis
  - Graph → Neo4j
- NewSQL: CockroachDB, Spanner — distributed ACID

**Daily Problem:** Given a data access pattern, pick the right database and justify.

---

### Chapter 6: Caching (Days 29–33)

**Topics:**
- Cache levels: client, CDN, reverse proxy, app, DB
- Eviction policies: LRU, LFU, TTL — when each fits
- Cache patterns: cache-aside, write-through, write-behind, read-through
- Cache invalidation — the hardest problem in CS, concretely
- Thundering herd, cache stampede — solutions
- Redis internals: single-threaded, data structures, persistence modes

**Problem:** Design a caching strategy for a social media feed.

---

### Chapter 7: Data Pipelines & Streaming (Days 34–40)

**Topics:**
- Batch vs stream processing — when each fits
- Kafka architecture: partitions, consumer groups, offsets, compaction — *the why behind the what*
- Exactly-once semantics — how Kafka achieves it, the tradeoffs
- Change Data Capture (CDC): Debezium, outbox pattern
- Stream processing: Flink vs Kafka Streams — stateful vs stateless
- Lambda vs Kappa architecture

**Problem:** Design a real-time analytics system for 10M events/sec.

---

## Phase 3 — Scalability Patterns (Days 41–55)

> **Goal:** Know every pattern to scale any component

---

### Chapter 8: Scaling Strategies (Days 41–45)

**Topics:**
- Vertical vs horizontal — ceiling and floor of each
- Stateless vs stateful services — why stateless is the default
- Service discovery: client-side vs server-side, Consul, K8s DNS
- Sidecar pattern, service mesh (Istio) — when it's worth the complexity

**Problem:** Your Spring Boot monolith is hitting limits. Design the decomposition.

---

### Chapter 9: Partitioning & Sharding (Days 46–50)

**Topics:**
- Consistent hashing — virtual nodes, rebalancing
- Hotspot detection and mitigation
- Cross-shard queries — why they're painful, how to avoid them
- Tenant isolation in multi-tenant systems

**Problem:** Design the sharding strategy for a messaging system (WhatsApp-scale).

---

### Chapter 10: Advanced Patterns (Days 51–55)

**Topics:**
- CQRS — read model vs write model, when it's overkill
- Event sourcing — audit log as first-class citizen, replay, snapshots
- Saga pattern — choreography vs orchestration, compensating transactions
- Outbox pattern — guaranteed event delivery without 2PC

**Problem:** Design an order management system with distributed transactions.

---

## Phase 4 — Reliability & Resilience (Days 56–68)

> **Goal:** Design systems that survive failure

---

### Chapter 11: Fault Tolerance (Days 56–62)

**Topics:**
- Failure mode analysis: what can fail, probability, impact
- Bulkhead pattern — isolating failures
- Circuit breaker — states, thresholds, half-open
- Retry strategies: exponential backoff, jitter — why naive retry kills systems
- Timeout design — the forgotten reliability tool
- Graceful degradation vs full failure
- Chaos engineering: inject failures, measure blast radius

**Problem:** Your payment service calls 5 downstream services. Design for resilience.

---

### Chapter 12: Observability (Days 63–68)

**Topics:**
- The three pillars: logs, metrics, traces
- Structured logging — why free-text logs are an anti-pattern at scale
- Metrics: counters, gauges, histograms — RED method, USE method
- Distributed tracing: trace ID propagation, sampling strategies
- SLI, SLO, SLA — how to define and measure reliability
- Alerting philosophy — alert on symptoms, not causes

**Problem:** Design the observability stack for a microservices system.

---

## Phase 5 — Infrastructure & Security (Days 69–78)

> **Goal:** Architect the platform, not just the application

---

### Chapter 13: Infrastructure Patterns (Days 69–73)

**Topics:**
- Multi-region architecture: active-active vs active-passive
- Data residency and replication lag across regions
- Disaster recovery: RPO vs RTO — concrete numbers
- Traffic management: failover, canary, blue-green at infrastructure level

**Problem:** Design a globally available system with <100ms latency anywhere.

---

### Chapter 14: Security Architecture (Days 74–78)

**Topics:**
- Zero trust architecture — never trust, always verify
- AuthN vs AuthZ: OAuth2, JWT, RBAC, ABAC
- Secret management: Vault, K8s secrets, rotation
- Encryption at rest vs in transit — where each applies
- DDoS protection layers

**Problem:** Design the auth system for a B2B SaaS platform.

---

## Phase 6 — Synthesis & Real Design (Days 79–90)

> **Goal:** Design any system, defend every choice

---

### Chapter 15: Classic System Designs (Days 79–85)

One full design per day — theory in the morning, design in afternoon, case study in evening.

| Day | System | Key Challenge |
|---|---|---|
| 79 | URL Shortener | Sharding, redirect latency |
| 80 | Twitter/X Feed | Fan-out on write vs read, celebrity problem |
| 81 | Uber/Lyft | Geospatial indexing, real-time matching |
| 82 | YouTube | Video encoding pipeline, CDN strategy |
| 83 | WhatsApp | Message ordering, presence, E2E encryption |
| 84 | Google Search | Crawling, indexing, ranking at scale |
| 85 | Stripe Payments | Exactly-once, ledger design, compliance |

---

### Chapter 16: Architecture Defense (Days 86–90)

| Day | Activity |
|---|---|
| 86–87 | Review a system you designed at work — tear it apart |
| 88 | Design a system from scratch, write a full ADR |
| 89 | Peer review simulation — every decision gets challenged |
| 90 | Design an unknown system, cold, in 45 minutes |

---

## How Each Chapter Works

1. **Theory** — concept taught anchored to your Spring/Kafka/K8s background
2. **Design problem** — you build a solution
3. **Critique** — your solution gets stress-tested, not just corrected
4. **Case study** — how Netflix/Uber/etc. solved the same problem

---

## Start

Say **"Start Chapter 1"** to begin Day 1.
