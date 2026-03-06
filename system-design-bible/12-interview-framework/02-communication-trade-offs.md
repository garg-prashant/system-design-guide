# Communication and Trade-off Reasoning

## How to Communicate

- **State your step.** “First I’ll clarify requirements.” “Now I’ll sketch the high-level architecture.”
- **Summarize before diving.** “The main components will be: API gateway, auth service, core service, DB, and cache. I’ll detail the core service and data layer.”
- **Invite feedback.** “Does this direction work, or would you like me to focus elsewhere?” “Should I go deeper on X?”
- **Listen to hints.** If they say “what about consistency?” or “how would this scale?” they’re steering you—answer that next.
- **No long monologues.** Short blocks; check in; let them ask follow-ups.

---

## How to Present Trade-offs

Don’t just pick a solution—**name the trade-off** so they see you understand the cost.

**Template:** “We could do A (pro: X, con: Y) or B (pro: X, con: Y). Given [scale / consistency / ops], I’d choose B because …”

**Examples:**

- **Consistency vs availability:** “We could use strong consistency, but that would add latency and reduce availability on partition; for a feed we’ll use eventual consistency and accept a short delay.”
- **Build vs buy:** “We could build our own rate limiter, but a managed API gateway or Redis-based solution gets us there faster; we’d build only if we need very custom logic.”
- **Latency vs freshness:** “We could read from the primary for every request for freshness, but that would increase latency and load; we’ll use read replicas with a short replication lag.”
- **Throughput vs simplicity:** “We could use a message queue for every write to decouple and buffer, but that adds complexity and eventual consistency; we’ll start with synchronous writes and add a queue if the DB becomes the bottleneck.”

**Interview:** Showing you see multiple options and choose with clear criteria is more important than picking the “perfect” answer.
