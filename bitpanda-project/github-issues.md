# Bitpanda Platform — GitHub Issues

Copy each block below as a separate GitHub issue. Label all as `enhancement`.

---

## Issue 1 — Project Scaffolding & Infrastructure Setup

**Title:** `[Phase 1] Project scaffolding — mono-repo structure, Docker Compose, CI`

**Labels:** `enhancement`, `phase-1`, `infrastructure`

**Body:**

### Overview
Set up the foundational project structure for the Bitpanda platform. All microservices will live in a single mono-repo for portfolio simplicity.

### Repo Structure
```
bitpanda-platform/
├── services/
│   ├── auth-service/          Spring Boot 3 + Java 21
│   ├── wallet-service/        Spring Boot 3 + Java 21
│   ├── trade-service/         Spring Boot 3 + Java 21
│   ├── order-service/         Spring Boot 3 + Java 21
│   ├── notification-service/  Spring Boot 3 + Java 21
│   └── api-gateway/           Spring Cloud Gateway
├── frontend/                  React + TypeScript + Vite
├── infrastructure/
│   ├── docker-compose.yml
│   ├── docker-compose.dev.yml
│   └── k8s/
├── docs/
│   ├── adr/                   Architecture Decision Records
│   └── diagrams/              C4 model diagrams
└── README.md
```

### Tasks
- [ ] Initialise mono-repo with above structure
- [ ] Spring Boot parent POM with shared dependencies
- [ ] `docker-compose.yml` with: Postgres, Redis, Kafka, Zookeeper, all services
- [ ] Each service has a `Dockerfile` (multi-stage build)
- [ ] GitHub Actions CI: build + test on every PR
- [ ] `.env.example` with all required env vars documented
- [ ] Root `README.md` with architecture overview and local setup instructions

### Acceptance Criteria
- `docker-compose up` starts the entire platform locally
- All services connect to their dependencies (Postgres, Redis, Kafka)
- CI passes on a clean checkout

---

## Issue 2 — Authentication Service

**Title:** `[Phase 1] Auth service — JWT, OAuth 2.0, refresh tokens, Redis session store`

**Labels:** `enhancement`, `phase-1`, `auth`

**Body:**

### Overview
Build the authentication service handling user registration, login, and token lifecycle. Stateless JWT — no sticky sessions.

### Architecture
```
POST /auth/register   → create user, send email verification
POST /auth/login      → issue access token (15 min) + refresh token (7 days)
POST /auth/refresh    → validate refresh token → issue new access token
POST /auth/logout     → revoke refresh token from Redis
GET  /auth/me         → return current user profile
```

### Token Design
- **Access token:** JWT, 15 min, carries `userId`, `email`, `tier`
- **Refresh token:** opaque UUID, stored in Redis with TTL 7 days, sent as HttpOnly cookie
- **Logout:** delete refresh token from Redis → immediate revocation
- Gateway injects `X-User-Id` header into downstream requests after JWT validation

### Security Requirements
- Passwords: bcrypt (strength 12)
- Refresh token: stored in Redis, not in DB (fast revocation)
- `X-User-Id` header only set by gateway — never trusted from client
- Rate limit login endpoint: 5 attempts / 15 min per IP (token bucket)

### Tech Stack
- Spring Boot 3 + Spring Security
- JJWT library for JWT generation/validation
- Redis (Lettuce client) for refresh token store
- Postgres for user accounts

### Tasks
- [ ] User registration endpoint with input validation
- [ ] Login endpoint — validate credentials, issue token pair
- [ ] JWT filter in API gateway — validate signature, inject X-User-Id
- [ ] Refresh endpoint — validate Redis entry, issue new access token, rotate refresh token
- [ ] Logout — delete Redis entry
- [ ] Unit tests for token generation and validation
- [ ] Integration test: full login → refresh → logout flow

### Acceptance Criteria
- Login returns access + refresh token
- Expired access token → 401
- Refresh with valid token → new access token issued
- Logout → refresh token no longer works
- Invalid JWT signature → 401

---

## Issue 3 — Wallet Service

**Title:** `[Phase 1] Wallet service — double-entry ledger, fiat + crypto balances`

**Labels:** `enhancement`, `phase-1`, `wallet`

**Body:**

### Overview
Wallet service manages user balances using double-entry accounting. Every transaction creates two ledger entries — no balance can disappear.

### Architecture
```
GET  /wallets/{userId}              → return fiat + crypto balances
GET  /wallets/{userId}/transactions → paginated transaction history (cursor)
POST /wallets/{userId}/deposit      → mock fiat deposit (internal, admin only)
```

