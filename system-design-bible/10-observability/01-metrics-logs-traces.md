# Metrics, Logs, and Traces

## The Three Pillars

| Pillar | What it is | Use case |
|--------|------------|----------|
| **Metrics** | Numeric time-series (counters, gauges, histograms). | Dashboards, alerting, SLOs. “What is the rate, latency, error %?” |
| **Logs** | Event records (message, timestamp, level, context). | Debugging, audit, search. “What happened for this request?” |
| **Traces** | Request flow across services (span per service, trace ID). | “Where did this request spend time? Which service failed?” |

---

## Metrics

- **Counter:** Monotonically increasing (requests, errors). Use rate() for req/s.
- **Gauge:** Current value (queue size, active connections).
- **Histogram / Summary:** Distribution (latency p50, p95, p99).
- **Labels:** dimensions (service, env, status_code). Don’t over-cardinality (e.g. user_id = bad).
- **Tools:** Prometheus, Datadog, CloudWatch. Store and query; alert on thresholds or SLOs.

---

## Logs

- **Structured (JSON):** Machine-readable; easy to query and aggregate.
- **Correlation:** Include trace_id, request_id so logs and traces link.
- **Level and sampling:** ERROR always; INFO/DEBUG sampled at high volume to control cost.
- **Retention:** Hot (search) vs cold (archive); define by compliance and cost.

---

## Distributed Tracing

- **Trace:** One request end-to-end. **Span:** One unit of work (e.g. one service call).
- **Trace ID:** Same for all spans of one request. **Span ID:** Per span. Parent-child relationship.
- **Propagation:** Trace ID (and span ID) passed via headers (e.g. W3C Trace Context); each service creates child spans.
- **Use:** Find latency breakdown per service; see which service failed or slowed.

**Interview:** “We emit metrics for SLOs and alerting, structured logs with trace_id for debugging, and distributed traces so we can see latency and errors across services for any request.”
