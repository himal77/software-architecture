# System Design: Live Order Book
**Designed:** Day 12 | **Scope:** Real-time order book for 500K concurrent users

---

## Requirements

- 500,000 concurrent users watching order book
- Updates reach users within 200ms
- Order book changes ~10,000 times/second
- Users watch different instruments (BTC/EUR, ETH/USD, etc.)
- Reconnecting users must see current state immediately

---

## Key Decisions

### 1. Transport: WebSockets
Persistent bidirectional connection. Server pushes without client polling. Sub-100ms latency.

### 2. Throttle: Batch every 100ms
```
10,000 updates/sec → aggregate → 1 delta per 100ms per instrument
10 updates/sec per user (not 10,000)
```

### 3. Snapshot + Delta
```
On connect:    GET /orderbook/BTC/EUR → full snapshot from Redis
After connect: WebSocket stream → deltas every 100ms

Reconnect after 30s offline:
  → Fetch fresh snapshot from Redis (always current)
  → Resume delta stream
  → No gap, no stale data
```

### 4. Instrument-based routing
```
Kafka topic per instrument: orderbook.BTC/EUR, orderbook.ETH/USD
WebSocket server subscribes to its assigned Kafka partitions
Users watching BTC/EUR → routed to servers consuming orderbook.BTC/EUR
```

---

## Architecture

```
Order Matching Engine
        │ order placed/filled
        ↓
     Kafka
  (orderbook.BTC/EUR, orderbook.ETH/USD, ...)
        │
        ↓
WebSocket Gateway Cluster (10 servers, ~50K connections each)
  Each server:
    - Consumes assigned Kafka partitions
    - Aggregates updates every 100ms
    - Pushes delta to connected users for that instrument
        │
  Redis (current snapshot per instrument — always latest state)
        │
        ↓
500K Browser/Mobile users
  On connect:  GET snapshot from Redis (REST call)
  Streaming:   receive deltas via WebSocket
```

---

## Scaling Math

```
500K connections ÷ 50K per server = 10 WebSocket gateway servers
10K updates/sec → 100ms batching → 10 deltas/sec per user
10 deltas/sec × 500K users = 5M messages/sec total (manageable)
vs unbatched: 10K × 500K = 5B messages/sec (impossible)
```

---

## Failure Scenarios

| Failure | Recovery |
|---|---|
| User disconnects | Reconnects, fetches fresh Redis snapshot, resumes delta stream |
| WebSocket server crashes | Load balancer removes it, user reconnects to another server, gets snapshot |
| Kafka partition slow | 100ms batching absorbs spikes, no data loss (Kafka retains messages) |
| Redis snapshot stale | Order matching engine updates Redis on every fill — TTL + push pattern |
