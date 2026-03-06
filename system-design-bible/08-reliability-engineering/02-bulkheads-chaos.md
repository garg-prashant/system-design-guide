# Bulkheads and Chaos Engineering

## Bulkheads

**Idea:** Isolate resources (threads, connections, CPU) so one failing dependency doesn’t exhaust all resources and kill the whole process.

- **Thread pool per dependency:** e.g. 10 threads for payment service, 20 for inventory. If payment is slow, only those 10 are blocked; rest of app can serve other traffic.
- **Connection limits:** Max connections per downstream; queue or reject when full.
- **Kubernetes:** CPU/memory limits per container; separate deployments for critical vs best-effort.

**Interview:** “We use bulkheads so a slow downstream (e.g. recommendation service) can’t consume all our threads; we limit connections per dependency and return degraded response when pool is full.”

---

## Chaos Engineering

**Purpose:** Prove that the system tolerates failure by **intentionally** injecting faults (kill pod, add latency, drop packets) in a controlled way.

- **Principles:** Start with hypothesis (e.g. “if DB primary fails, replicas take over within 30s”); run in production-like env first; start small (e.g. one node); automate and repeat.
- **Examples:** Chaos Monkey (random instance kill); latency injection; disk full; network partition.
- **Blast radius:** Limit who is affected (e.g. canary fleet, single AZ); have rollback and kill switch.

**Interview:** “We run chaos experiments (e.g. kill a replica, inject latency) in staging and small production segments to validate failover and recovery; we limit blast radius and have rollback.”
