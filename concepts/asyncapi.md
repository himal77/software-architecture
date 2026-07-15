# AsyncAPI
**Introduced:** Day 18

---

## What It Is

Open standard for documenting event-driven APIs (Kafka topics, WebSocket, AMQP).
REST equivalent: OpenAPI (Swagger). Event equivalent: AsyncAPI.

## Why

Partners integrating with your Kafka stream need to know: what topics exist, what schema, what events.
AsyncAPI generates human-readable + machine-readable docs — just like Swagger for REST.

## Example

```yaml
asyncapi: '2.6.0'
info:
  title: Bitpanda Event API
  version: '1.0.0'

channels:
  trade-events:
    subscribe:
      summary: Receive trade completion events
      message:
        payload:
          type: object
          properties:
            tradeId:  { type: string }
            userId:   { type: string }
            asset:    { type: string }
            amount:   { type: number }
            price:    { type: number }
            timestamp: { type: string, format: date-time }
```

## Bitpanda Usage

Document all Kafka topics in AsyncAPI so:
- Internal teams know what each topic publishes
- Partner integrations have a contract to code against
- Schema changes are visible and versioned
