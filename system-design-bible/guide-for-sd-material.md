# Complete Guide to System Design Material

How to use this repository to prepare for Senior, Staff, and Principal engineer system design interviews—and to build design intuition that lasts beyond the interview.

---

## What This Guide Is For

You’re not here for a quick cram. This repo is built for **experienced engineers (L5+)** who want depth: trade-offs, failure modes, production practices, and the kind of discussion that happens in real system design interviews.

This guide answers: **in what order to study, at what depth, and how to connect concepts to practice.** By the end you should be able to (1) approach any design problem with a clear framework, (2) justify decisions with numbers and trade-offs, and (3) talk about failure and operations without hand-waving.

---

## The Design Process: Use It Every Time

Before diving into material, internalize this. In interviews and when reading problem write-ups, follow the same sequence:

1. **Requirements** — Functional and non-functional; clarify ambiguity before you draw a single box.
2. **Constraints** — Scale, SLAs, budget, team. These bound your design.
3. **API** — Contract with callers (REST/gRPC, idempotency, error semantics).
4. **High-level architecture** — Major components and data flow. One diagram.
5. **Deep dives** — Bottlenecks, critical paths, how each part scales.
6. **Failure** — What breaks, replication, failover, backpressure.
7. **Operations** — Deployment, monitoring, migrations, on-call.

Each step feeds the next. In an interview, name the step you’re in so the interviewer can follow. Skipping to “boxes and arrows” without requirements is how good ideas get dismissed as unrealistic.

---

## How to Use This Repo

**Three ways to use it:**

