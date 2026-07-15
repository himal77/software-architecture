# CloudEvents
**Introduced:** Day 18

---

## What It Is

CNCF open standard for event envelope structure. Solves inconsistent schemas across services.
Used by: Google Cloud, Azure Event Grid, Knative.

## Structure

```json
{
  "specversion": "1.0",
  "id":          "evt-abc123",
  "source":      "/bitpanda/trade-service",
  "type":        "com.bitpanda.trade.completed",
  "time":        "2026-07-15T10:00:00Z",
  "datacontenttype": "application/json",
  "data": { ...payload... }
}
```

| Field | Purpose |
|---|---|
| `id` | Unique — used for deduplication |
| `source` | Which service produced it |
| `type` | Reverse-DNS naming convention |
| `time` | ISO 8601 |
| `data` | Actual payload |

## Why

Without standard: each service invents its own envelope → inconsistent logging, tracing, deduplication.
With CloudEvents: one parser, one logging pattern, one deduplication strategy across all topics.
