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

## Commit Hygiene

Per the assignment, the design was drafted with AI assistance and then critically refined. The git history reflects:

1. **Initial AI-assisted draft** of the six-area design.
2. **Critical review pass** — capacity math reconciled at 300 RPS/pod and max 100 replicas, latency budget added per hop, PgBouncer ambiguity resolved, idempotency / cardinality discipline / queueing & backpressure addressed.
3. **README and rendered version** added for reviewer context.
4. **Coverage extensions** — second runbook (voucher rejection), PgBouncer drawn into v1 diagram with `v1.5` annotation, concrete day-one load-test plan with pass criteria and decision matrix.

The delta between the AI draft and the final design is intentionally visible across the commits — each one names what was changed and why.

---

## Service Reference (from assignment)

The discount service exposes the interface defined in [assignment_SRE.md](assignment_SRE.md#appendix-service-reference). No code changes — the design treats this interface as a fixed contract.

Endpoints:
- `calculate_cart_discounts(cart_items, customer, payment_info) → DiscountedPrice`
- `validate_discount_code(code, cart_items, customer) → bool`

Both are synchronous from the caller's perspective and on the checkout critical path.
