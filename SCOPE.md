# Scope — who this guide is for, and who it is not for

This guide is opinionated. The opinions only hold inside a specific
organisational and technical envelope. This page names that envelope so
readers can decide, in 60 seconds, whether the rest of the guide will
help them or mislead them.

If your situation falls outside this envelope, individual chapters may
still be useful as background reading — but the *defaults* will be
wrong for you, and you should treat the recommendations as input, not
verdict.

---

## 1. Who it's for

Concrete reader profiles. If two or more of these apply, you are the
target reader.

- A staff/principal engineer or tech lead inheriting an existing cloud
  estate and asked to "make it boring".
- A platform / SRE / DevOps engineer at a product company with
  multiple stream-aligned teams (Team Topologies sense; ch11).
- An engineering manager scoping a platform team's first 12-month
  charter and trying to avoid the "renamed ops team" anti-pattern
  (ch11 §16).
- A first-time IaC owner picking between Terraform / OpenTofu / Pulumi
  / Bicep / CDK / Crossplane for a new repo (ch02 §2).
- An engineer drafting a design doc for an architecture review where
  reviewers will push back on Kubernetes adoption, multi-region, or
  multi-cloud (decision-trees §3, §6).
- A security/compliance engineer needing a defensible default posture
  for supply chain (SLSA, SBOM, signing, OIDC) and workload identity
  (ch03, ch06).
- A founding engineer at a Series-A/B SaaS who has to lay down account
  topology, IP plan, and IdP before the org grows past the point where
  those decisions can be edited (decision-trees §1).

---

## 2. Organisational shape it assumes

The defaults assume a *mid-sized cloud SaaS or product organisation*.
Specifically:

- **Multiple stream-aligned teams** sharing infrastructure — not a
  single squad, not a 10,000-engineer enterprise. Roughly 30–500
  engineers is the sweet spot; below that, parts of ch11 are
  premature, above that, parts are under-specified.
- **Git-based delivery** as the source of truth for application code
  *and* infrastructure. Pipelines are code in the same repo as the
  thing they build (ch03 §1). If your delivery is ticket-driven
  through a change-advisory board, this guide will not fit.
- **A paid on-call rotation** for at least one user-facing service.
  SLO and error-budget chapters (ch09) assume incidents have a
  responder with a pager and a runbook, not a best-effort rota.
- **An existing or emerging platform / SRE function** — even if it is
  one engineer with a 30% allocation. The "platform as a product"
  framing in ch11 assumes you have, or are about to have, internal
  customers.
- **Cloud-first, SaaS-heavy procurement.** The IdP, registry,
  observability backend, and secret store are expected to be SaaS or
  managed services, not on-prem appliances.
- **Engineering owns production.** "You build it, you run it" (ch01
  §6) is assumed, not negotiated. If a separate ops group owns prod,
  re-read ch01 §6 before anything else.

---

## 3. Clouds and runtimes it assumes

- **Hyperscaler-managed defaults.** AWS, Azure, GCP. Where vendor
  primitives diverge, ch07 spells the parallel out; the
  *recommendation* is the pattern, not the product.
- **Managed cloud-native primitives by default.** Managed K8s
  (AKS/EKS/GKE) before self-hosted; serverless containers (Cloud Run,
  ACA, App Runner, Fargate) before K8s at all (ch04 §1).
- **Kubernetes-adjacent, not Kubernetes-mandatory.** Ch04 §1 is
  explicit: default to "no K8s" until three workloads need it. The
  guide assumes you might run K8s, not that you do.
- **OpenTelemetry as the instrumentation contract** (ch05 §2). Vendor
  backends are pluggable behind the OTel collector; SDK choice is not.
- **SLO-centric reliability.** Google SRE Book + Workbook canon (ch09
  §1). If "five nines because the contract says so" is your model,
  ch09 §1.3 will read as heretical.
- **Primary-source citations from CNCF, IETF, Google SRE, NIST,
  ThoughtWorks Tech Radar, FinOps Foundation,** and the cloud
  Well-Architected frameworks. Industry voices (Majors, Fong-Jones,
  Hightower, Sridharan, Butow, Gregg, Morris, Skelton/Pais) are cited
  when they are the original popularisers (CONTRIBUTING.md §
  Sourcing).

---

## 4. What it is NOT

Stating the negatives so the guide doesn't get blamed for being bad at
things it never claimed to be good at.

- **Not a vendor-specific cookbook.** No portal click-paths, no
  Terraform-module-of-the-week, no "10 things to do in the AWS
  console". Decisions are at the IaC / pattern level.
