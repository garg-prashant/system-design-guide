# System Design: YouTube

## Problem

Design a video platform: upload, transcode, store, and stream video to users; support search, recommendations, and comments.

## Requirements

**Functional:** Upload video; process/transcode (multiple qualities); stream (adaptive bitrate); search; recommendations; comments, likes; subscriptions (optional).

**Non-functional:** Low start latency (video plays quickly); smooth playback; handle large files and high bandwidth; durability of uploads.

## High-Level Design

```
  Upload → API → Queue → Transcoding workers → Object store (variants)
  Stream: Client → CDN (video segments) ← Origin (object store)
  Metadata, search, recommendations: API + DB + search index
```

- **Upload:** Chunked upload to object store or upload service; metadata in DB; job in queue for transcoding.
- **Transcode:** Workers pull jobs; produce multiple resolutions/formats (e.g. HLS/DASH segments); store in object store; CDN in front.
- **Stream:** Client requests manifest + segments from CDN; adaptive bitrate based on bandwidth.
- **Search/recs:** Metadata in DB; search index (Elasticsearch) for full-text; recommendation service (collaborative filtering, embeddings) for related videos.

## Key Components

- **Upload & storage:** Resumable uploads (chunks); object store for raw and transcoded output; lifecycle/retention policies.
- **Transcoding pipeline:** Queue (e.g. SQS); worker pool; output per resolution (e.g. 360p, 720p, 1080p); segment format (HLS/DASH) for streaming.
- **CDN:** Cache segments at edge; reduce origin load and latency.
- **Metadata DB:** Videos, user, playlists, comments; shard by video_id or user_id.
- **Search:** Inverted index on title, description, tags; filter by upload date, duration, etc.
- **Recommendations:** Offline training (collaborative filtering, neural); serve from cache/API; A/B test ranking.

## Data Model

- **videos:** video_id, user_id, title, description, status (processing/live), created_at, segment_urls (or pointer).
- **segments:** Stored in object store (per quality); manifest references segments.
- **comments, likes:** Standard relational or document store.
- **recommendations:** Precomputed per user or video; served from cache.

## Scale

- Massive storage (videos); transcoding CPU-heavy; streaming bandwidth-dominated. Use CDN for most bytes; origin for long-tail. Transcode queue scales with workers; storage scales with object store and tiering.

## Trade-offs

- **Transcode sync vs async:** Async: user gets “processing”; better UX and scale. Sync: simple but blocks.
- **Adaptive bitrate:** Multiple qualities increase storage and transcode cost; better QoE.
- **Recommendations:** Real-time vs batch; batch is simpler; real-time adds latency and complexity.

## Failure & Operations

- **Transcode failures:** Retry; dead-letter queue; alert; allow re-upload or re-process.
- **CDN:** Origin failover; multiple CDN providers for critical path.
- **Monitoring:** Upload success, transcode lag, buffer ratio, error rate, search latency.