### Database Schema
```sql
-- Accounts table (one per user per currency)
CREATE TABLE accounts (
    id          UUID PRIMARY KEY,
    user_id     UUID NOT NULL,
    currency    VARCHAR(10) NOT NULL,  -- EUR, BTC, ETH
    balance     DECIMAL(20,8) NOT NULL DEFAULT 0,
    version     BIGINT NOT NULL DEFAULT 0,  -- optimistic lock
    created_at  TIMESTAMPTZ NOT NULL
);

-- Ledger entries (append-only, double-entry)
CREATE TABLE ledger_entries (
    id              UUID PRIMARY KEY,
    transaction_id  UUID NOT NULL,
    account_id      UUID NOT NULL,
    amount          DECIMAL(20,8) NOT NULL,  -- positive = credit, negative = debit
    balance_after   DECIMAL(20,8) NOT NULL,
    type            VARCHAR(50) NOT NULL,    -- DEPOSIT, WITHDRAWAL, TRADE_BUY, TRADE_SELL
    created_at      TIMESTAMPTZ NOT NULL
);
```

### Key Rules
- Balance updates use optimistic locking (`version` column) — prevents double-spend
- Ledger is append-only — no UPDATE or DELETE on `ledger_entries`
- Cursor pagination on transaction history (not offset — financial data grows indefinitely)
- BOLA check on every endpoint: JWT userId must match path userId

### Tasks
- [ ] Schema + Flyway migrations
- [ ] GET /wallets/{userId} with BOLA check
- [ ] Double-entry ledger write (two entries per transaction, in single DB transaction)
- [ ] Optimistic lock on balance update
- [ ] Cursor-paginated transaction history
- [ ] Unit tests for double-entry logic
- [ ] Integration test: deposit → balance increases, two ledger entries created

### Acceptance Criteria
- Balance change always creates exactly 2 ledger entries
- Concurrent balance updates do not corrupt balance (optimistic lock)
- User cannot read another user's wallet (403)
- Transaction history paginates correctly using cursor

---

## Issue 4 — Trade Service + Saga

**Title:** `[Phase 1] Trade service — buy/sell BTC, Saga pattern, idempotency`

**Labels:** `enhancement`, `phase-1`, `trading`

**Body:**

### Overview
Trade service handles buy/sell execution for a single asset (BTC). Uses Saga pattern to coordinate wallet changes — compensates on failure so no funds are lost.

### Architecture
```
POST /trades          → initiate buy or sell
GET  /trades/{id}     → get trade status
GET  /trades?userId=  → trade history (cursor paginated)
```

### Trade Flow (Saga — Choreography via Kafka)
```
1. trade-service:    validate request, create trade (PENDING), publish TradeInitiated
2. wallet-service:   reserve funds (debit from available), publish FundsReserved
3. trade-service:    execute at mock price, publish TradeExecuted
4. wallet-service:   complete transfer (credit other currency), publish FundsTransferred
5. trade-service:    mark trade COMPLETED

Compensation (any step fails):
- FundsReserved but TradeExecuted fails → publish TradeFailed → wallet-service releases reserved funds
```

### Idempotency
- Client sends `Idempotency-Key` header (UUID)
- Trade service stores `(idempotency_key, response)` in Postgres with TTL 24h
- Duplicate request → return stored response, do not re-execute

### Mock Price Feed
- Phase 1: fixed mock price (€60,000 per BTC) stored in Redis
- Phase 2: replace with CoinGecko WebSocket feed

### Tasks
- [ ] Trade entity + Flyway migration
- [ ] POST /trades with idempotency key handling
- [ ] Kafka events: TradeInitiated, FundsReserved, TradeExecuted, TradeFailed
- [ ] Wallet service Kafka consumer for fund reservation + release
- [ ] Saga compensation on failure
- [ ] Trade status polling endpoint
- [ ] Integration test: full buy flow (happy path + compensation path)

### Acceptance Criteria
- Successful trade: fiat decreases, BTC increases, trade status = COMPLETED
- Failed trade: funds returned to available balance, trade status = FAILED
- Duplicate idempotency key: returns same response, no double-execution
- Trade history paginated with cursor

---

## Issue 5 — API Gateway

**Title:** `[Phase 1] API Gateway — JWT validation, routing, rate limiting`

**Labels:** `enhancement`, `phase-1`, `infrastructure`

**Body:**

### Overview
Single entry point for all client traffic. Validates JWT, injects user context, routes to microservices, and enforces rate limits.

### Responsibilities
```
Client → API Gateway → downstream service

Gateway does:
1. JWT validation (signature + expiry)
2. Inject X-User-Id, X-User-Tier headers (trusted by downstream)
3. Route to correct microservice
4. Rate limiting (token bucket per userId)
5. Request/response logging
```

### Rate Limits
| Client tier | Limit |
|---|---|
| Free | 100 req/min |
| Premium | 1,000 req/min |
| Partner | 10,000 req/min |

Tier is read from JWT claim — no DB lookup per request.

### Tech Stack
- Spring Cloud Gateway (reactive, WebFlux)
- Redis for rate limit counter (token bucket)
- No auth logic — only validates tokens issued by auth-service

