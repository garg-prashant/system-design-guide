# System Design: API Gateway

## Problem

Design an API gateway that sits in front of multiple backend services: route requests, authenticate and authorize, rate limit, and provide observability (logging, metrics, tracing).

## Requirements

**Functional:** Route by path/host to backend; authenticate (JWT, API key); authorize (scope/role); rate limit per user/key; request/response transform (optional); versioning (path or header).

**Non-functional:** Low latency overhead; high throughput; highly available; observable (logs, metrics, traces).

## High-Level Design

```
  Clients → Load balancer → API Gateway (N instances) → Backend A, B, C
                Gateway: route, auth, rate limit, log, metric, trace
```

- **Request flow:** Receive request → identify client (API key, JWT) → auth → authz → rate limit (check Redis or local) → route to backend → forward request → response (optional transform) → log and metric → return.
- **Routing:** Config: path prefix or host → backend URL. Canary: route by header or weight to different backends.
- **Rate limiting:** Key = user_id or api_key; algorithm = token bucket or sliding window; store state in Redis for global limit; 429 when exceeded.
- **Observability:** Log request_id, latency, status; emit metrics (count, latency histogram); inject trace context and create span.

## Key Components

- **Gateway process:** Stateless; run behind LB; handle HTTP/gRPC; plugins or config for route, auth, rate limit.
- **Auth:** Validate JWT (signature, exp) or lookup API key; attach user/claims to request context.
- **Rate limiter:** Redis: INCR + EXPIRE (fixed window) or Lua (sliding); local cache for hot keys to reduce Redis load.
- **Router:** Table: path/host → backend; timeout and retry to backend; circuit breaker if backend failing.
- **Observability:** Structured logs; metrics (request count, latency by route, status); trace ID propagation.

## Data Model

- **Route config:** path_prefix or host → backend_url, timeout, retry_policy.
- **API keys:** key_id → user_id, scopes, rate_limit. Optional in DB or cache.
- **Rate limit state:** key (e.g. user_id) → counter, window; in Redis.

## Scale

- Gateway is stateless; scale horizontally. Rate limit: Redis cluster; cache allow/deny per key for hot path. Backend calls: connection pooling; timeouts to avoid blocking.

## Trade-offs

- **Centralized vs per-service:** Gateway centralizes cross-cutting concerns; single place to update auth and rate limits. Per-service: more flexibility, more duplication.
- **Sync vs async auth:** Sync validation (JWT local, API key lookup) adds latency; cache API keys; keep validation fast.
- **Rate limit storage:** Redis for global; in-memory for per-instance (not global but no dependency).

## Failure & Operations

- **Backend failure:** Timeout and circuit breaker; return 503 or fallback; don’t let one backend exhaust gateway connections.
- **Redis down (rate limit):** Fail open (allow) or fail closed (reject); document and monitor.
- **Monitoring:** Gateway latency, error rate, rate limit 429 rate, backend latency and errors.
