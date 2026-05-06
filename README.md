# Discount Service — SRE Design

> 🌐 **Live rendered design:** https://karanbarwa10.github.io/unifize-sre-assignment/design.html
> *(Hover any component in the diagram to see its role and design rationale. Light/dark mode toggle in top-right.)*

A reliability, observability, and operations plan for a Python discount calculation service on the **synchronous checkout critical path** (every cart-pricing request goes through us — if we're slow, customers can't buy) of a fashion e-commerce platform.

**Stack:** AWS · EKS · Aurora PostgreSQL · ElastiCache Redis · Datadog
**Scope:** v1 — single-region multi-AZ (one geographic region, multiple availability zones for in-region resilience)
**Targets:** 2K RPS steady-state · 20K RPS peak · p99 < 150 ms · 99.9% availability

---

## Repository Layout

```
.
├── README.md          ← entry point: prioritization, assumptions, trade-offs
├── design.html        ← visual version with interactive component diagram (hover any box)
└── design/
    └── design.md      ← canonical six-area design document (Markdown)
```

**Read order:**
1. **This README** — what was prioritized, assumptions, explicit trade-offs.
2. **[design.html](design.html)** — rendered version. Hover any component in the architecture diagram to see its role and design rationale. Light/dark mode toggle.
3. **[design/design.md](design/design.md)** — the same content as plain Markdown for GitHub viewing.

---

## What This Repo Contains

The six required deliverables from the assignment:

