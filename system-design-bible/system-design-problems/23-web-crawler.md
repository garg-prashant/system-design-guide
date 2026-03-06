# System Design: Web Crawler

## Problem

Design a web crawler that discovers and fetches web pages at scale: URL frontier, politeness (rate limit per domain), deduplication, and storage of content for indexing or analysis.

## Requirements

**Functional:** Seed URLs; discover new URLs from fetched pages; fetch pages (HTTP); respect robots.txt and rate limits; deduplicate URLs; store raw or parsed content; prioritize URLs (optional).

**Non-functional:** High throughput (millions of pages/day); politeness (don’t overload hosts); scalable and fault-tolerant; handle failures and retries.

## High-Level Design

```
  Seed URLs → URL Frontier (queue, per-domain priority) → Dedup (Bloom filter / store)
                                                                  ↓
  Fetcher workers → HTTP fetch → Parse → Extract links → Add to frontier (if not seen)
                    ↓
  Content store (raw or parsed) for indexing
```

- **Frontier:** Queue of URLs to crawl; partition by domain for politeness (one worker or rate-limited queue per domain). Priority: importance (PageRank, sitemap) or FIFO.
- **Dedup:** Before add to frontier: canonicalize URL (lowercase, strip fragment); check if already seen (Bloom filter + DB for persistence). Skip if seen.
- **Fetch:** Worker picks URL from frontier; check robots.txt (cache per domain); rate limit (e.g. 1 req/s per domain); fetch; parse; extract links; add new links to frontier (after dedup); store content.
- **Politeness:** Delay between requests to same domain; respect Crawl-delay if in robots.txt; back off on 429/5xx.

## Key Components

- **URL frontier:** Distributed queue (e.g. Kafka, or custom); partitions by host; priority queue per host or global. Scale with workers.
- **Dedup store:** Bloom filter (fast, may have false positives; confirm with DB) + DB (exact) for URLs already crawled. Or only DB with index on canonical URL.
- **Fetcher:** HTTP client; timeout; retry with backoff; parse HTML; extract <a href>; normalize to absolute URLs.
- **Robots.txt:** Fetch and cache per domain; check before each URL of that domain; respect Disallow and Crawl-delay.
- **Content store:** Object store or DB; key = URL; value = raw HTML or parsed text; optional checksum for dedup (same content).

## Data Model

- **frontier:** url (canonical), priority, last_fetched, host. Queue structure.
- **seen_urls:** url (canonical); for dedup.
- **content:** url, content_hash, raw_or_parsed, fetched_at.

## Scale

- Billions of URLs; millions of pages/day. Frontier: many partitions; workers per partition. Dedup: Bloom filter in memory; DB for persistence. Politeness limits concurrency per host.

## Trade-offs

- **Breadth-first vs priority:** BFS: simple. Priority: crawl important pages first; need scoring (e.g. PageRank, sitemap).
- **Dedup:** In-memory Bloom + DB vs only DB. Bloom reduces DB load; DB is source of truth.
- **Storage:** Store raw vs only parsed text; raw for replay, parsed for index; retention policy.

## Failure & Operations

- **Fetcher failures:** Retry with backoff; skip after N failures; dead-letter queue for manual review.
- **Politeness:** Monitor request rate per domain; alert on blocks (429) or bans.
- **Monitoring:** Crawl rate, frontier size, dedup hit rate, fetch errors, content store growth.
