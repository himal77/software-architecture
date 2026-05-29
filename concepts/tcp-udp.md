# TCP vs UDP
**Introduced:** Day 11

---

## TCP

Guarantees delivery, order, no duplicates. Requires 3-way handshake.

```
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
(one full round trip before any data)
```

Head-of-line blocking: one lost packet blocks all subsequent packets on that connection.

Used by: HTTP, gRPC, Kafka, Redis, Postgres, databases.

## UDP

No delivery guarantee, no ordering. No handshake — send immediately. No retransmission.

Used by: DNS, video streaming, gaming, QUIC (HTTP/3).

## When to Choose

| TCP | UDP |
|---|---|
| Data correctness required | Low latency more important than reliability |
| APIs, databases, messaging | Streaming, gaming, DNS |
| Any financial transaction | Live scores, video calls |
