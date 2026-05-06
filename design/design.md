# Discount Service — SRE Design

> **Stack:** AWS · EKS · Aurora PostgreSQL · ElastiCache Redis · Datadog
> **Scope:** v1 — single-region multi-AZ
> **Author:** Karan Barwa
> **Last updated:** 2026-05-06

---

## Prioritization

Given the 3-hour budget, depth went into **observability**, **resilience**, and the **runbook** — the things most likely to matter on day one of production. Capacity math is back-of-envelope but shown explicitly. CI/CD is designed but not exhaustively detailed.

---

## Architecture

```
                Checkout Service
                      │  HTTP, synchronous, p99 < 150 ms
                      ▼
            ┌─────────────────────────────┐
            │      discount-service       │
            │  Python async · Kubernetes  │
            │                             │
            │  /calculate    /validate    │
            └────────┬───────────┬────────┘
                     │           │
                ┌────▼───┐  ┌────▼───┐
                │ Redis  │  │ Redis  │
                │ rules  │  │vouchers│
                │ 5 min  │  │ 30 sec │
                └────┬───┘  └────┬───┘
                     │ miss      │ miss
                     └─────┬─────┘
                           ▼
                 ┌─────────────────┐
                 │ Aurora Postgres │
                 │  discount_rules │
                 └─────────────────┘
```

**Note on PgBouncer:** Connection pooler (multiplexes many client-side DB connections through a small pool of server-side connections). Omitted from this diagram for readability but **mandatory before peak-sale readiness**. Ships in v1.5 — see Capacity section.

**Request flow:**
1. Checkout calls `calculate_cart_discounts` synchronously.
2. Service checks Redis for cached rule set (key: `brand+category+bank`).
3. Cache miss → query Postgres → populate cache with 5-min TTL (time-to-live).
4. Apply discount rules → return `DiscountedPrice`.
5. `validate_discount_code` follows the same pattern; voucher cache TTL is 30 seconds because codes can be revoked mid-session.

---

## 1. Observability

Three pillars: **structured logging** (one machine-parseable line per request), **metrics** (numerical aggregates over many requests), and **distributed tracing** (the path one request took across services, broken into timed spans). All three feed Datadog.

### Structured Logging

Each request emits one JSON line with: `trace_id` (correlates a request across services), `request_id`, `duration_ms`, `cache_hit`, `discount_types_evaluated`, `customer_segment`, error fields when applicable.

**Cache hit, success:**
```json
{
  "level": "INFO",
  "event": "calculate_cart_discounts.success",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "duration_ms": 12,
  "cache_hit": true,
  "discount_types_evaluated": ["brand", "category"],
  "customer_segment": "loyalty_gold",
  "original_price_cents": 8500,
  "final_price_cents": 6800
}
```

**DB latency approaching SLO:**
```json
{
  "level": "WARN",
  "event": "calculate_cart_discounts.slow",
  "duration_ms": 134,
  "cache_hit": false,
  "db_query_ms": 121,
  "warning": "db_latency_approaching_slo_budget"
}
```

**Never logged:** raw voucher codes (SHA-256 hashed instead — log access ≠ replay surface), full customer PII (segment only), full cart contents.

### Metrics — RED + USE + Business

**RED** (Rate, Errors, Duration — request-level health) and **USE** (Utilization, Saturation, Errors — resource-level health) are the standard SRE base. Business-correctness metrics layer on top.

| Metric | Type | Why |
|---|---|---|
| `discount_request_total` | Counter | RED: rate, by `endpoint` and `outcome` |
| `discount_request_duration_seconds` | Histogram | p99 latency SLO is computed from this |
| `discount_cache_hit_ratio` | Gauge | Leading indicator — drops precede latency by 5–10 min |
| `discount_db_pool_utilization` | Gauge | USE saturation — first peak bottleneck |
| `discount_applied_amount_cents` | Histogram | **Business correctness** — catches silent miscalculation regressions |
| `discount_zero_applied_total` | Counter | Spike → voucher system or rule misconfiguration |

**Why `discount_applied_amount_cents` matters:** if a code change zeroes out all bank discounts, error rate is fine and latency is fine — but the average discount given drops sharply within minutes. RED metrics will never surface a correctness regression. This one will.

