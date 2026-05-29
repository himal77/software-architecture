# WebSockets vs SSE vs Long Polling
**Introduced:** Day 11

---

## The Problem

HTTP request-response cannot push to clients — the server has no open channel. The client must always initiate. For real-time notifications (trade confirmations, live scores, chat), you need the server to push.

---

## WebSockets

Persistent bidirectional connection. Server pushes any time without a client request.

```
Client opens connection → stays open
Server: "TradeCompleted: +1 BTC" → client receives instantly
Client can also send over same connection
```

Latency: sub-100ms typical.
Best for: trade notifications, chat, live collaboration, order book updates.

---

## Server-Sent Events (SSE)

Persistent connection, server → client only (one-way).
Simpler than WebSockets. Works over HTTP/2.

Best for: live scores, read-only notification streams.

---

## Long Polling

Client sends request → server holds it open until an event occurs → responds → client immediately re-polls.

Works everywhere, no special protocol. Higher latency and overhead than WebSockets.
Best for: fallback when WebSockets unavailable.

---

## Choosing

| Need | Choice |
|---|---|
| Bidirectional real-time | WebSockets |
| Server-only push, simple | SSE |
| Maximum compatibility, latency not critical | Long Polling |

**gRPC is NOT for client-facing push.** gRPC is for internal service-to-service calls. Browsers cannot initiate gRPC calls natively.
