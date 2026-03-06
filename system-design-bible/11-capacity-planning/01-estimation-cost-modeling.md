# Capacity Estimation and Cost Modeling

## Back-of-the-Envelope Estimation

**Purpose:** Size components (QPS, storage, bandwidth) so the design is plausible and you can choose the right scale.

### Steps

1. **Clarify scale:** DAU, requests per user per day, read/write ratio, payload sizes.
2. **QPS:** e.g. 100M DAU × 10 requests/day ≈ 100M×10 / 86400 ≈ 12K req/s average; peak 2–3× → ~30K req/s.
3. **Storage:** e.g. 100M users × 1 KB profile = 100 GB; growth per year.
4. **Bandwidth:** e.g. 30K req/s × 10 KB response = 300 MB/s ≈ 2.4 Gbps.

### Reference numbers (from guide)

- 1 server: order of 10K–100K req/s (depends on work).
- 1M rows × 100 bytes ≈ 100 MB; 1B rows × 100 bytes ≈ 100 GB.
- Use these to sanity-check (e.g. “30K QPS → we need at least a few app servers and a DB that can do that write rate”).

---

## Cost Modeling

- **Compute:** vCPU × hours; spot vs on-demand.
- **Storage:** GB/month; SSD vs HDD; egress often expensive.
- **Data transfer:** Inbound often free; outbound (e.g. to internet) charged.
- **Managed services:** DB, cache, queue — often dominant; compare self-hosted vs managed TCO.

**Interview:** “We estimated 30K QPS and 100 GB storage; we’d need N app servers, a DB with replicas, and a cache layer; we’d model cost for compute, storage, and egress to stay within budget.”