**Cardinality discipline:** Cardinality = number of unique label-value combinations on a metric. Each unique value = a separate stored time-series. High-cardinality labels explode cost (per-series billing in Datadog) or OOM the scrape target (in Prometheus).
- **Allowed labels** (bounded): `endpoint`, `outcome`, `discount_type` (4 values), `cache_hit`, `brand_tier` (3 values), `customer_segment` (~10 values).
- **Never as labels:** `voucher_code`, `customer_id`, `cart_id`, `request_id`, raw `brand`, full SKU. These belong in logs and traces, never metrics.

### Distributed Tracing (OpenTelemetry → Datadog APM)

A trace is the path one request took, broken into spans (each unit of work — one Redis call, one DB query). Spans nest to show the call tree.

```
[HTTP: calculate_cart_discounts]            # total span
  ├─ [redis.get rules_cache_lookup]         key=brand:zara|cat:dresses|bank:hdfc
  ├─ [postgres.query fetch_discount_rules]  # only on miss
  │     db.rows_returned=12
  ├─ [redis.set rules_cache_populate]       ttl=300s
  └─ [app: apply_discounts]
        discount.types_evaluated=["brand","bank"]
        discount.applied_count=2
```

**Span attributes that matter for debugging:**
- `cache_hit` — instantly bisects "Redis problem or Postgres problem?"
- `db.rows_returned` — 10× spike is the canonical signature of a missing index → sequential scan
- `discount.types_evaluated` — filters traces for bank-offer paths during incidents
- `error.type` + `error.message` on failure spans

### Grafana Dashboard — "Discount Service Health"

Designed around the on-call's attention path, not the system's component layout.

**Top row** (eye scans first) · RPS by endpoint · p99 latency 1-min rolling · error rate · cache hit ratio
**Second row** (where it hurts) · DB query p99 · DB pool utilization · Redis p99
**Third row** (correctness) · `discount_applied_amount_cents` p50/p99 · `discount_zero_applied_total` rate · errors by `error_type`

