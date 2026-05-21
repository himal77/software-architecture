# Case Study: Clash of Clans Leaderboard
**Covered:** Day 5 | **Topic:** Global leaderboard at game scale

---

## The Problem
Global clan leaderboards for 100M+ players across dozens of game servers worldwide. Real-time rankings at massive scale.

---

## Architecture
- Game servers write scores to **regional Redis Sorted Sets**
- Global leaderboard: top-N per region merged and re-sorted centrally **every 5 minutes**
- Merged result pushed to CDN globally
- 99% of queries hit CDN
- Redis handles own-rank lookups and regional top lists only

---

## The Key Insight
Players care about:
- **Regional ranking** → real-time (Redis ZSET per region)
- **Global ranking** → approximate is fine (5-min batch, CDN)

Split into two different systems with different consistency requirements instead of solving the hard problem of global real-time ranking.

---

## Why This Works
Global real-time leaderboard across 100M players = expensive coordination problem.
Regional real-time + global approximate = two cheap, independent problems.

---

## Lesson
**When global real-time is too expensive, ask if regional real-time + global approximate is good enough. It almost always is.**

Apply this pattern whenever you face a "global real-time" requirement — decompose it into regional + approximate global before assuming you need to solve the hard version.
