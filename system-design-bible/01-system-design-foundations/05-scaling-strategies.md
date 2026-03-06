# Scaling Strategies

## Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

Add more resources to a single machine: CPU, RAM, faster disks, better NIC.

```
Before:              After vertical scaling:
┌─────────────┐       ┌────────────────────┐
│  App Server │       │    App Server      │
│  4 CPU      │  →    │    32 CPU          │
│  16 GB RAM  │       │    256 GB RAM      │
│  SSD 500GB  │       │    NVMe 8TB        │
└─────────────┘       └────────────────────┘
```

**Pros:**
- Simple — no code changes required
- No distributed systems complexity
- Strong consistency (single machine)
- Low operational overhead

**Cons:**
- Hard limit — you can't buy an infinitely large machine
- Single point of failure — machine dies, everything dies
- Expensive — largest machines have diminishing returns
- Downtime during upgrade (usually)
- Memory/CPU can't be added "online" (usually)

**When to use:**
- Early stage products — simple wins
- Databases (up to a point — vertical scaling buys time)
- Workloads that don't parallelize (Amdahl's Law)
- Stateful systems that are hard to distribute

**Cloud reference sizes (AWS):**
```
m5.large:    2 vCPU, 8 GB RAM    (~$70/month)
m5.4xlarge:  16 vCPU, 64 GB RAM  (~$560/month)
m5.16xlarge: 64 vCPU, 256 GB RAM (~$2,200/month)
u-24tb1:     448 vCPU, 24 TB RAM (~$220,000/month)
```

### Horizontal Scaling (Scale Out)

Add more machines and distribute the load across them.

```
Before:              After horizontal scaling:
┌─────────────┐       ┌─────────────┐
│  App Server │       │  App Server │
│             │  →    │─────────────┤
│             │       │  App Server │  + Load Balancer
│             │       │─────────────┤
└─────────────┘       │  App Server │
                      └─────────────┘
```

**Pros:**
- Theoretically infinite scale
- High availability — no SPOF
- Commodity hardware (cheaper per unit)
- Independent failure domains
- Can scale incrementally

**Cons:**
- Requires stateless services (or shared state management)
- Distributed systems complexity (consistency, coordination)
- Operational overhead (more machines to manage)
- Data locality issues
- Testing and debugging harder

**When to use:**
- Stateless application servers (always — this is the standard)
- Read replicas for databases
- Cache clusters
- Message queue consumers

---

## Stateless vs Stateful Services

### Stateless Services

The service holds no client state. Any instance can handle any request.

```
Request 1  ──→ Server A
Request 2  ──→ Server B  ← same user, different server, works fine
Request 3  ──→ Server A
```

**What "stateless" means in practice:**
- Session state stored in shared store (Redis, database)
- No in-memory caches that must be consistent across instances
- No sticky sessions required

**Advantages:**
- Easy horizontal scaling
- Easy blue-green deployments
- Easy auto-scaling
- Load balancers can use round-robin (simple)
- Instances are interchangeable

**Making services stateless:**
1. Store sessions in Redis/Memcached
2. Use JWTs instead of server-side sessions (client carries state)
3. Move local caches to distributed caches
4. Use shared file systems (NFS, S3) for file state

### Stateful Services

The service holds state specific to a client or session. Requests from the same client must go to the same server.

```
Request 1  ──→ Server A  (uploads chunk 1)
Request 2  ──→ Server B  ← WRONG SERVER! chunk 1 is on Server A
Request 3  ──→ Server A  (uploads chunk 3)
```

**When stateful is necessary:**
- Long-lived connections (WebSockets for real-time chat)
- Streaming sessions
- In-progress multi-step operations
- When shared state coordination cost exceeds benefit

**Managing stateful services:**
- Consistent hashing (route client to same server)
- Sticky sessions in LB (based on cookie or IP)
- Stateful sets in Kubernetes
- Design for migration (what if that server dies?)

---

## Caching

Caching is the single highest-leverage scaling technique. Eliminates redundant computation and I/O.

### Cache Strategies

**Cache-Aside (Lazy Loading)**
```
Read path:
  1. Check cache for key
  2. If miss: query DB, store in cache, return
  3. If hit: return from cache

Write path:
  1. Write to DB
  2. Invalidate (delete) cache entry  ← OR skip invalidation (TTL-based expiry)
```

