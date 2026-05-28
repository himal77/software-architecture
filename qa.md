# Revision Q&A Log

All revision questions asked at the start of each day, with answers and corrections.

---

## Day 2 Revision — Covering Day 1 Content

---

**Q1. What are the 4 questions an architect asks before drawing anything?**

Your answer: They should know what is more important in CAP theorem. Also the architecture should know if the service is read heavy or write heavy.

Correct answer:
1. **Who uses it, and how?** — read/write ratio, sync/async, internal/consumer-facing
2. **What are the scale numbers?** — DAU, requests/sec, data volume
3. **What breaks, and what's the cost?** — defines RPO and RTO
4. **What are the constraints?** — budget, timeline, compliance, existing stack

Result: Partial — read/write ratio is part of Q1. CAP theorem is applied after the 4 questions, not one of them.

---

**Q2. You're designing a system. The client says "make it fast and cheap." What does the trade-off triangle tell you about what you're sacrificing?**

Your answer: Partition would be neglected, as the system will be built without thinking about performance.

Correct answer: Fast = Performance, Cheap = Cost. You can only pick two corners → **Reliability is sacrificed.** No redundancy, single points of failure, system goes down when something breaks.

Result: Incorrect — confused trade-off triangle with CAP theorem. These are separate tools.

---

**Q3. Your Spring Boot app works fine with one instance. You add a second instance for redundancy and users randomly lose their sessions. What's the root cause and how do you fix it?**

Your answer: The system is stateful, storing all data in-memory. Fix: use Redis as shared cache.

Correct answer: Root cause — sessions stored in local memory, load balancer routes to different instance which has no session. Fix — externalize sessions to Redis. Both instances read/write from same Redis. Design stateless from day 1.

Result: Correct ✅

---

**Q4. A URL shortener has a read/write ratio of 10,000:1. What's your first optimization before thinking about sharding — and why?**

Your answer: Use only one database, store data in cache like Redis as it's read-heavy, system will get instant data from cache.

Correct answer: Cache first, shard second. Redis cache handles ~95% of redirect traffic. DB only hit on cache misses. At 1M users/day, cache likely solves the problem without sharding. Cache buys 100× scale before sharding is needed.

Result: Correct ✅

---

**Q5. Why is sharding a URL shortener by userID wrong? What's the correct shard key?**

Your answer: Sharding by userID may not balance the shard. Sharding by short URL is better as it's the most used data.

Correct answer: The reason is not balance — it's that the primary query (`SELECT url WHERE short_code = ?`) contains no userID. Sharding by userID means you can't route to the correct shard without querying all shards (scatter-gather). Correct shard key: short_code — because that's what you query on.

Rule: Shard by the key you query on, not by logical ownership.

Result: Partial — right shard key, wrong reason.

---

**Q6. Name the 3 approaches to generating unique short codes. Which one fits best when codes expire and get recycled?**

Your answer: Generate all short codes at once and flag as used/unused. Generate on the fly. Generate using time + location. First one fits best.

Correct answer:
1. **Hash-based** — `base62(MD5(url)).first(6)` — deterministic, no coordination needed
2. **Centralized counter** — `Redis INCR` → base62 encode — atomic, sequential
3. **Pre-generated pool** — offline job fills pool, app picks a code — best for expiry/recycling

Best fit for expiry/recycling: pre-generated pool — uniqueness verified offline, not on critical path.

Result: Partial — right best fit, wrong names for the other two approaches.

---

**Q7. You use Redis to count clicks per link. It's 11:58pm and Redis crashes. What data do you lose, and how do you mitigate it?**

Your answer: Flush to DB every 5 min to shrink loss window. Use read Redis + write Redis and sync them.

Correct answer: Flush every 5 min — correct, shrinks loss window from hours to minutes. Read/write Redis split — wrong, doesn't solve crash data loss. Real mitigation: **Redis AOF persistence** — appends every operation to disk log, replays on restart. You lose seconds of data, not hours.

Result: Partial — right instinct on flush frequency, wrong mitigation for crash recovery.

---

**Q8. Why is flushing analytics to the DB every 5 minutes better than once per day?**

Your answer: Less data loss if crash, 5-min flush won't hamper performance.

Correct answer: Three reasons:
1. **Data loss window** — 5 min max vs 23h58min max
2. **DB performance** — 288 writes/day vs millions
3. **Dashboard freshness** — near real-time analytics vs yesterday's data only

Result: Correct ✅ (missed dashboard freshness)

---

**Q9. What is the critical path in a URL shortener? What is the non-critical path?**

Your answer: Short code is the critical path. Clicks are non-critical.

Correct answer: Critical path = operations the user is waiting on: `short_code lookup → fetch original URL → redirect`. Must be <10ms, served from Redis. Non-critical path = recording click analytics. User is already gone, can lag seconds or minutes.

Rule: Ask "is the user waiting for this?" Yes → critical. No → defer/batch/async.

Result: Partial — right components, wrong framing. Critical path is an operation, not a data element.

---

**Q10. bit.ly started with MySQL and later moved hot redirect data to Cassandra. Why Cassandra specifically for redirects?**

Your answer: bit.ly is not a heavy business website, key-value would be enough for their needs.

Correct answer: bit.ly moved to Cassandra *because* traffic became extremely high — MySQL was struggling. Reasons Cassandra fits:
- Optimized for single-key lookups at massive scale
- Linear horizontal scaling — add nodes, capacity grows
- Handles millions of reads/sec with low latency
- No complex queries needed — Cassandra's weaknesses don't matter

Result: Partial — right pattern, backwards reasoning.

---

