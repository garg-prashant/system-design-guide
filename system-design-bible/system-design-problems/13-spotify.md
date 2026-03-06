# System Design: Spotify

## Problem

Design a music streaming service: search and browse catalog, play tracks, playlists, and provide personalized recommendations (Discover Weekly, etc.).

## Requirements

**Functional:** Search tracks/artists/albums; play track (stream audio); create/edit playlists; follow playlists; recommendations (personalized playlists, radio); offline (optional).

**Non-functional:** Low latency to start playback; continuous playback; high availability; handle large catalog and many concurrent streams.

## High-Level Design

```
  Client → API → Catalog service, Playlist service, Recommendation service
  Playback: Client → CDN (audio segments) or streaming URL
  Metadata: DB + search index; Recommendations: batch + real-time
```

- **Catalog:** Tracks, artists, albums metadata in DB; full-text search (Elasticsearch); serve by ID or search.
- **Playback:** Audio files (or segments) in object store; CDN for delivery; client requests stream URL or segments; adaptive bitrate if multiple qualities.
- **Playlists:** Playlist metadata (name, owner, tracks list); shared or private; collaborative edit (optional) with conflict handling.
- **Recommendations:** Listen history; offline training (collaborative filtering, embeddings); generate Discover Weekly, radio; serve from cache/API.

## Key Components

- **Catalog service:** Metadata CRUD; search; filter by genre, artist, etc. Shard by entity type or ID.
- **Streaming:** Encode audio (e.g. multiple bitrates); store in object store; CDN; client gets URL or manifest.
- **Playlist service:** CRUD playlists; list of track_ids; version or timestamp for collaborative edit.
- **Recommendation service:** History ingestion; batch jobs (e.g. daily) to compute recommendations; store per user; API serves with cache.
- **Search:** Inverted index on track, artist, album; autocomplete; ranking by popularity and relevance.

## Data Model

- **tracks:** track_id, title, artist_id, album_id, duration, stream_url_or_segments.
- **playlists:** playlist_id, user_id, name, track_ids[], updated_at.
- **listen_history:** user_id, track_id, timestamp. For recommendations.
- **recommendations:** user_id → list of track_ids or playlist (precomputed).

## Scale

- Catalog: large but mostly read; cache and search index. Streaming: CDN for bytes; origin for ingest. Recommendations: batch + cache; scale with queue and workers.

## Trade-offs

- **Streaming:** Progressive download (simple) vs adaptive (multiple qualities, better QoE). DRM if required by licensing.
- **Recommendations:** Batch vs real-time; batch is common for weekly playlists; real-time for “now playing” suggestions.
- **Offline:** Pre-download encrypted segments; sync when online; device storage and key management.

## Failure & Operations

- **CDN:** Multiple regions; fallback to origin. Monitor buffer and errors.
- **Recommendation pipeline:** Fallback to popular/trending if pipeline fails; monitor freshness.
- **Monitoring:** Play start latency, skip rate, recommendation engagement, catalog API latency.
