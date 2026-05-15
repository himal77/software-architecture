# Sharding
**Introduced:** Day 1 | Deepened: Chapter 9 (Days 46–50)

---

## The Core Rule
**Shard by the key you query on most, not the key that feels logical.**

Always ask: *"What query hits this shard?"* If the shard key isn't in the query, you do a scatter-gather across all shards — which defeats the purpose.

---

## Example: URL Shortener

**Wrong:** Shard by userID
- Redirect query: `SELECT url WHERE short_code = 'xK9mP2'`
- No userID in this query → must query all shards → scatter-gather

**Correct:** Shard by short_code
- Hash the short_code → determine shard → direct single-shard query

---

## Sharding Strategies (preview — deepened in Chapter 9)

| Strategy | How | Best for |
|---|---|---|
| Hash sharding | `hash(key) % num_shards` | Even distribution, no range queries |
| Range sharding | Key ranges per shard (A–M, N–Z) | Range queries, but hotspot risk |
| Directory-based | Lookup table maps key → shard | Flexible, but lookup table is a bottleneck |

---

## Hotspot Problem
If one shard gets disproportionate traffic (e.g. a celebrity's data on one shard), that shard becomes the bottleneck regardless of total cluster capacity. Solution: virtual nodes, consistent hashing (Chapter 9).
