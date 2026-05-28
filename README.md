# System Architecture — 90-Day Learning Journal

**Goal:** Transition from software engineer to system architect
**Start date:** 2026-05-11 | **Target date:** 2026-08-09
**Schedule:** 5 hours/day × 90 days = 450 hours

---

## Files

| File | Purpose |
|---|---|
| [90-day-curriculum.md](90-day-curriculum.md) | Full curriculum — all 16 chapters, progress tracker |
| [rules.md](rules.md) | How each session works — revision quiz rules, note format |
| [qa.md](qa.md) | All revision Q&A — every question, your answer, correction, score |
| [bitpanda-project/](bitpanda-project/) | Capstone project plan — production-grade fintech app for portfolio |
| [concepts/](concepts/) | One file per major concept — reference glossary |
| [notes/](notes/) | One file per day — what was taught, designed, and learned |
| [designs/](designs/) | System designs drawn during problems — full architecture docs |
| [case-studies/](case-studies/) | Real company case studies (bit.ly, Netflix, Uber, etc.) |

---

## Progress

### Phase 1 — Foundations (Days 1–20)
| Day(s) | Chapter | Note | Status |
|---|---|---|---|
| 1 | Ch.1: The Architect's Mindset | [day1-architects-mindset.md](notes/day1-architects-mindset.md) | ✅ Done |
| 2 | Ch.1: The Architect's Mindset | [day2-architects-framework.md](notes/day2-architects-framework.md) | ✅ Done |
| 3 | Ch.1: The Architect's Mindset | [day3-unknown-systems-adr.md](notes/day3-unknown-systems-adr.md) | ✅ Done |
| 4 | Ch.2: Distributed Systems Theory | [day4-distributed-systems-cap.md](notes/day4-distributed-systems-cap.md) | ✅ Done |
| 5 | Ch.2: Distributed Systems Theory | [day5-acid-base-clocks.md](notes/day5-acid-base-clocks.md) | ✅ Done |
| 6 | Ch.2: Distributed Systems Theory | [day6-consensus-raft.md](notes/day6-consensus-raft.md) | ✅ Done |
| 7 | Ch.2: Distributed Systems Theory | [day7-replication-strategies.md](notes/day7-replication-strategies.md) | ✅ Done |
| 8 | Ch.2: Distributed Systems Theory | [day8-partitioning-sharding.md](notes/day8-partitioning-sharding.md) | ✅ Done |
| 9 | Ch.2: Distributed Systems Theory | [day9-distributed-transactions.md](notes/day9-distributed-transactions.md) | ✅ Done |
| 10 | Ch.2: Distributed Systems Theory | [day10-distributed-snapshots.md](notes/day10-distributed-snapshots.md) | ✅ Done |
| 11–15 | Ch.3: Networking & Protocols | — | ⬜ |
| 16–20 | Ch.4: API Design Patterns | — | ⬜ |

### Phase 2 — Data Layer Mastery (Days 21–40)
| Day(s) | Chapter | Note | Status |
|---|---|---|---|
| 21–28 | Ch.5: Database Internals | — | ⬜ |
| 29–33 | Ch.6: Caching | — | ⬜ |
| 34–40 | Ch.7: Data Pipelines & Streaming | — | ⬜ |

### Phase 3 — Scalability Patterns (Days 41–55)
| Day(s) | Chapter | Note | Status |
|---|---|---|---|
| 41–45 | Ch.8: Scaling Strategies | — | ⬜ |
| 46–50 | Ch.9: Partitioning & Sharding | — | ⬜ |
| 51–55 | Ch.10: Advanced Patterns | — | ⬜ |

### Phase 4 — Reliability & Resilience (Days 56–68)
| Day(s) | Chapter | Note | Status |
|---|---|---|---|
| 56–62 | Ch.11: Fault Tolerance | — | ⬜ |
| 63–68 | Ch.12: Observability | — | ⬜ |

### Phase 5 — Infrastructure & Security (Days 69–78)
| Day(s) | Chapter | Note | Status |
|---|---|---|---|
| 69–73 | Ch.13: Infrastructure Patterns | — | ⬜ |
| 74–78 | Ch.14: Security Architecture | — | ⬜ |

### Phase 6 — Synthesis & Real Design (Days 79–90)
| Day(s) | Chapter | Note | Status |
|---|---|---|---|
| 79–85 | Ch.15: Classic System Designs | — | ⬜ |
| 86–90 | Ch.16: Architecture Defense | — | ⬜ |

---

## Concepts Learned

