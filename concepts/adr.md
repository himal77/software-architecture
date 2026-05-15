# Architecture Decision Records (ADRs)
**Introduced:** Day 3

---

## What It Is
A short document capturing a significant architectural decision — the context, the choice, the alternatives rejected, and the consequences.

---

## When to Write One
- You chose technology A over B and someone will ask why later
- You made a tradeoff not obvious from the code
- You rejected a "better" solution for a valid but non-obvious reason
- A future engineer might be tempted to "fix" your design

## When NOT to Write One
- Obvious decisions
- Standard well-known patterns
- Things fully explained by the code itself

---

## Format
```markdown
# ADR-00X: Title of decision

## Status
Accepted / Rejected / Deprecated

## Context
Why was this decision needed? What constraints existed?

## Decision
What exactly was decided — specific, not vague.

## Alternatives Considered
What else was evaluated and why it was rejected.
Name each alternative + specific rejection reason.

## Consequences
Honest pros AND cons — not just the good parts.
```

---

## The Most Important Element
**Alternatives Considered** — without it, the ADR looks like you didn't evaluate options.
With it, it proves you made a reasoned choice under real constraints.

---

## Example
See [designs/notification-system.md](../designs/notification-system.md) for ADR-001: Centralised preference check before channel routing.