**Q11. You have 3 app instances generating short codes simultaneously. Why does the hash-based approach fail when the same URL is used by two different marketing campaigns?**

Your answer: Due to clash of the short key, only 1 app will generate the code, 2 will fail.

Correct answer: The opposite — no app fails. Hash is deterministic: same URL always produces same code. Both campaigns get identical short code → cannot track clicks separately → analytics merged → useless for marketing. This is a business failure, not a technical failure.

Result: Incorrect.

---

**Q12. What are RPO and RTO? Give a concrete example of each.**

Your answer: Not sure.

Correct answer:
- **RPO (Recovery Point Objective)** — how much data loss is acceptable. Example: payment system RPO = 0 (cannot lose any transaction). URL shortener analytics RPO = 5 min (losing 5 min of click counts is fine).
- **RTO (Recovery Time Objective)** — how fast must you recover. Example: payment system RTO = seconds (automatic failover). Internal reporting dashboard RTO = 4 hours (manual fix acceptable).

Result: Not answered — taught as new content.

---

## Day 3 Revision — Covering Day 1 + Day 2 Content

---

**Q1. What are the 5 steps you follow in order before writing a single line of architecture?**

Your answer: Budget/timeline, metrics DAU/MAU req/s, who are the users, data loss tolerance, reliability vs cost vs performance.

Correct answer:
1. Clarify requirements (functional + non-functional) — 2 min
2. Estimate scale — 3 min
3. Define data model — 5 min
4. High-level design — 10 min
5. Deep dive bottlenecks — remaining time

Your answers map to inputs inside Step 1 and Step 2 only. Steps 3, 4, 5 were missing.

Result: Partial.

---

**Q2. What is the difference between a functional and non-functional requirement? Give one example of each for a ride-sharing app like Uber.**

Your answer: Functional = business need. Non-functional = technical need.
Functional example: User should be able to find nearest available taxi ✅
Non-functional example: Use Google Maps API, send via Kafka, taxi driver gets notification ❌

Correct answer: Non-functional describes measurable behavior, not implementation.
Correct NFR examples: Match driver in <3 sec. 99.99% uptime. One driver per ride (consistency). 10M rides/day (scale).

Rule: If you're naming a technology in an NFR, you've skipped ahead to Step 4.

Result: Partial — functional correct, non-functional had solutions not requirements.

---

**Q3. A Pastebin has 10M DAU and a 5:1 read/write ratio. What is the peak reads/sec and peak writes/sec? Show your working.**

Your answer: Peak reads ~480 req/sec (split users 5:1). Peak writes ~18 req/sec.

Correct answer:
```
10M DAU, each user: 1 write/day, 5 reads/day
Writes/day:  10M × 1 = 10M → avg 116/sec → peak 350/sec
Reads/day:   10M × 5 = 50M → avg 580/sec → peak 1,750/sec
Peak = average × 3
```

Mistake: 5:1 is actions per user, not types of users. Also missing peak multiplier (×3).

Result: Partial — wrong approach and missing peak multiplier.

---

**Q4. Why should you never store large content (like a 10MB paste) directly in Postgres? Where should it go instead?**

Your answer: Only metadata in Postgres. Large content in S3/blob storage. Postgres is for small structured data, not large content.

Correct answer: Correct. Additional reasons not mentioned:
1. Query performance — Postgres reads entire rows into memory, large blobs pollute buffer cache
2. Cost — Postgres storage is 10–50× more expensive than S3 per GB

Result: Correct ✅

---

**Q5. You're writing a paste to Postgres and S3. Postgres succeeds, S3 fails. What pattern prevents a broken paste being shown to users?**

Your answer: S3 first, then Postgres. If Postgres fails, cleanup via Postgres metadata. Not business critical so data loss doesn't matter.

Correct answer: S3-first has a flaw — if Postgres write fails after S3 succeeds, user loses their content silently with no recovery path.

Correct pattern — PENDING status:
1. Write metadata → Postgres (status = PENDING)
2. Upload content → S3
3. Update → Postgres (status = ACTIVE)
4. Cache → Redis
Only ACTIVE pastes shown. Background job retries PENDING records.

Result: Partial — S3-first has silent data loss flaw. PENDING status is the correct pattern.

---

**Q6. What is a thundering herd? Give a concrete scenario and the solution.**

Your answer: Sudden spike in traffic reaching system capacity limit. Blog goes viral, 100–1000× traffic spike.

Correct answer: Thundering herd is specifically about simultaneous cache misses on the same key — not a general traffic spike.
```
10,000 users click viral link simultaneously
→ All hit Redis → all miss (cold cache)
→ All 10,000 query Postgres for same row
→ DB gets 10,000 identical queries → crashes
```
Solution: Cache mutex — first request acquires lock, fetches DB, populates cache. Others wait and read from cache. Result: 1 DB query instead of 10,000.

Result: Partial — described traffic spike, not the cache miss mechanism.

---

**Q7. What does CDN stand for and what problem does it solve? When should you NOT use a CDN?**

Your answer: Content Delivery Network. Reduces access time. Content cached at nearest edge node. Heavy/frequently requested content only moved.

Correct answer: Correct. When NOT to use CDN:
- Private content (CDN caches publicly)
- Highly dynamic/real-time data
- User-specific personalized responses
- Low-traffic content (high miss rate, pay CDN cost with little benefit)

Result: Correct ✅

---

**Q8. What is the difference between RPO and RTO? For a payment system, what values would you set?**

Your answer: RPO = data loss tolerance. RTO = recovery time tolerance. Both near zero for payments.

Correct answer: ✅ Both definitions correct. Both near-zero for payments.
- RPO = 0 → synchronous replication, write confirmed only after all replicas acknowledge
- RTO = seconds → automatic failover, standby promoted with no human intervention

