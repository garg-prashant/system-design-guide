# System Design: Recommendation System

## Problem

Design a recommendation system that suggests items (products, videos, articles) to users based on behavior (clicks, purchases, ratings), content similarity, and optional real-time context. Support “similar items” and “for you” feeds.

## Requirements

**Functional:** “Recommended for you” (personalized); “similar items” (item-to-item); “trending” or “popular”; filter by category or context (optional).

**Non-functional:** Low latency (p99 < 100 ms for serving); fresh recommendations (update daily or real-time); scalable to millions of users and items.

## High-Level Design

```
  Offline: History (clicks, purchases) → Training pipeline → Model (embeddings, matrix) → Store (cache/DB)
  Online:  Request (user_id, context) → Retrieval (candidates) → Ranking (model or rules) → Top K → Response
```

- **Offline:** Ingest user-item interactions; train collaborative filtering (matrix factorization) or sequence model; produce user embeddings and item embeddings; or item-item similarity matrix. Store in model store or key-value (user_id → item_ids).
- **Retrieval:** Given user_id (and context), get candidates: collaborative (users like you liked X), content-based (similar to what you liked), trending, or random. Retrieve hundreds to thousands of candidates.
- **Ranking:** Score candidates (dot product of embeddings, or learned ranker); apply diversity and business rules; take top K; return.
- **Serving:** Cache precomputed “for you” per user (refreshed periodically); or compute on read with cached embeddings; fallback to popular if cold user or failure.

## Key Components

- **Training pipeline:** Batch (daily) or streaming; compute embeddings or similarity; store in model registry and feature store.
- **Candidate retrieval:** Approximate nearest neighbor (ANN) for embeddings (e.g. FAISS, Annoy); or lookup tables (user → items, item → similar items). Scale with sharding.
- **Ranking:** Lightweight model (linear, shallow NN) or rules (diversity, freshness, business rules). Run in latency budget (e.g. 20 ms).
- **Serving API:** Input: user_id, limit, context. Output: list of item_ids. Cache per user; TTL (e.g. 1h); fallback to popular.
- **Feature store (optional):** User features, item features, context; for real-time ranking.

## Data Model

- **user_embeddings:** user_id → vector; or user_id → list of (item_id, score).
- **item_embeddings:** item_id → vector; or item_id → similar item_ids.
- **interactions:** user_id, item_id, event_type, timestamp. For training and logging.
- **candidates / precomputed:** user_id → [item_ids] (refreshed periodically).

## Scale

- Millions of users and items; retrieval: ANN or table lookup; ranking: thousands of candidates in ms. Training: batch on cluster; embeddings stored and replicated. Serving: cache and horizontal scale.

## Trade-offs

- **Batch vs real-time:** Batch: simpler, daily refresh. Real-time: incorporate latest actions; more complex (streaming pipeline, online model).
- **Recall vs latency:** More candidates = better recall, more ranking cost. Two-stage: fast retrieval (many) + heavier ranker (top hundreds).
- **Cold start:** New user/item: use content features, popular, or explore; improve over time with data.

## Failure & Operations

- **Model freshness:** Monitor staleness; fallback to previous model or popular if training fails.
- **Serving:** Cache and fallback to popular; circuit breaker for ranking service.
- **Monitoring:** Latency p99, CTR/engagement, coverage, diversity metrics, training pipeline success.
