# Load Balancers

## Role

Distribute incoming traffic across multiple backend servers so no single server is overwhelmed. Essential for horizontal scaling and high availability.

```
                    ┌─────────────────┐
   Clients ───────→│ Load Balancer   │───────→ Server 1
                    │ (L4 or L7)      │───────→ Server 2
                    └─────────────────┘───────→ Server 3
```

---

## Layer 4 vs Layer 7

| Aspect | L4 (Transport) | L7 (Application) |
|--------|----------------|------------------|
| **Works on** | IP + port (TCP/UDP) | HTTP headers, path, cookies |
| **Visibility** | No payload inspection | Full request content |
| **Use case** | Raw TCP/UDP, high throughput | HTTP/HTTPS, path-based routing, sticky sessions |
| **Examples** | AWS NLB, HAProxy (TCP), LVS | AWS ALB, NGINX, HAProxy (HTTP) |

**When to use L4:** Maximum throughput, minimal latency, non-HTTP (e.g. database proxies, game servers).

**When to use L7:** HTTP routing by path/host, TLS termination at LB, cookie-based stickiness, request/response manipulation.

---

## Load-Balancing Algorithms

| Algorithm | Description | When to use |
|-----------|-------------|-------------|
| **Round robin** | Rotate through servers in order | Servers equal; stateless |
| **Weighted round robin** | Round robin with weights by capacity | Heterogeneous servers |
| **Least connections** | Send to server with fewest active connections | Long-lived connections (e.g. WebSocket, DB) |
| **Least response time** | Send to server with lowest latency | When latency varies by server |
| **IP hash** | Hash client IP → server | Stickiness without cookies (e.g. L4) |
| **Consistent hash** | Hash key (e.g. user id) → server | Cache affinity, minimal reassignment |

---

## Health Checks

The LB must stop sending traffic to unhealthy nodes.

- **Active:** LB periodically sends HTTP/TCP request to backend; mark unhealthy after N failures.
- **Passive:** Observe connection failures from real traffic; mark unhealthy after N failures.
- **Typical:** 10–30s interval, 2–3 failures to unhealthy, 2–3 successes to healthy.

**Interview point:** Without health checks, LB keeps sending traffic to dead or overloaded nodes; availability drops.

---

## Sticky Sessions (Session Affinity)

Send all requests from the same client to the same server.

- **Why:** In-memory session, local cache, or non-shared state.
- **How (L7):** Cookie (server-set or LB-set); or hash on header (e.g. session ID).
- **Trade-off:** Reduces horizontal scalability and complicates failover (session lost if that server dies). Prefer stateless + shared session store when possible.

---

## TLS Termination

LB terminates TLS (decrypts), then talks to backends over HTTP or re-encrypts (TLS to backend).

- **Pros:** Centralized cert management; backends stay simple; LB can do L7 routing.
- **Cons:** LB sees plaintext; CPU cost on LB; backends may be in same trust domain (no re-encrypt).

---

## Failure Modes and HA

- **Single LB = SPOF.** Use active-passive or active-active with a floating IP or DNS failover.
- **Keep backends stateless** so any server can serve any request after failover.
- **Connection draining:** On scale-in or deploy, stop new connections and wait for existing ones to drain (e.g. 30–60s).

---

## Trade-offs (Interview)

| Decision | Trade-off |
|----------|-----------|
| L4 vs L7 | L4: simpler, faster. L7: routing, TLS, headers, but more CPU and latency. |
| Sticky vs stateless | Sticky: simpler for stateful apps; stateless: better scale and failover. |
| Health check interval | Faster = quicker failover but more overhead; slower = more traffic to bad node. |