Full forms: RPO = Recovery Point Objective. RTO = Recovery Time Objective.

Result: Correct ✅

---

**Q9. Back-of-envelope: 500M DAU, 3 writes/day, 50 reads/day. Peak writes/sec and reads/sec?**

Your answer: Peak writes ~52,000/sec ✅. Peak reads ~87,000/sec ❌

Correct answer:
```
Writes: 500M × 3 = 1.5B/day → 17,361 avg → 52,000 peak ✅
Reads:  500M × 50 = 25B/day → 289,352 avg → 868,000 peak
```

Mistake: Used 25M instead of 25B for reads — dropped 3 zeros.

Rule: Always write out full multiplication before dividing by 86,400.

Result: Partial — writes correct, reads 10× off.

---

**Q10. Redis has 30-min TTL on 1M keys all set at 2am. What problem occurs at 2:30am and how do you fix it?**

Your answer: All keys expire simultaneously → cache miss on all → if users spike, all hit DB → DB crashes. Fix: reload cache when count drops, or use TTL jitter (30 ± 5 sec).

Correct answer: ✅ This is a cache avalanche (different from thundering herd — many different keys, not one cold key).
Jitter range too small — 5 sec spreads 1M expirations over 10 sec = 100K/sec still a spike.
Better: `30min ± random(0–10min)` → spreads over 20-min window → ~833 expirations/sec.

Result: Correct ✅ (jitter range too small)

---

**Q11. What is the metadata + blob storage pattern? Name two real companies that use it.**

Your answer: Metadata = info about content. Content in blob/S3. Two separate storage systems. Pastebin and GitHub use it.

Correct answer: ✅ Correct pattern and companies.
Pastebin: Postgres stores paste_id, s3_key, metadata (~200B). S3 stores text content (up to 10MB).
GitHub Gist: MySQL stores gist_id, owner, metadata. Git objects store actual code content.
Why: Performance + cost + scalability.

Result: Correct ✅

---

**Q12. A stakeholder says "make it 99.999% available." What two questions do you ask back?**

Your answer: What is the budget? What is RPO?

Correct answer:
1. ✅ "What is the budget?" — five nines costs 10× more than 99.9%. Stakeholders often don't know this.
2. "Which components need this?" — usually only the critical path needs five nines. Building the whole system to five nines wastes money.

RPO is a separate requirement, not a question about availability.

Availability reference:
- 99% = 3.65 days downtime/year
- 99.9% = 8.7 hours/year
- 99.99% = 52 minutes/year
- 99.999% = 5 minutes/year

Result: Partial — budget correct, RPO misplaced.

---

## Day 4 Revision — Covering Days 1–3 Content

---

**Q1. What are the 3 system archetypes? For each, name the dominant hard problem.**

Your answer: Read-heavy (cache strategy, storage cost, read latency), Write-heavy (write throughput, durability, ordering, consumer lag), Compute-heavy (job scheduling, worker scaling, partial failure, idempotency).

Result: Perfect ✅

---

**Q2. What is an ADR and what is the single most important element — and why?**

Your answer: ADR documents a decision, shows tradeoffs, explains why one solution was chosen over others. Most important element: the Decision.

Correct answer: Most important element is **Alternatives Considered** — not the Decision.
The Decision is visible from the code/architecture anyway. Alternatives Considered is the reasoning that exists nowhere else. Without it, a future engineer might "fix" your design without knowing you already evaluated and rejected their approach.

Result: Partial — purpose correct, wrong most important element.

---

**Q3. Why can't you run SELECT user_id WHERE push=true at flash sale start on 50M users? What do you do instead?**

Your answer: Will have latency of 30s–120s+. Pre-populate segment in Redis cache via nightly offline job, read from Redis at sale start.

Correct answer: ✅ Full table scan on 50M rows takes 30–120s+. Flash sale might be 10 min total — notifications arrive after sale ends. Fix: nightly offline job pre-computes segment → Redis SET → read instantly at sale start.

Result: Correct ✅ (initially skipped the why, added when prompted)

---

**Q4. What check does a worker perform before sending a push notification, and what happens if the check fails?**

Your answer: Check expiration time of sale. If expired, don't forward.

Correct answer: Check `now > expires_at`. If expired → write status=EXPIRED to Cassandra, then discard. Two reasons to write EXPIRED: business visibility (how many users missed it) and ops visibility (spike in EXPIRED = consumer lag too high).

Result: Partial — missed writing EXPIRED status to DB initially.

---

**Q5. What is thundering herd? How is it different from cache avalanche?**

Your answer: Thundering herd — 10M users, cache miss, DB hit 10M times for same key. Fix: lock row, read once, cache, all others read from cache. Cache avalanche — all cache items expire at same time, DB hit with surge of traffic.

Result: Correct ✅ (minor: thundering herd is about one cold key, not necessarily 10M users)

---

**Q6. A stakeholder NFR says "Use Kafka for all async communication." What's wrong with this?**

Your answer: Need to understand system first — which parts are critical, how important is lag.

Correct answer: "Use Kafka" is a **technology choice / solution**, not a non-functional requirement. NFRs must describe measurable behavior, not implementation.
Correct NFR: "Async events must be delivered within 30 seconds" / "Handle 10,000 events/sec without data loss."

Rule: If you're naming a technology in an NFR, you've skipped ahead to Step 4.

Result: Partial — answered how to evaluate, not what's wrong with the statement itself.

---

**Q7. 50M users, 200 notifications/year, 500 bytes each. Storage needed and which DB?**

Your answer: ~5TB. Postgres.

