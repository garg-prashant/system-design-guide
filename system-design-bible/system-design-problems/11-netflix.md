# System Design: Netflix

## Problem

Design a video streaming platform: catalog, recommendations, playback (adaptive streaming), and support for many concurrent viewers and regions.

## Requirements

**Functional:** Browse catalog; search; recommendations; play video (adaptive bitrate); resume; multiple profiles (optional).

**Non-functional:** Fast start (play in seconds); smooth playback; global scale; high availability; catalog and recommendations personalized.

## High-Level Design

```
  Client → API (catalog, search, recs) → Catalog service, Recommendation service
  Playback: Client → CDN (video segments) ← Origin (encoded assets)
  Catalog/recs: DB + cache; recommendations: offline model + real-time serving
```

- **Catalog:** Metadata (title, description, artwork, segment URLs) in DB; search index; serve via API; cache hot data.
- **Recommendations:** Offline: train on watch history, ratings; produce per-user or per-title embeddings/scores. Online: serve from cache/API; A/B test ranking.
- **Playback:** Client gets manifest (URLs for segments per quality); fetches segments from CDN; adaptive bitrate; CDN caches at edge; origin holds encoded library.
- **Encoding:** Ingest video; transcode to multiple qualities and formats (HLS/DASH); store in object store; CDN in front (same as YouTube pattern).

## Key Components

- **Catalog service:** Metadata CRUD; search (Elasticsearch); filter by genre, region, etc. Regional catalog (licensing).
- **Recommendation service:** Precomputed rankings per user; real-time features (current session); combine and rank; cache results.
- **Playback/CDN:** Same as YouTube—segments on CDN; origin for long-tail; DRM if required.
- **Encoding pipeline:** Ingest → queue → transcode workers → store; multiple resolutions and codecs.

## Data Model

- **titles:** title_id, name, type (movie/series), metadata, region_availability.
- **episodes/segments:** References to segment URLs (per quality).
- **user_watch_history:** user_id, title_id, progress, rating. For recommendations.
- **recommendations:** user_id → list of title_ids (precomputed or key for real-time).

## Scale

- Catalog and recommendations: read-heavy; cache and CDN. Video bytes: CDN; origin for encoding and long-tail. Recommendations: batch + real-time; scale with cache and API.

## Trade-offs

- **Recommendation freshness:** Batch daily vs real-time. Real-time improves relevance; batch is simpler and cheaper.
- **Catalog regions:** Per-region licensing; filter catalog by user region; replicate metadata.
- **Start latency:** Prefetch next segment; CDN close to user; lower first-bit latency.

## Failure & Operations

- **CDN failure:** Multi-CDN or failover to origin; monitor buffer ratio and errors.
- **Recommendation service:** Degrade to popular or random; cache fallback.
- **Monitoring:** Start time, buffer ratio, error rate, recommendation CTR, catalog API latency.