| Concept | Introduced | File |
|---|---|---|
| Architect's 4 questions | Day 1 | [architects-4-questions.md](concepts/architects-4-questions.md) |
| Trade-off triangle | Day 1 | [tradeoff-triangle.md](concepts/tradeoff-triangle.md) |
| Scale inflection points | Day 1 | [scale-inflection-points.md](concepts/scale-inflection-points.md) |
| Shard by query pattern | Day 1 | [sharding.md](concepts/sharding.md) |
| Write-behind caching | Day 1 | [caching-patterns.md](concepts/caching-patterns.md) |
| Critical vs non-critical path | Day 1 | [critical-path.md](concepts/critical-path.md) |
| Pre-generated code pool | Day 1 | [id-generation.md](concepts/id-generation.md) |
| 5-step design framework | Day 2 | [day2-architects-framework.md](notes/day2-architects-framework.md) |
| Metadata + blob storage pattern | Day 2 | [metadata-blob-pattern.md](concepts/metadata-blob-pattern.md) |
| PENDING status pattern | Day 2 | [pending-status-pattern.md](concepts/pending-status-pattern.md) |
| Thundering herd | Day 2 | [thundering-herd.md](concepts/thundering-herd.md) |
| CDN | Day 2 | [cdn.md](concepts/cdn.md) |
| 3 System Archetypes | Day 3 | [system-archetypes.md](concepts/system-archetypes.md) |
| Architecture Decision Records | Day 3 | [adr.md](concepts/adr.md) |
| Message TTL Check | Day 3 | [message-ttl.md](concepts/message-ttl.md) |
| Kafka Priority Lanes | Day 3 | [kafka-priority-lanes.md](concepts/kafka-priority-lanes.md) |
| CAP Theorem | Day 4 | [cap-theorem.md](concepts/cap-theorem.md) |
| Consistency Models | Day 4 | [consistency-models.md](concepts/consistency-models.md) |
| WebSockets — Push vs Poll | Day 4 | [websockets-push-vs-poll.md](concepts/websockets-push-vs-poll.md) |
| ACID vs BASE | Day 5 | [acid-base.md](concepts/acid-base.md) |
| Distributed Clocks | Day 5 | [distributed-clocks.md](concepts/distributed-clocks.md) |
| Redis Sorted Sets | Day 5 | [redis-sorted-sets.md](concepts/redis-sorted-sets.md) |
| Consensus & Raft | Day 6 | [consensus-raft.md](concepts/consensus-raft.md) |
| Distributed Locks | Day 6 | [distributed-locks.md](concepts/distributed-locks.md) |
| Replication Strategies | Day 7 | [replication-strategies.md](concepts/replication-strategies.md) |
| Polyglot Persistence | Day 7 | [polyglot-persistence.md](concepts/polyglot-persistence.md) |
| Partitioning Strategies | Day 8 | [partitioning-strategies.md](concepts/partitioning-strategies.md) |
| Consistent Hashing | Day 8 | [consistent-hashing.md](concepts/consistent-hashing.md) |
| Two-Phase Commit (2PC) | Day 9 | [two-phase-commit.md](concepts/two-phase-commit.md) |
| Saga Pattern | Day 9 | [saga-pattern.md](concepts/saga-pattern.md) |
| Outbox Pattern | Day 9 | [outbox-pattern.md](concepts/outbox-pattern.md) |
| Idempotency | Day 9 | [idempotency.md](concepts/idempotency.md) |
| Distributed Snapshots | Day 10 | [distributed-snapshots.md](concepts/distributed-snapshots.md) |

---

## System Designs Built

| System | Day | File | Scale |
|---|---|---|---|
| URL Shortener | Day 1 | [url-shortener.md](designs/url-shortener.md) | 100 → 1B users |
| Pastebin | Day 2 | [pastebin.md](designs/pastebin.md) | 10M DAU |
| Notification System | Day 3 | [notification-system.md](designs/notification-system.md) | 50M users |
| Live Scoreboard | Day 4 | [scoreboard.md](designs/scoreboard.md) | 500M users |
| Game Leaderboard | Day 5 | [leaderboard.md](designs/leaderboard.md) | 100M players |
| Distributed Lock Service | Day 6 | [distributed-lock-service.md](designs/distributed-lock-service.md) | 100 servers |
| E-commerce Data Layer | Day 7 | [ecommerce-data-layer.md](designs/ecommerce-data-layer.md) | 100M users |
| Messaging System | Day 8 | [messaging-system.md](designs/messaging-system.md) | 2B users |
| Bitpanda Trade Saga | Day 9 | [bitpanda-trade-saga.md](designs/bitpanda-trade-saga.md) | Fintech trade flow |
| Stock Exchange Order Matching | Day 10 | [stock-exchange-order-matching.md](designs/stock-exchange-order-matching.md) | 500K orders/sec |

---

## Case Studies

| Company | System | Day | File |
|---|---|---|---|
| bit.ly / TinyURL | URL Shortener | Day 1 | [bitly.md](case-studies/bitly.md) |
| GitHub Gist | Pastebin at scale | Day 2 | [github-gist.md](case-studies/github-gist.md) |
| Uber Eats | Notification system | Day 3 | [uber-eats-notifications.md](case-studies/uber-eats-notifications.md) |
| ESPN | Live scoreboard at scale | Day 4 | [espn-scoreboard.md](case-studies/espn-scoreboard.md) |
| Clash of Clans | Global game leaderboard | Day 5 | [clash-of-clans-leaderboard.md](case-studies/clash-of-clans-leaderboard.md) |
| Kubernetes etcd | Raft consensus in production | Day 6 | [kubernetes-etcd.md](case-studies/kubernetes-etcd.md) |
| Amazon | Polyglot persistence | Day 7 | [amazon-polyglot.md](case-studies/amazon-polyglot.md) |
| WhatsApp | Messaging at 2B-user scale | Day 8 | [whatsapp.md](case-studies/whatsapp.md) |
| Stripe | Idempotency at fintech scale | Day 9 | [stripe-idempotency.md](case-studies/stripe-idempotency.md) |
