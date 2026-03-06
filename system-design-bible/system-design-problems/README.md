# System Design Problems

This directory contains 30 end-to-end system design problems. Each file follows a consistent structure: problem statement, requirements, high-level design, key components, data model, scale, trade-offs, and failure/operations.

Use these for timed practice (25–35 min per problem). See the [Complete Guide to System Design Material](../guide-for-sd-material.md) for study paths and the problem index with key concepts.

## Index

| # | Problem | Key concepts |
|---|---------|--------------|
| 01 | [URL Shortener](./01-url-shortener.md) | Hashing, KV store, redirects |
| 02 | [Rate Limiter](./02-rate-limiter.md) | Token bucket, Redis, sliding window |
| 03 | [Distributed Cache](./03-distributed-cache.md) | Consistent hashing, eviction, replication |
| 04 | [Twitter/X](./04-twitter.md) | Fanout, timeline, social graph |
| 05 | [Instagram](./05-instagram.md) | Media storage, feeds, CDN |
| 06 | [YouTube](./06-youtube.md) | Video transcoding, streaming, CDN |
| 07 | [WhatsApp](./07-whatsapp.md) | Real-time messaging, presence, e2e encryption |
| 08 | [Uber](./08-uber.md) | Geospatial indexing, matching, surge pricing |
| 09 | [Google Docs](./09-google-docs.md) | OT/CRDT, real-time collaboration |
| 10 | [Dropbox](./10-dropbox.md) | File sync, deduplication, chunking |
| 11 | [Netflix](./11-netflix.md) | Video delivery, CDN, recommendation |
| 12 | [Reddit](./12-reddit.md) | Voting, ranking, communities |
| 13 | [Spotify](./13-spotify.md) | Audio streaming, playlist, recommendation |
| 14 | [Facebook News Feed](./14-facebook-news-feed.md) | Ranking, fanout, social graph |
| 15 | [Search Engine](./15-search-engine.md) | Crawling, indexing, ranking |
| 16 | [Ad Click Tracking](./16-ad-click-tracking.md) | High-write, aggregation, fraud detection |
| 17 | [Notification System](./17-notification-system.md) | Push, email, SMS, fan-out |
| 18 | [Log Aggregation](./18-log-aggregation.md) | ELK-style, streaming, retention |
| 19 | [Metrics Monitoring](./19-metrics-monitoring.md) | Time-series DB, aggregation, alerting |
| 20 | [API Gateway](./20-api-gateway.md) | Auth, routing, rate limiting, observability |
| 21 | [CDN](./21-cdn.md) | Edge nodes, cache invalidation, anycast |
| 22 | [Payment System](./22-payment-system.md) | ACID, idempotency, fraud, reconciliation |
| 23 | [Web Crawler](./23-web-crawler.md) | BFS, politeness, dedup, frontier |
| 24 | [Autocomplete](./24-autocomplete.md) | Trie, prefix search, ranking |
| 25 | [Chat System](./25-chat-system.md) | WebSocket, message delivery, group chat |
| 26 | [Recommendation System](./26-recommendation-system.md) | Collaborative filtering, embeddings, serving |
| 27 | [Real-time Analytics](./27-realtime-analytics.md) | Stream processing, Flink-style, aggregation |
| 28 | [Feature Flag System](./28-feature-flags.md) | Targeting, rollout, consistency |
| 29 | [Distributed Job Scheduler](./29-job-scheduler.md) | Cron, leader election, at-least-once |
| 30 | [Distributed Lock Service](./30-distributed-lock.md) | Fencing tokens, lease, Redlock |
