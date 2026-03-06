# System Design: Facebook News Feed

## Problem

Design a social news feed: users see a ranked stream of posts from friends and followed pages. Posts can be text, image, video; support likes, comments, shares; feed is algorithmically ranked.

## Requirements

**Functional:** Post (text, photo, video); follow friends/pages; feed (ranked); like, comment, share; notifications (optional).

**Non-functional:** Low latency (feed load p99 < 200 ms); ranking reflects relevance and recency; handle viral content and high read volume.

## High-Level Design

```
  Clients → API → Post service, Graph service, Feed service, Ranking service
                        ↓            ↓             ↓              ↓
                    Post store   Graph (friends)  Feed cache   Ranking model
```

- **Post:** Create post; store with author, content, timestamp; fanout or pull for feed.
- **Social graph:** Friends, follow (pages); stored and cached; used to know whose posts to consider for feed.
- **Feed generation:** Candidate retrieval (posts from friends/pages in time window); ranking (ML model: engagement, recency, affinity); truncate to N; store in feed cache per user (push) or compute on read (pull/hybrid).
- **Ranking:** Features: post age, relationship strength, past engagement, type (photo vs text). Model: neural or gradient boosting; run offline for batch or online for real-time; latency budget ~50–100 ms for ranking step.

## Key Components

- **Post service:** Store posts; media in object store + CDN; index by user_id and time for candidate retrieval.
- **Graph service:** Friends, follows; cache hot; query “friends of user” for candidate set.
- **Feed service:** Push: on new post, add to followers’ feed cache (or only for small accounts). Pull: get candidates, rank, return. Hybrid: push for most, pull for heavy posters.
- **Ranking service:** Load model; score candidates; return ordered list; cache result for short TTL.
- **Like/Comment:** Store in DB; update counts; invalidate or refresh feed cache if needed (or eventual).

## Data Model

- **posts:** post_id, user_id, content, type, created_at.
- **graph:** user_id, friend_id or page_id (follows).
- **feed_cache:** user_id → list of (post_id, score, position); TTL.
- **engagement:** likes, comments; stored per post; aggregated counts.

## Scale

- Billions of users; millions of posts per day; feed reads 10–20× writes. Candidate retrieval: filter by graph and time. Ranking: limit candidates (e.g. 500); rank to 100; cache feed for seconds. Shard posts and graph by user_id.

## Trade-offs

- **Push vs pull vs hybrid:** Same as Twitter/Instagram; hybrid with ranking adds complexity. Push for normal users; pull or lightweight push for celebrities/pages.
- **Ranking latency:** Heavier model = better relevance, higher latency. Lighter model or two-stage (retrieve then rank) to stay in budget.
- **Freshness vs relevance:** Real-time inject (new post from friend) vs pure ranking; blend both.

## Failure & Operations

- **Ranking service down:** Fallback to chronological or cached feed; degrade gracefully.
- **Feed cache:** Invalidate on new post (push); or accept staleness (pull). Monitor cache hit and latency.
- **Monitoring:** Feed latency p50/p99, ranking latency, model quality (CTR, engagement), cache hit rate.
