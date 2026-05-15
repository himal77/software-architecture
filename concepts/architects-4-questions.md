# The Architect's 4 Questions
**Introduced:** Day 1

Ask these before drawing a single box on a whiteboard.

---

## 1. Who uses it, and how?
- Internal tooling (50 engineers) vs consumer-facing (50M users)?
- Read-heavy (social feed) or write-heavy (logging)?
- Synchronous (user waits) or async (fire and forget)?

## 2. What are the scale numbers?
- Users: DAU, MAU, peak concurrent
- Data: how much per day, per year, total
- Requests: reads/sec, writes/sec at peak

> Never design without numbers. Vague scale = vague design.

## 3. What breaks, and what's the cost of it breaking?
- Inconvenience (Twitter down) vs catastrophic (payment system down)?
- Can you lose data? For 1 second? 1 minute? Never?
- Defines **RPO** — how much data loss is tolerable
- Defines **RTO** — how fast you must recover

## 4. What are the constraints?
- Budget: startup with 3 engineers vs enterprise with 300
- Timeline: 2-week MVP vs 6-month proper build
- Compliance: GDPR, PCI-DSS, HIPAA changes the entire data layer
- Existing stack: greenfield vs must integrate with legacy
