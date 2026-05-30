# gRPC
**Introduced:** Day 12

---

## What It Is

Remote procedure call framework built on HTTP/2 + Protocol Buffers. Service A calls Service B like a local method call. Type-safe, binary, fast.

## Protocol Buffers vs JSON

```
JSON:     52 bytes, string parsing, no type safety
Protobuf: ~15 bytes, binary, strongly typed, compile-time safety
          3-5x smaller, 5-10x faster serialization
```

## Four Communication Patterns

1. **Unary** — one request, one response (most common)
2. **Server streaming** — one request, stream of responses (live prices)
3. **Client streaming** — stream of requests, one response (batch orders)
4. **Bidirectional streaming** — both sides stream simultaneously (trading terminal)

## gRPC vs REST

| | gRPC | REST |
|---|---|---|
| Use for | Internal services | External/public APIs |
| Payload | Protobuf (binary) | JSON (human-readable) |
| Browser | No (needs grpc-web) | Yes |
| Streaming | Built-in | Limited |

## Rule for Bitpanda

- Service → Service: gRPC + Protobuf
- Client → API Gateway: REST or WebSocket
- Server → Browser push: WebSocket + JSON (browsers can't parse Protobuf natively)

## Spring Boot

```java
@GrpcClient("wallet-service")
private WalletServiceGrpc.WalletServiceBlockingStub walletStub;

BalanceResponse balance = walletStub.getBalance(
    BalanceRequest.newBuilder().setUserId(userId).build()
);
```
