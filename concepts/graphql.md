# GraphQL
**Introduced:** Day 16

---

## What It Solves

REST over-fetching (too many fields) and under-fetching (multiple requests needed). Client specifies exactly what it needs in one query.

## Three Operations

- **Query** — read data
- **Mutation** — write data  
- **Subscription** — real-time push (WebSocket under the hood)

## N+1 Problem

```
query { trades { user { name } } }
Naive: 1 query for trades + N queries for users = N+1 total
Fix: DataLoader batches all user IDs → 1 query
Always use DataLoader for relationship queries.
```

## When to Use

| GraphQL | REST |
|---|---|
| Mobile apps (bandwidth-sensitive) | Public/partner APIs |
| Complex dashboards (many data sources) | Simple CRUD |
| Rapidly evolving APIs | Heavy caching needs |

## Bitpanda

- External/partner API → REST (industry standard)
- Mobile internal → GraphQL worth considering
- Service-to-service → gRPC (never GraphQL)
