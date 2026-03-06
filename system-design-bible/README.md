# System Design Bible

A comprehensive reference repository for Senior, Staff, and Principal engineer interview preparation at top-tier technology companies.

This repository is written for **experienced engineers** who need depth — not beginner tutorials. Every topic includes trade-offs, failure modes, production practices, and interview discussion points.

---

## Repository Structure

| Directory | Contents |
|-----------|----------|
| [01-system-design-foundations](./01-system-design-foundations/) | Core concepts, distributed systems theory, data management |
| [02-core-infrastructure](./02-core-infrastructure/) | Load balancers, API gateways, service meshes, CDNs |
| [03-distributed-systems-concepts](./03-distributed-systems-concepts/) | Consensus, replication, partitioning, clocks, transactions |
| [04-scalability-patterns](./04-scalability-patterns/) | CQRS, event sourcing, saga, BFF, sidecar patterns |
| [05-data-management](./05-data-management/) | Sharding, indexing, OLTP vs OLAP, time-series, search |
| [06-networking](./06-networking/) | TCP/UDP, DNS, TLS, HTTP/2, gRPC, WebSockets |
| [07-performance-engineering](./07-performance-engineering/) | Profiling, query optimization, caching, latency budgets |
| [08-reliability-engineering](./08-reliability-engineering/) | Circuit breakers, retries, bulkheads, chaos engineering |
| [09-security](./09-security/) | AuthN/AuthZ, OAuth, encryption, secrets, API security |
| [10-observability](./10-observability/) | Metrics, logs, traces, alerting, SLO/SLA/SLI |
| [11-capacity-planning](./11-capacity-planning/) | QPS/storage/bandwidth estimation, cost modeling |
| [12-interview-framework](./12-interview-framework/) | Interview strategy, communication, trade-off reasoning |
| [system-design-problems](./system-design-problems/) | 30+ complete end-to-end system designs |

---

## How to Use This Repository

**For interview prep:** Start with `12-interview-framework` to learn the meta-skill of *how* to approach any design problem, then study the concept files, and finally practice with `system-design-problems`.

**As a reference:** Jump directly to the topic or problem you need.

**Study path by level:**

```
Senior Engineer (L5):
  Week 1: Foundations + Core Infrastructure
  Week 2: Distributed Systems Concepts + Data Management
  Week 3: 10-15 System Design Problems

Staff Engineer (L6):
  Week 1: All foundations (fast pass) + Scalability Patterns
  Week 2: Reliability + Observability + Security + Capacity Planning
  Week 3: All 30+ Problems + Trade-off deep dives

Principal Engineer (L7+):
  Focus on: Cross-cutting concerns, org-scale trade-offs,
  industry case studies, build vs buy decisions,
  migration strategies, and cost/complexity trade-offs
```

---

## Core Reference Numbers (Memorize These)

```
Latency numbers:
  L1 cache:          ~1 ns
  L2 cache:          ~4 ns
  RAM access:        ~100 ns
  SSD random read:   ~100 us
  HDD random read:   ~10 ms
  Network RTT same DC: ~0.5 ms
  Network RTT cross-region: ~100 ms

Storage:
  1 char = 1 byte
  1 int = 4 bytes
  1 UUID = 16 bytes (raw) or 36 bytes (string)
  1 timestamp = 8 bytes
  1 million rows * 100 bytes = 100 MB
  1 billion rows * 100 bytes = 100 GB

Throughput:
  Gigabit NIC: ~125 MB/s
  10 Gigabit NIC: ~1.25 GB/s
  SSD: ~500 MB/s read, ~200 MB/s write
  HDD: ~100 MB/s sequential, ~100 IOPS random

Scale:
  1 server handles: 10K-100K req/sec (depending on work)
  DNS TTL: 300s (5 min) typical
  TCP connection setup: 1 RTT (with TLS: 2-3 RTT)
  HTTP/2 multiplexing: N requests over 1 TCP connection
```

---

## System Design Problem Index

| # | Problem | Key Concepts |
|---|---------|-------------|
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
| 18 | [Log Aggregation](./system-design-problems/18-log-aggregation.md) | ELK stack, streaming, retention |
| 19 | [Metrics Monitoring](./system-design-problems/19-metrics-monitoring.md) | Time-series DB, aggregation, alerting |
| 20 | [API Gateway](./system-design-problems/20-api-gateway.md) | Auth, routing, rate limiting, observability |
| 21 | [CDN](./system-design-problems/21-cdn.md) | Edge nodes, cache invalidation, anycast |
| 22 | [Payment System](./system-design-problems/22-payment-system.md) | ACID, idempotency, fraud, reconciliation |
| 23 | [Web Crawler](./system-design-problems/23-web-crawler.md) | BFS, politeness, dedup, frontier |
| 24 | [Autocomplete](./system-design-problems/24-autocomplete.md) | Trie, prefix search, ranking |
| 25 | [Chat System](./system-design-problems/25-chat-system.md) | WebSocket, message delivery, group chat |
| 26 | [Recommendation System](./system-design-problems/26-recommendation-system.md) | Collaborative filtering, embeddings, serving |
| 27 | [Real-time Analytics](./system-design-problems/27-realtime-analytics.md) | Stream processing, Flink, aggregation |
| 28 | [Feature Flag System](./system-design-problems/28-feature-flags.md) | Targeting, rollout, consistency |
| 29 | [Distributed Job Scheduler](./system-design-problems/29-job-scheduler.md) | Cron, leader election, at-least-once |
| 30 | [Distributed Lock Service](./system-design-problems/30-distributed-lock.md) | Fencing tokens, lease, Redlock |

---

## Key Principles for Senior+ Interviews

1. **Requirements first, always.** Never assume scope. Clarify before designing.
2. **Numbers anchor decisions.** Every design choice should be justified by scale.
3. **Trade-offs over silver bullets.** No perfect solution exists. Show you understand the costs.
4. **Failure is normal.** Design for failure explicitly, not as an afterthought.
5. **Operational concerns matter.** Deployment, monitoring, migrations, on-call burden.
6. **Evolution over perfection.** Start simple, identify bottlenecks, evolve incrementally.
