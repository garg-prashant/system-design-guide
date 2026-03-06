# Circuit Breakers and Retries

## Retries

**When:** Transient failures (network blip, temporary overload). **When not:** Permanent errors (4xx validation, 404); non-idempotent operations without care.

### Best practices

- **Exponential backoff:** Wait 1s, 2s, 4s, … (with jitter) before retry to avoid thundering herd.
- **Max retries:** Cap total attempts (e.g. 3) so failing calls don’t hang forever.
- **Idempotency:** If the operation isn’t idempotent, use idempotency keys so the server can deduplicate.
- **Timeout:** Each attempt should have a timeout; total time = attempts × timeout (plus backoff).

**Interview:** “We retry with exponential backoff and jitter, up to N times, only on retryable errors (5xx, timeouts); we use idempotency keys for writes.”

---

## Circuit Breaker

**Purpose:** Stop calling a failing dependency so it can recover and so the caller doesn’t pile on (cascading failure). Fail fast and optionally fallback.

### States

```
  CLOSED (normal)  ── failure count > threshold ──→  OPEN
       ↑                                                    │
       │                                                    │ after timeout
       └──────────────  HALF-OPEN  ←────────────────────────┘
                            │
                    success → CLOSED   failure → OPEN
```

- **CLOSED:** Requests go through; count failures.
- **OPEN:** Requests fail immediately (no call); after a timeout, move to HALF-OPEN.
- **HALF-OPEN:** Allow a few test requests; success → CLOSED; failure → OPEN.

### Parameters

- **Failure threshold:** e.g. 5 failures or 50% error rate → open.
- **Timeout:** How long to stay OPEN before trying again (e.g. 30s).
- **Half-open trial:** How many requests to allow in HALF-OPEN.

**Interview:** “We use a circuit breaker so when the payment service is down we don’t keep hammering it; we fail fast and show a fallback (e.g. ‘try later’) and let the dependency recover.”
