# System Design: Twitter / X

## Problem

Design a system that allows users to post short messages (tweets), follow other users, and view a timeline of tweets from people they follow (and optionally algorithmic ranking).

## Requirements

**Functional:** Post tweet; follow/unfollow; view user timeline (tweets by that user); view home timeline (tweets from followees, chronological or ranked); search (if in scope).

**Non-functional:** Low latency for timeline (p99 < 200 ms); handle viral users (millions of followers); high read volume vs write volume.

## High-Level Design

```
  Clients → API → Tweet service, Timeline service, Social graph service
                        ↓              ↓                    ↓
                    Tweet store    Timeline store      Graph store (followers/following)
```

- **Write path:** Post tweet → store tweet; update fanout (if push) or leave for read-time (pull).
- **Read path – user timeline:** Fetch tweets by user_id, ordered by time.
- **Read path – home timeline:** Push model: pre-compute and store “timeline” per user; read from timeline store. Pull model: get followees, fetch their tweets, merge and sort (expensive for large followee sets).

## Key Components

- **Tweet service:** Store tweets (tweet_id, user_id, text, timestamp). Shard by user_id or tweet_id.
- **Social graph:** Followers and following per user. Graph DB or relational; cached for hot users.
- **Timeline service:** Push: on each tweet, fan out to all followers’ timeline caches (async). Pull: on timeline read, get followees, fetch recent tweets, merge. Hybrid: push for normal users, pull for celebrities (to avoid write storm).
- **Feed ranking (optional):** Scoring model (recency, engagement); run offline or in a separate ranking service; merge with real-time tweets.

## Data Model

- **tweets:** tweet_id, user_id, text, created_at. Index: user_id, created_at.
- **follows:** follower_id, followee_id. Index: follower_id (for “who I follow”); followee_id (for “my followers”).
- **timelines (push):** user_id, list of (tweet_id, timestamp) or pointer to tweet store. Stored per user; truncated (e.g. last 800).

## Scale

- Assume 300M users; 500M tweets/day → ~6K writes/s. Timeline reads 20× writes → 120K reads/s. Celebrities with 50M followers → fanout 50M timeline writes (use pull or hybrid).
- Shard tweets by user_id; timelines by user_id; graph by user_id. Cache hot timelines and graph in Redis.

## Trade-offs

- **Push vs pull:** Push: fast read, heavy write for high-follower users. Pull: simple write, slow read for users following many. Hybrid: push for most, pull for celebrities.
- **Consistency:** Timeline eventually consistent; accept “new tweet appears in a few seconds.”
- **Ranking:** Chronological is simple; ranking improves engagement but adds ML and latency.

## Failure & Operations

- **Fanout queue:** Async workers for fanout; backpressure if backlog grows; drop or defer for celebrities (pull path).
- **Caching:** Timeline cache per user; invalidation on new tweets (append or refresh). Cache stampede: single-flight or TTL.
- **Monitoring:** Timeline latency, fanout lag, tweet write throughput, cache hit rate.
