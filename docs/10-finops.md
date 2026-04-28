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

**Do** adopt the FinOps Foundation framework explicitly: **Inform → Optimize →
Operate**, executed continuously, not as a one-off cost-cutting project
([finops.org/framework](https://www.finops.org/framework); Storment & Fuller,
*Cloud FinOps* 2nd ed., O'Reilly 2023, ch. 1–3). The three phases map to:

- **Inform** — visibility, allocation, benchmarking. You cannot optimize what
  you cannot attribute.
- **Optimize** — rightsizing, commitment-based discounts, architectural change.
- **Operate** — policies, automation, KPIs, anomaly response. The steady state.

**Don't** treat FinOps as a finance reporting function. The decisions
(turn off the dev cluster at 18:00, move logs to cold storage, re-architect
the cross-AZ chatty service) are engineering decisions; finance owns the
forecast and the commitment ladder. Storment & Fuller call this the
"crawl → walk → run" maturity model — every team is at a different stage and
that is fine; what is not fine is no stage at all.

**Do** anchor the practice on the six **FinOps Principles**: teams need to
collaborate, decisions are driven by business value, everyone takes ownership
of cloud usage, FinOps reports are accessible and timely, a centralized team
drives FinOps, and take advantage of the variable cost model of the cloud
([finops.org/framework/principles](https://www.finops.org/framework/principles/)).

---

## 2. The cost model: know your levers

Cloud compute is sold on (at least) four pricing dimensions. You should be
able to explain the break-even of each on a whiteboard.

| Model | Discount vs on-demand | Commit | Risk |
|---|---|---|---|
| On-demand | 0% | none | none |
| Reserved Instance / 1-yr no-upfront | ~30–40% | 1y | unused capacity |
| Savings Plan / 3-yr all-upfront | ~50–70% | 3y | wrong family / cash |
| Spot / preemptible / Azure Spot | 60–90% | none | eviction, ~2-min notice |

Numbers per AWS Well-Architected Cost Optimization pillar (COST07-BP01) and
the published AWS / Azure / GCP rate cards as of 2024.

**Do** compute break-even before committing. Rule of thumb: a 1-year
no-upfront RI breaks even at ~7–8 months of continuous use; a 3-year
all-upfront SP breaks even around month 14–16. If your workload may not
exist in 14 months, do not buy a 3-year commitment for it.

**Prefer** Savings Plans / Compute Savings Plans over classic RIs for compute
that is likely to change instance family or region. RIs lock you to a family;
Compute Savings Plans apply to *any* EC2/Fargate/Lambda spend. Same broad
pattern exists on Azure (Savings Plan for Compute) and GCP (Committed Use
Discounts — flexible CUDs since 2022).

---

## 3. Visibility before optimization

**Do** make tagging a build-time concern, not a cleanup-time one. Required
tags on every cloud resource: `cost-center`, `service`, `env`, `owner`,
`data-classification`. Enforce in IaC with policy-as-code (Azure Policy,
AWS SCP + Config rules, GCP Org Policy + custom constraints).
Untagged resources are a finding, not a warning. Storment & Fuller, ch. 7
("Allocation"): "if it is not tagged, it cannot be charged back".

**Do** standardize on the **FOCUS** specification (FinOps Open Cost &
Usage Specification, [focus.finops.org](https://focus.finops.org/)) for any
cross-cloud cost dataset. FOCUS 1.0 (June 2024) is exported natively by
AWS CUR 2.0, Azure Cost Management, GCP Billing, OCI, and Snowflake. Building
custom ETL from raw CUR/EA exports is now an anti-pattern; start FOCUS-first.
ThoughtWorks Tech Radar Vol. 30/31 places FOCUS in **Adopt**.

**Prefer** the cloud-native cost UI (Cost Explorer, Azure Cost Management,
GCP Billing reports) for daily/weekly use, and a unified view (Vantage,
CloudZero, OpenCost+Grafana, or your own FOCUS warehouse) for monthly
exec/cross-cloud views. The cloud-native tools are free, fresh, and good
enough for 80% of questions.

**Don't** ship cost dashboards that nobody opens. A dashboard with no
reviewing cadence and no owner is decoration. Either put it on a weekly
service-team review with a named owner, or delete it.

---

## 4. Budgets and the kill-switch

**Do** create a budget at every account / subscription / project, with at
least three alert thresholds: 50% (informational, email service owner),
80% (warning, ping team channel), 100% (page on-call + automated action).
AWS Budgets supports SNS → Lambda for automated action; Azure Budgets fires
Action Groups; GCP Billing Budgets fire Pub/Sub. Documented in the
[AWS Well-Architected Cost Optimization pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
(COST01-BP05) and [Azure Cost Management best practices](https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices).

**Do** implement a **kill-switch** for non-prod: at 100% of monthly budget,
automatically stop or scale-to-zero the dev/test environment. Yes, this will
wake somebody up the first time. That is the point; an alert with no action
is a feeling, not a control. The pattern is well-documented in the
[AWS Builders' Library "Reducing your cloud costs"](https://aws.amazon.com/builders-library/)
materials and in Forrest Brazeal's writing on cost guardrails.

**Don't** put a kill-switch on production. Production gets paged and
human-decided. The kill-switch lives in dev, test, sandbox, and personal
accounts.

---

## 5. Rightsizing

**Do** drive rightsizing from observed p95/p99 utilization plus a documented
headroom (commonly **p99 + 30%** for steady-state services, more for spiky
ones). The cloud rightsizing recommenders (AWS Compute Optimizer, Azure
Advisor, GCP Recommender) all use percentile-based heuristics — read the
recommendation, do not just click apply.

**Prefer** Kubernetes-native rightsizing: **VPA** in *recommendation* mode
(not auto-apply) feeding requests, plus **Karpenter** (AWS / now CNCF) or
the cluster autoscaler with consolidation enabled. Karpenter's
*consolidation* feature is the single highest-ROI knob most EKS clusters
have not turned on (Karpenter docs; AWS re:Invent 2023 talks).

**Don't** rightsize from synthetic benchmarks. Production has correlated
spikes, GC pauses, and noisy neighbors that no benchmark reproduces. Use
real percentiles from real load.

**Don't** chase a 100%-utilized service. CPU at 100% means you are queueing;
your latency SLO is already breached. Headroom is the cost of reliability.

---

## 6. Scale-to-zero

**Do** prefer scale-to-zero compute for spiky or low-volume workloads:
Lambda / Functions for sub-second event handlers; Cloud Run / Azure
Container Apps / App Runner for HTTP services; KEDA for Kubernetes workloads
that should drop to zero pods between events
([KEDA docs](https://keda.sh/), CNCF graduated 2023).

**Do** measure cold-start cost honestly. Lambda cold-start on a JVM/.NET
runtime is hundreds of milliseconds to seconds; on Node/Python typically
tens to low hundreds of ms. Provisioned Concurrency / Always-Ready
instances trade scale-to-zero economics for latency. Pick consciously,
document in the service README, and verify with p99 latency in production.

**Prefer** Cloud Run / ACA over raw FaaS for HTTP services with non-trivial
startup; you keep scale-to-zero economics with full container portability
and longer execution limits. The Andreessen Horowitz "Cost of Cloud" essay
and Vantage write-ups consistently show that managed-container offerings
hit a sweeter price/performance point than FaaS once requests exceed ~50ms
of work.

---

## 7. Spot / preemptible

**Do** use spot/preemptible/Azure Spot for any workload that is **stateless,
retry-able, idempotent, and tolerant of 1–2 minute eviction notice**: CI
runners, batch ETL, ML training, async workers, dev clusters. Discounts of
60–90% are real and stable enough to plan around (per AWS / Azure / GCP
spot pricing history; see also Vantage's spot interruption tracker).

**Do** apply the **dual-pool pattern** for production Kubernetes: an
on-demand floor (e.g., 30% of capacity, system pods, stateful sets) plus a
spot bulk pool for stateless workloads. Karpenter and GKE node auto-
provisioning both support this natively with weighted node pools / instance
diversification.

**Don't** put databases, primary brokers, or anything with local state on
spot. The 2-minute drain window is not enough for a 4-hour replica rebuild.

**Don't** use a single instance type in your spot pool. Diversify across
3–5 families/sizes/AZs; eviction is correlated within a pool. AWS Spot Best
Practices and the Karpenter docs both call this out as the #1 reliability
pattern.

---

## 8. Egress: the surprise bill

Egress (data leaving the cloud, leaving a region, or crossing AZs) is the
line item that breaks budgets. Cross-AZ traffic on AWS is $0.01/GB *each
way*; internet egress is $0.05–0.09/GB after the free tier. Cloudflare and
the Andreessen Horowitz "Cost of Cloud" piece (2021) both made the case
that egress economics fundamentally shape architecture; nothing has
changed since.

**Do** put a CDN in front of any object/blob storage that serves the
public internet (CloudFront, Azure Front Door / CDN, Cloud CDN, or
Cloudflare/Fastly). Origin egress to a CDN is discounted or free; CDN egress
is dramatically cheaper at volume. Quinn's *Last Week in AWS* has
returned to this point repeatedly (e.g., "The compelling economics of
Cloudflare R2", 2022).

**Do** keep chatty services in the same AZ where possible (topology-aware
routing in Kubernetes; placement groups on EC2). A microservice mesh
splattered across three AZs with no topology hints is a hidden 2–3×
cost multiplier.

**Don't** pull large datasets out of object storage to a different region
or cloud "to process them where the team works". Move the compute, not the
data. This is the lift-and-shift egress trap.

---

## 9. Storage lifecycle

**Do** define lifecycle policies as IaC for every bucket / container /
GCS bucket: hot (frequent access) → warm (IA / Cool) → cold (Glacier IR /
Archive / Coldline) → delete. Typical defaults: warm at 30 days, cold at
90, archive at 365, delete at policy-defined retention. AWS S3 Intelligent-
Tiering automates the hot→warm boundary if access patterns are unknown.
Documented in the [AWS Well-Architected Cost Optimization pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
(COST04) and Azure / GCP equivalents.

**Do** review **request costs**, not just storage GB cost. Many cold tiers
have per-retrieval fees that dwarf the storage saving if access patterns
were misjudged. S3 Glacier Deep Archive is $0.00099/GB/month — and
$0.02/GB to retrieve. Wrong tier = a worse bill, not a better one.

**Don't** lifecycle data you may need for compliance retrieval out to a
tier whose retrieval SLA you cannot meet.

---

## 10. The cost of governance

**Don't** spend a $500 platform-team week chasing a $5/day spike. The cost
of the FinOps process must be less than the cost it controls. Storment &
Fuller call this out repeatedly: a mature FinOps practice prioritizes by
**dollar impact × confidence × ease**, not by alert noise.

**Prefer** a written threshold (e.g., investigate anomalies > $100/day or
> 20% of service spend; otherwise log and move on). This is
the single most common reason internal FinOps programs stall — they try to
fix everything and burn out their advocates.

---

## 11. Showback vs chargeback

**Showback** = tell the team what they spend. **Chargeback** = the team's
P&L is debited.

**Prefer** showback first, chargeback once the data is trusted (typically
6–12 months of clean tagging). Chargeback before tagging discipline causes
finance disputes that poison the program (Storment & Fuller, ch. 10).

**Do** make per-service unit economics ($/order, $/active user, $/inference)
the headline metric in the engineering review, not the absolute monthly
bill. Absolute spend goes up with the business; unit cost is the signal.
This is the "value" lens in the FinOps Foundation framework's
**Quantify Business Value** capability.

---

## 12. Anomaly detection

**Do** turn on cloud-native anomaly detection (AWS Cost Anomaly Detection,
Azure Cost Management anomaly alerts, GCP Recommender / budget anomaly).
They are free, ML-based, and tuned for the platform.

**Do** add **OpenCost** (CNCF, the upstream of KubeCost) to any
multi-tenant Kubernetes cluster. Cluster-level cloud bills cannot tell you
which namespace burned the spike; OpenCost can
([opencost.io](https://www.opencost.io/), ThoughtWorks Tech Radar:
**Adopt**). KubeCost is the commercial superset; pick based on whether you
want the OSS UX or the enterprise features.

**Prefer** "alert at significant deviation" (e.g., +30% week-over-week, or
> $X absolute) over "alert on first dollar of new service". The latter
generates so much noise it gets muted within a week.

---

## 13. Reservations: the ladder

**Do** run a **laddered commitment** strategy: stagger 1-year and 3-year
commitments so a portion expires every quarter. You never reach 100%
coverage and you never face a cliff. Documented in Cloudability and Vantage
write-ups (e.g., Vantage blog "How to build a Reserved Instance ladder",
Ben Schaechter; Cloudability "Reserved Instance Ladder Strategy").

**Do** target a **60–80% commitment coverage** band on steady-state
compute. Below 60% you are leaving money on the table; above 80% any
modest workload change strands commitments. The exact band is workload-
specific; the principle is "never 100%".

**Don't** delegate commitment purchases to "the platform team will handle
RIs". Commitments are a finance + engineering decision; engineering owns
the forecast of what will still be running in 12–36 months.

---

## 14. Sustainability as a cost metric

**Do** report carbon alongside dollars. The AWS Customer Carbon Footprint
Tool, Azure Emissions Impact Dashboard, and GCP Carbon Footprint reports
all expose Scope 2 estimates per service / region. For a multi-cloud or
detailed view, use **Cloud Carbon Footprint**
([cloudcarbonfootprint.org](https://www.cloudcarbonfootprint.org/),
Thoughtworks OSS; Tech Radar: **Trial/Adopt**).

**Prefer** lower-carbon regions when latency and data-residency permit.
Region carbon intensity differs by an order of magnitude; this is one of
the few decisions where the cheap choice and the green choice agree.

---

## 15. The "move to a cheaper cloud" trap

**Don't** lift-and-shift to a "cheaper" cloud or back to on-prem to "save
money" without a TCO model that includes engineering time, retraining,
re-tooling, and the loss of managed-service leverage. The Andreessen
Horowitz piece "[The Cost of Cloud, a Trillion Dollar Paradox](https://a16z.com/the-cost-of-cloud-a-trillion-dollar-paradox/)"
(2021) is widely misquoted as advocating repatriation; read carefully, the
authors argue **for** repatriation only at scale, with a dedicated
infra team, and after exhausting cloud-native optimization. Most teams are
not at that scale. Optimize first; repatriate only with a defended TCO
model.

---

## 16. Anti-patterns

- **Cost dashboards nobody looks at.** No owner, no cadence, no decision —
  delete.
- **"Let the platform team buy more RIs".** Commitments without engineering
  forecast are a finance bet, not FinOps.
- **Weekly cost meetings with no kill-switches.** Discussion without
  authority to act is theatre.
- **Tagging policies without enforcement.** Optional tags = no tags.
- **Single-AZ spot pool.** One eviction event drains everything.
- **Cross-region S3 reads as a default.** The egress line will catch up
  with you in month 2.
- **Lifecycle policies set up once, never reviewed.** Access patterns
  drift; archived hot data costs more, not less.
- **Per-team cost guilt-tripping with no allocation.** Without showback
  data the conversation is unfounded.

---

## Further reading

- Storment, J.R. & Fuller, Mike — *Cloud FinOps*, 2nd ed., O'Reilly, 2023.
- FinOps Foundation Framework — <https://www.finops.org/framework/>
- FOCUS specification — <https://focus.finops.org/>
- AWS Well-Architected Cost Optimization Pillar —
  <https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html>
- Azure Cost Management best practices —
  <https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices>
- GCP Cost Optimization on GKE —
  <https://cloud.google.com/architecture/best-practices-for-running-cost-effective-kubernetes-applications-on-gke>
- OpenCost — <https://www.opencost.io/>
- Cloud Carbon Footprint — <https://www.cloudcarbonfootprint.org/>
- A16z, "The Cost of Cloud, a Trillion Dollar Paradox" (2021) —
  <https://a16z.com/the-cost-of-cloud-a-trillion-dollar-paradox/>
- Quinn, Corey — *Last Week in AWS* — <https://www.lastweekinaws.com/>
- Brazeal, Forrest — selected essays on cloud cost discipline.
