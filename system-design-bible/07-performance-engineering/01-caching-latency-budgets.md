# Caching and Latency Budgets

## Caching

**Purpose:** Serve data from a faster store (memory, local cache, distributed cache) to reduce latency and load on the primary store (e.g. DB).

### Cache strategies

| Strategy | When to use |
|----------|-------------|
| **Cache-aside** | App checks cache; on miss, loads from DB and populates cache. App owns logic. |
| **Read-through** | Cache layer loads from DB on miss; app only talks to cache. |
| **Write-through** | Write goes to cache and cache writes to DB. Read miss rare. |
| **Write-behind** | Write to cache; cache asynchronously writes to DB. Risk of loss if cache dies. |

### Eviction

- **LRU (Least Recently Used):** Evict least recently used. Good general default.
- **TTL:** Expire after time; simple; may serve stale.
- **Combination:** TTL + LRU (e.g. max memory + TTL).

### Consistency

- Cache can be **stale** vs primary. Accept eventual consistency or invalidate on write (e.g. delete key on DB write).
- **Cache stampede:** Many requests miss at once and hit DB. Mitigate with lock, probabilistic early expiry, or single-flight.

**Interview:** “We use cache-aside with TTL and invalidate on write for user-facing data; we accept eventual consistency for non-critical reads.”

---

## Latency Budgets

**Idea:** Allocate a **max latency per component** so end-to-end SLO is met.

Example: p99 target 200 ms.

| Component | Budget (p99) |
|-----------|--------------|
| API gateway | 5 ms |
| Auth | 10 ms |
| Service A | 50 ms |
| Service B | 30 ms |
| DB | 40 ms |
| Cache | 5 ms |
| **Total** | ~140 ms (under 200 ms) |

- Each team owns their budget; if a component exceeds it, it’s a bug or capacity issue.
- Use **percentiles** (p50, p95, p99), not averages.
- **Tail latency:** One slow dependency can blow the whole request; use timeouts, circuit breakers, and fallbacks.
