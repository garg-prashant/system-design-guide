# System Design: Feature Flag System

## Problem

Design a feature flag (feature toggle) system: define flags (on/off or gradual rollout), evaluate flags per request (user, context), and serve the result with low latency and high availability. Support targeting (user segment, percentage) and consistency (optional).

## Requirements

**Functional:** Create/update flag (key, rules: on/off, percentage, segment); evaluate flag for (user_id, context) and return on/off or variant; audit log of changes; optional A/B (multiple variants).

**Non-functional:** Low latency (single-digit ms); high availability (flags should not block app); eventually or strongly consistent; scale to many flags and evaluations.

## High-Level Design

```
  SDK (in app) → Local cache (flags, rules) ← Poll or stream ← Flag service
       ↓
  Evaluate (user_id, context) against rules → return on/off or variant
```

- **Evaluation:** Input: flag_key, user_id, context (device, country, etc.). Rules: e.g. “on for 10% of users,” “on for segment X,” “on for user_ids [list].” Evaluate in order (first match); default if no match. Return boolean or variant.
- **SDK:** In-process; hold rules in memory; evaluate locally for latency. Refresh rules by poll (e.g. every 60s) or stream (SSE, WebSocket) from flag service.
- **Flag service:** CRUD flags and rules; store in DB; serve rules to SDKs (poll or push); optional: evaluate server-side for consistency (e.g. for experiments).

## Key Components

- **Flag store:** flag_key, rules (JSON or structured: percentage, segments, user list), default value. Version or timestamp for cache invalidation.
- **Segment store:** segment_id, definition (e.g. “country IN (US, UK)” or “user_id IN list”). Used in flag rules.
- **SDK:** Load rules; evaluate with hash(user_id) % 100 for percentage; segment lookup (cached); return value. Minimal dependency; fallback to default if service unreachable.
- **Flag service API:** Get all flags (or changed since version) for SDK poll; or evaluate server-side (user_id, flag_key) for strict consistency.
- **Audit:** Log flag changes (who, when, old/new rules) for compliance.

## Data Model

- **flags:** flag_key, rules[], default_value, updated_at.
- **segments:** segment_id, name, definition (predicate or list).
- **rules:** e.g. { “percentage”: 10, “segment_id”: “beta” } or { “user_ids”: [...] }.

## Scale

- Millions of evaluations/s; SDK evaluates locally so no network for each request. Flag service: serve rule payload to SDKs (poll or stream); scale with cache (e.g. CDN) and DB read replicas.
- Percentage: deterministic hash(user_id + flag_key) % 100 for stable rollout per user.

## Trade-offs

- **Poll vs stream:** Poll: simple; delay to propagate changes (e.g. 60s). Stream: faster propagation; more complexity.
- **Client vs server evaluation:** Client (SDK): lowest latency, no extra hop; rules can be stale. Server: consistent, can log for experiments; extra latency and load.
- **Consistency:** Eventually consistent is fine for most flags; use server evaluation if you need strict consistency for A/B analysis.

## Failure & Operations

- **SDK resilience:** If flag service down, use cached rules; if no cache, use default (e.g. off). Don’t block app.
- **Rule propagation:** Version or timestamp so SDK knows when to refresh; invalidate cache on update.
- **Monitoring:** Evaluation latency (SDK), flag service availability, rule fetch latency, error rate.
