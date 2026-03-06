# SLO, SLA, SLI, and Alerting

## Definitions

| Term | Meaning |
|------|---------|
| **SLI (Service Level Indicator)** | A measurable metric (e.g. availability = successful requests / total requests; latency = p99). |
| **SLO (Service Level Objective)** | Target for an SLI (e.g. availability ≥ 99.9%; p99 latency ≤ 200 ms). |
| **SLA (Service Level Agreement)** | Contract with customers; consequences if SLO is missed (e.g. credits). Internal SLOs are not SLAs. |

---

## Good SLOs

- **User-centric:** Reflect what users care about (e.g. “successful checkout” not “DB up”).
- **Measured:** You can actually measure the SLI in production.
- **Achievable:** Team can meet them with normal effort; not “100%” unless you mean it.
- **Error budget:** 99.9% = 0.1% failure budget. Use it for release risk (e.g. freeze launches when budget exhausted).

---

## Alerting

- **Alert on SLO burn rate or error budget**, not on “any failure.” Avoid alert fatigue.
- **On-call:** Page when user impact (e.g. SLO violation or imminent); use runbooks and escalation.
- **Tiers:** Critical (page), warning (ticket), info (dashboard only).

**Interview:** “We define SLOs from user-facing SLIs (e.g. availability of API, p99 latency); we alert when we’re burning error budget too fast so we don’t miss our SLO; we use error budget to decide launch risk.”
