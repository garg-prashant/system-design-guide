# System Design: CDN

## Problem

Design a Content Delivery Network: cache content at edge locations close to users, reduce origin load and latency, and support cache invalidation and optional dynamic content at the edge.

## Requirements

**Functional:** Cache static assets (and optionally dynamic) at edge; serve on cache hit; fetch from origin on miss; invalidate (purge) by URL or prefix; optional edge compute (custom logic at edge).

**Non-functional:** High cache hit ratio; low latency (single-digit ms at edge); high availability; minimal origin load.

## High-Level Design

```
  User request → DNS (resolve to nearest edge) → Edge PoP
                      ↓
  Cache hit → return from edge
  Cache miss → fetch from origin → store in cache → return
```

- **Request:** User resolves hostname via DNS (anycast or geo DNS) → IP of nearest edge. Edge receives request; lookup cache by URL (and optionally vary by header).
- **Hit:** Return cached object; no origin call.
- **Miss:** Edge fetches from origin (your server or object store); caches with TTL; returns to user. Optional: origin shield (extra layer) to coalesce requests from many edges to origin.
- **Invalidation:** Purge API: “invalidate URL or prefix”; edge removes or marks stale; next request refetches from origin.
- **Versioned URLs:** e.g. /app.abc123.js; new version = new URL; no purge needed.

## Key Components

- **Edge servers:** PoPs in many regions; cache store (memory/disk); HTTP server; optional edge compute (Lambda@Edge, Workers).
- **Origin:** Your application or object store; only hit on miss or purge.
- **DNS:** Geo or anycast so client reaches nearest edge.
- **Purge/invalidation:** API to invalidate by URL or prefix; propagate to edges (eventual).
- **Origin shield (optional):** Layer between edges and origin; one request per object from shield to origin when many edges miss.

## Data Model

- **Cache entry:** Key = URL + vary headers; value = response body, headers; TTL, size. Stored at edge.
- **Config:** TTL defaults, origin URL, purge rules.

## Scale

- Millions of requests/s; origin sees only miss traffic. Hit ratio target 90%+ for static. Scale edges by adding PoPs; scale origin with shield and capacity.

## Trade-offs

- **TTL:** Long TTL = fewer origin hits, more staleness. Short TTL = fresher, more load. Versioned URLs avoid trade-off for immutable assets.
- **Purge scope:** Purge by URL (precise) vs prefix (broad). Broad purge can cause thundering herd to origin.
- **Dynamic at edge:** Enables personalization at edge; more complex and cost; use for lightweight logic.

## Failure & Operations

- **Origin down:** Serve stale from cache (if TTL allows); improve availability. Have fallback origin or multi-origin if critical.
- **Purge storm:** Avoid purging huge prefixes at once; use versioned URLs to avoid purge.
- **Monitoring:** Hit ratio, origin request rate, latency p50/p99 per PoP, error rate.