Correct answer: Storage = 5TB ✅. Database = Cassandra, not Postgres.
- 10B rows — Postgres degrades badly at this scale
- Access pattern: append-only writes, query by user_id + created_at → Cassandra's native pattern
- Cassandra: ~100K writes/sec/node vs Postgres ~10K
- Horizontal scale built-in — add nodes, capacity grows linearly

Rule: Match DB to access pattern AND scale. 10B append-only rows queried by partition key = Cassandra.

Result: Partial — storage correct, wrong DB choice.

---

**Q8. What are the two Kafka operational problems in a notification system at scale?**

Your answer: Not sure.

Correct answer (taught as new content):
1. **Consumer lag** — producers push faster than workers consume. Lag grows → messages expire → users missed. Detect: Kafka consumer group lag metric → alert + auto-scale.
2. **Duplicate notifications** — worker sends to FCM, ACK lost due to network blip, worker retries, user gets same notification twice. Fix: idempotency key per notification_id sent to provider.

Result: Not answered — taught as new content.

---

**Q9. Walk through all 5 steps of creating a Pastebin paste using the PENDING status pattern.**

Your answer: Metadata in Postgres as PENDING → content in S3 → background process updates to ACTIVE → background process updates Redis → return 200.

Correct answer: ACTIVE update happens in the **same request**, not a background process.
```
1. INSERT metadata → Postgres (status=PENDING)
2. PUT content → S3
3. UPDATE metadata → Postgres (status=ACTIVE)
4. SET metadata → Redis
5. Return paste_id + 200 to client

Background job (every 5 min) handles FAILURES only:
  PENDING > 10 min → S3 exists? retry step 3 | S3 missing? retry steps 2+3
  N retries failed → status=FAILED, alert
```

Result: Partial — steps correct, background job in wrong role.

---

**Q10. What is metadata + blob pattern and the 3 reasons not to store large content in Postgres?**

Your answer: Store metadata in DB, content in S3. Reasons: DB meant for small data, memory fills faster, S3 scales to petabytes and is cheaper.

Correct answer: ✅ Correct. Precise versions:
1. **Query performance** — large columns pollute buffer cache, slow index scans
2. **Cost** — DB storage 10–50× more expensive than S3 per GB
3. **Scalability** — S3 scales to exabytes transparently; Postgres does not

Result: Correct ✅

---

## Day 5 Revision — Covering Days 1–4 Content

---

**Q1. CP vs AP — what happens during a network partition in each? Give a concrete example.**

Your answer: CP — system unavailable, focuses on consistency. Example: banking balance consistent regardless of location. AP — eventual consistency ok, lag acceptable. Example: ESPN scoreboard, 5 sec lag fine.

Result: Correct ✅

---

**Q2. 4 consistency models strongest to weakest — with real-world example each.**

Your answer: Linearizable (banking). Sequential (FB post/comment). Causal (post + related comment). Eventual (ESPN scoreboard).

Correct answer: Sequential example is wrong — FB post/comment is causal (cause-effect relationship). Sequential = agreed global order without real-time guarantee. Better example: distributed message log, multi-player game move sequence.

Result: Partial — sequential consistency wrong example.

---

**Q3. Why does polling fail at 500M users? What do you use instead? Show the math.**

Your answer: 500M requests at once causes lag and crash. Use WebSocket push. 100M req/sec polling, 3,000 push updates/sec.

Correct answer: ✅
```
Polling: 500M / 5sec = 100M req/sec → impossible
Push:    10,000 matches × 1/30sec = ~333 updates/sec → trivial
300,000× difference from one architectural decision
```

Result: Correct ✅

---

**Q4. Difference between causal and eventual consistency. When choose causal over eventual?**

Your answer: Causal = related data consistent in order (post before comments). Eventual = same data updated, lag acceptable (scoreboard).

Correct answer: Causal = reader never sees effect before cause. If you can see the reply, you must be able to see the original post. Independent operations can appear in any order.
Eventual = all replicas converge eventually, no timing guarantee.

Choose causal: collaborative document editing — User B edits paragraph based on User A's paragraph. Without causal, reader might see B's edit without A's original content.

Result: Partial — right idea, imprecise definition of causal.

---

**Q5. Kafka CAP default and how to change to CP — what do you sacrifice?**

Your answer: AP by default. Change with "time to consistency" parameter. Sacrifice availability.

Correct answer: AP by default. Change with:
- `acks=all` — producer waits for ALL in-sync replicas to acknowledge
- `min.insync.replicas=2` — minimum 2 replicas must acknowledge

Sacrifice: write latency increases 4–10× (5ms → 20–50ms). Producer blocks waiting for replication.

Result: Partial — right concept, wrong parameter names.

---

**Q6. 10,000 matches × 200 bytes score record. Memory needed at CDN edge?**

Your answer: 2000mb (initially). Corrected to 2MB after working through.

Correct answer: 10,000 × 200 bytes = 2,000,000 bytes = 2MB.
Architectural implication: 2MB fits in RAM of any server. Cache ALL scores on every edge node simultaneously. No eviction strategy needed. Push all updates on every score change. Origin gets zero read traffic.

Result: Correct ✅ (after working through calculation)

---

**Q7. Consumer lag and duplicate delivery — causes, detection, fix for each.**

Your answer: Consumer lag = producer writes faster than consumer reads. Duplicate = no ACK, message delivered twice. Fix duplicates with idempotency key.

Correct answer:
Consumer lag:
- Cause ✅ — producer faster than consumer
- Detect: Kafka consumer group lag metric, alert when lag > 100K messages
- Fix: auto-scale workers, rate limit producers, expires_at check discards stale messages