| # | Section | Where |
|---|---------|-------|
| 1 | **Observability Plan** — structured logging, metrics, distributed tracing, dashboard | [design.md §1](design/design.md#1-observability) |
| 2 | **SLI / SLO and Error Budget Policy** | [design.md §2](design/design.md#2-sli--slo--error-budget) |
| 3 | **Deployment & Resilience Design** — K8s topology, failure modes, graceful degradation | [design.md §3](design/design.md#3-deployment--resilience) |
| 4 | **CI/CD and Change Safety** | [design.md §4](design/design.md#4-cicd--change-safety) |
| 5 | **Capacity & Cost** — latency budget, throughput math, bottleneck, cost optimization | [design.md §5](design/design.md#5-capacity--cost) |
| 6 | **Runbooks** — DB connection pool saturation + voucher code rejection | [§6](design/design.md#6-runbook--db-pool-saturated) · [§6b](design/design.md#6b-runbook--voucher-codes-being-rejected) |

---

## Prioritization

Three hours doesn't allow equal depth. Priorities:

| Priority | Area | Reasoning |
|----------|------|-----------|
| **High** | Observability | Without it, nothing else can be debugged at 3 AM |
| **High** | Resilience & failure modes | Checkout-blocking service — failures need named mitigations |
| **High** | Runbook | The acid test for whether the design holds up under stress |
| **Medium** | SLI/SLO | Small section, high leverage; **correctness as an SLI** is the unusual choice that matters |
| **Medium** | Capacity & cost | Math shown explicitly; per-pod throughput flagged as the assumption to validate first |
| **Skimmed** | CI/CD | Designed but not exhaustive — pipeline structure, key gates, rollback policy |

**What I'd do with another week** is documented at the end of [design.md](design/design.md#what-id-do-with-another-week).

---

## Assumptions

These are the assumptions the design rests on:

- **Python async** (asyncio + FastAPI/uvicorn), not synchronous WSGI. Capacity math changes substantially under sync workers.
- **Aurora PostgreSQL**, not vanilla RDS. Aurora's 15–30s failover (the automatic switch to a healthy replica when the primary dies) vs. RDS's 60–120s is material on the checkout path.
- **Datadog** as the observability platform — unified traces/metrics/logs in one UI matters at 3 AM more than the vendor cost. Prometheus + Grafana + Jaeger is a valid alternative.
- **Voucher TTL = 30 seconds** because voucher codes can be revoked (fraud, expiry, single-use redemption). 5-min TTL is acceptable if revocation isn't a feature.
- **Per-pod throughput ≈ 300 RPS** at 20 ms avg latency. **The single most important number to validate by load test on day one** — the entire capacity model rests on it. A concrete day-one k6 protocol with pass criteria and a decision matrix for what to do when results miss is in [design.md §5](design/design.md#validation-plan-day-one-load-test).
- **Single region, multi-AZ**, per the assignment. Multi-region active-active is out of v1 scope.

---

## Explicit Trade-offs

Senior decisions are about which discomfort to accept:

### Reliability vs. complexity

- **99.9%, not 99.95%.** Honest for single-region multi-AZ. 99.95% requires multi-region active-active (both regions serving concurrently with conflict resolution), which adds significant operational cost.
- **PgBouncer drawn in the v1 diagram with a `v1.5` label.** PgBouncer is a connection pooler that multiplexes many client-side DB connections through a smaller pool of server-side connections. Drawn with a double-line border so reviewers see the deployment phasing visually rather than reading a footnote. Adds operational complexity, but **mandatory before the first peak-sale event** — see Capacity section for connection-ceiling math.

### Customer experience vs. revenue

- **Fail-open with zero-discount fallback, not fail-closed.** When dependencies break, customers complete checkout at full price rather than getting blocked. The cost of not applying a discount is bounded; the cost of blocking checkout is unbounded. **Must be an explicit product decision, not an engineering one.**

### Cost vs. predictability

- **Spot instances for non-baseline pods.** Spot = spare AWS capacity at 60–70% discount, reclaimable with 2-min warning. ~$200–300/month savings above the 6-pod on-demand floor. **Reverted to 100% on-demand during sale windows** — Spot reclamation rates spike when we can least afford it.

### Performance vs. safety

- **HPA target 60% CPU, not 70%.** Python GC (garbage collection — periodic memory cleanup that pauses the process) creates uneven utilization. At 70% average, individual pods spike to 95%+ during GC, async event-loop queueing kicks in, and p99 breaches. The 10% headroom is insurance.
- **Backpressure via fast-fail at queue thresholds.** Tail latency collapse (uniformly slow under sustained overload) is more dangerous than controlled rejection.

### Speed of rollout vs. safety

- **Discount logic gets extra gates** (50-cart fixture suite, applied-amount drift canary, optional shadow runs) on top of standard error/latency checks. Pricing bugs are silent — generic gates don't catch them.

---

## Key Design Decisions at a Glance

| Decision | Choice | Why |
|----------|--------|-----|
| Cloud | AWS (EKS, Aurora, ElastiCache) | Mature, multi-AZ, faster Aurora failover than RDS |
| Observability | Datadog | Unified UI for 3 AM incident triage |
| Cache strategy | Two TTLs (rules 5 min, vouchers 30s) | Vouchers need shorter TTL because revocable |
| Eviction policy | `volatile-lru` + pin rule keys without TTL | Bound the eviction blast radius (limit how much breaks under memory pressure) |
| Min replicas | 6 (3 AZ × 2 pods) | Survive one full AZ loss with buffer |
| Max replicas | 100 | ~1.5× peak demand for assumption-error margin |
| HPA target | 60% CPU | Headroom for Python GC bursts |
| Availability SLO | 99.9% / 28d | Honest for single-region multi-AZ |
| Correctness SLO | 99.99% / 7d | Wrong price worse than no price (asymmetric cost) |
| Degradation | Fail open (zero discount) | Checkout completion > discount accuracy under failure |
| Canary metric | `discount_applied_amount_cents` distribution | Catches silent miscalculation bugs |

---

## Architecture (Simplified)

```
Checkout Service
      │  HTTP, sync, p99 < 150 ms
      ▼
discount-service (Python async, K8s, 6–100 pods)
      │
      ├──► Redis (rules, 5 min TTL)     ──┐
      ├──► Redis (vouchers, 30s TTL)    ──┤  cache miss
      │                                   ▼
      │                          ╔══════════════╗
      └──────────────────────────►  PgBouncer   ║   ← v1.5
                                 ╚══════╤═══════╝
                                        ▼
                                 Aurora PostgreSQL
                                 (main + read replica)
```

PgBouncer is drawn with a double-line box and a `v1.5` label — it ships in the next iteration, mandatory before the first peak-sale event. The interactive diagram in [design.html](design.html) has hover tooltips on every component with role + design rationale.

---

## AI Critique Log — Draft vs. Final

The assignment specifically evaluates *"the delta between the AI's first draft and your final design."* Below is the explicit list of places where the AI draft was generic, cargo-culted, or internally inconsistent, and what was changed (with reasoning).

| # | What the AI draft said | What was changed | Why |
|---|------------------------|------------------|-----|
| 1 | HPA target = **70% CPU** (the textbook number) | **60% CPU** | Python's GC pauses spike individual pods to 95%+ during collection; at 70% average, async event-loop queueing kicks in and p99 breaches. The 10% extra headroom is insurance, not waste. |
| 2 | Availability SLO = **99.95%** (the AI default for "high reliability") | **99.9%** | Honest for single-region multi-AZ. Aurora failover alone consumes 15–30s of the budget; 99.95% would require multi-region active-active, out of v1 scope. Promising 99.95% on this design would be lying. |
| 3 | Capacity math: **100 RPS/pod → 200 pods at peak**, but `maxReplicas = 60`. **Internally inconsistent** — peak demand exceeds the configured ceiling. | **300 RPS/pod, max 100 pods** | Reconciled with realistic Python async throughput. Now peak demand (~67 pods) sits comfortably under the ceiling with assumption-error margin. The number is also flagged explicitly as the day-one load-test priority. |
| 4 | "**p99 < 150 ms**" stated as a single end-to-end target | **Per-hop latency budget** (network 20 / Redis 10 / DB 50 / app 20 / headroom 50 = 150 ms) | An end-to-end SLO is meaningless without allocating it across hops. Without a budget, every component thinks it has 150 ms and consumes the whole thing. |
| 5 | Only **availability + latency** SLIs (the standard pair) | Added **correctness as a separate SLI** at **99.99%** — *higher than availability* | Wrong price > no price. Overcharging is a legal/trust issue; undercharging is silent revenue loss. Asymmetric cost means correctness must have a tighter bar. Most candidates miss this. |
| 6 | PgBouncer **mentioned as mitigation in prose but absent from the architecture diagram** | **Drawn into the diagram with a `v1.5` annotation** (double-line border + label) | Architectural ambiguity. Deployment phasing made visually explicit so reviewers see "ships in the next iteration" rather than reading a footnote. |
| 7 | Failure modes were **generic** ("DB slow", "Redis down") | Three **specific, named** failure modes: <br>(a) Sequential scan on `discount_rules` after a new bank offer with new `card_type` array values defeats the GIN index <br>(b) Redis `allkeys-lru` eviction at 20K RPS cascades 20K q/s onto a DB sized for 200–400 q/s <br>(c) Rule-priority bug applies brand+bank discounts multiplicatively for one combination — silent revenue loss | The rubric explicitly says *"the database is slow is not a failure mode."* Specific reproducible failure modes also give us specific mitigations. |
| 8 | **No backpressure / admission control** under sustained overload | Three controls added: deadline propagation, bounded request queue per worker (max 50 in-flight, fail fast above), DB pool wait cap (30 ms) | At 20K RPS, queueing causes tail-latency collapse — system becomes uniformly broken instead of partially serving. Controlled rejection beats cascading timeout. |
| 9 | **No idempotency discussion** despite checkout retrying on transient errors | Explicit invariants: pure functions, no `datetime.now()` except voucher expiry, idempotency key on `validate_discount_code`, no side effects in calculate path | Without these, a retry on a flaky network can charge a customer twice or double-decrement a single-use voucher. |
| 10 | Suggested **high-cardinality metric labels** freely (e.g. `voucher_code`, `customer_id`) | Added **cardinality allowlist + forbidden list** | Each unique label value = a separate stored time-series. Voucher codes as a label would explode Datadog cost or OOM the Prometheus scrape target. They go in logs, never on metrics. |
| 11 | **Standard CI/CD gates** (errors, latency) for all changes including discount logic | Three **discount-specific gates**: 50-cart fixture suite with exact expected prices, applied-amount distribution canary (>15% drift = auto-rollback), shadow mode for major logic changes | Pricing bugs are silent — error/latency gates can't catch "discount is now 0 for all bank cards." Needs a different defense. |
| 12 | **No graceful-degradation policy** | Explicit fail-open strategy with named fallbacks, plus a **never-acceptable invariant** (`final_price > original_price` is a legal issue) | The central trade-off on a checkout-blocking service must be named, not avoided. |

### Coverage extensions (after the critical review pass)

- **Second runbook** added (voucher code rejection) — different failure pattern from pool saturation: cache poisoning, read-replica lag, recent rule deploy. Different ranked causes, different mitigations.
- **Concrete day-one load-test plan** with k6 protocol, pass criteria (p99 < 120 ms, error < 0.1%, CPU < 80%, GC pause < 10 ms), and a decision matrix mapping load-test results to capacity-model adjustments.
- **PgBouncer drawn into the v1 diagram** with double-line border and `v1.5` label.

### Commit history
The git log reflects this progression — each commit names what was changed and why. Commit `91e5c55` in particular lists the critical-review refinements explicitly. See `git log` for the full sequence.

---

## Service Reference (from assignment)

The discount service exposes the interface from the assignment brief. No code changes — the design treats this interface as a fixed contract.

Endpoints:
- `calculate_cart_discounts(cart_items, customer, payment_info) → DiscountedPrice`
- `validate_discount_code(code, cart_items, customer) → bool`

Both are synchronous from the caller's perspective and on the checkout critical path.