### Routing Table
```yaml
routes:
  - id: auth-service
    uri: lb://auth-service
    predicates: [Path=/auth/**]
  - id: wallet-service
    uri: lb://wallet-service
    predicates: [Path=/wallets/**]
  - id: trade-service
    uri: lb://trade-service
    predicates: [Path=/trades/**]
```

### Tasks
- [ ] Spring Cloud Gateway setup with service discovery
- [ ] JWT validation global filter
- [ ] X-User-Id header injection
- [ ] Rate limiting filter (Redis token bucket)
- [ ] Public routes whitelist (POST /auth/register, POST /auth/login)
- [ ] Request logging filter
- [ ] Integration test: valid JWT → routed; invalid JWT → 401; rate exceeded → 429

### Acceptance Criteria
- Valid JWT → request reaches downstream service with X-User-Id header
- Expired/invalid JWT → 401 at gateway, request never reaches downstream
- Rate limit exceeded → 429 with Retry-After header
- Public endpoints accessible without token

---

## Issue 6 — Real-Time Price Feed (WebSocket)

**Title:** `[Phase 2] Real-time price feed — WebSocket push, Kafka fan-out`

**Labels:** `enhancement`, `phase-2`, `real-time`

**Body:**

### Overview
Push live BTC/ETH/SOL prices to connected clients over WebSocket. Replaces the mock fixed price from Phase 1.

### Architecture
```
CoinGecko/Binance WS → price-feed-service → Kafka (price-updates topic)
                                                    ↓
                                          websocket-service (Kafka consumer)
                                                    ↓
                                          Connected clients (WS push)
```

### Price Update Flow
1. `price-feed-service` connects to Binance public WebSocket
2. On each tick → normalize → publish to Kafka topic `price-updates`
3. `websocket-service` consumes Kafka, batches updates every 100ms
4. Pushes delta update to all subscribed clients
5. New client connects → send snapshot (current prices), then deltas

### Snapshot + Delta Pattern
```
Client connects → server sends: { BTC: 62000, ETH: 3200, SOL: 145 }   ← snapshot
Every 100ms →  { BTC: 62015 }   ← delta (only changed assets)
```

### Spring Boot WebSocket Routing
- Client authenticates via JWT on WebSocket CONNECT
- userId → STOMP session mapping maintained in memory
- For private messages (trade confirmations): `convertAndSendToUser(userId, ...)`
- For broadcast (prices): `convertAndSend("/topic/prices", ...)`

### Tasks
- [ ] Binance WebSocket client (or CoinGecko polling fallback)
- [ ] Kafka producer: price-updates topic
- [ ] WebSocket server with STOMP
- [ ] JWT authentication on CONNECT handshake
- [ ] Snapshot on new connection
- [ ] 100ms batched delta broadcast
- [ ] Trade confirmation push to specific user (via Kafka consumer)

### Acceptance Criteria
- Client receives price snapshot on connect
- Price updates arrive within 200ms of source tick
- Trade confirmation delivered to correct user only
- Disconnected client receives snapshot on reconnect (no missed-state problem)

---

## Issue 7 — Observability Stack

**Title:** `[Phase 4] Observability — Prometheus, Grafana, structured logging, distributed tracing`

**Labels:** `enhancement`, `phase-4`, `observability`

**Body:**

### Overview
Production-grade observability: metrics, logs, and traces. Required for the portfolio to demonstrate operational maturity.

### Metrics (Prometheus + Grafana)
- Spring Boot Actuator + Micrometer → Prometheus scrape
- Custom metrics:
  - `trades_executed_total` (counter, by asset)
  - `trade_latency_ms` (histogram)
  - `websocket_connections_active` (gauge)
  - `kafka_consumer_lag` (from Kafka JMX)

### Dashboards (Grafana)
- Service health (CPU, memory, GC, thread pools)
- Trade volume over time
- WebSocket connection count
- Kafka consumer lag per topic
- API Gateway: request rate, p99 latency, error rate (RED method)

### Structured Logging
- Logback JSON encoder → each log line is a JSON object
- All logs include: `traceId`, `spanId`, `userId` (when available), `service`, `level`
- Log aggregation: ELK stack or Loki (Docker Compose)

### Distributed Tracing
- Spring Cloud Sleuth / Micrometer Tracing + Zipkin
- Trace spans across: Gateway → Service A → Kafka → Service B
- TraceId propagated in Kafka message headers

### Tasks
- [ ] Prometheus + Grafana in Docker Compose
- [ ] Micrometer in all services
- [ ] Custom business metrics (trade counter, latency histogram)
- [ ] Grafana dashboards (import JSON)
- [ ] Structured JSON logging with traceId
- [ ] Zipkin for distributed tracing
- [ ] Loki for log aggregation

### Acceptance Criteria
- All services visible in Grafana
- A single trade can be traced end-to-end across all services via traceId
- Alert fires if trade latency p99 > 500ms