Duplicate delivery:
- Cause ✅ — ACK lost, worker retries
- Detect: monitor delivery count per notification_id > 1
- Fix ✅ — idempotency key at provider level (not worker level)

Result: Partial — causes correct, detection methods missing.

---

**Q8. You chose AP. Users see 30-sec staleness instead of 5-sec. Two most likely causes?**

Your answer: Consistency happening in system (described CP behaviour). Then: Kafka not properly fanning out, single Kafka making it slow.

Correct answer:
1. **Consumer lag** — fan-out consumers fall behind producer. Score update queued in Kafka for 25+ sec before reaching edge nodes. Fix: monitor lag, auto-scale fan-out workers.
2. **CDN TTL too high** — CDN caches score with TTL=30sec. Even after fan-out delivers update, CDN serves stale version. Fix: CDN TTL must match lag SLA (TTL=5sec for 5-sec guarantee).

Result: Partial — one cause found after prompting, CDN TTL missed entirely.

---

## Day 6 Revision — Covering Days 1–5 Content

---

**Q1. 3 clock problems and the rule architects follow.**

Your answer: Clock skew, clock drift, time goes backward. Rule (after prompting): never use wall clock for ordering, use counter between servers.

Result: Partial — 3 problems correct, missed rule initially.

---

**Q2. Difference between Lamport timestamps and vector clocks. When use vector clocks?**

Your answer: Lamport counter is same in all servers, vector clock tracks per node.

Correct answer: Lamport tells you A happened before B. Vector clocks additionally detect **concurrent events** (no causal relationship). Use vector clocks when you need to detect conflicts from concurrent writes (DynamoDB shopping carts).

Result: Partial — right idea, missed concurrent event detection.

---

**Q3. BASE — name all 3 properties and contrast with ACID.**

Your answer: Eventual consistency (E), Basically available (B), Soft state (S — after prompt). Soft state defined as "background running happens."

Correct: Soft state = state changes over time without new input as replicas converge. Contrast with ACID Isolation which freezes state during transactions.

Result: Partial — 2/3 properties initially, soft state definition vague.

---

**Q4. 4 isolation levels weakest to strongest. Postgres default?**

Your answer: Atomicity, Consistency, Isolation, Durability (confused with ACID properties).

Correct (taught as new content):
- Read Uncommitted (allows dirty reads)
- Read Committed (Postgres default — prevents dirty reads)
- Repeatable Read (prevents non-repeatable reads)
- Serializable (prevents all anomalies including phantoms)

Result: Not answered — taught as new content.

---

**Q5. Redis Sorted Set — 3 commands: add score, increment, get top-100.**

Your answer: Don't know.

Correct (taught as new content):
- ZADD leaderboard 1500 "player_alice"
- ZINCRBY leaderboard 50 "player_alice"
- ZREVRANGE leaderboard 0 99 WITHSCORES

Bonus: ZREVRANK leaderboard "player_alice" for own rank.

Result: Not answered — taught as new content.

---

**Q6. Payment startup needs global scale + strong consistency. What database?**

Your answer: Google Spanner — ACID in distributed environment.

Result: Correct ✅ (added: CockroachDB as open-source alternative)

---

**Q7. Same consistency model, different lag SLAs (5s vs 30s). How?**

Your answer: Use CDN to push updates so SLA is less.

Correct: CDN solves top-100 delivery but can't serve own rank (personalized).
Real answer: **tunable consistency** — same eventual consistency, different quorum:
- Top-100: consistency=ONE → fastest, 30s lag
- Own rank: consistency=QUORUM → fresher, 5s lag

Rule: Don't change consistency model for tighter lag — tune the quorum.

Result: Partial — CDN correct, missed tunable quorum.

---

**Q8. What is quorum? If N=3, W=2, R=2, what does W+R>N guarantee?**

Your answer: Don't know.

Correct (taught as new content):
- Quorum = majority must agree before operation succeeds
- W+R>N guarantees overlap between write set and read set
- Therefore guaranteed to read at least one node with latest write
- Same concept as Kafka min.insync.replicas

Result: Not answered — taught as new content.

---

**Q9. 100M players, 10 updates/session, 1hr session, 20% active. Updates/sec?**

Your answer: ~55K (after working through).

Correct: 100M × 20% = 20M active. 20M × 10 = 200M/hr. 200M / 3600 = ~55,556/sec average. Peak (×3) = ~166,000/sec.

Add: at 166K peak, write batching needed (buffer per-player 100ms, write sum once).

Result: Correct ✅

---

**Q10. AP system after partition heals — 3 conflict resolution strategies?**

Your answer: Eventual consistency (the goal, not a strategy). Then: based on counter (LWW with logical clock).

Correct (1 of 3 named):
1. **Last Write Wins** ✅ — highest counter wins (Cassandra default)
2. **Merge** — combine values (DynamoDB carts, CRDTs)
3. **Ask the user** — present conflict, let human decide (Google Docs, Git, Dropbox)

Result: Partial — 1/3 named, others taught.

---

## Day 7 Revision — Covering Days 1–6 Content

---

**Q1. What is consensus and why does FLP impossibility matter?**

Your answer: Consensus is the way to make the system consistent. Not sure about FLP.

Correct: Consensus = getting unreliable nodes to agree on a single value despite failures. FLP (1985): in async network with even one faulty node, no consensus algorithm can guarantee both safety AND liveness. Real systems sacrifice liveness during partitions to preserve safety.

Result: Partial — consensus correct, FLP unknown.

---

**Q2. Why does Raft use odd numbers (3, 5, 7) and never even?**

Your answer: So decision can be made — even numbers can split decision.

