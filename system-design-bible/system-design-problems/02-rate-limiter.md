# System Design: Rate Limiter

## Problem

Design a rate limiter that restricts the number of requests a user (or API key / IP) can make in a time window. When the limit is exceeded, return 429 Too Many Requests.

## Requirements

**Functional:** Limit by identifier (user_id, API key, IP); configurable limit (e.g. 100 req/min); return 429 when exceeded; optional per-endpoint limits.

**Non-functional:** Low latency (add minimal overhead); accurate across distributed servers; high availability.

## High-Level Design

```
  Request → API Gateway / Middleware → Rate Limiter (e.g. Redis) → Allow / 429
                                              ↓
                                    Check counter for (key, window)
```

- **Key:** user_id or api_key or IP (or composite).
- **Window:** Fixed window (e.g. 1-min buckets) or sliding window (more accurate, more work).
- **Storage:** Distributed store (Redis) so all app instances see the same counters.

## Key Components

- **Algorithm:** Token bucket (smooth rate) or sliding window / fixed window (request count). Sliding window log: store timestamps, count in window; or approximate with Redis sorted set.
- **Redis:** INCR + EXPIRE for fixed window; or Lua script for atomic check-and-increment. Key = rate_limit:{identifier}:{window_id}.
- **Placement:** In API gateway or as middleware before app; must run before expensive logic.

## Data Model

- **Key:** e.g. `rl:user_123:60` (user, 60s window). **Value:** counter. **TTL:** 60s (or window size).
- Sliding: store (timestamp, count) or use Redis sorted set (member = request_id, score = timestamp); count members in [now-window, now].

## Scale

- Assume 100K req/s; each request one Redis read + optional write. Redis can do 100K+ ops/s per node; use cluster or partition by key prefix. Latency: add ~1–2 ms for Redis round-trip.

## Trade-offs

- **Fixed vs sliding window:** Fixed: simple; can allow 2× limit at boundary. Sliding: fairer; more state or computation.
- **In-memory vs Redis:** In-memory per server: no shared state, limit is per-server. Redis: global limit; extra latency and dependency.
- **Strict vs best-effort:** Under overload, can fail open (allow) or fail closed (reject); fail closed is safer for backend protection.

## Failure & Operations

- **Redis down:** Fail open (allow) or fail closed (reject). Document and choose; often fail open to avoid dropping all traffic.
- **Clock skew:** Use Redis time or NTP; sliding window sensitive to clock.
- **Monitoring:** Rate of 429s, latency of rate-limit check, Redis availability.
