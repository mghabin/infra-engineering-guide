# Chapter 10 — FinOps

Opinionated, cloud-agnostic defaults for running cloud workloads at a defensible
unit cost. FinOps is not "the cloud team will fix our bill". It is a
cross-functional operating model — engineering, finance, and product
co-owning cloud spend the same way SRE co-owns reliability. This chapter is
about the practice, the controls, and the traps. AWS/Azure/GCP examples appear
where the mechanics differ; the principles are the same.

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint.

---

## 1. What FinOps actually is

- **Do** adopt the FinOps Foundation framework explicitly: **Inform → Optimize
  → Operate**, executed continuously, not as a one-off cost-cutting project
  ([finops.org/framework](https://www.finops.org/framework); Storment & Fuller,
  *Cloud FinOps* 2nd ed., O'Reilly 2023, ch. 1–3).
- **Inform** — visibility, allocation, benchmarking. You cannot optimize what
  you cannot attribute.
- **Optimize** — rightsizing, commitment-based discounts, architectural change.
- **Operate** — policies, automation, KPIs, anomaly response. The steady state.
- **Don't** treat FinOps as a finance reporting function. The decisions
  (turn off the dev cluster at 18:00, move logs to cold storage, re-architect
  the cross-AZ chatty service) are engineering decisions; finance owns the
  forecast and the commitment ladder.
- **Do** anchor on the six **FinOps Principles**: teams collaborate; decisions
  are driven by business value; everyone owns their cloud usage; FinOps
  reports are accessible and timely; a centralized team drives FinOps;
  take advantage of the variable cost model
  ([finops.org/framework/principles](https://www.finops.org/framework/principles/)).

### 1.1 Operating model by scale

- **Do** size the FinOps function to spend and complexity, not to fashion.
  Storment & Fuller's "crawl → walk → run" maturity model is explicit that
  early-stage practices are virtual, not headcount.
- Small org / single cloud / < ~$1M/yr: **virtual FinOps function** —
  platform engineer + finance partner + product/eng leader meeting monthly.
  No dedicated headcount.
- Mid org / multi-account / ~$1M–$20M/yr: part-time FinOps lead inside
  platform plus the virtual function. Cloud-native tooling plus one
  unified view.
- Large / multi-cloud / > ~$20M/yr: dedicated FinOps team for enablement
  and standards, not approving every purchase.
- **Don't** stand up a dedicated FinOps team before you have tagging
  discipline and a monthly cadence; year one will be data-quality fights.

---

## 2. The cost model: per-cloud commitment mechanics

Cloud compute is sold on (at least) four pricing dimensions: on-demand,
commitment-based discounts, spot/preemptible, and free-tier/credits. The
commitment products are *not* equivalent across clouds — confusing them is
the most expensive FinOps mistake.

### 2.1 Commitment mechanics matrix

| Product | Commit unit | Term | Flexibility | Notes |
|---|---|---|---|---|
| **AWS Standard RI** | Instance family + size + region (size-flex within family for Linux shared-tenancy) | 1y / 3y | Lowest; can sell on Marketplace | Highest discount among RIs ([AWS RI docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html)) |
| **AWS Convertible RI** | Family-flexible (exchange permitted) | 1y / 3y | Exchange to different family/OS | Lower discount than Standard |
| **AWS Compute Savings Plan** | $/hour committed spend | 1y / 3y | Any EC2/Fargate/Lambda, any region/family | Broadest flexibility ([AWS SP docs](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)) |
| **AWS EC2 Instance Savings Plan** | $/hour, locked to family + region | 1y / 3y | Size/OS/tenancy flex within family | Higher discount than Compute SP |
| **Azure Reservation** | VM size or service SKU + region (instance-size flexibility within group) | 1y / 3y | Exchange/refund subject to policy | Per-SKU; covers VM, SQL, Cosmos, etc. ([Azure Reservations](https://learn.microsoft.com/azure/cost-management-billing/reservations/save-compute-costs-reservations)) |
| **Azure Savings Plan for Compute** | $/hour committed spend | 1y / 3y | Compute services across regions/families | Spend-based, similar to AWS Compute SP ([Azure SP](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/)) |
| **GCP Resource-based CUD** | vCPU + memory in a region (or specific SKU) | 1y / 3y | Locked to machine family + region | Higher discount; classic CUD ([GCP CUDs](https://cloud.google.com/docs/cuds)) |
| **GCP Flexible/Spend-based CUD** | $/hour spend on Compute Engine flexible families | 1y / 3y | Across families/regions for covered services | Similar role to AWS Compute SP |

- **Do** classify each workload as **base-load** (predictable 24/7),
  **variable** (predictable shape, varying volume), or **bursty/unknown**
  before mapping to a commitment product. Base-load → highest-discount
  product (Standard RI / EC2 Instance SP / resource-based CUD). Variable →
  spend-based (Compute SP / Azure SP / flexible CUD). Bursty/unknown →
  on-demand or spot.
- **Prefer** spend-based commitments when family or region may change inside
  the term; prefer resource-based when the family is stable and the extra
  discount matters.

### 2.2 Break-even and unused-commitment math

- Let `D` = effective hourly discount fraction vs on-demand, `U` = expected
  utilization of the commitment (0–1), `C` = on-demand cost per hour for the
  covered capacity, `F` = annualized financing/cash cost (upfront × WACC).
- Net hourly saving ≈ `C × (D × U − (1 − U) × wasted_fraction) − F/H` where
  `H` is hours in the term.
- **Break-even rule**: only commit when expected `D × U` exceeds your
  required hurdle and `(1 − U) × C` (expected unused commitment) plus
  financing cost is comfortably less than the gross discount.
- Practical heuristic, calibrate per workload: 1-year no-upfront breaks even
  near continuous use of ~60–70% of the term; 3-year all-upfront requires
  longer runway and higher confidence because cash is out the door.
- **Do** model upfront purchases against your cost of capital and
  local-currency exposure (USD-denominated commitments hit non-USD P&Ls
  through FX). Coordinate with finance on amortization treatment — most
  orgs amortize upfront commitments over the term per ASC 606 / IFRS 15
  guidance from the controller.
- **Don't** buy a 3-year commitment for a workload whose roadmap does not
  exist 14 months out.

---

## 3. Visibility before optimization

- **Do** make tagging a build-time concern, not a cleanup-time one.
  Required tags on every cloud resource: `cost-center`, `service`, `env`,
  `owner`, `data-classification`. Enforce in IaC with policy-as-code
  (Azure Policy, AWS SCP + Config rules, GCP Org Policy + custom
  constraints). Untagged resources are a finding, not a warning. Storment &
  Fuller, ch. 7 ("Allocation"): "if it is not tagged, it cannot be charged
  back".
- **Prefer** the **FOCUS** specification (FinOps Open Cost & Usage
  Specification, [focus.finops.org](https://focus.finops.org/)) for any
  cross-cloud cost dataset, **where your provider and tooling natively
  support it**. Raw provider exports (AWS CUR 2.0, Azure Cost Management
  exports, GCP Billing BigQuery export) remain a valid fallback when FOCUS
  fields are missing or stale.
- FOCUS provider support, as of FOCUS 1.1 (2024–2025): AWS publishes a
  FOCUS-aligned data export from the Data Exports service; Azure Cost
  Management offers a FOCUS export in preview/GA depending on tenant; GCP
  Billing supports FOCUS via BigQuery export; OCI publishes a FOCUS report.
  Coverage and field completeness vary — check the FinOps Foundation
  conformance page before assuming parity.
- **Prefer** the cloud-native cost UI (Cost Explorer, Azure Cost Management,
  GCP Billing reports) for daily/weekly use, and a unified view for
  monthly exec/cross-cloud views. Cloud-native tools are free, fresh, and
  cover ~80% of questions.
- **Don't** ship cost dashboards that nobody opens. A dashboard with no
  reviewing cadence and no owner is decoration. Either put it on a weekly
  service-team review with a named owner, or delete it.

### 3.1 Tooling comparison

| Tool | Scope | Data freshness | K8s depth | Shared-cost allocation | Multi-cloud | Overhead |
|---|---|---|---|---|---|---|
| Cloud-native (Cost Explorer / Azure CM / GCP Billing) | Single cloud | Hours | None | Tag/account-based | No | None — included |
| OpenCost (CNCF) | K8s clusters | ~Minutes (in-cluster) | High (namespace/pod/workload) | Idle/shared cost models | Per cluster | OSS install + Prom |
| Kubecost | K8s + cloud bill | Minutes–hours | High; commercial extras | Built-in models, RI/SP amortization | Limited | SaaS or self-host license |
| CloudHealth (VMware) | Multi-cloud | Daily | Medium (via OpenCost-style integrations) | Mature shared-cost & chargeback | Yes | Commercial; enterprise effort |
| Vantage | Multi-cloud + SaaS bills | Hours | Medium (Kubecost integration) | Cost reports & custom allocation | Yes | SaaS, low ops |
| CloudZero | Multi-cloud, unit-economics focus | Hours | Medium | Per-customer/feature allocation | Yes | SaaS, integration effort |

- **Do** pick by the question you must answer monthly (K8s namespace
  attribution → OpenCost/Kubecost; per-customer unit cost → CloudZero or a
  warehouse model; enterprise governance → CloudHealth). Do not stack three
  overlapping tools.

---

## 4. Budgets, burn-rate, and the kill-switch

- **Do** create a budget at every account / subscription / project, with
  multiple alert *types*, not just monthly percentages:
- Monthly thresholds (e.g., 50% / 80% / 100%) for steady-state awareness.
- **Forecast** alerts (cloud-native: AWS Budgets forecast, Azure Cost
  Management forecast, GCP Billing forecasted-spend) catch trajectory
  changes before the month closes.
- **Burn-rate** alerts mirror SRE error-budget practice: alert when the
  rolling 1h/6h/24h spend rate, projected over the period, would exceed
  the budget by N×. This catches overnight runaways that monthly
  percentages miss.
- AWS Budgets supports SNS → Lambda for automated action; Azure Budgets
  fires Action Groups; GCP Billing Budgets fire Pub/Sub
  ([AWS Well-Architected Cost Optimization pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html);
  [Azure Cost Management best practices](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices)).
- **Do** scope **kill-switches** narrowly: only on **pre-approved non-prod**
  resources tagged for automatic teardown, plus a default **TTL** on
  ephemeral environments (e.g., PR previews destroy after 24h, sandbox
  accounts nuke daily). The kill-switch is automation against a documented
  resource list, not a global "stop the cloud" button.
- **Don't** put a kill-switch on production. Production gets paged and
  human-decided. Production protection is anomaly detection plus on-call.
- **Don't** rely on a single 100%-of-budget action. By the time the month
  closes, the money is already spent; forecast and burn-rate alerts must
  fire days earlier.

---

## 5. Rightsizing

- **Do** drive rightsizing from observed p95/p99 utilization plus a
  documented headroom (commonly **p99 + 30%** for steady-state services,
  more for spiky ones). Cloud rightsizing recommenders (AWS Compute
  Optimizer, Azure Advisor, GCP Recommender) all use percentile-based
  heuristics — read the recommendation, do not just click apply.
- **Prefer** Kubernetes-native rightsizing: **VPA** in *recommendation* mode
  (not auto-apply) feeding requests, plus a node autoscaler with
  consolidation. Options include Cluster Autoscaler with overprovisioning
  buffers, **Karpenter** (CNCF) on AWS/AKS, **GKE node auto-provisioning**
  or **GKE Autopilot**, and AKS node-autoprovisioning preview. Each has
  different bin-packing and consolidation behavior — pick by platform, not
  by hype.
- For non-Kubernetes workloads, scale-to-zero via FaaS, Cloud Run, ACA, or
  App Runner often beats rightsizing a long-running VM.
- **Don't** rightsize from synthetic benchmarks; production has correlated
  spikes, GC pauses, and noisy neighbors no benchmark reproduces.
- **Don't** chase a 100%-utilized service. CPU at 100% means you are
  queueing; your latency SLO is already breached.
- **Note** — rightsizing changes *requests*, not limits. CPU-limit policy
  (omit on Burstable, latency-sensitive workloads to avoid CFS throttling)
  is owned by ch04 §5; do not tighten CPU limits in the name of
  utilization.

---

## 6. Scale-to-zero

- **Do** prefer scale-to-zero compute for spiky or low-volume workloads:
  Lambda / Functions for sub-second event handlers; Cloud Run / Azure
  Container Apps / App Runner for HTTP services; **KEDA** for Kubernetes
  workloads that should drop to zero pods between events
  ([KEDA docs](https://keda.sh/), CNCF graduated 2023).
- **Do** measure cold-start cost honestly. JVM/.NET cold starts are
  hundreds of ms to seconds; Node/Python typically tens to low hundreds of
  ms. Provisioned Concurrency / Always-Ready trade scale-to-zero economics
  for latency. Document and verify with p99 in production.
- **Prefer** Cloud Run / ACA over raw FaaS for HTTP services with
  non-trivial startup; you keep scale-to-zero economics with container
  portability and longer execution limits. Validate with your own
  benchmarks before standardizing.

---

## 7. Spot / preemptible

- **Do** use spot/preemptible/Azure Spot for any workload that is
  **stateless, retry-able, idempotent, and tolerant of interruption at any
  time**: CI runners, batch ETL, ML training, async workers, dev clusters.
  60–90% discounts are documented on each provider's pricing page.
- **Do** treat spot interruption notice as **best-effort and
  provider-specific**. Notice timing is not a guarantee:

| Provider | Product | Interruption notice | Notes |
|---|---|---|---|
| AWS | EC2 Spot | Best-effort **2 minutes** via instance metadata + EventBridge | Can be **shorter or absent** under capacity stress; Spot Blocks deprecated ([AWS Spot interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)) |
| Azure | Spot VMs / Spot Scale Sets | Best-effort **up to 30 seconds** via Scheduled Events | Eviction may also be cost-based; configurable max price ([Azure Spot VMs](https://learn.microsoft.com/azure/virtual-machines/spot-vms)) |
| GCP | Spot VMs / Preemptible VMs | **No guaranteed dedicated notice** for Spot; preemption signal via `ACPI G2 Soft Off` / shutdown script (~30 seconds) — features and timing vary by instance type | Plan for sudden termination ([GCP Spot VMs](https://cloud.google.com/compute/docs/instances/spot)) |

- **Do** apply the **dual-pool pattern** for production Kubernetes: an
  on-demand floor (e.g., 30% of capacity, system pods, stateful sets) plus
  a spot bulk pool for stateless workloads. All major node autoscalers
  (Cluster Autoscaler, Karpenter, GKE NAP, AKS NAP) support weighted pools
  or instance diversification.
- **Don't** put databases, primary brokers, or anything with local state on
  spot. The drain window — even when notice is given — is not enough for a
  multi-hour replica rebuild. The Kubernetes-side guardrails (PDBs,
  StatefulSet drain ordering) are owned by ch04 §5/§13; the resilience
  expectations (graceful degradation under sudden node loss, error budget
  treatment of evictions) are owned by ch09 §9.
- **Don't** use a single instance type in your spot pool. Diversify across
  3–5 families/sizes/AZs; eviction is correlated within a pool. Documented
  in [AWS Spot Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html).

---

## 8. Egress and network: the surprise bill

Egress and network-processing fees are the line items that break budgets,
and the SKUs differ materially by provider. Always price against the
current provider pricing page; rates and tiers change.

### 8.1 Per-cloud network model

| Charge | AWS | Azure | GCP |
|---|---|---|---|
| Internet egress | Tiered $/GB after monthly free tier; varies by source region | Tiered $/GB by zone & volume | Tiered $/GB by destination continent (Standard tier); Premium vs Standard network tiers |
| Intra-region cross-AZ | Charged each direction per GB | Charged for VNet peering & cross-zone in some SKUs | Cross-zone within region charged per GB |
| Inter-region | Charged per GB; varies by region pair | Charged per GB; varies by geography | Charged per GB; varies by region pair & network tier |
| CDN origin pull | Discounted/free from S3 → CloudFront in same partition | Discounted from Storage → Front Door / Azure CDN | Discounted from GCS → Cloud CDN |
| NAT gateway | Per-GB **processing** fee on top of egress | NAT Gateway data-processing per GB | Cloud NAT per-GB processing |
| Load balancer | LCU / data-processed fees on ALB/NLB | Standard LB / App Gateway data-processed | Forwarding-rule + per-GB processing |
| Common surprise SKUs | VPC endpoint hourly + data, PrivateLink, cross-region replication, CloudWatch Logs ingest | Private Endpoint per-hour + data, ExpressRoute egress, Log Analytics ingestion | VPC peering egress, Interconnect egress, Logging ingestion |

Sources: [AWS bandwidth pricing](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer),
[Azure bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/),
[GCP network pricing](https://cloud.google.com/vpc/network-pricing).

- **Do** put a CDN in front of object/blob storage serving the public
  internet; origin → CDN egress is discounted or free within the same
  provider, and CDN egress is dramatically cheaper at volume.
- **Do** keep chatty services in the same AZ where possible (topology-
  aware routing in Kubernetes; placement groups on EC2; proximity placement
  groups on Azure). Cross-AZ chatter is a hidden multiplier; measure it.
- **Do** audit NAT and load-balancer **processing** fees independently of
  egress. A high-throughput service behind a NAT gateway can spend more on
  per-GB NAT processing than on the destination egress itself.
- **Don't** pull large datasets out of object storage to a different
  region or cloud "to process them where the team works". Move compute,
  not data.

---

## 9. Storage lifecycle

- **Do** define lifecycle policies as IaC for every bucket / container:
  hot → warm (IA / Cool) → cold (Glacier IR / Archive / Coldline) → delete.
  Typical defaults: warm at 30 days, cold at 90, archive at 365, delete at
  policy-defined retention. AWS S3 Intelligent-Tiering automates the
  hot→warm boundary if access patterns are unknown
  ([AWS Well-Architected Cost Optimization pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html),
  COST04).
- **Do** review **request and retrieval costs**, not just storage GB cost.
  Many cold tiers have per-retrieval and per-request fees that dwarf
  storage savings if access patterns were misjudged. Check current
  provider rate cards before lifecycling.
- **Don't** lifecycle data you may need for compliance retrieval out to a
  tier whose retrieval SLA you cannot meet.

---

## 10. The cost of governance

- **Don't** spend a $500 platform-team week chasing a $5/day spike. The
  cost of the FinOps process must be less than the cost it controls.
  Storment & Fuller call this out repeatedly: a mature practice prioritizes
  by **dollar impact × confidence × ease**, not by alert noise.
- **Prefer** a written investigation threshold (e.g., investigate anomalies
  > $100/day or > 20% of service spend; otherwise log and move on). This
  is the single most common reason internal FinOps programs stall.
- This rule is consistent with the operating-model scaling in §1.1 — small
  orgs cannot afford the same governance overhead as large ones.

---

## 11. Showback, chargeback, and allocation

- **Showback** = tell the team what they spend. **Chargeback** = the team's
  P&L is debited. The hard problem is **allocation**: how shared, indirect,
  and amortized costs map to teams.
- **Prefer** showback first, chargeback once the data is trusted (typically
  6–12 months of clean tagging). Chargeback before tagging discipline
  causes finance disputes that poison the program (Storment & Fuller,
  ch. 10).

### 11.1 Allocation mechanics

- **Direct attribution** — costs with a clean owner tag (per-service VMs,
  databases, buckets) flow straight to the owning team.
- **Proportional usage allocation** — shared infrastructure (multi-tenant
  K8s clusters, shared databases, observability, log ingestion) is split
  by a measured driver: CPU·hour, memory·GB·hour, ingested log GB,
  request count. OpenCost/Kubecost implement this for Kubernetes; for
  shared SaaS, use vendor seat or usage exports.
- **Fixed shared-cost pools** — costs that resist measurement (control
  plane, support contracts, SSO, baseline networking) go into a pool
  allocated by a stable key (headcount, revenue share, % of direct
  spend). Document the key; revisit annually.
- **Unattributed spend** — anything still unallocated after the above
  belongs to a named **platform pool**, not silently smeared. If the
  pool grows, that is a tagging-debt signal for §3.
- **Amortization** — upfront commitments and prepaid credits must be
  amortized across the term in showback so monthly numbers reflect
  consumption, not cash. Cloud-native tools and OpenCost/Kubecost expose
  amortized views; pick one and use it consistently.
- **Credits & discounts** — decide policy: pass through to the consuming
  team, hold centrally as a platform discount, or amortize. Document.
- **Do** make per-service unit economics ($/order, $/active user,
  $/inference) the headline metric in the engineering review, not the
  absolute monthly bill. This is the **Quantify Business Value**
  capability in the FinOps Foundation framework.

---

## 12. Anomaly detection

- **Do** turn on cloud-native anomaly detection (AWS Cost Anomaly
  Detection, Azure Cost Management anomaly alerts, GCP budget anomaly).
  Free, ML-based, tuned for the platform.
- **Do** add **OpenCost** (CNCF) to any multi-tenant Kubernetes cluster;
  cluster-level cloud bills cannot tell you which namespace burned the
  spike ([opencost.io](https://www.opencost.io/)). Kubecost is the
  commercial superset.
- **Prefer** "alert at significant deviation" (e.g., +30% week-over-week
  or > $X absolute) over "alert on first dollar of new service".

---

## 13. Commitment portfolio management

- **Do** run a **laddered commitment** strategy: stagger 1-year and 3-year
  commitments so a portion expires every quarter. You never reach 100%
  coverage and you never face a cliff.
- **Do** target a **base-load coverage** band, not "% of total". Compute
  base-load as the rolling 30-day p10 (or trough) of on-demand-equivalent
  spend on the family/region you intend to cover. Cover **70–90% of
  base-load** with the highest-discount product the workload permits;
  cover the variable layer with spend-based commitments; leave the top
  bursty layer on-demand or spot.
- Coverage band math:
- `base_load = p10(hourly_on_demand_eq_spend, 30d)`
- `target_commit ≈ 0.7..0.9 × base_load`
- `expected_unused = max(0, target_commit − realized_usage)` — track
  monthly; chronic > ~5% means the band is too high.
- **Do** include cash, FX, and accounting in the decision: upfront
  commitments are a financing decision (compare vs. WACC), and
  USD-denominated commitments expose non-USD entities to FX. Confirm
  amortization treatment with the controller (commitments are typically
  amortized straight-line over the term).
- **Don't** delegate commitment purchases to "the platform team will handle
  RIs". Commitments are a finance + engineering decision; engineering owns
  the 12–36 month forecast.

---

## 14. Sustainability as a cost-adjacent metric

- **Do** report carbon alongside dollars **with the measurement basis
  attached**. Provider tools (AWS Customer Carbon Footprint, Azure
  Emissions Impact Dashboard, GCP Carbon Footprint) expose Scope 2
  estimates using different methodologies (location- vs market-based,
  monthly granularity, varying scope coverage).
- **Do** state, for each carbon metric: scope (1/2/3 covered), estimation
  method (location- vs market-based), region/temporal granularity, and the
  **internal carbon price** (if any) used to compare alternatives. Without
  a price, carbon and dollars cannot be traded off mechanically.
- For multi-cloud or finer granularity, **Cloud Carbon Footprint**
  ([cloudcarbonfootprint.org](https://www.cloudcarbonfootprint.org/))
  estimates from billing line items — an estimate, not a measurement.
- **Prefer** lower-carbon regions when latency and data-residency permit.
  Cheaper and greener often, but not always, align — verify per workload.

---

## 15. The "move to a cheaper cloud" trap

- **Don't** lift-and-shift to a "cheaper" cloud or back to on-prem without
  a TCO model that includes engineering time, retraining, re-tooling, and
  loss of managed-service leverage. The a16z "Cost of Cloud" piece (2021)
  is widely misquoted as advocating repatriation; it argues for it only at
  scale, with a dedicated infra team, after exhausting cloud-native
  optimization.

---

## 16. Anti-patterns

- **Cost dashboards nobody looks at.** No owner, no cadence, no decision —
  delete.
- **"Let the platform team buy more RIs".** Commitments without
  engineering forecast are a finance bet, not FinOps.
- **Weekly cost meetings with no kill-switches or TTLs.** Discussion
  without authority to act is theatre.
- **Tagging policies without enforcement.** Optional tags = no tags.
- **Single-AZ, single-family spot pool.** One eviction event drains
  everything.
- **Cross-region object-storage reads as a default.** The egress line will
  catch up with you in month 2.
- **Lifecycle policies set up once, never reviewed.** Access patterns
  drift; archived hot data costs more, not less.
- **Per-team cost guilt-tripping with no allocation.** Without showback the
  conversation is unfounded.
- **Carbon dashboards with no measurement basis.** Invites greenwashing.

---

## Further reading

- Storment, J.R. & Fuller, Mike — *Cloud FinOps*, 2nd ed., O'Reilly, 2023.
- FinOps Foundation Framework & Principles — <https://www.finops.org/framework/>
- FOCUS specification — <https://focus.finops.org/>
- AWS Well-Architected Cost Optimization Pillar — <https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html>
- AWS Savings Plans — <https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html>
- AWS Spot interruptions — <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html>
- AWS bandwidth pricing — <https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer>
- Azure Reservations — <https://learn.microsoft.com/azure/cost-management-billing/reservations/save-compute-costs-reservations>
- Azure Savings Plan for Compute — <https://learn.microsoft.com/azure/cost-management-billing/savings-plan/>
- Azure Spot VMs — <https://learn.microsoft.com/azure/virtual-machines/spot-vms>
- Azure bandwidth pricing — <https://azure.microsoft.com/pricing/details/bandwidth/>
- GCP Committed Use Discounts — <https://cloud.google.com/docs/cuds>
- GCP Spot VMs — <https://cloud.google.com/compute/docs/instances/spot>
- GCP network pricing — <https://cloud.google.com/vpc/network-pricing>
- OpenCost — <https://www.opencost.io/>
- Cloud Carbon Footprint — <https://www.cloudcarbonfootprint.org/>