Correct ✅. 3=tolerates 1 failure, 5=2 failures, 7=3 failures. Even numbers split exactly during partition → no majority on either side → cluster halts.

Result: Correct ✅

---

**Q3. TTL/lease pattern in distributed locks. Why not "release on crash"?**

Your answer: Releasing on crash can be false info, could be due to high request for heartbeat. TTL=30s, renew every 10s.

Correct ✅. Lock service can't distinguish: crashed, slow, network blip. TTL must be > processing time but short enough for fast recovery.

Result: Correct ✅

---

**Q4. Cassandra N=3, W=1, R=1. Consistency for banking?**

Your answer: Not good enough. With W=1, R=1 leader and one node guarantee consistency — but two different data if leader goes down.

Correct: Cassandra is leaderless — no leader/follower. Real reason: W+R = 2, not > 3. No overlap guaranteed → can read replica without latest write. For banking need W+R>N: W=2, R=2 with N=3.

Result: Partial — right verdict, leader/follower framing wrong for Cassandra.

---

**Q5. What is split brain and why with 2-node setups?**

Your answer: Don't know.

Correct (taught): Network partition divides cluster. Each side thinks it's authoritative, both accept writes. Result: conflicting data impossible to safely reconcile.
2 nodes: each thinks the other crashed → both accept writes → split brain.
Solved by odd numbers + majority quorum: minority side halts.

Result: Not answered — taught as new content.

---

**Q6. 50M users, 5min sessions, 10 page views/session, 30% active. Page views/sec at peak?**

Your answer: 1.5 million views/sec. Working: 15M × 3 × 10 / 300.

Correct: 50M × 30% = 15M active. 15M × 10 = 150M views/day. 150M / 86,400 = ~1,736/sec avg. Peak ×3 = ~5,208/sec.

Mistake: divided by 300 (not 86,400). Multiplied by 3 before averaging. Recurring formula gap.

Rule:
1. Active users = total × active%
2. Daily events = active users × events/user
3. Avg/sec = daily / 86,400
4. Peak = avg × 3 (apply LAST)

Result: Wrong — recurring capacity formula gap.

---

**Q7. Lamport vs vector clocks. When use vector clocks?**

Your answer: Lamport counter per node, vector clocks counter from all nodes preventing inconsistency.

Correct: Lamport tells you A happened before B. Vector clocks ALSO detect concurrent events (no causal relationship). Use vector clocks specifically to detect concurrent writes for conflict resolution (DynamoDB shopping carts).

Result: Partial — structural difference correct, missed concurrent detection purpose.

---

**Q8. @Transactional without isolation level — what's used and what anomaly does it allow?**

Your answer: Atomic commit guaranteed by @transactional.

Correct: Postgres default = Read Committed. Allows non-repeatable reads (same query returns different result mid-transaction if another commits). Step up to Serializable for financial transactions.

Result: Not answered — recurring gap from Day 6.

---

**Q9. Name all 3 BASE properties without prompting.**

Your answer: Basically available, soft state, eventual consistency.

Result: Correct ✅ — recall improving!

---

**Q10. Global scale + strong consistency — category and 2 products?**

Your answer: NewSQL — Google Spanner (enterprise) or CockroachDB (startup, open-source).

Result: Correct ✅

---

## Day 8 Revision — Covering Days 1–7 Content

---

**Q1. 200M users, 40% daily, 4 videos × 30 min each. Avg + peak concurrent viewers?**

Your answer: 80M × 4 × 30 × 60 = 576K views/sec × 3 = 1.73M/sec.

Correct: This is concurrent VIEWERS, not views/sec.
80M × 4 × 30 = 9.6B user-minutes/day
9.6B / 1,440 (minutes/day) = ~6.67M concurrent avg
Peak (×3) = ~20M concurrent

Formula: `(active × time_per_user) / total_time_window`

Result: Wrong — concurrent users is a different formula than events/sec.

---

**Q2. Postgres default isolation level + anomaly it allows?**

Your answer: Read write commit. Returns stale data.

Correct: **Read Committed**. Allows non-repeatable reads (same query mid-transaction returns different result if another commits).

Result: Partial — wrong term, wrong anomaly. Recurring gap.

---

**Q3. Split brain — why dangerous in 2-node?**

Your answer: Both can't ping each other, both become leader, hard to tell actual leader. (Then prompted) Both have different data, need 3 conflict mechanisms.

Result: Correct ✅

---

**Q4. Single-leader vs multi-leader vs leaderless. One DB each.**

Your answer: Single-leader = Postgres ✅. Multi-leader = Spanner (wrong — that's NewSQL). Leaderless = CockroachDB (wrong — that's NewSQL).

Correct:
- Single-leader = Postgres / MySQL / MongoDB ✅
- Multi-leader = CouchDB / MySQL circular
- Leaderless = Cassandra / DynamoDB / Riak
- NewSQL (separate category) = Spanner / CockroachDB

Result: Partial — single-leader correct, others mixed up.

---

**Q5. Polyglot persistence + why?**

Your answer: Multiple DBs based on need. Amazon: Postgres for payment, Cassandra for viewing.

Result: Correct ✅

---

**Q6. Why DynamoDB for cross-device cart? Which mechanism handles concurrent writes?**

Your answer: DynamoDB best because reads happen across leaderless DBs, viewable immediately.

Correct: DynamoDB chosen because of **vector clocks** detecting concurrent writes from different devices → MERGE both items into cart. Single-leader with last-write-wins would silently lose items.

Result: Wrong — missed vector clocks + merge purpose.

---

**Q7. Product catalog: 10M products, 100K reads/sec, 100 writes/day. DB + replication?**

Your answer: CDN + Cassandra.

