# HTTP/1.1 vs HTTP/2 vs HTTP/3
**Introduced:** Day 11

---

## HTTP/1.1

- One request at a time per connection
- Text protocol
- Head-of-line blocking at application level
- Workaround: browsers open 6 parallel connections per domain

## HTTP/2

- Multiplexing: many requests over one connection in parallel
- Binary protocol — faster parsing
- Header compression (HPACK)
- TCP head-of-line blocking still exists (transport layer)
- Used by gRPC

## HTTP/3

- Built on QUIC (UDP-based)
- Independent streams — lost packet in stream 1 does NOT block stream 2
- 0-1 round trips before data (0-RTT for known servers)
- True elimination of head-of-line blocking

## Round Trip Comparison

```
HTTP/1.1 + TLS 1.2: 3 round trips before first data byte
HTTP/2   + TLS 1.3: 2 round trips
HTTP/3   + QUIC:    1 round trip (0-RTT for returning clients)
```

## When to Use

| Protocol | Use Case |
|---|---|
| HTTP/1.1 | Legacy systems, simple internal tools |
| HTTP/2 | gRPC internal microservice calls, modern APIs |
| HTTP/3 | Client-facing APIs, CDN, mobile (high packet loss networks) |
