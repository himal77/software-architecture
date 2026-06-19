# Webhooks
**Introduced:** Day 16

---

## What It Is

Server makes outbound HTTP POST to client's registered URL when an event occurs. No persistent connection needed.

```
Partner registers: POST https://bank.com/webhooks/trades
Trade completes → Bitpanda POSTs event to that URL
```

Used by: Stripe, GitHub, Twilio, most modern platforms.

## vs WebSocket

```
WebSocket:  client maintains persistent connection, server pushes over it
Webhook:    server makes outbound HTTP POST to client's URL
            no persistent connection
            client just needs an HTTP endpoint
```

## Reliability

```
Partner down → store in Kafka → retry with exponential backoff up to 24h
Kafka ensures no webhook lost even if Bitpanda itself restarts
```

## Security — HMAC Signature

```
Bitpanda: HMAC-SHA256(body, shared_secret) → X-Bitpanda-Signature header
Partner:  verify by computing same HMAC with their copy of secret

Why HMAC: signature is different for every request
Intercepting one webhook gives attacker nothing reusable
```
