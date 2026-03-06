# API Gateways

## Role

Single entry point for client requests to backend services. Handles cross-cutting concerns: auth, routing, rate limiting, observability, and sometimes protocol translation.

```
  Clients (web, mobile)  ─────→  API Gateway  ─────→  Service A
                                                    →  Service B
                                                    →  Service C
```

---

## Core Responsibilities

| Responsibility | What it does |
|----------------|--------------|
| **Routing** | Path/host → backend service (e.g. `/users` → user-service). |
| **Authentication** | Validate JWT, API key, or OAuth; reject unauthenticated requests. |
| **Authorization** | Check permissions (e.g. scope, role) before forwarding. |
| **Rate limiting** | Throttle by user, API key, or IP to protect backends. |
| **Request/response transform** | Add headers, aggregate responses, translate REST ↔ gRPC. |
| **Observability** | Logs, metrics, distributed trace IDs for every request. |

---

## Why Use an API Gateway?

- **Single place** for auth, rate limits, and routing — no duplication in every service.
- **Protocol translation** — clients use REST; backends can be gRPC.
- **Backend hiding** — internal service topology and versions stay hidden.
- **Centralized policies** — security and throttling in one layer.

---

## Routing Patterns

- **Path-based:** `/api/v1/users` → user-service; `/api/v1/orders` → order-service.
- **Host-based:** `users.api.com` vs `orders.api.com`.
- **Version in path or header:** `/v1/` vs `/v2/` or `Accept-Version: 2`.
- **Canary/blue-green:** Route a fraction of traffic to new version (header or weight).

---

## Rate Limiting at the Gateway

- **Token bucket or sliding window** — often implemented in Redis for distributed limit.
- **Key:** User ID, API key, or IP (with risk of IP sharing).
- **Response:** 429 Too Many Requests; optional `Retry-After` header.
- **Interview:** Protects backends from abuse and gives a single place to enforce quotas.

---

## Failure Modes

- **Gateway as SPOF:** Run multiple instances behind a load balancer; stateless.
- **Cascading failure:** If a backend is slow, gateway threads/connections can exhaust; use timeouts and circuit breakers from gateway to backend.
- **Latency:** Every request gets an extra hop; keep gateway logic lean; avoid heavy logic in hot path.

---

## Build vs Buy

| Option | Pros | Cons |
|--------|------|------|
| **Managed (AWS API GW, Kong Cloud, etc.)** | No ops, scaling, features | Cost, less flexibility, vendor lock-in |
| **Self-hosted (Kong, APISIX, Envoy)** | Control, any infra | Operate, scale, upgrade |
| **Custom (NGINX + Lua, Envoy filters)** | Exact behavior | Build and maintain |

---

## Interview Takeaways

1. Gateway = **single entry point** for routing, auth, rate limiting, observability.
2. Keep it **stateless** and **horizontally scalable**.
3. Use **timeouts and circuit breakers** to backends so one slow service doesn’t kill the gateway.
4. **Rate limit by user/key** when possible; fall back to IP with clear trade-offs.
