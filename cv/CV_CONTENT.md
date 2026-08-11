# HIMAL PURI — CV content

Source of truth for the CV text. Edit here, then copy into the .docx.

---

## HEADER

```
HIMAL PURI
+43 68864346750  ·  Klosterneuburg, Austria
mr.himalpuri@gmail.com  ·  LinkedIn
Red-White-Red Card plus
```

---

## PROFILE  ← NOT YET IN THE DOCX

Full width, above the two-column section.

```
Senior backend engineer with 5 years modernising legacy systems in the insurance domain.
Led a greenfield microservices project end-to-end (team of 7, delivered in 9 months) and
designed an event-driven integration platform serving 5,000+ agents. Java, Spring Boot,
Kafka, Kubernetes, AWS. EU work authorisation (Red-White-Red Card plus, Austria).
```

---

## EXPERIENCE

**Allianz Technology** · Vienna, Austria — 10/2021 – Present

### Senior Software Developer (01/2024 – Present)

- As a lead developer, I interviewed and gathered a team of 7 and led the delivery of Allianz
  Italy's Intercompany Agreement greenfield project in 9 months, leveraging Spring Boot, Kafka,
  MongoDB, and AWS (ECR, EKS).
- Designed and implemented a middleware platform Enterprise Process Component (EPC) using
  Apache Camel and Kafka to synchronize multiple systems used by over 5K agents in Allianz France.
- Migrating legacy PL/I mainframe modules to Java, reverse-engineering undocumented business
  logic from production code and verifying functional equivalence against the original
  implementation.

### Software Developer (10/2021 – 12/2023)

- Maintained and enhanced customer management system with REST API, wrote quality and reusable
  code, and resolved about 150+ bug reports, resulting in a more stable product version.
- Built a PoC for a modernized backend customer management system using Spring Boot and MongoDB,
  where data was synced with new system in real time by using Kafka Connect and presented to 10k
  viewers during Allianz Global Forum.

---

## PROJECTS  ← NOT YET IN THE DOCX

**Trading Platform** — event-driven fintech backend (in progress)
`github.com/himal77/trading-platform`

- Microservices architecture: 6 Spring Boot services, PostgreSQL, Redis, Kafka (KRaft),
  Docker Compose, GitHub Actions CI.
- Double-entry ledger with optimistic locking and database-enforced idempotency; balance
  correctness verified by replaying the ledger in integration tests.
- Stateless JWT authentication with rotating opaque refresh tokens in Redis for immediate
  revocation.
- Architecture Decision Records documenting each trade-off, including rejected alternatives.

---

## SKILLS

**Backend**
- Java, Python, C, C++
- Spring Boot, Flask, FastAPI
- REST, OpenAPI

**Data**
- PostgreSQL, DB2, MongoDB, Redis
- Apache Kafka (Connect, Streams)
- JPA/Hibernate, Flyway

**Frontend**
- JavaScript, HTML, CSS
- Vue.js, AngularJS

**Testing**
- JUnit, Mockito, Postman

**DevOps & Tools**
- GitHub Actions, Jenkins, ArgoCD
- Docker, Kubernetes
- Terraform (IaC)
- Prometheus & Grafana
- AWS (EC2, Lambda, EKS, S3, RDS)
- IntelliJ IDEA, Eclipse

**Architecture & Principles**
- Hexagonal
- Layered Architecture
- Domain-Driven Design (DDD)
- Event-Driven Architecture
- SOLID

**Languages**
- English C1 · German B1 · Hindi C1 · Nepali Native

---

## EDUCATION

**Universität Wien** — Austria
Master's in Computer Science (part-time) · 2020 – Present

**Bangalore University** — India
Bachelor's in Computer Application · 2013 – 2016

---

# Outstanding items

- [ ] Add PROFILE section (above Skills, full width)
- [ ] Add PROJECTS section (after Experience, before Skills/Education)
- [ ] Fix `PostgreSQL,DB2` → `PostgreSQL, DB2` (missing space)
- [ ] Remove stray parentheses: `using (Apache Camel and Kafka)` → `using Apache Camel and Kafka`
- [ ] Make the trading-platform repo public with a presentable README before listing it
- [ ] Move "Data" above "Frontend" — backend roles care about data, not Vue.js

# Interview preparation

**The PL/I bullet will be probed.** Expect: *"Walk me through a tricky piece of business logic
you found in the PL/I and how you handled it."* Collect one concrete example now — an edge case,
dead code, an undocumented rule, a rounding behaviour that didn't translate naively. Without a
story ready, drop the bullet; it does more harm than good.

**The ADR bullet is the strongest differentiator.** Almost no candidate can show a document
explaining why they *rejected* an alternative. Be ready to talk through ADR-001 (why the ledger
is not full double-entry) and ADR-002 (why optimistic over pessimistic locking, and when that
decision would flip).

**Do not list the 90-day curriculum on the CV.** "Currently studying architecture" reads as
*not yet an architect*. The knowledge should surface in how you answer questions.
