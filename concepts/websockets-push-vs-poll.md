# WebSockets — Push vs Poll
**Introduced:** Day 4

---

## The Problem with Polling at Scale
Client asks server "any updates?" on a fixed interval.

```
500M users × poll every 5 sec = 100M req/sec → impossible
Most responses: "no update" → wasted requests
```

Polling works at small scale. Fails at consumer scale.

---

## Push Model (WebSockets / Server-Sent Events)
Server pushes data to client when something changes. Client is passive.

```
500M users connected via WebSocket
Score changes → server pushes update to all connected clients
10,000 matches × 1 update/30sec = ~333 score pushes/sec → trivial
```

---

## WebSocket vs Server-Sent Events (SSE)

| | WebSocket | SSE |
|---|---|---|
| Direction | Bidirectional | Server → client only |
| Protocol | WS/WSS | HTTP |
| Use when | Chat, gaming, collaborative editing | Live feeds, scoreboards, notifications |
| Complexity | Higher | Lower |

---

## Architecture Pattern for Global Push at Scale
```
Data change → Kafka topic
→ Fan-out service (Kafka consumer)
→ Regional WebSocket server clusters
→ Connected clients in that region
```

New client connection:
```
1. Fetch current state from CDN/Redis (snapshot, <50ms)
2. Subscribe to WebSocket for live updates from that point
```

Never poll. Separate initial load (CDN) from live updates (WebSocket).

---

## ESPN Rule
```
Polling model:  100M users × poll/5sec = 20M req/sec → impossible
Push model:     100M users × WebSocket = ~1,000 score updates/sec → trivial
```

**At consumer scale, always push. Never poll.**
