# Content Delivery Networks (CDNs)

## Role

Cache static (and some dynamic) content at **edge locations** close to users. Reduces origin load, lowers latency, and improves availability.

```
  User (Tokyo)  ──→  Edge (Tokyo)  ── cache hit  →  response
                            │
                            └── cache miss  ──→  Origin (US)  ──→  response + cache store
```

---

## How It Works

1. **First request (miss):** Edge fetches from origin (e.g. your server or S3), caches, returns to user.
2. **Subsequent requests (hit):** Edge serves from cache; origin not contacted.
3. **TTL:** Cached object expires after TTL (e.g. 24h for images); next request triggers revalidation or refetch.

---

## What to Put in a CDN

| Content type | Typical use |
|--------------|-------------|
| **Static assets** | JS, CSS, images, fonts (long TTL). |
| **Media** | Video/audio segments (HLS/DASH), images (thumbnails). |
| **Downloadable files** | Software binaries, docs. |
| **Dynamic (careful)** | Some CDNs support edge compute (Lambda@Edge, Workers) for personalized or dynamic content; more complex. |

---

## Cache Invalidation

When origin content changes, cache must be updated.

- **TTL only:** Wait for expiry. Simple but stale content until TTL.
- **Purge (invalidation):** Tell CDN to remove specific URLs or prefixes. Next request refetches from origin.
- **Versioned URLs:** e.g. `app.abc123.js` — new version = new URL; no purge needed. Preferred when possible.

**Interview:** “We use versioned filenames so we never purge; we only purge for emergency fixes.”

---

## Key CDN Concepts

| Concept | Meaning |
|---------|---------|
| **Edge / PoP** | Data center close to users; cache lives here. |
| **Origin** | Your server or bucket that the CDN fetches from on miss. |
| **Anycast** | Same IP announced from many locations; BGP routes user to nearest edge. |
| **Origin shield** | Optional layer between edges and origin; reduces origin load when many edges miss. |

---

## Metrics That Matter

- **Cache hit ratio** — % of requests served from edge. Target high for static (e.g. >90%).
- **Origin load** — Requests that hit origin. Lower is better.
- **Latency (p50, p99)** — Should drop for cached content vs origin-only.

---

## Failure Modes

- **Origin down:** CDN can serve stale cached content (if TTL allows); improves availability.
- **CDN outage:** Fallback to origin (if you have one); possible overload. Some DNS failover to backup CDN or origin.
- **Bad purge:** Purge too much → origin thundering herd. Purge by prefix with care.

---

## Trade-offs (Interview)

| Decision | Trade-off |
|----------|-----------|
| Long TTL | Fewer origin hits, lower cost, but stale content longer. |
| Short TTL | Fresher content, more origin load and latency on miss. |
| Versioned URLs | No purge needed, but build/deploy must generate new names. |
| Dynamic at edge | Lower latency for personalization; more complexity and cost. |