Pros: Only caches what's actually needed; cache failure doesn't break reads.
Cons: Cache miss = 3 operations (cache miss + DB read + cache write); stale data during TTL window.

**Write-Through**
```
Write path:
  1. Write to cache
  2. Write to DB
  3. Return success only when both succeed
```
Pros: Cache always has fresh data; reads are fast.
Cons: Write latency is double; may cache data that's never read.

**Write-Back (Write-Behind)**
```
Write path:
  1. Write to cache only
  2. Return success immediately
  3. Asynchronously flush to DB

DB write happens out-of-band, possibly batched.
```
Pros: Very low write latency; batching reduces DB pressure.
Cons: Risk of data loss if cache fails before flush; complex consistency.

**Read-Through**
```
Application talks only to cache.
Cache handles DB reads on misses.
```
Pros: Simplifies application code.
Cons: Cache library must support DB integration; cold start problem.

### Cache Eviction Policies

**LRU (Least Recently Used):** Evict the item that was accessed least recently.
- Best general-purpose policy
- Works well for temporal locality
- Implementation: doubly linked list + hashmap

**LFU (Least Frequently Used):** Evict the item accessed least frequently overall.
- Better for items with stable long-term popularity (static files)
- Suffers from "cache pollution": old popular items never evicted
- More complex to implement

**ARC (Adaptive Replacement Cache):** Combines LRU and LFU dynamically.
- Maintains two lists: recently used and frequently used
- Automatically adapts to access pattern
- Used in ZFS, some database buffer pools

**TTL-based:** Items expire after a fixed time.
- Simple and predictable
- Works well when freshness is more important than hit rate
- Often combined with LRU

### Cache Invalidation

The hardest part of caching. Three strategies:

1. **TTL expiry** — Let items expire naturally. Simple, but stale window = TTL.
2. **Event-driven invalidation** — On write, send invalidation event. Lower staleness, higher complexity.
3. **Write-through** — Update cache synchronously with DB. No staleness, higher write cost.

**Cache stampede (thundering herd):**
When a popular cached item expires, many simultaneous requests hit the DB:
```
Cache expires for popular-key
1000 concurrent requests: all miss cache, all query DB simultaneously
DB overloaded, slow/failing
```

**Solutions:**
1. **Probabilistic early expiry:** Re-compute cache before it expires (randomly, before TTL)
2. **Mutex/lock:** Only one request refreshes cache; others wait
3. **Background refresh:** Serve stale data while async refresh runs
4. **Jitter on TTL:** Spread expiration times across the cluster

---

## Database Scaling Patterns

### Read Replicas

```
                    ┌────────────────┐
  Write requests →  │  Primary DB    │
                    └───────┬────────┘
                            │ replicate
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
         ┌─────────┐  ┌─────────┐  ┌─────────┐
         │ Replica │  │ Replica │  │ Replica │
         └─────────┘  └─────────┘  └─────────┘
              ↑             ↑             ↑
                    Read requests
```

- Offloads reads from primary (typically 80-90% of traffic)
- Replication lag means replicas may be slightly stale (usually < 100ms)
- Replicas can serve as hot standbys for failover
- For analytics queries: route to dedicated replica to avoid impacting OLTP

### Connection Pooling

Database connections are expensive (memory, file descriptors). Pool reuses them:

```
App Servers (100 instances, each wants 10 connections)
                    ↓
           ┌─────────────────┐
           │  PgBouncer /    │  ← maintains 20-50 actual DB connections
           │  ProxySQL       │
           └─────────────────┘
                    ↓
           ┌─────────────────┐
           │    Database     │
           └─────────────────┘

Without pooler: 1000 connections → huge memory overhead
With pooler: 20 connections → efficient
```

### Sharding

Partition data across multiple database instances:

```
User ID → Shard determination → Appropriate shard

Example: user_id % 4
  user_id=1001 → shard 1
  user_id=1002 → shard 2
  user_id=1003 → shard 3
  user_id=1004 → shard 0
```

See `05-data-management/sharding.md` for deep dive.

---

## Microservices vs Monolith

### Monolith

All components deployed as a single unit.

