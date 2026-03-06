# Saga Pattern

## Problem

In a **distributed system**, a business operation may span multiple services and databases. You need **eventual consistency** across them without a single global transaction (2PC is costly and often avoided at scale).

---

## Saga Definition

A **saga** is a sequence of **local transactions**, each in one service. If any step fails, **compensating transactions** run to undo prior steps (in reverse order). No global lock; eventual consistency.

```
  Order Service          Payment Service          Inventory Service
       │                        │                         │
       │  1. Create order       │                         │
       │──────────────────────→│                         │
       │  2. Reserve payment    │                         │
       │                        │  3. Reserve inventory   │
       │                        │────────────────────────→│
       │                        │  OK                     │
       │  OK                    │                         │
       │                        │                         │
  If step 3 fails:
       │                        │  3'. Release payment    │
       │                        │←────────────────────────│
       │  2'. Cancel order      │                         │
       │←──────────────────────│                         │
```

---

## Choreography vs Orchestration

| Style | How it works | Pros | Cons |
|-------|----------------|------|------|
| **Choreography** | Each service reacts to events and publishes its own; no central coordinator. | Loose coupling, no SPOF | Hard to reason about; flow scattered; harder to add compensations. |
| **Orchestration** | Central orchestrator (saga coordinator) calls each service in sequence and triggers compensations on failure. | Clear flow, easier to add steps and compensations | Orchestrator can become bottleneck; coupling to coordinator. |

**Interview:** Prefer **orchestration** when the flow is complex or compensations matter a lot (e.g. payments); choreography for simpler, event-driven flows.

---

## Compensating Transactions

Each step has a **compensation** that semantically undoes it (e.g. “release inventory”, “refund payment”). Compensations are **not** exact inverse of the original step—they are business-level undo.

- **Idempotency:** Compensations may run more than once (retries); make them idempotent.
- **Order:** Run compensations in **reverse** order of completed steps.
- **Failure in compensation:** Design for it (e.g. manual intervention, alerting, retry).

---

## Trade-offs

| Aspect | Trade-off |
|--------|-----------|
| **Consistency** | Saga gives eventual consistency, not strong. User may see temporary inconsistency. |
| **Complexity** | Many steps and compensations are hard to design and test. |
| **Semantic lock** | Sometimes you “lock” a resource (e.g. hold inventory) until confirm or timeout; then release or confirm. |

**When to use:** Cross-service workflows (order → payment → inventory → shipping) where you don’t want 2PC. When to avoid: when strong consistency is required (consider single service or 2PC if acceptable).
