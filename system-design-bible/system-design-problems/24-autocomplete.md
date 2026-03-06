# System Design: Autocomplete

## Problem

Design an autocomplete (typeahead) system: as the user types a query, return a list of top suggestions (e.g. search queries, product names) that match the prefix. Low latency (tens of ms) and relevance (popularity, recency).

## Requirements

**Functional:** Given prefix, return top K suggestions (e.g. 10); suggestions match prefix; ranked by relevance (popularity, recency, or learning). Support multiple languages and correction (optional).

**Non-functional:** Low latency (p99 < 50 ms); high QPS; suggestions update periodically (e.g. daily) or near real-time.

## High-Level Design

```
  User types "net" → API → Autocomplete service → Trie or prefix index → Top-K by score
                                                                              ↓
  Return ["netflix", "network", "net worth", ...]
```

- **Data structure:** Trie (prefix tree): each node = character; path from root = prefix; store at nodes or leaves: candidate strings and score (e.g. query count, recency). Or: inverted index keyed by prefix (prefix → list of (phrase, score)); sorted by score; top K.
- **Query:** Lookup prefix in trie or index; get candidates; return top K by score. Cache hot prefixes (Redis) for latency.
- **Update:** Batch: aggregate search logs (query, count); build/rebuild trie or index; deploy. Near real-time: stream updates; merge into structure or rebuild incrementally.

## Key Components

- **Trie or prefix index:** In-memory for latency; shard by first character or prefix range if too large. Each leaf or bucket holds (phrase, score); sorted for top K.
- **Scoring:** Count of past queries (popularity); recency; or learned ranking (CTR, relevance model). Precomputed and stored with phrase.
- **Cache:** Redis: prefix → top K list; TTL (e.g. 5 min). Reduce load on main structure.
- **Update pipeline:** Log queries; aggregate (e.g. daily); compute scores; rebuild trie/index; swap or rolling deploy.

## Data Model

- **Trie:** Node: char, children, optional (phrase, score) at terminal. Or key-value: prefix → [(phrase, score)].
- **Query log (for building):** query, count or timestamp. Aggregated to (phrase, score).

## Scale

- Millions of unique phrases; thousands of QPS. Trie in memory (GBs); replicate across instances. Cache covers hot prefixes. Update: batch job; optional streaming for trending.

## Trade-offs

- **Trie vs prefix index:** Trie: compact, flexible for traversal. Prefix index: simpler (prefix → list); good for fixed format.
- **Batch vs real-time update:** Batch: simpler, slight staleness. Real-time: trending phrases faster; more complex.
- **Personalization:** Global top K vs per-user (add user_id to key); increases cache keys and storage.

## Failure & Operations

- **Latency:** P99 under 50 ms; cache and in-memory structure; avoid DB in hot path.
- **Fallback:** If service down, return empty or cached; degrade gracefully.
- **Monitoring:** Latency p50/p99, cache hit rate, update lag, suggestion CTR (if instrumented).
