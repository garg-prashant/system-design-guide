# BFF and Sidecar Patterns

## BFF (Backend for Frontend)

**Idea:** One backend API per **client type** (e.g. web, mobile, desktop). The BFF aggregates and shapes data from multiple downstream services so the client gets one tailored API.

```
  Web app    ──→  Web BFF   ──→  Service A, B, C  (aggregates, shapes for web)
  Mobile app ──→  Mobile BFF ──→  Service A, B, C  (different fields, pagination, etc.)
```

### Why use a BFF

- **Different clients need different data** — mobile may want fewer fields, different pagination, or different auth flow.
- **Reduce round-trips** — BFF calls several services and returns one response.
- **Ownership** — frontend team can own the BFF and change API without changing every backend.

### Trade-offs

- **Pros:** Client-specific APIs; fewer round-trips; clear ownership.
- **Cons:** Multiple BFFs to maintain; BFF can become a thick layer (keep it thin: aggregate and delegate).

---

## Sidecar Pattern

**Idea:** Run a **helper process** next to each instance of your app (same node/pod). The app talks to the sidecar for cross-cutting concerns; the sidecar handles networking, observability, or policy.

```
  ┌─────────────────────────────────┐
  │  Node / Pod                      │
  │  ┌─────────────┐  ┌────────────┐ │
  │  │   App       │  │  Sidecar   │ │
  │  │             │──│  (proxy,   │─┼──→  Other services
  │  │             │  │  metrics)  │ │
  │  └─────────────┘  └────────────┘ │
  └─────────────────────────────────┘
```

**Examples:** Envoy proxy (service mesh), Prometheus metrics exporter, log shipper.

### What the sidecar does

- **Proxy:** Outbound (and sometimes inbound) traffic goes through sidecar; can do retries, timeouts, mTLS, routing.
- **Observability:** Export metrics, trace propagation, log collection.
- **Policy:** Auth, rate limit, access control at the edge of the process.

### Trade-offs

- **Pros:** App stays unaware of mesh/observability; uniform behavior across languages; upgrade sidecar independently.
- **Cons:** Extra latency and resource (CPU/memory) per pod; more moving parts; debugging across app + sidecar.

**Interview:** “We use a BFF so each client gets an API shaped for it; we use a sidecar (e.g. Envoy) so retries, TLS, and metrics are consistent across all services without changing app code.”
