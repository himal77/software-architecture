# Backend for Frontend (BFF) Pattern
**Introduced:** Day 16

---

## What It Is

A dedicated API layer per client type. Each BFF aggregates internal microservice calls and shapes the response for its specific client.

## Why

One API serving all clients → compromises for everyone. BFF per client → optimal for each.

```
Mobile BFF  → minimal fields, bandwidth-optimised, JWT auth
Web BFF     → rich aggregated data for dashboard
Partner BFF → stable versioned contract, API key auth
```

## How It Works

```
Client makes 1 request → BFF makes N parallel gRPC calls → combines response

CompletableFuture<Trades>  trades  = supplyAsync(() -> tradeClient.get(userId));
CompletableFuture<Wallet>  wallet  = supplyAsync(() -> walletClient.get(userId));
CompletableFuture.allOf(trades, wallet).join();
// Total time = slowest call, not sum
```

## Benefits

- Each client evolves independently
- Different auth strategies per client
- Internal services unaffected by client-specific changes