```
┌──────────────────────────────────────────┐
│           Monolithic Application          │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │   Auth   │  │  Orders  │  │Catalog │ │
│  └──────────┘  └──────────┘  └────────┘ │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Payments│  │   Users  │  │ Notify │ │
│  └──────────┘  └──────────┘  └────────┘ │
└──────────────────────────────────────────┘
              │
        Single database
```

**Pros:**
- Simple to develop early on (single codebase, single deploy)
- Easy to test (unit + integration tests in same process)
- No network overhead for inter-component calls (in-process)
- Simple transactions (single DB, ACID)
- Easy debugging (single log stream, single trace)

**Cons:**
- Single deploy unit — all-or-nothing rollouts
- Scaling: must scale the entire app, not just the hot component
- Long build/test cycles as codebase grows
- Technology lock-in — can't use Python for ML and Java for API
- Team coupling — stepping on each other's code

### Microservices

Decomposed into independent services, each with its own process and (ideally) database.

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│  Auth   │  │ Orders  │  │ Catalog │
│ Service │  │ Service │  │ Service │
└────┬────┘  └────┬────┘  └────┬────┘
     │             │             │
  ┌──┴──┐      ┌──┴──┐      ┌──┴──┐
  │  DB │      │  DB │      │  DB │
  └─────┘      └─────┘      └─────┘
```

**Pros:**
- Independent scaling (scale only the bottleneck)
- Independent deployments (faster releases)
- Technology heterogeneity (right tool for each job)
- Team autonomy (team owns entire service)
- Fault isolation (one service failure doesn't bring down everything)

**Cons:**
- Distributed systems complexity (network calls, partial failures)
- No atomic transactions across services (saga pattern needed)
- Operational overhead (many services to monitor, deploy)
- Service discovery and routing complexity
- Testing is harder (need to mock/spin up dependencies)
- Data consistency is harder (each service owns its DB)

### When to Choose Which

**Start with a monolith when:**
- Early-stage product (< 12 months)
- Small team (< 10 engineers)
- Unclear domain boundaries
- Need to iterate fast

**Move to microservices when:**
- Specific scaling bottlenecks that can't be solved otherwise
- Clear team/domain boundaries exist
- Different parts need different technology
- Deployment frequency is high and all-or-nothing is a problem

**Rule of thumb:** Don't start with microservices. The complexity cost is real. Modular monolith → extract services as needed.

---

## Event-Driven Architecture

Components communicate via events rather than direct calls.

```
Producer → Event → [Message Broker] → Consumer A
                                    → Consumer B
                                    → Consumer C
```

**Benefits:**
- Decoupling — producer doesn't know who consumes
- Fan-out — one event, many consumers
- Resilience — consumer failure doesn't affect producer
- Temporal decoupling — consumer can process when it's ready

**Challenges:**
- Eventual consistency (event processing is async)
- Debugging is harder (event chains vs stack traces)
- Ordering guarantees require careful design
- Schema evolution (events are API contracts)

See `01-system-design-foundations/messaging-queues.md` for details.

---

## Synchronous vs Asynchronous Communication

| Aspect | Synchronous | Asynchronous |
|--------|------------|--------------|
| Latency | Client waits for response | Client gets acknowledgment immediately |
| Coupling | Tight (caller needs callee up) | Loose (callee can be down) |
| Error handling | Immediate feedback | Errors discovered later |
| Throughput | Limited by slowest link | Queue absorbs spikes |
| Complexity | Simple | Requires idempotency, ordering, DLQ |
| Best for | User-facing reads | Background processing, fan-out |

**Hybrid approach (recommended):**
```
User request → Sync API → validate → store → enqueue → 202 Accepted
                                              ↓ async
                                    Worker processes event
                                    (send email, update feed, etc.)
```

---

## Interview Discussion Points

- "I'd start with vertical scaling here — simpler to operate and we have headroom. When we hit the limit, we add read replicas."
- "The service needs to be stateless so we can auto-scale horizontally. Sessions should live in Redis, not in-process."
- "At 100K QPS, a single relational DB won't hold up. We need to think about sharding strategy or offloading reads to replicas and caches."
- "I'd propose a cache-aside strategy with a 60-second TTL and background refresh to avoid thundering herd on popular items."
- "We should keep this as a modular monolith now and extract the notification component as a service only when team boundaries or scaling needs justify it."
