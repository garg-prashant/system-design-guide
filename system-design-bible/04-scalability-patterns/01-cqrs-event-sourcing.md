# CQRS and Event Sourcing

## CQRS (Command Query Responsibility Segregation)

**Idea:** Separate the model (and often the storage) for **writes** (commands) from **reads** (queries). Read and write paths can then scale and evolve independently.

```
  Commands (write)  ──→  Write model / Write DB  ──→  (optional) events
                                                              │
  Queries (read)   ──→  Read model / Read DB   ←──────────────┘
                       (views, caches, replicas)
```

### When to use CQRS

- **Read and write load or shape differ a lot** — e.g. many different read views (lists, dashboards, search) vs simple write model.
- **Different consistency needs** — writes strong, reads eventually consistent from read replicas or denormalized views.
- **Team or domain split** — one team owns writes, another owns read pipelines.

### Trade-offs

- **Pros:** Scale reads independently; optimize read models (denormalized, indexed) without touching write path; can add new read views without changing writes.
- **Cons:** Eventual consistency on reads; more moving parts (write store, read store(s), sync pipeline); complexity and ops.

---

## Event Sourcing

**Idea:** Store **all changes** as an immutable log of events. Current state is derived by replaying or folding events. No “current row” overwrites; append-only event store.

```
  Command  ──→  Append event to store  ──→  Event log: [E1, E2, E3, ...]
                                                    │
                                                    └──→  Projections / Read models (build from events)
```

### When to use event sourcing

- **Audit and compliance** — full history of what happened.
- **Time travel / debugging** — replay to any point in time.
- **Multiple read models** — each projection consumes same events and builds its own view.
- **Complex domain** — events capture intent and decisions, not just state snapshots.

### Trade-offs

- **Pros:** Full history; flexible read models; replay and debugging; natural fit with CQRS.
- **Cons:** Event store grows forever (need retention/snapshots); replay can be slow (snapshots + replay from snapshot); learning curve; eventual consistency for read models.

---

## CQRS + Event Sourcing Together

- **Writes:** Command → validation → append event(s) to event store.
- **Reads:** Projections consume events and update read-model DBs (e.g. PostgreSQL, Elasticsearch). Queries hit read models only.
- **Consistency:** Read models are eventually consistent. Use saga or process manager for multi-entity writes; use version/sequence in events for ordering.

**Interview:** “We use CQRS so read and write scale independently; we use event sourcing so we have a full audit trail and can build multiple read views from the same event log.”