- **Interview in 2–6 weeks** — Start with [12-interview-framework](./12-interview-framework/) so you know *how* to run the interview. Then hit the concept files, then timed practice with [system-design-problems](./system-design-problems/).
- **Look something up** — Use the [Concept Index](#concept-index-all-12-areas) and [Quick Reference](#quick-reference-where-to-go-when) below to jump to a topic or problem.
- **Deep study** — Follow a [level-based path](#study-paths-by-level), read everything, do all problems, and keep a short “trade-off journal” for each design.

**Five rules to keep in mind:**

1. **Requirements first.** Never assume scope. Clarify before designing.
2. **Numbers anchor decisions.** Use the [Core Reference Numbers](#core-reference-numbers-memorize-these) to justify scale, latency, and throughput.
3. **Trade-offs over silver bullets.** There is no perfect solution. Show you understand the costs.
4. **Failure is normal.** Design for it explicitly—replication, failover, graceful degradation.
5. **Evolution over perfection.** Start simple, find bottlenecks, evolve. Don’t over-engineer the whiteboard.

---

## Study Paths by Level

### Senior (L5) — about 3 weeks

**Goal:** A solid framework, core concepts, and 10–15 problems so you can pass typical L5 system design rounds.

| Week | What to study | How to practice |
|------|----------------|-----------------|
| **1** | [01-system-design-foundations](./01-system-design-foundations/) (all five). [02-core-infrastructure](./02-core-infrastructure/) (load balancers, API gateway, CDN). | After each concept: “How would I use this in a real design?” |
| **2** | [03-distributed-systems-concepts](./03-distributed-systems-concepts/) (all). [05-data-management](./05-data-management/) (sharding, indexing, when to use what). | Pick 3–5 problems from [system-design-problems](./system-design-problems/) that lean on replication/sharding. |
| **3** | [12-interview-framework](./12-interview-framework/). | 10–15 full problems, 25–35 min each, out loud or with a peer. |

**Reality check:** Can you explain CAP, consistency models, replication, and partitioning in one minute each? Can you draw a URL shortener, a rate limiter, and one feed-style system (Twitter or Instagram) in under 30 minutes?

---

### Staff (L6) — about 3 weeks (after you have the L5 base)

**Goal:** Broader and deeper—reliability, observability, security, capacity; all problems; articulate trade-offs clearly.

| Week | What to study | How to practice |
|------|----------------|-----------------|
| **1** | Re-read [01-system-design-foundations](./01-system-design-foundations/) quickly. [04-scalability-patterns](./04-scalability-patterns/) (CQRS, event sourcing, saga, BFF). | Map each pattern to a problem (e.g. saga → payment system). |
| **2** | [08-reliability-engineering](./08-reliability-engineering/), [10-observability](./10-observability/), [09-security](./09-security/), [11-capacity-planning](./11-capacity-planning/). | For each problem you’ve done: add failure modes, SLOs, and a rough capacity estimate. |
| **3** | All 30+ problems in [system-design-problems](./system-design-problems/). | For each: write down three trade-offs you’d mention in an interview. Do 2–3 full mock interviews. |

**Reality check:** Can you add circuit breakers, retries, and bulkheads to a design? Define SLO/SLI and alerting? Do back-of-the-envelope QPS, storage, and bandwidth? Discuss auth (OAuth, API keys) and encryption at a high level?

---

### Principal (L7+) — ongoing

**Goal:** Cross-cutting concerns, org-scale trade-offs, migrations, build vs buy, cost vs complexity.

- **Cross-cutting:** Revisit [04-scalability-patterns](./04-scalability-patterns/), [08-reliability-engineering](./08-reliability-engineering/), [10-observability](./10-observability/) with an “org-wide” lens. How would you roll out a new pattern (e.g. event sourcing) across teams?
- **Case studies:** Use [system-design-problems](./system-design-problems/) as product scenarios. For 5–10 of them: “How would we migrate from the current system to this design? What’s the rollout and risk?”
- **Build vs buy:** Tie [02-core-infrastructure](./02-core-infrastructure/), [05-data-management](./05-data-management/), [09-security](./09-security/) to “build vs buy” and TCO. For each major component (API gateway, DB, cache, auth): when would you buy vs build?

**Reality check:** Can you lead a whiteboard discussion on migration strategy, team boundaries, and cost/complexity trade-offs without disappearing into implementation details?

---

## Interview Meta-Skills

- **Clarify before designing.** Ask about scale, latency, consistency, and must-haves vs nice-to-haves.
- **Think out loud.** Your interviewer can’t read your mind. Narrate your process.
- **Start high-level, then go deep.** One component at a time. Don’t jump into implementation.
- **Use the whiteboard.** Boxes, arrows, data flow. Label everything.
- **State trade-offs.** e.g. “We could use strong consistency here, but it would hurt latency and availability; for a feed we’ll choose eventual consistency.”
- **Bring up failure and operations.** Replication, failover, circuit breakers, monitoring, rollbacks. Show you think beyond the happy path.

---

## Core Reference Numbers (Memorize These)

Use these to anchor every design and every interview answer.

**Latency**

| Operation | Order of magnitude |
|-----------|--------------------|
| L1 cache | ~1 ns |
| L2 cache | ~4 ns |
| RAM | ~100 ns |
| SSD random read | ~100 µs |
| HDD random read | ~10 ms |
| Network RTT, same DC | ~0.5 ms |
| Network RTT, cross-region | ~100 ms |

**Storage**

| Item | Size |
|------|------|
| 1 char | 1 byte |
| 1 int | 4 bytes |
| 1 UUID | 16 bytes (raw) or 36 (string) |
| 1 timestamp | 8 bytes |
| 1M rows × 100 bytes | 100 MB |
| 1B rows × 100 bytes | 100 GB |

**Throughput**

| Resource | Rough throughput |
|----------|------------------|
| Gigabit NIC | ~125 MB/s |
| 10 Gbps NIC | ~1.25 GB/s |
| SSD | ~500 MB/s read, ~200 MB/s write |
| HDD | ~100 MB/s sequential, ~100 IOPS random |

**Scale and protocols**

| Concept | Typical value |
|---------|----------------|
| Single server | 10K–100K req/s (depends on work) |
| DNS TTL | 300 s (5 min) |
| TCP setup | 1 RTT (with TLS: 2–3 RTT) |
| HTTP/2 | N requests over one TCP connection |

When you propose a component, tie it to a number: e.g. “We need a cache because the DB is ~1–5 ms and we want p99 under 100 ms.”

---

## Concept Index (All 12 Areas)

Use this to find topics and track what you’ve covered. **Bold** = high impact for interviews.

| # | Area | Key topics | Note |
|---|------|------------|------|
| 01 | [01-system-design-foundations](./01-system-design-foundations/) | **What is system design**, **latency/throughput/availability**, **CAP**, **consistency models**, **scaling strategies** | Start here. Non-negotiable. |
| 02 | [02-core-infrastructure](./02-core-infrastructure/) | **Load balancers**, **API gateways**, service meshes, **CDNs** | Every design touches these. |
| 03 | [03-distributed-systems-concepts](./03-distributed-systems-concepts/) | **Consensus (Raft/Paxos)**, **replication**, **consistent hashing**, **distributed transactions** | Core distributed systems. |
| 04 | [04-scalability-patterns](./04-scalability-patterns/) | **CQRS**, event sourcing, **saga**, BFF, sidecar | When to use each pattern. |
| 05 | [05-data-management](./05-data-management/) | **Sharding**, indexing, OLTP vs OLAP, time-series, search | Drives scalability and latency. |
| 06 | [06-networking](./06-networking/) | TCP/UDP, DNS, TLS, **HTTP/2**, **gRPC**, **WebSockets** | Protocol choices matter at scale. |
| 07 | [07-performance-engineering](./07-performance-engineering/) | Profiling, query optimization, **caching**, latency budgets | p99 and tail latency. |
| 08 | [08-reliability-engineering](./08-reliability-engineering/) | **Circuit breakers**, **retries**, bulkheads, chaos engineering | Failure handling. |
| 09 | [09-security](./09-security/) | **AuthN/AuthZ**, **OAuth**, encryption, secrets, API security | Required for most designs. |
| 10 | [10-observability](./10-observability/) | **Metrics**, **logs**, **traces**, alerting, **SLO/SLA/SLI** | How you know the system is healthy. |
| 11 | [11-capacity-planning](./11-capacity-planning/) | **QPS/storage/bandwidth estimation**, cost modeling | Back-of-the-envelope. |
| 12 | [12-interview-framework](./12-interview-framework/) | **Interview strategy**, communication, **trade-off reasoning** | How to run the interview. |

---

## System Design Problem Index (30+)

Aim for at least 10–15 (L5) or all 30+ (L6). Use this table to pick by concept or to see what each problem exercises.

| # | Problem | Key concepts |
|---|---------|--------------|
| 01 | [URL Shortener](./system-design-problems/01-url-shortener.md) | Hashing, KV store, redirects |
| 02 | [Rate Limiter](./system-design-problems/02-rate-limiter.md) | Token bucket, Redis, sliding window |
| 03 | [Distributed Cache](./system-design-problems/03-distributed-cache.md) | Consistent hashing, eviction, replication |
| 04 | [Twitter/X](./system-design-problems/04-twitter.md) | Fanout, timeline, social graph |
| 05 | [Instagram](./system-design-problems/05-instagram.md) | Media storage, feeds, CDN |
| 06 | [YouTube](./system-design-problems/06-youtube.md) | Video transcoding, streaming, CDN |
| 07 | [WhatsApp](./system-design-problems/07-whatsapp.md) | Real-time messaging, presence, e2e encryption |
| 08 | [Uber](./system-design-problems/08-uber.md) | Geospatial indexing, matching, surge pricing |
| 09 | [Google Docs](./system-design-problems/09-google-docs.md) | OT/CRDT, real-time collaboration |
| 10 | [Dropbox](./system-design-problems/10-dropbox.md) | File sync, deduplication, chunking |
| 11 | [Netflix](./system-design-problems/11-netflix.md) | Video delivery, CDN, recommendation |
| 12 | [Reddit](./system-design-problems/12-reddit.md) | Voting, ranking, communities |
| 13 | [Spotify](./system-design-problems/13-spotify.md) | Audio streaming, playlist, recommendation |
| 14 | [Facebook News Feed](./system-design-problems/14-facebook-news-feed.md) | Ranking, fanout, social graph |
| 15 | [Search Engine](./system-design-problems/15-search-engine.md) | Crawling, indexing, ranking |
| 16 | [Ad Click Tracking](./system-design-problems/16-ad-click-tracking.md) | High-write, aggregation, fraud detection |
| 17 | [Notification System](./system-design-problems/17-notification-system.md) | Push, email, SMS, fan-out |
| 18 | [Log Aggregation](./system-design-problems/18-log-aggregation.md) | ELK-style, streaming, retention |
| 19 | [Metrics Monitoring](./system-design-problems/19-metrics-monitoring.md) | Time-series DB, aggregation, alerting |
| 20 | [API Gateway](./system-design-problems/20-api-gateway.md) | Auth, routing, rate limiting, observability |
| 21 | [CDN](./system-design-problems/21-cdn.md) | Edge nodes, cache invalidation, anycast |
| 22 | [Payment System](./system-design-problems/22-payment-system.md) | ACID, idempotency, fraud, reconciliation |
| 23 | [Web Crawler](./system-design-problems/23-web-crawler.md) | BFS, politeness, dedup, frontier |
| 24 | [Autocomplete](./system-design-problems/24-autocomplete.md) | Trie, prefix search, ranking |
| 25 | [Chat System](./system-design-problems/25-chat-system.md) | WebSocket, message delivery, group chat |
| 26 | [Recommendation System](./system-design-problems/26-recommendation-system.md) | Collaborative filtering, embeddings, serving |
| 27 | [Real-time Analytics](./system-design-problems/27-realtime-analytics.md) | Stream processing, Flink-style, aggregation |
| 28 | [Feature Flag System](./system-design-problems/28-feature-flags.md) | Targeting, rollout, consistency |
| 29 | [Distributed Job Scheduler](./system-design-problems/29-job-scheduler.md) | Cron, leader election, at-least-once |
| 30 | [Distributed Lock Service](./system-design-problems/30-distributed-lock.md) | Fencing tokens, lease, Redlock |

---

## Self-Assessment

**Concepts**

- [ ] 01 – System design foundations (all 5)
- [ ] 02 – Core infrastructure
- [ ] 03 – Distributed systems concepts (all 4)
- [ ] 04 – Scalability patterns
- [ ] 05 – Data management
- [ ] 06 – Networking
- [ ] 07 – Performance engineering
- [ ] 08 – Reliability engineering
- [ ] 09 – Security
- [ ] 10 – Observability
- [ ] 11 – Capacity planning
- [ ] 12 – Interview framework

**Problems**

- [ ] 10 problems timed (L5 target)
- [ ] 15 problems timed (L5 strong)
- [ ] 30+ problems with trade-off notes (L6)
- [ ] 2+ full mock interviews with feedback

**Basics**

- [ ] Core latency numbers memorized
- [ ] Core storage/throughput numbers memorized
- [ ] Can run the 7-step design process without looking

---

## Quick Reference: Where to Go When

| I want to… | Go to |
|------------|--------|
| Understand the design process and requirements | [01-what-is-system-design](./01-system-design-foundations/01-what-is-system-design.md) |
| Justify latency, throughput, availability | [02-latency-throughput-availability](./01-system-design-foundations/02-latency-throughput-availability.md) |
| Reason about CAP and consistency | [03-cap-theorem](./01-system-design-foundations/03-cap-theorem.md), [04-consistency-models](./01-system-design-foundations/04-consistency-models.md) |
| Scale a system (vertical/horizontal, caching, async) | [05-scaling-strategies](./01-system-design-foundations/05-scaling-strategies.md) |
| Design load balancing, API gateway, or CDN | [02-core-infrastructure](./02-core-infrastructure/) |
| Understand consensus, replication, partitioning | [03-distributed-systems-concepts](./03-distributed-systems-concepts/) |
| Choose patterns (CQRS, saga, event sourcing) | [04-scalability-patterns](./04-scalability-patterns/) |
| Design for failure (retries, circuit breakers) | [08-reliability-engineering](./08-reliability-engineering/) |
| Define SLOs and observability | [10-observability](./10-observability/) |
| Do back-of-the-envelope capacity | [11-capacity-planning](./11-capacity-planning/) |
| Improve how I run the interview | [12-interview-framework](./12-interview-framework/) |
| Practice a specific product/system | [system-design-problems](./system-design-problems/) |

---

Treat this as your map. Adjust the study paths to your timeline and level; the structure stays the same: **concepts → problems → mocks → refine.**
