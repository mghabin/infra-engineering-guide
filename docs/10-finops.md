# Chapter 10 — FinOps

Opinionated, cloud-agnostic defaults for running cloud workloads at a
defensible unit cost. FinOps is a cross-functional operating model
where engineering, finance, and product co-own cloud spend the same
way SRE co-owns reliability. AWS/Azure/GCP examples appear where
mechanics differ.

> Conventions: **Do** = required default. **Don't** = reject in review
> unless a written exception exists. **Prefer** = strong default;
> deviate only with measurement or a documented constraint.

---

## 1. What FinOps actually is

- **Do** adopt the FinOps Foundation framework: **Inform → Optimize →
  Operate** (visibility/allocation → rightsizing/commitments/
  architecture → policy/automation/KPIs), run continuously, not as a
  one-off cost-cutting project
  ([finops.org/framework](https://www.finops.org/framework); Storment
  & Fuller, *Cloud FinOps* 2nd ed., O'Reilly 2023). **Don't** treat
  FinOps as a finance reporting function — the decisions (turn off the
  dev cluster at 18:00, move logs to cold storage, re-architect the
  cross-AZ chatty service) are engineering decisions; finance owns the
  forecast and the commitment ladder.
- **Do** anchor on the six **FinOps Principles**
  ([principles](https://www.finops.org/framework/principles/)). The 2025
  Foundation mission update — "managing the value of Cloud" → "managing
  the value of **Technology**" — signals scope: practitioners now manage
  SaaS (~90%), licensing (~64%), private cloud (~57%), data center
  (~48%), and AI (~98%, up from 31% two years prior); AI cost is the #1
  forward-looking priority and skillset gap
  ([State of FinOps 2025](https://www.finops.org/insights/state-of-finops-2025/)).

### 1.1 Operating model by scale

- **Do** size the function to spend and complexity (Storment & Fuller
  "crawl → walk → run"; early-stage practices are virtual, not
  headcount). < ~$1M/yr, single cloud: **virtual function** (platform
  engineer + finance partner + product/eng leader, monthly).
  ~$1M–$20M/yr, multi-account: part-time FinOps lead inside platform
  plus the virtual function. > ~$20M/yr, multi-cloud: dedicated FinOps
  team for enablement and standards (not approving every purchase),
  with explicit **AI/GPU** and **SaaS/licensing** workstreams.
  **Don't** stand up a dedicated team before tagging discipline and a
  monthly cadence exist; year one will be data-quality fights.

---

## 2. The cost model: per-cloud commitment mechanics

Cloud compute is sold on four pricing dimensions: on-demand,
commitments, spot/preemptible, and free-tier/credits. Commitment
products are *not* equivalent across clouds — confusing them is the
most expensive FinOps mistake.

### 2.1 Commitment mechanics matrix

| Product | Commit unit | Term | Flexibility | Documented max discount | Notes |
|---|---|---|---|---|---|
| **AWS Standard RI** | Instance family + size + region (size-flex within family for Linux shared-tenancy) | 1y / 3y | Lowest; can sell on Marketplace | Up to ~72% (3y AURI, family-locked) | Highest discount among RIs ([AWS RI docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html)) |
| **AWS Convertible RI** | Family-flexible (exchange permitted) | 1y / 3y | Exchange to different family/OS | Lower than Standard | Exchange — not refund — path |
| **AWS Compute Savings Plan** | $/hour committed spend | 1y / 3y | Any EC2/Fargate/Lambda, any region/family | **Up to 66%** | Broadest flexibility ([AWS SP pricing](https://aws.amazon.com/savingsplans/compute-pricing/)) |
| **AWS EC2 Instance Savings Plan** | $/hour, locked to family + region | 1y / 3y | Size/OS/tenancy flex within family | **Up to 72%** | Higher discount than Compute SP |
| **Azure Reservation** | VM size or service SKU + region (instance-size flexibility within group) | 1y / 3y | Exchange/refund subject to policy | Vendor-quoted up to ~72% (VM, 3y) | Per-SKU; covers VM, SQL, Cosmos, etc. ([Azure Reservations](https://learn.microsoft.com/azure/cost-management-billing/reservations/save-compute-costs-reservations)) |
| **Azure Savings Plan for Compute** | $/hour committed spend | 1y / 3y | VMs, App Service, Functions Premium, Container Instances, Container Apps, Dedicated Host, Spring Apps Enterprise — **not** software, networking, or storage | Vendor-quoted up to ~65% (3y) | **Non-cancellable, non-refundable.** USD for MCA/MPA (FX exposure for non-USD entities), local currency for EA ([Azure SP overview](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/savings-plan-compute-overview)) |
| **GCP Resource-based CUD** | vCPU + memory in a region (or specific SKU) | 1y / 3y | Locked to machine family + region | Vendor-quoted up to ~70% (3y) | Higher discount; classic CUD ([GCP CUDs](https://cloud.google.com/docs/cuds)) |
| **GCP Flexible/Spend-based CUD** | $/hour spend on Compute Engine flexible families | 1y / 3y | Across families/regions for covered services | Lower than resource-based | Similar role to AWS Compute SP |

- **Do** classify each workload as **base-load** (24/7), **variable**
  (predictable shape, varying volume), or **bursty/unknown** before
  mapping. Base-load → highest-discount product (Standard RI / EC2
  Instance SP / resource-based CUD). Variable → spend-based (Compute SP
  / Azure SP / flexible CUD). Bursty/unknown → on-demand or spot.
  **Prefer** spend-based when family/region may change inside the term;
  resource-based when the family is stable and the extra discount
  matters.
- **Note** — AWS moved EC2 RHEL to per-vCPU-hour pricing on **1 Jul
  2024**; recompute break-even on RHEL-heavy fleets before renewing.

### 2.2 Break-even and unused-commitment math

- Let `D` = effective hourly discount fraction vs on-demand, `U` =
  expected utilization (0–1), `C` = on-demand cost/hr for covered
  capacity, `F` = annualized financing cost (upfront × WACC), `H` =
  hours in the term. Net hourly saving ≈
  `C × (D × U − (1 − U) × wasted_fraction) − F/H`. Only commit when
  `D × U` exceeds your hurdle and `(1 − U) × C` plus financing is
  comfortably less than the gross discount.
- **Do** model upfront purchases against cost of capital and
  local-currency exposure (USD commitments hit non-USD P&Ls through
  FX); confirm amortization with the controller.
- **Do** weight **organizational volatility** as a first-class input —
  pending M&A, platform migrations, and headcount swings invalidate
  forecasts. Asymmetry matters: Azure SPs are non-cancellable and
  non-refundable, AWS Convertible RIs allow exchange. Lean shorter
  and on exchangeable products when volatility is high. **Don't** buy
  a 3-year commitment for a workload whose roadmap does not exist 14
  months out.

---

## 3. Visibility before optimization

- **Do** make tagging a build-time concern, not cleanup-time. Required
  tags on every resource: `cost-center`, `service`, `env`, `owner`,
  `data-classification`. Enforce in IaC with policy-as-code (Azure
  Policy, AWS SCP + Config rules, GCP Org Policy). Untagged resources
  are a finding, not a warning.
- **Prefer** the **FOCUS** specification
  ([focus.finops.org](https://focus.finops.org/)) for any cross-cloud
  cost dataset where your provider and tooling support it. Raw provider
  exports (AWS CUR 2.0, Azure CM exports, GCP Billing BigQuery export)
  remain a valid fallback when FOCUS fields are missing or stale.
  **FOCUS 1.2** (29 May 2025) materially changes the data plane:
  unified **Cloud+ schema** for SaaS/PaaS alongside core cloud spend;
  **virtual-currency** lifecycle columns for credits/tokens/DBUs
  (Snowflake, Databricks); separate `PricingCurrency`/`BillingCurrency`
  for auditable multi-currency normalization; `InvoiceID` per row;
  `BillingAccountType`/`SubAccountType` for shared-cost allocation.
  Generators now include AWS, Azure, GCP, OCI, Alibaba, Databricks, and
  Grafana ([FOCUS 1.2](https://www.finops.org/insights/focus-1-2/)).
- **Prefer** the cloud-native cost UI for daily/weekly use; a unified
  FOCUS-backed view for monthly exec/cross-cloud and SaaS+AI rollups.
  **Don't** ship cost dashboards nobody opens — put them on a weekly
  service-team review with a named owner, or delete.

### 3.1 Tooling comparison

| Tool | Scope | Data freshness | K8s depth | Shared-cost allocation | Multi-cloud | Overhead |
|---|---|---|---|---|---|---|
| Cloud-native (Cost Explorer / Azure CM / GCP Billing) | Single cloud | Hours | None | Tag/account-based | No | None — included |
| **OpenCost** (CNCF **Incubating**, since Oct 2024) | K8s clusters + plugins (Datadog, OpenAI, MongoDB Atlas) | ~Minutes (in-cluster) | High (namespace/pod/workload) | Idle/shared cost models | Per cluster + plugins | OSS install + Prom |
| Kubecost | K8s + cloud bill | Minutes–hours | High; commercial extras | Built-in models, RI/SP amortization | Limited | SaaS or self-host license |
| CloudHealth (VMware) | Multi-cloud | Daily | Medium (via OpenCost-style integrations) | Mature shared-cost & chargeback | Yes | Commercial; enterprise effort |
| CloudZero / Vantage | Multi-cloud + SaaS, unit-economics focus | Hours | Medium (Kubecost integration) | Per-customer/feature allocation | Yes | SaaS, integration effort |

- **Do** pick by the question you must answer monthly (K8s namespace
  attribution → OpenCost/Kubecost; per-customer unit cost → CloudZero or a
  warehouse model; enterprise governance → CloudHealth). The OpenCost
  plugin framework is now the obvious instrument for **AI/SaaS cost**
  allocation alongside K8s. Do not stack three overlapping tools.

---

## 4. Budgets, burn-rate, and the kill-switch

- **Do** create a budget at every account/subscription/project with
  multiple alert *types*: monthly thresholds (50/80/100%); **forecast**
  alerts (AWS Budgets / Azure CM / GCP forecasted-spend) catching
  trajectory before month close; **burn-rate** alerts mirroring SRE
  error-budget practice — alert when the rolling 1h/6h/24h projected
  spend would exceed budget by N×, catching overnight runaways monthly
  percentages miss. Wire to AWS Budgets → SNS → Lambda; Azure Budgets
  → Action Groups; GCP Billing Budgets → Pub/Sub
  ([AWS Cost-Opt pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html);
  [Azure CM best practices](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices)).
- **Do** scope **kill-switches** narrowly: only on **pre-approved
  non-prod** resources tagged for automatic teardown, plus a default
  **TTL** on ephemeral environments (PR previews destroy after 24h,
  sandbox accounts nuke daily). **Don't** put a kill-switch on
  production — page and let humans decide via anomaly detection plus
  on-call. **Don't** rely on a single 100%-of-budget action; by month
  close the money is spent.

---

## 5. Rightsizing

- **Do** drive rightsizing from observed p95/p99 utilization plus
  documented headroom (commonly **p99 + 30%** for steady-state
  services). Cloud recommenders (AWS Compute Optimizer, Azure Advisor,
  GCP Recommender) all use percentile heuristics — read the
  recommendation, do not just click apply.
- **Prefer** Kubernetes-native rightsizing: **VPA** in *recommendation*
  mode (not auto-apply) feeding requests, plus a node autoscaler with
  consolidation. Compute-scaler choice (Cluster Autoscaler, **Karpenter**,
  GKE NAP/Autopilot, AKS NAP) is owned by ch04 §11; ch10 only asserts the
  **cost property** — pick a scaler that does live consolidation and
  supports diversified spot pools, otherwise §7's economics are out of
  reach. For non-K8s workloads, scale-to-zero (FaaS, Cloud Run, ACA,
  App Runner) often beats rightsizing a long-running VM.
- **Do** default new AWS block storage to **gp3**, not gp2 — up to
  20% cheaper per GiB plus 3,000 IOPS / 125 MiB/s baseline regardless
  of size, in-place migration via Elastic Volumes
  ([AWS Storage Blog](https://aws.amazon.com/blogs/storage/migrate-your-amazon-ebs-volumes-from-gp2-to-gp3-and-save-up-to-20-on-costs/)).
  Highest-ROI single AWS knob most teams have not pulled.
- **Don't** rightsize from synthetic benchmarks (production has
  correlated spikes, GC pauses, and noisy neighbors no benchmark
  reproduces). **Don't** chase a 100%-utilized service — at 100% you
  are queueing and your latency SLO is breached. **Note** —
  rightsizing changes *requests*, not limits; CPU-limit policy is
  owned by ch04 §5.

---

## 6. Scale-to-zero

- **Do** prefer scale-to-zero compute for spiky or low-volume
  workloads: Lambda / Functions for sub-second event handlers; Cloud
  Run / Azure Container Apps / App Runner for HTTP services; **KEDA**
  for K8s workloads that should drop to zero pods between events
  ([KEDA docs](https://keda.sh/), CNCF graduated 2023). **Prefer**
  Cloud Run / ACA over raw FaaS for HTTP services with non-trivial
  startup — same economics with container portability.
- **Do** measure cold-start cost honestly: JVM/.NET cold starts are
  hundreds of ms to seconds; Node/Python typically tens to low hundreds
  of ms. Provisioned Concurrency / Always-Ready trade scale-to-zero
  economics for latency; verify with p99 in production.
- **Note (ACA)** — Azure Container Apps scale-to-zero is real *only
  when replicas reach zero*; Microsoft documents that "replicas that
  aren't processing, but remain in memory might be billed at a lower
  idle rate"
  ([ACA scaling rules](https://learn.microsoft.com/azure/container-apps/scale-app)).
  Verify `minReplicas: 0` and that your KEDA scaler actually releases
  the warm replica (same caveat for AKS+KEDA).

---

## 7. Spot / preemptible

- **Do** use spot/preemptible/Azure Spot for any **stateless,
  retry-able, idempotent, interruption-tolerant** workload: CI runners,
  batch ETL, ML training, async workers, dev clusters. 60–90% discounts
  are documented on each provider's pricing page.
- **Do** treat spot interruption notice as **best-effort and
  provider-specific**. AWS docs state: "It is always possible that
  your Spot Instance might be interrupted", and a user-set max price
  causes more frequent interruption. Causes (capacity, host
  maintenance, hardware decommission, price, constraint) are correlated
  provider-side events you cannot predict.

| Provider | Product | Interruption notice | Notes |
|---|---|---|---|
| AWS | EC2 Spot | Best-effort **2 minutes** via instance metadata + EventBridge | "Always possible" interruption; max-price increases rate; Spot Blocks deprecated ([AWS Spot interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)) |
| Azure | Spot VMs / Spot Scale Sets | Best-effort **up to 30 seconds** via Scheduled Events | Eviction may also be cost-based; configurable max price ([Azure Spot VMs](https://learn.microsoft.com/azure/virtual-machines/spot-vms)) |
| GCP | Spot VMs (preemptible legacy) | **~30 seconds** ACPI G2 Soft Off / shutdown script | Documented across machine types; plan for sudden termination ([GCP Spot VMs](https://cloud.google.com/compute/docs/instances/spot)) |

- **Do** apply the **dual-pool pattern** for production Kubernetes: an
  on-demand floor (e.g., 30% of capacity, system pods, stateful sets)
  plus a spot bulk pool for stateless workloads (ch04 §11 owns scaler
  choice).
- **Don't** put databases, primary brokers, or anything with local
  state on spot — drain window is not enough for a multi-hour replica
  rebuild. K8s guardrails (PDBs, StatefulSet drain) are owned by ch04
  §5/§13; resilience under sudden node loss by ch09 §9. **Don't** use
  a single instance type in your spot pool — diversify across 3–5
  families/sizes/AZs; eviction is correlated within a pool
  ([AWS Spot Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)).

---

## 8. Egress and network: the surprise bill

Egress and network-processing fees break budgets, and SKUs differ by
provider — always price against the current pricing page.

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

- **Note — egress-on-exit (2024)**: AWS (Mar 2024, expanded Sep 2025),
  GCP (Jan 2024), and Azure all waive data-transfer-out for customers
  *migrating off* the platform (EU Data Act response). Does **not**
  change in-life egress, cross-AZ, NAT, PrivateLink, or Private Endpoint
  pricing — Cloudflare's 2021 markup analysis stays directionally
  correct for day-to-day egress
  ([AWS](https://aws.amazon.com/blogs/aws/free-data-transfer-out-to-internet-when-moving-out-of-aws/);
  [GCP](https://cloud.google.com/blog/products/networking/eliminating-data-transfer-fees-when-migrating-off-google-cloud)).
- **Do** put a CDN in front of object/blob storage serving the public
  internet; origin → CDN egress is discounted or free within the same
  provider, and CDN egress is dramatically cheaper at volume.
- **Do** keep chatty services in the same AZ where possible
  (topology-aware routing in Kubernetes; placement / proximity
  placement groups on EC2/Azure) — cross-AZ chatter is a hidden
  multiplier.
- **Do** audit NAT and load-balancer **processing** fees independently
  of egress; a high-throughput service behind a NAT gateway can spend
  more on per-GB NAT processing than on the destination egress itself.

---

## 9. Storage lifecycle

- **Do** define lifecycle policies as IaC for every bucket/container:
  hot → warm (IA/Cool) → cold (Glacier IR/Archive/Coldline) → delete.
  Typical defaults: warm at 30d, cold at 90d, archive at 365d, delete
  per retention. AWS S3 Intelligent-Tiering automates the hot→warm
  boundary if access is unknown
  ([AWS WA Cost-Opt pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html), COST04).
- **Do** review **request and retrieval costs**, not just GB cost —
  many cold tiers have per-retrieval/per-request fees that dwarf
  storage savings if access patterns were misjudged. **Don't**
  lifecycle data you may need for compliance retrieval out to a tier
  whose retrieval SLA you cannot meet.

---

## 10. The cost of governance

- **Don't** spend a $500 platform-team week chasing a $5/day spike;
  prioritize by **dollar impact × confidence × ease**, not alert noise.
  **Prefer** a written investigation threshold (e.g., > $100/day or
  > 20% of service spend; otherwise log and move on) — most common
  reason internal FinOps programs stall.

---

## 11. Showback, chargeback, and allocation

- **Showback** = tell the team what they spend. **Chargeback** = the
  team's P&L is debited. The hard problem is **allocation**: how
  shared, indirect, and amortized costs map to teams. **Prefer**
  showback first, chargeback once data is trusted (typically 6–12
  months of clean tagging) — chargeback before tagging discipline
  poisons the program with finance disputes (Storment & Fuller, ch. 10).

### 11.1 Allocation mechanics

- **Direct attribution** — costs with a clean owner tag flow straight
  to the owning team. With FOCUS 1.2 use `BillingAccountType` /
  `SubAccountType` plus `InvoiceID` as the standardized cross-cloud
  join key rather than hand-rolled account hierarchies.
- **Proportional usage allocation** — shared infrastructure (multi-
  tenant K8s, shared DBs, observability, log ingestion) is split by a
  measured driver (CPU·hour, memory·GB·hour, ingested log GB, request
  count). OpenCost/Kubecost implement this for K8s; for shared SaaS
  use vendor exports or OpenCost plugins (Datadog, MongoDB Atlas,
  OpenAI).
- **Fixed shared-cost pools** — control plane, support, SSO, baseline
  networking go into a pool allocated by a stable key (headcount,
  revenue share, % of direct spend); revisit annually. Anything still
  unallocated belongs to a named **platform pool**, not silently
  smeared — pool growth = tagging-debt signal.
- **Amortization & virtual currency** — upfront commitments and
  prepaid credits must be amortized across the term in showback so
  monthly numbers reflect consumption, not cash. Snowflake credits,
  Databricks DBUs, AI tokens, and cloud credits each have lifecycle
  (issue → consume → expire → refund); FOCUS 1.2 adds dedicated
  columns. Treat them as first-class allocation units.
- **Do** make per-service unit economics ($/order, $/active user,
  $/inference, $/training-run) the headline metric in engineering
  review, not the absolute monthly bill — the **Quantify Business
  Value** capability in the FinOps Foundation framework.

---

## 12. Anomaly detection

- **Do** turn on cloud-native anomaly detection. On AWS, enable
  **AWS-managed monitors** in Cost Anomaly Detection in every payer
  account — they auto-track all accounts/teams/BUs and auto-include
  new accounts, tag values, and cost categories with severity weighted
  by historical variance
  ([AWS CAD](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)).
  Azure CM anomaly alerts and GCP budget anomaly are equivalents.
- **Do** add **OpenCost** (CNCF Incubating since Oct 2024) to any
  multi-tenant K8s cluster — cluster-level cloud bills cannot tell you
  which namespace burned the spike. The plugin framework (Datadog,
  OpenAI, MongoDB Atlas) extends the same model to SaaS and AI cost
  ([opencost.io](https://www.opencost.io/);
  [CNCF Incubation](https://www.opencost.io/blog/cncf-incubation));
  Kubecost is the commercial superset.
- **Prefer** "alert at significant deviation" (e.g., +30%
  week-over-week or > $X absolute) over "alert on first dollar".

---

## 13. Commitment portfolio management

- **Do** run a **laddered commitment** strategy: stagger 1y/3y so a
  portion expires every quarter — never reach 100% coverage; never
  face a cliff. Target a **base-load coverage** band, not "% of total":
  compute base-load as `p10(hourly_on_demand_eq_spend, 30d)`, set
  `target_commit ≈ 0.7..0.9 × base_load`. Cover base-load with the
  highest-discount product the workload permits, the variable layer
  with spend-based commitments, the bursty top with on-demand or spot.
  Track `expected_unused = max(0, target_commit − realized_usage)`
  monthly; chronic > ~5% means the band is too high. Cash, FX, and
  amortization come from §2.2.
- **Note — data warehouse / virtual-currency commitments are not VM
  commitments.** BigQuery Editions (2023) replaced flat-rate slot
  reservations with autoscaling slot pools — Google estimates 30–40%
  lower committed capacity vs the prior fixed model
  ([BigQuery Editions](https://cloud.google.com/blog/products/data-analytics/introducing-new-bigquery-pricing-editions)).
  Snowflake credits and Databricks DBUs follow virtual-currency
  mechanics (first-class in FOCUS 1.2). Model these portfolios
  separately from VM compute.

---

## 14. AI / GPU cost

- **Do** treat AI cost as a first-class FinOps domain — *State of
  FinOps 2025* lists it as the #1 forward-looking priority and skillset
  gap. Split: **training** (large, batch, schedulable, often
  spot-tolerant with checkpointing) vs **inference** (latency-bound,
  scale-to-zero for spiky traffic, reserved/committed for steady).
  Allocate GPU·hour and token cost the same way you allocate CPU·hour
  (per service/feature/customer); OpenCost plugins (OpenAI etc.) and
  FOCUS 1.2 virtual-currency columns are the off-the-shelf path.
- **Prefer** smaller / quantized / distilled models, batched inference,
  KV-cache reuse, and prompt-caching where supported before buying more
  GPU capacity — usually the largest controllable line items.
- **Don't** put inference for latency-critical user paths on spot GPUs
  (correlated reclamation, slow recovery); training and offline batch
  inference are the spot/preemptible candidates. **Don't** sign a
  multi-year reserved-GPU commitment off a 30-day usage curve — AI
  workload shape is volatile and §2.2's organizational-volatility rule
  applies double.

---

## 15. Sustainability as a cost-adjacent metric

- **Do** report carbon alongside dollars **with measurement basis
  attached**: scope (1/2/3 covered), method (location- vs market-based),
  region/temporal granularity, and the **internal carbon price** (if
  any) used to trade off alternatives. Without a price, carbon and
  dollars cannot be traded off mechanically.
- Provider tools — **AWS Sustainability console** (2025 successor to
  the in-billing Customer Carbon Footprint Tool, with its own IAM,
  APIs/SDK, and Scope 1/2/3 breakdown by region/service/account back to
  2022), Azure Emissions Impact Dashboard, GCP Carbon Footprint —
  expose Scope 2 estimates with different methodologies. For
  multi-cloud or finer granularity, **Cloud Carbon Footprint**
  ([cloudcarbonfootprint.org](https://www.cloudcarbonfootprint.org/))
  estimates from billing line items (estimate, not measurement).
- **Prefer** lower-carbon regions when latency/residency permit, and
  shift batch/training to lower-carbon time windows where the scheduler
  supports it. Cheaper and greener often, but not always, align —
  verify per workload.

---

## 16. Anti-patterns

- **Cost dashboards nobody looks at.** No owner, no cadence — delete.
- **"Let the platform team buy more RIs".** Commitments without
  engineering forecast are a finance bet, not FinOps.
- **Tagging policies without enforcement.** Optional tags = no tags.
- **Single-AZ, single-family spot pool.** One eviction event drains it.
- **Cross-region object-storage reads as a default.** Egress catches
  up in month 2.
- **Lifecycle policies set up once, never reviewed.** Archived hot
  data costs more, not less.
- **Per-team cost guilt-tripping with no allocation.** Without
  showback the conversation is unfounded.
- **Carbon dashboards with no measurement basis.** Greenwashing.
- **Multi-year reserved GPU off a 30-day curve.** Commit the floor,
  burst the rest.
- **"Egress is free now" as architecture.** The 2024 waivers cover
  *exit*, not in-life cross-AZ/NAT/PrivateLink.
- **Lift-and-shift to a "cheaper" cloud or back to on-prem** without
  a TCO model (engineering time, retraining, lost managed-service
  leverage). The a16z "Cost of Cloud" piece (2021) argues for
  repatriation only at scale, after exhausting cloud-native
  optimization; the 2024 egress-on-exit waivers lower transfer cost
  but do not change this calculus.

---

## Further reading

- Storment, J.R. & Fuller, M. — *Cloud FinOps*, 2nd ed., O'Reilly, 2023.
- FinOps Foundation — <https://www.finops.org/framework/> ·
  [Principles](https://www.finops.org/framework/principles/) ·
  [State of FinOps 2025](https://www.finops.org/insights/state-of-finops-2025/) ·
  [FOCUS 1.2](https://www.finops.org/insights/focus-1-2/) (<https://focus.finops.org/>)
- AWS — [Cost-Optimization pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html) ·
  [Savings Plans](https://aws.amazon.com/savingsplans/compute-pricing/) ·
  [Spot interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html) ·
  [bandwidth](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer) ·
  [Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html) ·
  [Sustainability console](https://aws.amazon.com/aws-cost-management/aws-customer-carbon-footprint-tool/)
- Azure — [Reservations](https://learn.microsoft.com/azure/cost-management-billing/reservations/save-compute-costs-reservations) ·
  [Savings Plan](https://learn.microsoft.com/azure/cost-management-billing/savings-plan/savings-plan-compute-overview) ·
  [Spot VMs](https://learn.microsoft.com/azure/virtual-machines/spot-vms) ·
  [Container Apps scaling](https://learn.microsoft.com/azure/container-apps/scale-app) ·
  [bandwidth](https://azure.microsoft.com/pricing/details/bandwidth/)
- GCP — [CUDs](https://cloud.google.com/docs/cuds) ·
  [Spot VMs](https://cloud.google.com/compute/docs/instances/spot) ·
  [network pricing](https://cloud.google.com/vpc/network-pricing) ·
  [BigQuery Editions](https://cloud.google.com/blog/products/data-analytics/introducing-new-bigquery-pricing-editions)
- OpenCost (CNCF Incubating) — <https://www.opencost.io/> ·
  [Incubation](https://www.opencost.io/blog/cncf-incubation) ·
  Cloud Carbon Footprint — <https://www.cloudcarbonfootprint.org/>
