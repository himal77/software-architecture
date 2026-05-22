# Bitpanda Clone — Production-Grade Capstone Project

**Goal:** Portfolio piece for Senior Developer / Junior Architect roles
**Context:** Austrian fintech, builds on Bitpanda (Vienna-based)
**Scope:** Demo platform, not real exchange (no MiCA license, no real funds)

---

## Why This Project

- Exercises ~80% of the 90-day curriculum
- Demonstrates fintech-grade architecture (auth, payments, transactions, real-time data)
- Shippable result: live demo + ADRs + load test results + multi-region deployment
- Strong CV piece for senior/architect roles in Austria/EU

---

## Build Phases

### Phase 1 — MVP (4-6 weeks)
- User registration + login (JWT + refresh token, Redis)
- Mock KYC (toggle "verified")
- Mock fiat balance
- Buy/sell ONE crypto (BTC) with mock price feed
- Transaction history
- Wallet shows balance

**Stack:** Spring Boot + Postgres + Redis + Kafka + WebSockets + React/TS + Tailwind + shadcn/ui

### Phase 2 — Real-Time + Multi-Asset (3-4 weeks)
- Real price feed (CoinGecko / Binance public WebSocket)
- WebSocket push of live prices
- Multiple assets (BTC, ETH, SOL)
- Order book + simple matching engine (Redis ZSET)
- Live charts (TradingView Lightweight Charts + TimescaleDB / InfluxDB)

### Phase 3 — Financial-Grade Robustness (4-6 weeks)
- Wallet system with double-entry accounting
- Event sourcing for wallet
- Idempotency keys on all financial operations
- Audit log (Cassandra, append-only, partition by user_id)
- 2FA + email verification

### Phase 4 — Production Polish (3-4 weeks)
- Multi-region deployment (Frankfurt + Singapore)
- Observability (Prometheus + Grafana — already in user's toolkit)
- Load testing — simulate 1,000 concurrent users
- Security review — penetration testing basics
- ADRs for every major decision

**Total timeline:** 4-5 months part-time (after curriculum Day 90)

---

## Architecture Notes (Captured During Curriculum)

### Authentication
- **JWT + Refresh Token** (NOT sticky session, NOT HttpSession)
- Access token: 15 min lifetime
- Refresh token: 30 days, stored in Redis (revocable)
- Logout = delete refresh token from Redis
- 2FA on login (TOTP)
- Re-auth for high-value actions (withdrawals)

### Workflow / Session State
- Server **stateless** — no sticky session
- For multi-step workflows (KYC):
  - Solution 2 from session discussion: dedicated `workflow_drafts` table
  - JSONB column for accumulated data
  - `expires_at` for cleanup
  - Survives server crash + cross-device + audit-friendly

### Database Choices (Polyglot)
| Component | Database | Why |
|---|---|---|
| User accounts, balances, orders | Postgres | ACID required, relational |
| Wallet ledger | Postgres | Double-entry accounting needs ACID |
| Transaction history (long-term) | Cassandra | Append-only, query by user+time, scale |
| Audit log | Cassandra | Append-only, regulatory retention |
| KYC documents | Postgres metadata + S3 blob | Metadata + blob pattern |
| Hot prices, order book, sessions | Redis | In-memory, ZSET for orderbook |
| Live price history (1-min bars) | TimescaleDB / InfluxDB | Time-series optimized |
| Notifications | Kafka topic + workers | Day 3 pattern |
| Files / images | S3 + CDN | Cheap, scalable blob storage |

### Trading Workflow (Saga Pattern Required)
"Buy 1 BTC for €60,000":
1. Reserve fiat (€60K) — Wallet service `@Transactional`
2. Match in order book — Trading service
3. Transfer fiat: buyer → seller — Wallet service
4. Transfer crypto: seller → buyer — Crypto wallet service
5. Record trade for tax — Audit service

External Stripe charge → must use Saga (compensating refund if downstream fails).

### What NOT to Use
- ❌ Sticky session (not modern, can't scale globally)
- ❌ Single Postgres for everything (will break at scale)
- ❌ HttpSession for workflow state (stateful server, fragile)
- ❌ Cassandra for the wallet (need ACID)
- ❌ Polling for prices (use WebSocket push)

---

## Frontend

**Stack:**
- React + TypeScript + Vite
- Tailwind CSS + shadcn/ui (copy-paste components)
- Zustand for state
- React Hook Form + Zod for forms
- React Router
- TradingView Lightweight Charts (free)
- WebSocket connection for live prices

**Approach:** Use AI tools heavily for component generation. Modify a crypto exchange template instead of building from scratch — what matters is the architecture, not pixel-perfect custom design.

---

## Curriculum Mapping

| Curriculum Concept | Where Used in Bitpanda |
|---|---|
| Day 1: Stateless services | Backend services, no sticky session |
| Day 1: Critical vs non-critical path | Trading = critical, analytics = non-critical |
| Day 2: PENDING status pattern | KYC document upload |
| Day 2: Metadata + blob | KYC docs in S3 |
| Day 3: Notification system | Email/SMS/push for trades, alerts |
| Day 3: ADRs | Document every major decision |
| Day 4: CAP / WebSocket push | Live price ticker |
| Day 5: BASE / Redis ZSET | Order book |
| Day 6: Distributed locks | Critical operation coordination |
| Day 7: Polyglot persistence | Multiple DBs (table above) |
| Day 8: Sharding | Audit log partitioned by user_id |
| Day 9: Saga pattern | Trading workflow with Stripe |
| Day 14: Security architecture | Auth, 2FA, secrets, encryption |

---

## Important Caveats

**Don't make it real:**
- Use testnet crypto (free, fake)
- Use mock fiat balances
- Don't connect real bank APIs
- Don't take real payments

**Why:** EU MiCA regulation (effective 2024-2025) requires:
- License (~€125-350K capital)
- AML/KYC compliance infrastructure
- Banking partnerships
- Auditing

For solo dev → impossible. Build it as a **demo platform** that proves architectural skills.

---

## Success Criteria for Portfolio Use

- [ ] Live demo deployed (multi-region)
- [ ] GitHub repo with clean architecture
- [ ] ADRs documenting major decisions
- [ ] Load test results (1,000 concurrent users)
- [ ] Observability dashboards
- [ ] Architecture diagram (C4 model)
- [ ] Walkthrough video / blog post

This becomes a 30-minute architecture conversation in any interview.
