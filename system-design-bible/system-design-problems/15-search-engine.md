# System Design: Search Engine

## Problem

Design a web search engine: crawl the web, build an index, rank pages, and serve search results for user queries. Scale to billions of pages and millions of queries per second.

## Requirements

**Functional:** Crawl and discover pages; extract and store content; build index (inverted index); rank pages for a query; serve result list with snippets.

**Non-functional:** Fresh index (crawl periodically); low query latency (p99 < 100 ms); high throughput; relevance and ranking quality.

## High-Level Design

```
  Crawlers → URL frontier → Fetch → Parse → Extract links + content → Index pipeline
                                                                           ↓
  Query → Query parser → Index lookup (inverted index) → Rank → Result formatter → Response
```

- **Crawl:** Seed URLs; frontier (queue of URLs to crawl); politeness (rate limit per domain); fetch; parse; extract links (add to frontier) and text; deduplicate (URL canonicalization, fingerprint).
- **Index:** Inverted index: term → list of (doc_id, position or frequency). Built from parsed content; sharded by term hash. Store doc store (doc_id → url, title, snippet).
- **Query:** Parse query (terms); lookup each term in index; intersect or merge postings lists; rank (e.g. TF-IDF, PageRank, learning-to-rank); fetch doc metadata; return top K with snippets.

## Key Components

- **Crawler:** Distributed workers; URL frontier (priority queue); dedup (Bloom filter or store); politeness (robots.txt, delay per host); store raw or parsed content.
- **URL frontier:** Prioritize by importance (PageRank, freshness); partition by domain for politeness.
- **Index builder:** Parse; tokenize; build inverted index; merge segments (batch). Shard index by term.
- **Query service:** Parse query; lookup index shards (parallel); merge results; rank; top K; snippets from doc store.
- **Ranking:** Static (PageRank) + query-dependent (TF-IDF, BM25) + learning-to-rank; run in latency budget.

## Data Model

- **Inverted index:** term → [(doc_id, tf, positions)]; distributed by term.
- **Doc store:** doc_id → url, title, snippet, last_crawled.
- **URL frontier:** url, priority, last_fetched; dedup by canonical URL.

## Scale

- Billions of docs; index petabytes; queries millions/s. Crawl: massive parallelism; politeness and dedup critical. Index: sharded; merge at query time. Ranking: precompute PageRank; query-time scoring with pruning.

## Trade-offs

- **Freshness vs coverage:** Crawl frequently (fresh) vs broadly (coverage); prioritize by importance and change rate.
- **Index update:** Batch rebuild vs incremental; incremental is complex; batch is simpler, stale until next build.
- **Ranking:** Quality vs latency; heavy features in second stage; light features in first stage.

## Failure & Operations

- **Crawler:** Respect robots.txt and rate limits; handle failures and retries; avoid duplicate URLs.
- **Index:** Replication for availability; backup and restore; versioned deploys.
- **Monitoring:** Crawl rate, index size, query latency, relevance metrics (CTR, A/B).
