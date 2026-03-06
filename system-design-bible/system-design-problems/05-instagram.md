# System Design: Instagram

## Problem

Design a photo (and video) sharing platform: upload media, follow users, view feed of media from followees, like/comment, and view user profiles.

## Requirements

**Functional:** Upload photo/video; follow/unfollow; feed (chronological or ranked); like, comment; user profile with media grid; stories (optional).

**Non-functional:** Fast feed load (p99 < 200 ms); efficient media delivery (CDN); high read-to-write ratio; support large media files.

## High-Level Design

```
  Clients → API Gateway → Auth, Feed service, Media service, Social graph
                ↓              ↓           ↓
            CDN (media URLs)   Timeline   Object store (S3) + metadata DB
```

- **Upload:** Client uploads to app or presigned URL to object store; metadata (user_id, caption, timestamp) stored in DB; media processed (resize, thumbnail) if needed; CDN URL stored.
- **Feed:** Similar to Twitter—push or pull timeline; each item is post metadata + media URL (CDN); client loads images from CDN.
- **Profile:** User info + list of post IDs; media URLs from CDN.

## Key Components

- **Media storage:** Object store (S3/GCS) for originals and variants (thumbnail, different resolutions); CDN in front for delivery. Upload via presigned URL to reduce app load.
- **Metadata store:** Posts (post_id, user_id, media_urls, caption, created_at); likes, comments. Shard by user_id.
- **Feed/timeline:** Push model: on new post, fanout to followers’ timeline cache. Pull for celebrities. Timeline stores post_ids + metadata; client gets CDN URLs from metadata.
- **Social graph:** Same as Twitter (followers/following); cached.

## Data Model

- **posts:** post_id, user_id, media_urls (thumbnail, full), caption, created_at.
- **likes:** user_id, post_id. **comments:** comment_id, post_id, user_id, text, created_at.
- **follows:** follower_id, followee_id.
- **timelines:** user_id → list of (post_id, timestamp).

## Scale

- 1B users; 100M uploads/day → ~1.2K writes/s. Feed reads 50× → 60K reads/s. Media: serve from CDN; origin stores only originals and generated sizes. Storage: large (photos/videos); use tiering and retention.

## Trade-offs

- **Upload path:** Direct to object store (presigned) vs via app. Direct reduces load and scales; app can do validation and processing async.
- **Media processing:** Synchronous (user waits) vs async (queue + worker); async for heavy transcoding.
- **Feed:** Same push/pull/hybrid as Twitter for timeline.

## Failure & Operations

- **CDN:** Cache media; invalidate on delete or versioned URLs. Origin behind CDN for availability.
- **Processing queue:** Retry failed jobs; dead-letter queue; monitor backlog.
- **Monitoring:** Upload success rate, feed latency, CDN hit ratio, storage growth.
