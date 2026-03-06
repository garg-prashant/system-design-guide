# System Design: Reddit

## Problem

Design a link and discussion platform: post content, vote (up/down), comment (threaded), organize by communities (subreddits), and rank by hot/new/top.

## Requirements

**Functional:** Create post (link or text); vote; comment (nested); subscribe to subreddits; feed (home = subscribed subreddits); sort by hot, new, top, controversial.

**Non-functional:** High read volume; low latency for feed; voting and ranking must scale; handle viral posts and subreddits.

## High-Level Design

```
  Clients → API → Post service, Vote service, Comment service, Feed service
                        ↓           ↓              ↓              ↓
                    Post store   Vote store    Comment tree   Ranked feed (cache)
```

- **Post:** Store post (title, url, subreddit_id, user_id, score, created_at). Score = upvotes - downvotes (or Wilson score); updated on vote.
- **Vote:** Store vote (user_id, post_id, direction); aggregate score; avoid double-vote (idempotent).
- **Comments:** Tree structure (comment_id, parent_id, post_id, user_id, text); store in DB with path or adjacency; fetch subtree for post.
- **Feed:** Home = posts from subscribed subreddits, sorted. Pre-compute hot/top per subreddit and globally; cache; merge for home feed. New = chronological; Top = by score in window.

## Key Components

- **Post service:** CRUD posts; shard by subreddit_id or post_id; index by (subreddit, sort_key) for listing.
- **Vote service:** Record votes; update post score (async or sync); prevent duplicate vote per user (unique constraint).
- **Ranking:** Hot = f(score, time decay). Precompute periodically (e.g. every minute) per subreddit; store in cache or materialized view. Top = sort by score in time window.
- **Comment service:** Store comments with parent_id; fetch tree (recursive or nested set or path); paginate deep threads.
- **Feed service:** For home: get subscribed subreddits; for each, get hot/top posts; merge and sort; cache per user or merge on read.

## Data Model

- **posts:** post_id, subreddit_id, user_id, title, url, score, created_at.
- **votes:** user_id, post_id, direction (+1/-1). Unique (user_id, post_id).
- **comments:** comment_id, post_id, parent_id, user_id, text, score, created_at.
- **subreddits:** subreddit_id, name. **subscriptions:** user_id, subreddit_id.

## Scale

- Millions of posts and comments; votes very high (one per user per post). Score update: batch or async to avoid write storm. Feed: cache ranked lists; merge subscribed subreddits; paginate.

## Trade-offs

- **Score consistency:** Strong consistency (update on every vote) vs eventual (async aggregation). Strong is simpler but write-heavy; eventual scales better.
- **Feed merge:** Pre-merge and cache per user (storage) vs merge on read (compute). Depends on number of subscriptions and churn.
- **Comment tree:** Recursive query vs path/materialized path vs limit depth. Trade-off between query simplicity and depth.

## Failure & Operations

- **Vote storm:** Idempotent votes; batch score updates; rate limit if needed.
- **Cache:** Invalidate or refresh ranked lists on schedule; accept short staleness.
- **Monitoring:** Feed latency, vote throughput, comment load, cache hit rate.
