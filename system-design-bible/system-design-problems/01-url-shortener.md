# System Design: URL Shortener

## Problem

Design a service that takes a long URL and returns a short URL (e.g. `https://short.com/abc12`). When users visit the short URL, they are redirected to the original long URL.

## Requirements

**Functional:** Generate short URL from long URL; redirect short → long; optionally track click count; custom short codes (if scope included).

**Non-functional:** High availability; low redirect latency (p99 < 100 ms); short URLs are durable and redirect for years.

## High-Level Design

```
  Client → API / Web → Shortener Service → KV Store (short → long)
                ↓
         Redirect (HTTP 302) to long URL
```

- **Write path:** POST long URL → generate or accept short code → store mapping → return short URL.
- **Read path:** GET short URL → lookup long URL → 302 redirect to long URL.

## Key Components

- **API server:** Accept create and redirect requests; stateless; horizontal scaling.
- **Storage:** KV store (short key → long URL, optional metadata). Use consistent hashing if sharding.
- **Short code generation:** Base62 encode of auto-increment ID, or hash(long URL + salt) truncated to avoid collisions. If collision, retry with new salt or use longer length.
- **Redirect:** 302 (temporary) so we can change destination; 301 (permanent) if we never change. 302 allows analytics.

## Data Model

- **short_code** (PK) → long_url, created_at, optional user_id, optional expires_at.
- **Index:** short_code (primary lookup). Optional: long_url for dedup “same long → same short” if desired.

## Scale

- Assume 100M new short URLs/month → ~40 writes/s. 10B redirects/month → ~4K reads/s. Storage: 100M × (8 + 200) bytes ≈ 20 GB for mappings.
- Single KV store (e.g. Redis + DB for durability, or DynamoDB) can handle this; shard by short_code hash if needed.

## Trade-offs

- **Hash vs auto-increment:** Hash: no single counter; need collision handling. Auto-increment: simple, but counter is a bottleneck (use range allocation or DB sequence).
- **302 vs 301:** 302 = we can track and change; 301 = cached by browsers, fewer hits to our system.
- **Storage:** KV/NoSQL fits; RDBMS with index on short_code also works.

## Failure & Operations

- **Availability:** Replicate KV store; multiple API instances behind LB. Redirect is read-heavy; cache short→long in Redis with TTL for hot URLs.
- **Durability:** Persist mapping in DB; Redis as cache. Don’t lose mappings.
- **Monitoring:** Redirect latency, error rate, storage size; alert on high 4xx/5xx.