Correct: CDN ✅. **Postgres single-leader + read replicas + CDN** — NOT Cassandra.
Cassandra is for write throughput, not read throughput. With 100 writes/day, Cassandra is overkill and gives weaker consistency than needed for price updates.

Result: Partial — CDN correct, Cassandra wrong choice.

---

**Q8. Read-after-write trap + most common solution?**

Your answer: Don't know.

Correct (taught): User writes to leader, immediately reads from follower (still has old data) → "I just updated this!" Solution: route the writer's reads to leader for ~60 seconds after write.

Result: Not answered — taught.

---

**Q9. 80M users, 5 API calls/day. Peak/sec? Show steps.**

Your answer: 80M × 5 = 400M/day. 400M / 86,400 = 4,629/sec. × 3 = 14,184/sec peak.

Result: Correct ✅ — capacity formula clean for events/sec.

---

**Q10. 3-device cart updates — why multi-leader poor fit, what's better?**

Your answer: Multi-leader gets data from different devices, will be inconsistent. Better: DynamoDB.

Correct ✅. Sharpened: multi-leader uses LWW timestamps → silent data loss. Leaderless with vector clocks → detects concurrent → merges → no data loss.

Result: Correct ✅

---

## Day 9 Revision — Covering Days 1–8 Content

---

**Q1. Name 3 partitioning strategies + 1 real DB each.**

Your answer: hash-based ✅, directory-based ✅, location-based ❌.

Correct: range-based (HBase, BigTable), hash-based (Cassandra), directory-based (Vitess).

Result: Partial — 2/3.

---

**Q2. What is consistent hashing + what problem does it solve?**

Your answer: hashing based on num of shards. Naive hash loads one shard a lot.

Correct: Circle of values 0 to 2^32-1. Hash nodes onto positions. Hash keys onto positions. Owner = first node clockwise from key. Solves resharding pain — adding a node moves only ~1/N of keys instead of 80%.

Result: Wrong — confused with virtual nodes, missed resharding pain.

---

**Q3. Natural shard key for messaging system?**

Your answer: chat_id. Unique, holds chat info, same for individual and group chats.

Result: Correct ✅

---

**Q4. Postgres default isolation level + Serializable use cases?**

Your answer: read write. Result of transaction will change in between (good non-repeatable read example).

Correct: Read Committed (recurring gap — 4th time wrong). Use Serializable for: bank transfers, inventory deduction, sequential ID generation.

Result: Partial — example correct, term wrong.

---

**Q5. 100B msg/day × 100 bytes × 365 × 3 replication?**

Your answer: 100B × 100 = 10TB × 365 = 3.65PB × 3 = 10.95PB.

Result: Correct ✅

---

**Q6. Why single-leader Postgres bad for global payments? What instead?**

Your answer: Synchronous replication slow. Use Spanner/CockroachDB.

Sharpened: bigger problem is geography — single leader can't be near global users. NewSQL solves with regional writes + Paxos/Raft for ACID across regions.

Result: Correct ✅

---

**Q7. 50M users, 5% trade daily, 10 trades each. Peak/sec?**

Your answer: 2.5M × 10 = 25M/day. /86,400 = 289/sec. ×3 = 868/sec peak.

Result: Correct ✅

---

**Q8. Metadata + blob pattern + 2 Bitpanda components?**

Your answer: metadata in Postgres, blob in S3. Not sure about Bitpanda.

Correct (taught): KYC documents (passport, ID, address proof), historical price data archives, transaction PDFs, profile pictures.

Result: Partial — pattern correct, applications taught.

---

**Q9. Read-after-write trap + most common solution?**

Your answer: Replication lag means follower has old data when read happens immediately after write. Solution: route writer's reads to leader for ~60 seconds after write.

Result: Correct ✅ (cause description was muddled but landed)

---

**Q10. Name all 3 BASE properties without prompting.**

Your answer: Basically available, soft state, eventual consistency.

Result: Correct ✅ — instant recall

---

## Score Summary

| Day | Score | Main Gaps |
|---|---|---|
| Day 2 revision (Day 1 content) | 8.5/12 | CAP vs trade-off triangle, RPO/RTO unknown, hash-based collision misunderstood |
| Day 3 revision (Day 1+2 content) | 8.5/12 | 5-step framework, NFR vs solution, dropping zeros in calculations, thundering herd mechanism |
| Day 4 revision (Days 1–3 content) | 7.5/10 | Alternatives Considered is most important ADR element, NFR vs solution (recurring), wrong DB for 10B rows, consumer lag + idempotency unknown |
| Day 5 revision (Days 1–4 content) | 7/8 | Sequential consistency wrong example, Kafka CP params unknown, detection methods missing, stale data causes needed prompting |
| Day 6 revision (Days 1–5 content) | 6/10 | Isolation levels unknown, Redis ZSET commands unknown, quorum math unknown, conflict strategies (only 1 of 3) |
| Day 7 revision (Days 1–6 content) | 7/10 | FLP unknown, Cassandra leader/follower confusion, split brain unknown, capacity formula scrambled, isolation levels still unknown |
| Day 8 revision (Days 1–7 content) | 6.5/10 | Concurrent users formula new variant, Postgres default term wrong, multi-leader/leaderless mixed up, vector clocks for carts missed, Cassandra wrong for read-heavy catalog |
| Day 9 revision (Days 1–8 content) | 7.5/10 | Range-based as 3rd partitioning strategy missed, consistent hashing confused with vnodes, Read Committed term still wrong, metadata+blob applications missed initially |
| Day 10 revision (Days 1–9 content) | 8.2/15 (55%) | Raft quorum formula wrong (said 2, correct is 3), 2PC fatal flaw unknown, hot spot patterns unknown, Lamport clocks unknown, read-after-write unknown, consistent hashing direction imprecise, range partition advantage missed (range queries), Read Committed anomaly missed (non-repeatable read) |

