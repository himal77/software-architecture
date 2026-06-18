# Back-pressure
**Introduced:** Day 14

---

## What It Is

Consumer signals producer to slow down when it cannot keep up. Prevents queue overflow, OOM crashes, and stale data.

## Without Back-pressure

```
Producer: 50,000/sec
Consumer: 5,000/sec
Queue fills → OOM crash OR endless backlog
```

## Kafka Back-pressure

```
Consumer slow → stops polling Kafka
Messages accumulate in Kafka partition (retained up to 7 days)
Consumer catches up when recovered
No data loss

Monitor: consumer lag = distance between latest offset and consumer offset
High lag = consumer overwhelmed
```

## gRPC Back-pressure

Built into HTTP/2 flow control. Consumer sends WINDOW_UPDATE=0 → producer pauses automatically. No code needed.

## Reactive (WebFlux)

```java
stream.onBackpressureBuffer(1000).subscribe(item -> process(item));
```

## vs Rate Limiting

| | Rate Limiting | Back-pressure |
|---|---|---|
| Direction | External → your system | Internal producer → consumer |
| Purpose | Protect from outside abuse | Protect downstream from upstream |
| Mechanism | Reject with 429 | Signal to slow down |
