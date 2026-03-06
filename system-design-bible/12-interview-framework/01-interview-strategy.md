# Interview Strategy for System Design

## Before You Start Drawing

1. **Clarify the problem.** Restate in your words: “So we’re building X; users can do A, B, C; and we care about scale and latency. Is that right?”
2. **Ask about scope.** “Are we focusing on the core flow first, or should I include auth, analytics, and admin?” “Is this mobile-only or web too?”
3. **Get numbers.** “Roughly how many users? DAU? Reads vs writes? Latency target?” If they don’t give numbers, propose: “Assume 10M DAU, 100 reads per user per day, p99 under 200 ms—should I adjust?”
4. **Identify constraints.** Single region or global? Must we use specific tech? Compliance (e.g. data residency)?

---

## How to Run the Interview

- **Time-box.** Allocate time: e.g. 5 min requirements, 3 min API, 10 min high-level design, 10 min deep dives, 5 min failure/ops. Adjust if interviewer steers.
- **Think out loud.** “I’m going to start with requirements, then API, then a high-level diagram.” Say what you’re doing so they can follow and correct.
- **Start broad, then go deep.** One diagram with 5–7 boxes first. Then: “I’ll go deeper on the database and caching layer.” Don’t jump into code or tiny details early.
- **Use the whiteboard.** Draw client, LB, services, DB, cache, queue. Label. Arrows for data flow. It’s the main artifact.

---

## What They’re Evaluating

- **Requirements:** Do you ask good questions and avoid over/under-scoping?
- **Structured thinking:** Clear process (requirements → API → design → deep dive → failure).
- **Trade-offs:** Do you name options and trade-offs (e.g. consistency vs availability, build vs buy)?
- **Scale and numbers:** Do you use rough QPS, storage, latency to justify choices?
- **Failure and operations:** Do you mention replication, failover, monitoring, deployment?
- **Communication:** Can they follow your reasoning? Do you listen to hints?

---

## If You Get Stuck

- **Narrow the problem.** “Can I focus on the write path first?” or “Should I assume we already have a user service?”
- **Use what you know.** Fall back to patterns: LB, cache, queue, DB with replicas, async processing.
- **State assumptions.** “I’ll assume we use a managed DB and add read replicas for scale.”
- **Ask.** “Would you like me to go deeper on the DB schema or on the caching strategy?”