- **Not a data engineering or lakehouse-ops guide.** Ch08 covers
  *operating* stateful infrastructure (backups, restore, HA, RPO/RTO,
  tenancy). Schema design, ELT pipelines, dbt, Airflow, Spark tuning,
  ML feature stores are out of scope.
- **Not an enterprise cloud-foundation playbook for VMware /
  OpenStack-heavy estates.** Hybrid is acknowledged in ch07, but the
  centre of mass is hyperscaler IaaS/PaaS.
- **Not a regulated-industry compliance manual.** Ch06 names the
  control families (NIST SSDF, SLSA, threat modelling, zero trust);
  it does not map them to FedRAMP, HIPAA, PCI-DSS, DORA (EU), or
  SOC 2 line by line. That work is downstream of this guide.
- **Not a beginner's intro to DevOps.** Readers are assumed to know
  what a CI pipeline, a VPC, and a container image are. The guide is
  about *what to default to* and *what to reject*, not *what these
  things mean*.
- **Not a frontend, mobile, or end-user accessibility guide.** Edge,
  CDN, and anycast appear only in ch07 as transport concerns.
- **Not an AI/ML training-infrastructure guide.** GPU pools are
  mentioned as a K8s adoption trigger (ch04 §1) and a cost line item
  (ch10); training pipelines, model registries, and inference serving
  are not covered.

---

## 5. "If you are X, this guide is Y for you"

| Reader archetype | Recommendation |
|---|---|
| Staff engineer at a 200-person SaaS, multi-team, hyperscaler-native | **Primary reader.** Read decision-trees + checklist first; chapters as backing material. |
| Platform / SRE lead chartering a new platform team | **Primary reader.** Start at ch11 §16 (readiness gate) before any tooling choice. |
| Solo founder, one product, pre-PMF | **Skim only.** Decision-trees §1 (topology, IP plan, identity) and ch10 §4 (budgets) are worth an hour. The rest is premature. |
| Enterprise architect at a 10k-engineer regulated bank | **Background reading.** Useful as a vocabulary and a sanity check on hyperscaler-native patterns; do not adopt defaults without mapping to your control framework. |
| Data / ML platform engineer | **Adjacent.** Ch08 §1–9 (stateful ops), ch05 (observability), ch06 (supply chain) apply. Family choice for warehouses and lakehouses is named but not detailed. |
| VMware / OpenStack / on-prem-heavy operator | **Background reading.** Ch07 (network), ch09 (SRE), ch01 (IaC principles) translate; ch04 (containers) and ch10 (FinOps) assume cloud economics. |
| Mobile / frontend engineer touching infra for the first time | **Wrong guide.** Read the companion .NET / language guide and the cloud's own Well-Architected first; come back when you own a backend service. |
| Compliance / GRC engineer mapping controls | **Reference, not source of truth.** Use ch06 and ch12 as the engineering-side counterpart to your control catalog; the mapping work is yours. |

---

## 6. Glossary of disclaimers used in chapters

The guide tries to be honest about where opinions are strong and where
they are conditional. The recurring qualifiers:

- **"default unless …"** — the recommendation holds for the assumed
  envelope (§2, §3); the listed exceptions are the only sanctioned
  reasons to deviate, and they require a written ADR.
- **"high-maturity baseline"** — assumes a platform/SRE function
  exists and on-call is paid. Pre-maturity teams should read the
  recommendation as *aspirational* and stage adoption.
- **`must` / `should` / `prefer` / `avoid`** — strength markers
  defined in CONTRIBUTING.md. `must` is load-bearing (≥2 independent
  primary sources); `should` is a strong default with a written-reason
  escape hatch; `prefer` is a taste call backed by evidence; `avoid`
  is explicitly rejected with a named replacement.
- **"Do / Don't / Prefer"** — Form-B equivalent of the severity
  markers; same semantics (CONTRIBUTING.md § Style).
- **"Concrete check:"** — turns an opinion into something a reviewer
  can verify in a PR or a repo. If a recommendation has no concrete
  check, treat it as a heuristic, not a rule.
- **"Cross-ref ch_X §_Y"** — the decision is *owned* by the cited
  chapter; this mention is a pointer, not a re-derivation. Do not
  resolve disagreements between chapters by reading the pointer; read
  the owner.
- **"Anti-pattern" / "Smell test"** — a rejected pattern with a named
  replacement (CONTRIBUTING.md § Style). Smell tests are
  one-screen heuristics, not gates.
- **"Primary source"** — vendor docs, CNCF/IETF/NIST specs, RFCs,
  books, named industry voices when they are the original
  popularisers. A single recent team blog post does not qualify
  (CONTRIBUTING.md § Sourcing).

If a chapter ever drops these qualifiers and asserts a universal,
treat it as a bug and open an issue.
