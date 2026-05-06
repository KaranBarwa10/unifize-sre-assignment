# Unifize SRE Assignment

Please don't spend more than 3 hours on this assignment. The goal is to evaluate your ability to think like a production owner — translating a service specification into a credible reliability, observability, and operations plan. **You are NOT expected to implement the solution.** We are looking for clear design thinking, justified trade-offs, and the kind of judgment that shows up at 3 AM on a holiday.

## Submission Expectations

* Generate your initial design draft using Cursor, Claude, ChatGPT, or any AI tool of your choice. Treat this as your first commit.
* Review the AI-generated content critically as you would in a professional design review. AI tends to produce generic, cargo-culted reliability advice — your job is to make it specific, justified, and honest.
* Identify and refine the design in subsequent commits. Use clear commit messages explaining what you changed and why.
* Document all assumptions and trade-offs in a `README.md`.
* Diagrams are strongly encouraged — architecture, request flow, deployment topology, failure modes, whatever helps communicate your thinking. Hand-drawn, Excalidraw, Mermaid, draw.io — the format does not matter.
* Share a public GitHub link.

You will not have time to design all of this in equal depth. Your README should explain what you prioritized, what you skimmed, and what you would do next with another week. **This trade-off articulation is part of the evaluation.**

## Business Scenario

You've inherited a Python-based discount calculation service for a fashion e-commerce platform. The service handles four discount types: brand-specific discounts, bank card offers, category-specific deals, and voucher codes. It has been running in internal beta and is about to be exposed to production traffic.

**Traffic assumptions** (use these unless you choose to challenge them, in which case state your alternative and why):

- Steady-state: ~2,000 RPS
- Peak (sale events, e.g., Black Friday): ~20,000 RPS for 4-6 hour windows
- p99 latency target: under 150 ms
- The service is on the synchronous checkout path — if it is down or slow, customers cannot complete purchases
- Multi-region not required for v1; single region with multi-AZ is acceptable
- Stack: Python service, PostgreSQL for discount rules, Redis for caching, deployed to Kubernetes on a major cloud (AWS/GCP/Azure — your pick)

A reference of the service interface is included at the end of this document for context. You do not need to modify it.

## Technical Deliverables

Produce a **design document** (in the repo's README or a `/design` folder) covering the six areas below. Each area has a suggested time budget — treat these as guidance, not gospel.

### 1. Observability Plan (~30 min)

- Which structured log fields will you emit, and at what level? Show 2-3 example log lines for a typical request.
- Which metrics (RED / USE / custom business metrics) will you instrument? Briefly justify each.
- Which spans / trace attributes matter for this service, and what would a typical trace look like?
- Sketch one Grafana dashboard (described in text or a screenshot of a mock — no need to build it). What does the on-call engineer look at first?

### 2. SLI/SLO and Error Budget Policy (~30 min)

- Define 2-3 SLIs. For each, state precisely how it is measured (which events, which window, success criteria).
- Set SLO targets and justify them against the business context (this service blocks checkout).
- Define an error budget policy: what happens when the budget is 50% consumed? 100%? Who decides, and what changes operationally?

### 3. Deployment & Resilience Design (~30 min)

- Sketch the Kubernetes deployment topology: replicas, resource requests/limits, HPA triggers and thresholds (with justification — not just "70% CPU" because a tutorial said so), PDB, probes.
- Identify the top 3 failure modes for this service and what mitigates each. Be specific — "the database is slow" is not a failure mode; "p99 query latency on the discount_rules table degrades when we add a new bank offer that triggers a sequential scan" is.
- How do dependencies (Postgres, Redis, downstream services) fail, and how does your service degrade gracefully? What is acceptable degraded behavior here, given that this is on the checkout path?

### 4. CI/CD and Change Safety (~25 min)

- Pipeline stages from PR to production. What gates exist where?
- How do you safely roll out a change to discount calculation logic, given that bugs translate directly into revenue loss or overcharging customers?
- What is your rollback strategy and target rollback time?
- Where do security scans, secrets management, and supply chain controls fit in?

### 5. Capacity & Cost (~25 min)

- Given the traffic assumptions above, estimate the steady-state and peak resource footprint (pods, DB size/IOPS, Redis memory). Show the math, even if rough — back-of-envelope is fine, hand-waving is not.
- Identify the first bottleneck the system will hit at peak, and what you would do about it.
- One cost-optimization proposal with an estimated impact.

### 6. One Runbook (~20 min)

Pick one realistic failure scenario and write the runbook an on-call engineer would actually use at 3 AM. Examples:
- "Discount calculation p99 latency has breached SLO for 10 minutes."
- "Customers are reporting that valid voucher codes are being rejected."
- "Database connection pool is saturated."

A good runbook has: symptoms, dashboards/queries to confirm, likely causes ranked by probability, mitigation steps, escalation path, and post-incident actions.

## Evaluation Rubric

We are evaluating four things:

1. **Production thinking.** Does the design hold up under "what happens when X breaks?" Are SLOs and HPA thresholds justified or copy-pasted? Would the runbook actually unblock someone at 3 AM?
2. **Trade-off articulation.** Senior engineers make choices and explain them. Cost vs. reliability, simplicity vs. flexibility, speed of rollout vs. safety — we want to see these tensions named, not avoided.
3. **Specificity over generality.** "Add monitoring" is not a plan. "Emit a `discount_calculation_duration_seconds` histogram with `discount_type` and `outcome` labels, alert when p99 over 5min exceeds 150ms for 2 consecutive windows" is.
4. **Commit hygiene and AI critique.** We want to see your edits to the AI-generated draft and your reasoning in commit messages. The delta between the AI's first draft and your final design is a strong signal.

## What We Are NOT Evaluating

- Implementation completeness — no code is required.
- Diagram polish — Mermaid, Excalidraw, or a phone photo of a whiteboard sketch are all fine.
- Tool-specific expertise — pick the cloud / IaC / observability stack you know best and justify it briefly.
- Exhaustive coverage — depth on the things you prioritized beats shallow coverage of everything.

---

## Appendix: Service Reference

The discount service exposes the following interface. You do not need to modify it; it is provided so you can reason about request shape, dependencies, and failure modes.

```python
from dataclasses import dataclass
from typing import List, Optional, Dict
from decimal import Decimal
from enum import Enum

class BrandTier(Enum):
    PREMIUM = "premium"
    REGULAR = "regular"
    BUDGET = "budget"

@dataclass
class Product:
    id: str
    brand: str
    brand_tier: BrandTier
    category: str
    base_price: Decimal
    current_price: Decimal

@dataclass
class CartItem:
    product: Product
    quantity: int
    size: str

@dataclass
class PaymentInfo:
    method: str
    bank_name: Optional[str]
    card_type: Optional[str]

@dataclass
class DiscountedPrice:
    original_price: Decimal
    final_price: Decimal
    applied_discounts: Dict[str, Decimal]
    message: str


class DiscountService:
    async def calculate_cart_discounts(
        self,
        cart_items: List[CartItem],
        customer: "CustomerProfile",
        payment_info: Optional[PaymentInfo] = None
    ) -> DiscountedPrice:
        ...

    async def validate_discount_code(
        self,
        code: str,
        cart_items: List[CartItem],
        customer: "CustomerProfile"
    ) -> bool:
        ...
```

The service reads discount rules from PostgreSQL, caches hot rules and computed results in Redis, and is called synchronously by the checkout service on every cart-pricing request.