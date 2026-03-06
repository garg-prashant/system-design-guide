# Latency, Throughput, and Availability

## Latency

**Definition:** Time from when a request is initiated to when a response is received.

### Latency Breakdown

```
Total Request Latency
├── Client-side: serialization, network stack
├── Network transit: propagation delay
├── Server-side processing:
│   ├── Request parsing
│   ├── Auth check
│   ├── Business logic
│   ├── Database query (often dominates)
│   └── Response serialization
└── Network return transit
```

### Reference Latency Numbers

```
Operation                          Latency
─────────────────────────────────────────────
L1 cache hit                       ~0.5 ns
L2 cache hit                       ~7 ns
L3 cache hit                       ~30 ns
RAM access                         ~100 ns
SSD sequential read (4KB)          ~150 us
SSD random read                    ~100 us
HDD random seek + read             ~10 ms
Mutex lock/unlock                  ~25 ns
Context switch                     ~1-2 us
System call                        ~500 ns
Network same datacenter            ~0.5 ms
Network same region (cross-DC)     ~5 ms
Network cross-continent            ~100 ms
Network cross-ocean                ~150 ms
Memcached/Redis get                ~0.5 ms
PostgreSQL simple query            ~1-5 ms
PostgreSQL complex join            ~50-500 ms
```

### Percentiles vs Averages

**Never use averages for latency.** They hide tail latency, which is what users experience.

```
Example: 1000 requests
  990 requests: 10ms
  9 requests: 100ms
  1 request: 10,000ms

Average: 109ms (misleading — 99% of users see 10ms)
p50: 10ms
p95: 100ms
p99: 100ms
p99.9: 10,000ms (that one bad request)
```

**Why tail latency matters:**

For a service with N dependencies, the overall p99 = 1 - (1 - p99_dependency)^N

If each downstream call has 1% chance of being slow, and you make 10 calls, you have ~10% chance of being slow. At scale, tail latency compounds.

### Latency Budget

A "latency budget" is the maximum time allocated to each component:

```
User-facing API target: 200ms p99
├── Network (client → LB → server): 20ms
├── Auth validation: 5ms
├── Cache lookup: 2ms
│   └── Cache miss → DB query: 50ms (reserve)
├── Business logic: 10ms
├── Downstream service call: 50ms
└── Response serialization: 5ms

Remaining budget for retries/overhead: ~58ms
```

### Reducing Latency

1. **Caching** — Avoid recomputation; serve from memory
2. **Async processing** — Don't block on non-critical paths
3. **Parallelism** — Concurrent downstream calls instead of sequential
4. **Data locality** — Colocate computation and data
5. **Connection pooling** — Eliminate TCP handshake overhead
6. **Batching** — Amortize per-request overhead
7. **Read replicas** — Serve reads from geographically closer nodes
8. **Compression** — Reduce network transfer time for large payloads

---

## Throughput

**Definition:** Amount of work completed per unit of time. Typically measured in:
- Requests per second (RPS/QPS)
- Transactions per second (TPS)
- Bytes per second (bandwidth)
- Messages per second (for queues)

### Little's Law

```
L = λ × W

L = average number of requests in the system
λ = average arrival rate (requests/sec)
W = average time a request spends in the system (latency)
```

**Example:** If your service handles 1000 req/sec with average latency of 50ms, at any given moment there are 50 concurrent requests in flight.

Implications:
- To increase throughput, reduce latency or add capacity
- High concurrency with low throughput = high latency
- System capacity limit: when arrival rate > service rate, queue grows unboundedly

### Throughput Bottlenecks

```
Client → [LB] → [App Server] → [DB]
                                 ↑
                         Usually the bottleneck

Common limits:
- App server CPU: 50K-200K req/sec (stateless, simple logic)
- App server I/O bound: 5K-50K req/sec
- Postgres single master: 10K-50K simple queries/sec
- Redis: 100K-1M ops/sec
- Kafka partition: 100MB/sec throughput
- Network NIC: 1-10 Gbps
```

### Increasing Throughput

1. **Horizontal scaling** — Add more stateless servers
2. **Vertical scaling** — More CPU/RAM/network
3. **Sharding** — Distribute database load
4. **Read replicas** — Offload reads from primary
5. **Caching** — Reduce downstream pressure
6. **Async/queuing** — Smooth traffic spikes
7. **Batching** — Amortize fixed costs

### Latency vs Throughput Trade-off