---

## Day 10 Revision — Covering Days 1–9

---

**Q1. Postgres default isolation level — exact term — and what anomaly does it allow?**

Your answer: Read Committed. Only final committed reads allowed.

Correct answer: **Read Committed** (name correct — first time in 5 quizzes). Anomaly it *still allows*: **non-repeatable reads** — same row read twice in same transaction can return different values if another transaction commits between reads. Score: 50%.

---

**Q2. CAP theorem — 3 properties and which you must give up during partition?**

Your answer: Consistency, Availability, Partition Tolerance. Must give up either C or A during partition. Banking → CP. Shopping → AP.

Correct answer: Exactly right. P is not optional — networks always partition. Score: 100%.

---

**Q3. Raft 5-node cluster — how many nodes must ACK before leader commits?**

Your answer: One leader and one follower (= 2 nodes).

Correct answer: **Quorum = ⌊N/2⌋ + 1 = 3 nodes** for 5-node cluster. Leader + 2 followers. "2 nodes" is not a majority — allows split brain. Score: 0%.

---

**Q4. Difference between orchestration and choreography in Saga?**

Your answer: Orchestration = one system responsible for commit/revert. Choreography = not centralized, has instructions for positive/negative scenarios.

Correct answer: Correct. Fixed from Day 9 where these were reversed. Score: 100%.

---

**Q5. What problem does the Outbox Pattern solve, and how?**

Your answer: DB commit success + Kafka fail = lost message. Outbox: commit message + data in same DB transaction. Other service picks up and sends to Kafka.

Correct answer: Perfect. The "other service" is the outbox publisher (polling or CDC via Debezium). Score: 100%.

---

**Q6. Consistent hashing — when a new node is added, which data moves and where?**

Your answer: Only data from previous ring moves to next ring.

Correct answer: When Node C is inserted between A and B, keys in range **(A → C]** move **from Node B to Node C**. Only 1/N of total data moves. Score: 60% — direction imprecise.

---

**Q7. Range vs hash partitioning — one advantage and one disadvantage each?**

Your answer: Range = easier setup, can hot spot. Hash = even distribution, hard to setup.

Correct answer: Range advantage = **range queries hit only relevant partitions** (not "easier setup"). Hash disadvantage = **range queries require scatter-gather** (not "hard to setup"). Score: 60%.

---

**Q8. Back-of-envelope: 500M DAU, 10 msgs/day each. Peak messages/sec?**

Your answer: (500M × 10 / 86400) × 3 = ~173,600/sec peak.

Correct answer: Exactly right. ×3 peak multiplier last. Score: 100%.

---

**Q9. What is linearizability? How does it differ from eventual consistency?**

Your answer: Eventual consistency = data consistent eventually via messages. Linearizability = unsure.

Correct answer: **Linearizability** = once a write completes, every subsequent read from any node sees it immediately. System behaves as single copy. Required for: distributed locks, leader election, bank balances. Eventual = window of inconsistency (seconds to minutes). Score: 50%.

---

**Q10. What problem does 2PC solve, and what is its fatal flaw?**

Your answer: Not sure.

Correct answer: **Solves:** atomic commits across multiple DBs/services — all commit or all rollback. **Fatal flaw:** if coordinator crashes after "Prepare" but before "Commit", all participants hold locks indefinitely waiting. System is **blocked** until coordinator recovers. Second flaw: external systems (Stripe, Kafka, Redis) don't support 2PC. Score: 0%.

---

**Q11. What does CP mean in CAP? Name a real CP database and why.**

Your answer: CP = consistency + partition tolerance. Availability negotiable. Google Spanner is CP.

Correct answer: Correct. CP system refuses requests during partition rather than risk stale data. Spanner uses TrueTime + Paxos. Other CP: etcd, ZooKeeper, HBase. Score: 100%.

---

**Q12. What is a hot spot? Two causes and fixes?**

Your answer: Don't know.

Correct answer: One partition gets disproportionate traffic. **Cause 1:** Time-based partition key → all writes go to "today" → fix: add random shard suffix (date-0, date-1...date-9). **Cause 2:** Celebrity/viral key → fix: cache hot key in Redis. **Cause 3:** Sequential ID → fix: use random UUIDs/Snowflake IDs. Score: 0%.

---

**Q13. What is a Lamport clock and what can it NOT tell you?**

Your answer: Don't know.

Correct answer: **Lamport clock** = logical counter per node. Rule 1: increment before every event. Rule 2: on receive, set to max(local, received) + 1. Gives causal ordering. **Cannot tell you:** if two events are concurrent (neither caused the other). For that: vector clocks (counter per node, used in DynamoDB/CRDTs). Score: 0%.

---

**Q14. What is the read-after-write problem and how do you solve it?**

Your answer: Leader reads/writes but not synced, leader is dead.

Correct answer: User writes profile → read routes to follower → follower not synced yet → user sees old data. **Three fixes:** (1) Route reads for own data to leader; (2) Track replication LSN, only use replicas caught up to that position; (3) Route all reads to leader for 1-2s after any write. Score: 0%.

---

**Q15. Same POST /trade received twice due to network retry — how to execute exactly once?**

Your answer: Use idempotency key. Generate key, verify if processed, if yes return old result.

Correct answer: Correct. 3 steps: (1) Client generates UUID before sending; (2) Server checks Redis for key; (3) If exists → return cached result, if not → execute + store result in Redis with 24h TTL. Score: 100%.