**Annotations:** deploy markers and sale-event windows (don't page on expected spikes).

---

## 2. SLI / SLO & Error Budget

An **SLI** (Service Level Indicator) is a precisely-defined measurement. An **SLO** (Service Level Objective) is the target for that SLI. The **error budget** is the allowed shortfall — when burned through, ops behavior changes.

### SLI Definitions

**SLI-1 — Availability.** Fraction of `calculate_cart_discounts` and `validate_discount_code` requests returning HTTP 2xx **AND** finishing within 150 ms, over a rolling 28-day window. *Combined definition:* a 200 ms response is functionally identical to a 5xx — checkout has timed out upstream.

**SLI-2 — Correctness.** Fraction of calculations where `final_price ≤ original_price` AND `applied_discounts` is non-empty when at least one rule matched. Validated by a shadow comparison job (runs alongside production traffic, checks expected outputs against fixtures) every 5 minutes.

**SLI-3 — Cache hit ratio (leading indicator, not SLO).** 5-min rolling. Tracked but not an SLO target — drops here predict latency SLO breach by 5–10 min.

### SLO Targets

| SLO | Target | Window | Justification |
|---|---|---|---|
| Availability | **99.9%** | 28d | 43.8 min/month error budget. Honest for single-region multi-AZ; 99.95% requires multi-region active-active (both regions serving traffic concurrently). |
| Latency | **p99 < 150 ms** | 5-min | Spec requirement. Owned despite Python GC pauses making it costly. |
| Correctness | **99.99%** | 7d | Tighter than availability: wrong price > no price (asymmetric cost). |

**Why correctness > availability:** Asymmetric cost. Overcharging is a legal/refund/trust issue. Undercharging is silent revenue loss — no customer complains when they pay less. The bar for "did we calculate correctly" must be stricter than "did we respond at all."

### Error Budget Policy

28-day budget for 99.9% = **43.8 minutes**.

| Consumed | Operational change |
|---|---|
| **< 50%** | Normal ops. Feature work and deployments proceed. |
| **50%** (~22 min) | Eng lead notified. New feature deploys need SRE sign-off. |
| **75%** (~33 min) | **Feature freeze** (no new functionality). Reliability fixes only. Causal review: one incident or chronic small failures? |
| **100%** | Full freeze. Incident retrospective required before resuming. SLO target reassessed with leadership. |

**Decision rights:** Eng manager + SRE lead at 50/75%. VP Eng at 100%. **Correctness breaches** bypass the budget gates and escalate immediately as incidents at any level. **External-event budget burn** (AWS AZ outage, upstream dependency failure) is reviewed separately — we don't penalize the team for failures outside our control.

---

## 3. Deployment & Resilience

### Kubernetes Topology

```yaml
namespace: discount-service        # isolated from checkout for blast radius control
replicas:
  min: 6                           # 3 AZ × 2 pods/AZ — survives full AZ loss
  max: 100                         # ~1.5× peak demand for assumption-error margin

resources:
  requests: { cpu: 500m,  memory: 512Mi }
  limits:   { cpu: 1000m, memory: 768Mi }    # OOM > silent slowdown

affinity:
  podAntiAffinity: required        # no two pods on same node — eliminates shared-fate
  topologySpreadConstraints:
    maxSkew: 1
    topologyKey: topology.kubernetes.io/zone

hpa:
  cpu_target: 60%                  # NOT 70% — see why below
  scaleUp:   stabilize=60s, +100%/min     # asymmetric: fast up
  scaleDown: stabilize=300s                # slow down — avoid post-sale thrash

pdb:
  maxUnavailable: 2                # PDB = Pod Disruption Budget; covers drain + 1 failure

probes:
  liveness:  /healthz (initialDelay=15s, period=10s, failureThreshold=3)
  readiness: /readyz  (initialDelay=5s,  period=5s,  failureThreshold=2)
  startup:   /readyz  (failureThreshold=30, period=2s)   # 60s cold-start budget
```

`/readyz` checks DB connection pool and Redis reachability. A pod that can't reach dependencies is removed from rotation but not killed — distinguishes "transient blip" from "broken pod."

**Why HPA at 60% CPU, not 70%:** Python GC (garbage collection — periodic memory cleanup that pauses the process) creates uneven utilization. At 70% average, individual pods spike to 95%+ during GC, async event-loop queueing kicks in, p99 breaches. 60% gives ~15% headroom. If load testing shows GC pressure is mild, 65% is defensible — the number is a hypothesis to validate.

**Why min 6:** 3 AZs × 2 pods/AZ. With only 3 pods (one per AZ), losing an AZ removes 33% capacity with no buffer for a concurrent failure.

### Top 3 Failure Modes

**1. Sequential scan on `discount_rules` after a new bank offer.** A new offer with `applicable_card_types = ARRAY['visa','mastercard','amex']` defeats the GIN index (Postgres index type for array/JSON columns) if it doesn't cover the new pattern. Plan flips to `Seq Scan` (full table read instead of index lookup); `db_query_p99` jumps from ~5 ms to 200+; `db.rows_returned` spikes 10×.

*Mitigation:* CI runs `EXPLAIN ANALYZE` (Postgres command that shows the actual query plan) against a production-scale staging dataset for any rule-column migration. Read replica for rule queries; primary for writes only. If it lands in production: bump Redis TTL 5m → 30m while `CREATE INDEX CONCURRENTLY` (creates an index without locking writes) runs.

**2. Redis eviction during Black Friday cascades to Postgres.** Default `allkeys-lru` evicts any key under memory pressure. Cold cache at 20K RPS = 20K queries/sec to a DB sized for 200–400/sec. Pool saturates in seconds — the **thundering herd** (many requests piling onto a slow dependency at once).

*Mitigation:* Pre-size Redis (rules dataset is < 100 MB — eviction implies a leak). Use `volatile-lru` so only TTL'd keys are evicted; pin critical rule keys without TTL. **Circuit breaker** (auto-trips when a downstream is failing): if pool utilization > 90% for 10s, serve stale rules instead of queueing more DB connections.

**3. Silent miscalculation for a brand+bank combination.** A rule-priority ordering bug applies brand and bank discounts multiplicatively instead of additively for Zara + HDFC. Final price 30% lower than intended. *Will not show in error rate or latency metrics.*

*Mitigation:* Correctness shadow job (SLI-2) every 5 min against known fixtures. Canary uses `discount_applied_amount_cents` distribution — auto-rollback on > 15% shift. Daily finance reconciliation against expected vs. applied totals; > 0.1% deviation pages.

### Graceful Degradation

The central tension on the checkout path: **fail-open** (degrade gracefully and serve a reduced response) vs. **fail-closed** (refuse the request entirely). **Decision: fail open.**

- **Redis unreachable** → bypass cache, query Postgres directly. Doubles DB load but stays correct.
- **Postgres unreachable + cold cache** → return zero discounts (`final_price = original_price`, `message = "Discounts temporarily unavailable"`). Customers complete checkout at full price. Explicit trade: business cost of not applying discounts < revenue cost of blocking checkout.
- **Postgres slow (> 100 ms)** → serve stale cached rules up to 10 min old.
- **Never acceptable:** `final_price > original_price` (legal/regulatory exposure) or substantially under-priced (revenue exposure). Returning original price is conservative-safe.

---

## 4. CI/CD & Change Safety

```
PR opened
  ├─ ruff · mypy
  ├─ pytest (fixture-based discount logic)
  ├─ Snyk (deps) · Bandit (SAST — static application security testing)
  ├─ Trivy image scan · blocks HIGH/CRITICAL CVEs
  └─ Review · 1 SRE + 1 domain engineer for logic changes

Merge → main
  ├─ Integration tests · ephemeral Postgres + Redis (testcontainers)
  ├─ Contract tests · DiscountedPrice schema (don't break checkout's contract)
  └─ Push → ECR (tagged with git SHA) → auto-deploy staging

Staging
  ├─ Smoke tests · /calc + /validate with known inputs
  ├─ k6 load · 5 min @ 2× steady-state RPS · p99 must stay < 120 ms
  ├─ Correctness fixture suite · 50 carts with exact expected prices
  └─ Manual sign-off for changes touching discount calculation logic

Production (manual trigger after staging green)
  ├─ Canary 5% traffic · 10 min
  │     auto-rollback if: error rate > 0.5% · p99 > 140 ms · applied_amount drift > 15%
  └─ Progressive 25% → 50% → 100% in 10-min increments (same gates)
```

### Discount-Logic Rollout Gates

Discount bugs produce wrong answers silently — standard error/latency gates won't catch them. Three additional gates:

1. **Correctness fixture suite** in staging — 50 pre-computed cart/rule combinations with exact expected prices. Primary safety gate.
2. **Canary metric — distribution comparison.** p25/p50/p75 of `discount_applied_amount_cents` compared between canary pods and stable. Shift > 15% triggers automated rollback. Catches the "discount is now 0 for everyone" bug class.
3. **Shadow mode** for major pricing logic changes — run new logic in parallel for 24 hours in production, return the *old* answer to users, alert on output divergence. Zero customer impact, full visibility.

### Rollback

| Type | Target time | How |
|---|---|---|
| Code rollback | **< 5 min** | `kubectl rollout undo deployment/discount-service` |
| Schema rollback | 15–30 min | Compensating migration. Avoided by policy. |

**Migration policy:** all schema changes must be **backward-compatible with N-1 code** (works with both the previous and current app version). Additive only — no breaking renames or constraint changes. Renames split: add column → backfill → drop, across separate deploys.

### Secrets & Supply Chain

- DB credentials and Redis auth via **AWS Secrets Manager**, injected at pod start via the Secrets Store CSI driver (Kubernetes plugin that mounts secrets from cloud secret managers as files). Never baked into images.
- `git-secrets` pre-commit hook + CI scan block accidental check-ins.
- **Image signing** via AWS Signer; **OPA admission policy** (Open Policy Agent — cluster-level rule engine) refuses unsigned images.

---

## 5. Capacity & Cost

### Latency Budget (p99 < 150 ms)

The end-to-end SLO is meaningless without allocating it across hops. These are p99 targets per component, not hard timeouts.

| Hop | Budget | Notes |
|---|---|---|
| Network in/out (LB + service mesh) | 20 ms | Round-trip from checkout |
| Redis lookup (rules + voucher) | 10 ms | Hard timeout 25 ms — fail to DB on breach |
| Postgres query (cache miss only) | 50 ms | Hard timeout 100 ms — fail to stale cache |
| Application logic (rule evaluation) | 20 ms | Includes JSON serialization |
| Headroom (GC pauses, queueing, jitter) | 50 ms | Absorbs Python GC pauses + tail noise |
| **Total** | **150 ms** | Matches SLO |

### Throughput Math

**Per-pod throughput:** ~300 RPS for Python async (FastAPI/uvicorn) at 20 ms avg, accounting for GC, connection-pool contention, TLS overhead. **The single most important assumption — validate by load test on day one.**

| Tier | Math | Result |
|---|---|---|
| Steady-state pods | 2,000 RPS ÷ 300 RPS/pod | **~7 pods** · run 10 for headroom · floor at 6 (AZ resilience) |
| Peak pods | 20,000 RPS ÷ 300 RPS/pod | **~67 pods** · `maxReplicas=100` for assumption-error margin |
| Peak nodes | 67 vCPU ÷ 8 vCPU/node | ~9 m5.2xlarge (or ~5 m5.4xlarge) |
| Postgres at peak (15% cache miss) | 20K × 0.15 | **3,000 q/s** · r6g.2xlarge + read replica |
| Connections at peak | 67 pods × 15 | **~1,000** · under Aurora's `max_connections` ~1,700 limit, but tight |
| Redis working set | 20MB rules + 100MB vouchers | **~200 MB** · r6g.large oversized but cheap insurance |

### Backpressure & Admission Control

At 20K RPS, queueing causes **tail latency collapse** — requests pile up faster than they drain, average latency climbs into multi-second territory before timing out, and the system becomes uniformly broken instead of partially serving. Three controls:

- **Deadline propagation:** checkout passes a deadline header (how long it can wait); we reject early with `503 deadline_exceeded` rather than start work we can't finish in time.
- **Bounded request queue per worker** (max 50 in-flight); above that, fail fast with `503 overloaded`. Controlled rejection > cascading timeout.
- **DB pool wait cap:** if a request waits > 30 ms for a connection, abort and return zero-discount fallback rather than block the worker.

### First Bottleneck — Postgres connection ceiling

At 67 pods × 15 = ~1,000 connections, comfortably under Aurora's ~1,700 limit. *But* if per-pod throughput is lower than estimated and pod count grows toward 100+, we re-enter the danger zone.

**Mitigation: PgBouncer in transaction-pooling mode** (multiplexes per transaction — reuses a server-side connection between requests rather than for the whole client session). Holds ~200 server-side connections; multiplexes thousands of client-side connections through them. **Mandatory before peak-sale readiness.** Omitted from the simplified architecture diagram for readability; ships in v1.5.

### Idempotency & Retry Safety

Checkout retries on transient failures. The calculation is naturally **idempotent** (same inputs → same output, every time), but invariants must hold:

- **Pure functions:** rule evaluation never depends on wall-clock time except explicit voucher expiry checks (deterministic given the input timestamp).
- **Idempotency key on `validate_discount_code`** — the request-ID short-circuits duplicate redemption-counter increments on retry. For single-use vouchers, retry could otherwise consume the redemption twice.
- **No side effects in the calculate path:** reads only. Cache populates are safe to repeat. Any future audit log writes must use upserts keyed by request ID.

### Cost Optimization — Spot Worker Tier

**Spot instances** (spare AWS capacity at 60–70% discount, reclaimable with 2-min warning) for non-baseline pods. Run 6 on-demand pods (the reliability minimum); elastic capacity above that floor on Spot. Estimated $200–300/month savings, scaling with peak fleet.

PDB ensures at least 4 on-demand pods survive a Spot reclamation event.

**Black Friday: switch to 100% on-demand for the 4–6 hour window.** Spot reclamation rates spike during high-demand windows — exactly when we can least afford a forced reclamation cascade. Pre-purchase Savings Plans for the baseline.

---

## 6. Runbook · DB Pool Saturated

**Trigger:** `discount_db_pool_utilization > 95%` for over 2 minutes. Requests queueing, p99 climbing toward SLO breach.

### Symptoms

- PagerDuty: `DiscountService_DBPoolSaturation` (threshold: pool utilization > 90% for 2 consecutive 1-min windows)
- Pool utilization gauge red on Grafana
- p99 latency climbing from baseline (~20 ms) toward 150 ms
- Logs: `WARNING pool_wait_ms=…` increasing, possibly `ERROR pool_timeout`

### Step 1 — Pool problem, or DB problem?

```promql
discount_db_pool_utilization                            # > 90% confirms pool
discount_db_query_duration_seconds{quantile="0.99"}     # is DB itself slow?
```

- Pool high + queries fast (< 10 ms) → connection-count problem. **Stay in this runbook.**
- Pool high + queries slow (> 50 ms) → DB is the root cause. **See: Runbook — DB Query Latency Spike.** Mitigations here will buy time but won't fix it.

### Step 2 — What changed (ranked by probability)

1. **Traffic spike** — check RPS panel. Sale event or unexpected traffic?
2. **Recent deploy bumped pool size** — `kubectl describe deploy discount-service | grep DB_POOL_SIZE`. Compare with deploy annotations on the timeline.
3. **HPA hasn't caught up** — `kubectl get hpa discount-service -n discount-service`. Current < desired?
4. **Cache hit ratio dropped** — drop from 85% to 60% doubles DB query rate.
5. **Slow query appeared** — each query holds a connection longer. Check Aurora Performance Insights.

### Step 3 — Mitigate

**If traffic spike, HPA lagging:**
```bash
# First: verify we won't blow Aurora's max_connections
psql -h $DB_HOST -U $DB_USER -c "SELECT count(*) FROM pg_stat_activity;"

# Force scale-up (don't wait for HPA stabilization window)
kubectl scale deployment discount-service --replicas=50 -n discount-service
```

**If bad deploy:**
```bash
kubectl rollout undo deployment/discount-service -n discount-service
kubectl rollout status deployment/discount-service -n discount-service
```

**Last resort — shrink per-pod pool:**
```bash
kubectl set env deployment/discount-service DB_POOL_SIZE=5 -n discount-service
```

### Step 4 — Verify recovery

All three trending right before closing:
- `discount_db_pool_utilization` < 70%
- `discount_request_duration_seconds{quantile="0.99"}` < 50 ms
- `discount_request_total{outcome="error"}` rate at zero

### Escalation Path

| Time | Action |
|---|---|
| 0 min | On-call works the runbook |
| 10 min | No improvement → page on-call lead + DB/infra team |
| 20 min | SLO breach confirmed → page eng manager. Consider activating zero-discount fallback. |
| 30 min | Incident commander declares SEV-1; checkout team notified. |

**Activate zero-discount fallback (last resort):**
```bash
kubectl set env deployment/discount-service DISCOUNT_FALLBACK_MODE=zero -n discount-service
```
Returns `original_price` for all requests. Customers unhappy (no discounts) but checkout works.

### Post-Incident Actions

- [ ] Root cause document within 24 hours
- [ ] If connection growth: evaluate PgBouncer (should be backlogged already)
- [ ] If cache eviction cascade: review Redis eviction policy and memory sizing
- [ ] Link runbook to PagerDuty alert definition
- [ ] If > 20% of error budget consumed: trigger budget review

---

---

## 6b. Runbook · Voucher Codes Being Rejected

**Trigger:** Spike in `discount_zero_applied_total` from voucher endpoint, or customer-service ticket surge ("my code says invalid"), or `/validate_code` 4xx rate elevated.

### Symptoms

- PagerDuty: `DiscountService_VoucherRejection_HighRate`
- `discount_zero_applied_total{endpoint="validate_code"}` rate >> baseline
- Support inbox flooded with "valid code being rejected" tickets
- `/validate_code` returning `false` at 5–10× normal rate

### Step 1 — Confirm scope

```promql
rate(discount_validate_code_total{outcome="invalid"}[5m])   # rejection rate
rate(discount_validate_code_total[5m])                       # total rate
```

Is rejection rate elevated *as a fraction*, or just because traffic is up? If fraction is normal, this isn't an incident — it's volume.

### Step 2 — What changed (ranked by probability)

1. **Recent voucher-rule deploy** — check deploy timeline. Did a new rule misconfigure expiry, brand scope, or stacking?
2. **Voucher cache poisoning** — Redis cached an "invalid" answer (e.g. due to a transient DB error during validation). 30s TTL means it self-heals fast, but during the window many requests see stale invalid.
3. **Read-replica lag** — admin just bulk-loaded new voucher codes into the primary; the replica doesn't have them yet, validation against the replica says "not found."
4. **Database row-level issue** — a recent migration left voucher table in inconsistent state.
5. **Timezone / clock bug** — expiry check using server-local time instead of UTC. Surfaces around midnight UTC.

### Step 3 — Mitigate

**If recent voucher-rule deploy:**
```bash
kubectl rollout undo deployment/discount-service -n discount-service
```

**If cache poisoning suspected (rejection rate is patchy, recovers in seconds then re-spikes):**
```bash
# Flush voucher keyspace only — leave rules cache alone
redis-cli -h $REDIS_HOST --scan --pattern 'voucher:*' | \
  xargs redis-cli -h $REDIS_HOST DEL
```

**If read-replica lag (admin bulk-load was recent):**
```bash
# Force voucher reads from primary until replica catches up
kubectl set env deployment/discount-service VOUCHER_DB_TARGET=primary -n discount-service
# Revert once Aurora replica lag returns to < 1s
```

**Customer-side workaround while debugging:**
- Customer service has a list of pre-approved manual override codes that bypass the validate path. They can reissue these to affected customers.

### Step 4 — Verify

- `discount_validate_code_total{outcome="invalid"}` rate back to baseline (typically < 5% of total)
- Spot-check 5 known-good codes via curl
- Customer-service ticket inflow normalising

### Escalation

| Time | Action |
|---|---|
| 0 min | On-call works the runbook |
| 15 min | No improvement → page on-call lead + voucher-platform owner |
| 30 min | Sustained customer impact → page eng manager + Customer Service lead |

### Post-incident

- [ ] Root-cause document within 24 hours
- [ ] If voucher-rule deploy: tighten the fixture suite to cover the failed case
- [ ] If cache poisoning: review `validate_code` retry logic — should we cache negative answers at all?
- [ ] If replica lag: document the "VOUCHER_DB_TARGET=primary" toggle in the runbook
- [ ] Reach out to affected customers with apology + recovered discount

---

## Assumptions

- **Python async** (asyncio + FastAPI/uvicorn), not synchronous WSGI. Capacity math changes substantially under sync workers.
- **Aurora PostgreSQL**, not vanilla RDS. Failover is faster (15–30s vs. 60–120s); read replicas are easier to add.
- **Datadog.** Unified UI for traces/metrics/logs at 3 AM matters more than vendor cost. Prom + Grafana + Jaeger is a valid alternative.
- **Voucher TTL = 30s** because codes can be revoked. 5-min TTL is fine if revocation isn't a feature.
- **Per-pod throughput ≈ 300 RPS.** Validate by load test before treating as fact.

## Explicit Trade-offs

- **99.9% availability, not 99.95%.** Honest for single-region multi-AZ. 99.95% requires multi-region active-active.
- **Fail-open with zero-discount fallback** instead of fail-closed. Customer experience tradeoff. Must be a product decision.
- **HPA target 60% CPU, not 70%.** Headroom for Python GC bursts.
- **PgBouncer ships in v1.5, not v1.** Operational complexity vs. simplicity. Mandatory before first peak-sale event.
- **Spot instances for non-baseline pods.** Reclamation risk vs. ~$200–300/mo savings. Reverted to on-demand during sales.

## What I'd Do With Another Week

1. Load test the async service to validate the 300 RPS/pod assumption — entire capacity model rests on it
2. Detail the correctness shadow job (named in SLI-2 but not fully specified)
3. Write the second runbook — voucher rejection (different Redis key patterns, different escalation)
4. Specify the PgBouncer deployment topology and failover behavior
5. Pre-scaling playbook for known sale events (cron pre-provisioning nodes 30 min ahead)