They often conflict:
- **Batching** increases throughput but increases latency (wait for batch to fill)
- **Connection pooling** increases throughput but may add queue latency at peak
- **Write buffering** increases write throughput but increases latency and risk of data loss

**Rule of thumb:** Optimize for the constraint that matters for your use case.
- Interactive systems (APIs, web): latency-sensitive
- Batch processing (ETL, analytics): throughput-sensitive
- Real-time streaming: both matter

---

## Availability

**Definition:** Fraction of time the system is operational and accessible.

### Availability Nines

| Availability | Downtime/Year | Downtime/Month | Downtime/Day |
|-------------|---------------|----------------|--------------|
| 90% | 36.5 days | 73 hours | 2.4 hours |
| 99% | 3.65 days | 7.3 hours | 14.4 minutes |
| 99.9% | 8.76 hours | 43.8 minutes | 1.44 minutes |
| 99.99% | 52.6 minutes | 4.38 minutes | 8.64 seconds |
| 99.999% | 5.26 minutes | 26.3 seconds | 864 ms |
| 99.9999% | 31.6 seconds | 2.63 seconds | 86.4 ms |

**6 nines is the limit** of what's practically achievable — requires redundancy everywhere, including power, cooling, network, and hardware.

### Availability in Series vs Parallel

**Series** (both must be up):
```
Service A (99.9%) → Service B (99.9%)
Combined: 99.9% × 99.9% = 99.8%

General: A1 × A2 × ... × An
Each dependency REDUCES availability
```

**Parallel** (either can serve):
```
Instance A (99.9%) ─┐
                    ├→ Combined
Instance B (99.9%) ─┘

Combined: 1 - (1-0.999)^2 = 1 - 0.000001 = 99.9999%
```

This is why replication dramatically improves availability: two 99.9% instances in parallel give 99.9999%.

### Causes of Unavailability

1. **Hardware failure** — Disk, NIC, CPU, power supply
2. **Software bugs** — Memory leaks, deadlocks, null pointers
3. **Configuration changes** — Bad deploy, misconfigured LB
4. **Dependency failures** — Downstream service, database, external API
5. **Capacity overload** — Traffic spike beyond capacity
6. **Network partitions** — Between nodes or to external network
7. **Human error** — Accidental deletion, misconfiguration

### Achieving High Availability

```
Strategy                 Impact
─────────────────────────────────────────
Redundant instances      Eliminates single point of failure
Health checks + LB       Automatically routes around failures
Multi-AZ deployment      Survives availability zone failure
Multi-region             Survives regional failure
Circuit breakers         Prevent cascade failures
Graceful degradation     Serve partial results vs complete failure
Blue-green deploys       Zero-downtime deployments
Feature flags            Instant rollback without redeployment
Chaos engineering        Find failure modes before users do
```

---

## The Tension: Availability vs Consistency

The fundamental distributed systems trade-off (from CAP theorem):

```
Availability                    Consistency
(always respond)                (respond correctly)
       \                              /
        \                            /
         ──────── Trade-off ────────

During a partition, choose one:
  AP: Return stale data or allow conflicting writes
  CP: Refuse to serve requests (503) until partition heals
```

**Real-world examples:**
- Shopping cart: AP — prefer availability, reconcile later
- Bank balance: CP — consistency matters more than availability
- Social media likes: AP — approximate count is fine
- Payment processing: CP — correctness is non-negotiable

---

## SLAs, SLOs, SLIs

**SLI (Service Level Indicator):** A measurement. "Our p99 latency is 150ms."

**SLO (Service Level Objective):** A target. "We want p99 latency < 200ms."

**SLA (Service Level Agreement):** A contract with consequences. "If we miss SLO, customer gets credit."

**Error Budget:** Time/requests you can afford to fail.
- 99.9% SLO = 0.1% error budget
- Over 30 days: 43.8 minutes of downtime, or 0.1% of requests can fail

**Error budget forces trade-offs:**
- Slow down deployments if budget is depleted
- Prioritize reliability work when burning through budget
- Gives engineering teams quantified room to experiment

---

## Interview Discussion Points

- "What's the latency requirement? p99? p999? This affects architecture significantly."
- "For availability, are we targeting 99.9% or 99.99%? Each nine is a different architecture."
- "These two services are in series, so our combined availability is their product — we should consider whether we need the second call to be synchronous."
- "Our error budget for a 99.99% SLO is 52 minutes per year. Any deploy that risks availability needs a fast rollback strategy."
- "I'd use async processing here: we don't need the user to wait for this operation, and it improves both latency and availability."
