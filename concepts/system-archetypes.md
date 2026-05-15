# The 3 System Archetypes
**Introduced:** Day 3

Recognising the dominant archetype in the first 2 minutes tells you where the hard problems live.

---

## Archetype 1: Read-Heavy Storage System
**Examples:** URL shortener, Pastebin, Google Drive, Dropbox, CDN

```
Signature:     reads >> writes, data stored and retrieved by key
Hard problems: cache strategy, storage cost, read latency, CDN
Default arch:  App → Cache → DB → Blob storage
```

---

## Archetype 2: Write-Heavy Streaming System
**Examples:** Twitter feed, logging pipeline, analytics, IoT sensors

```
Signature:     writes >> reads OR writes at massive volume
Hard problems: write throughput, durability, ordering, consumer lag
Default arch:  App → Message queue → Stream processor → DB
```

---

## Archetype 3: Compute-Heavy Processing System
**Examples:** YouTube encoding, search indexing, Uber matching, fraud detection

```
Signature:     complex computation on data, not just storage/retrieval
Hard problems: job scheduling, worker scaling, partial failure, idempotency
Default arch:  App → Job queue → Worker pool → Result store
```

---

## Real Systems Are Combinations
Twitter = Archetype 2 (writes) + Archetype 1 (reading feeds)
YouTube = Archetype 3 (encoding) + Archetype 1 (video delivery)

Identify the **dominant** archetype first, then layer secondary ones.

---

## How to Use
When given an unfamiliar system:
1. Identify which archetype dominates
2. Open the right mental model (cache strategy vs queue depth vs job scheduling)
3. Apply the 5-step framework from there
