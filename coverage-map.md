# Coverage map — what each chapter owns, defers, and excludes

Companion to [`SCOPE.md`](./SCOPE.md). SCOPE answers *who is this
for*; this page answers *what is in it, and where does each topic
live*. If two chapters look like they overlap, this page names the
owner.

---

## 1. Per-chapter coverage matrix

| Chapter | Owns (primary scope) | Defers to | Explicitly out of scope |
|---|---|---|---|
| [`ch01 — Foundations`](./docs/01-foundations.md) | IaC philosophy, declarative vs imperative, immutability, drift, GitOps definition (OpenGitOps), control loops, you-build-you-run, environment shape, account/sub/project topology framing. | Tool choice → ch02. K8s as a control loop → ch04. Workload identity → ch06 §3. | Specific cloud onboarding, cost models, language/framework choice. |
| [`ch02 — IaC tooling`](./docs/02-iac-tooling.md) | Tool selection (Terraform / OpenTofu / Pulumi / Bicep / CDK / Crossplane), module design, state model, repo topology, provider lock and provenance, plan-review discipline, IaC testing. | Pipeline that runs the IaC → ch03. Policy gates → ch06 §4. K8s-as-target IaC → ch04. | Application config management, OS-level configuration (Ansible/Chef/Puppet for hosts is mentioned only as anti-scope). |
| [`ch03 — CI/CD`](./docs/03-ci-cd.md) | Pipeline-as-code, runner trust, OIDC over secrets, SBOM and SLSA provenance, signing, digest-pinned promotion, progressive delivery (canary, blue/green, shadow), DORA metrics ownership. | Supply-chain attestations and verification policy → ch06 §7–11. Schema migrations as deploys → ch08 §15. SLO gates on rollouts → ch09. Mono vs poly philosophy → ch11. | Build-tool-of-the-month tutorials, language-specific test frameworks. |
| [`ch04 — Containers & K8s`](./docs/04-containers-k8s.md) | "Do you need K8s", managed vs self-hosted, image hygiene, requests/limits, probes, NetworkPolicy, Gateway API, ingress, autoscaling (HPA/KPA/VPA), service mesh (when *not* to), multi-tenancy in-cluster, batch. | Image signing and admission policy → ch06 §10–11. Mesh telemetry vs OTel → ch05 §3. CNI / VPC interplay → ch07. Persistent volumes operational tier → ch08. | Helm-vs-Kustomize religious wars (one is picked, alternatives noted), specific operator catalogs. |
| [`ch05 — Observability`](./docs/05-observability.md) | Wide structured events, OTel as instrumentation contract, SDK/protocol/collector/storage tiers, three-pillars trap, cardinality budget, exemplars, RUM, tracing sample policy, vendor neutrality. | SLO and error-budget mechanics → ch09 §1. Audit log transport → ch06 §15. Cost of telemetry → ch10 §10. Mesh-as-observability anti-pattern → ch04 §9. | Specific dashboard catalogs, vendor pricing teardowns. |
| [`ch06 — Security & supply chain`](./docs/06-security-supply-chain.md) | Threat modelling, secrets management, **workload identity** (IRSA, GKE WI, AKS WI, SPIFFE, OIDC from CI), policy-as-code, SBOM, SLSA levels, signing, admission verification, zero-trust networking, audit log transport. | Human/workforce identity → ch12. Network primitives the controls run on → ch07. CI/CD plumbing the supply chain runs through → ch03. Secret store *operations* → ch08. | Compliance-framework mapping (FedRAMP/HIPAA/PCI line-by-line), pentest methodology, application-layer authn/authz code patterns. |
| [`ch07 — Networking`](./docs/07-networking.md) | VPC/VNet primitives, IP plan, subnets, peering, transit, private endpoints, DNS, egress, hybrid circuits, IPv6, MTU, CDN/anycast, mTLS as transport. | L7 mesh policy and "do we need a mesh" → ch04 §9. Workload identity inside the mesh → ch06 §3. Egress *cost* → ch10 §8. | CDN content/cache strategy, frontend performance, BGP for ISPs. |
| [`ch08 — Data & state`](./docs/08-data-state.md) | Managed vs self-hosted DB, backups (3-2-1, restore drills), PITR, replicas, HA topology, RPO/RTO tiers, blast-radius isolation, cache patterns, message brokers, **multi-tenancy data isolation**, schema migrations as deploys, encryption at rest. | Network path to data → ch07. Per-tenant cost allocation → ch10 §11. Stateful workload SLOs → ch09 §1. Data-tier supply chain (image signing for engines) → ch06. | Schema design, ORMs, query tuning, dbt/Airflow, ML feature stores, lakehouse table-format wars. |
| [`ch09 — Reliability (SRE)`](./docs/09-reliability.md) | SLI/SLO/error-budget canon, multi-window burn-rate alerting, incident response roles, runbooks-as-code, on-call rotation hygiene, blameless postmortems, chaos engineering, DR mechanics. | Instrumentation that feeds SLIs → ch05. Progressive-delivery rollback hooks → ch03 §7. RPO/RTO data tier → ch08 §4. Cost of reliability → ch10. | HR/legal aspects of on-call pay, individual vendor incident-management UIs. |
| [`ch10 — FinOps`](./docs/10-finops.md) | FinOps Foundation framework (Inform → Optimize → Operate), tagging taxonomy, budgets and burn-rate alerts, anomaly detection, rightsizing, scale-to-zero, spot/preemptible, commitments (RIs/SPs/CUDs), unit economics, **per-tenant cost allocation**, egress as a cost line. | Telemetry that feeds unit cost → ch05. Account/sub/project topology that gates allocation → ch01 §6. Multi-cloud arbitrage anti-pattern → decision-trees §6b. | Procurement/vendor negotiation tactics, finance accounting (CapEx/OpEx treatment). |
| [`ch11 — Platform engineering`](./docs/11-platform-engineering.md) | Platform-as-product, golden paths, IDP (Backstage and alternatives), Team Topologies framing, cognitive load, platform readiness gate, surface/abstraction-level choice, **DORA metric ownership**, docs-as-product. | The plumbing the platform exposes → ch02, ch04, ch07, GitOps in ch01. Workload identity as a paved path → ch06 §3. SLOs of the platform itself → ch09. | Org-design at >5,000 engineers, M&A integration playbooks. |
| [`ch12 — Identity (workforce)`](./docs/12-identity.md) | Corporate IdP choice, federation across M&A, AAL and WebAuthn, SCIM JML, PAM/PIM, break-glass, IGA and access reviews, CIAM split from workforce. | Workload identity → ch06 §3. mTLS → ch07. Account/sub/project topology bound to IdP groups → ch01 §6. | End-user UX for sign-up flows, marketing-side CRM identity, customer-data-platform (CDP) identity resolution. |
| [`docs/decision-trees.md`](./docs/decision-trees.md) | One-screen synthesis: 12 highest-leverage decisions, decision trees for IaC choice, K8s yes/no, mesh, platform team, multi-region, multi-cloud, ship-a-change, SLO model, cost guardrails, data-store family, tenancy, IdP. | Each tree node cites the owning chapter — that chapter is the source of truth. | Anything not on the highest-leverage list. |
| [`checklist.md`](./checklist.md) | One-page `must`-only PR review card, copy-pasteable into PR templates. | Each line links the owning chapter for rationale. | `should`/`prefer`/`avoid` rules — those live in chapters, not on the card. |
| [`patterns/anti-patterns.md`](./patterns/anti-patterns.md) | Cross-cutting rejected patterns with named replacements. | Subsystem-specific anti-patterns stay in the chapter that owns the subsystem. | One-team-only smells, language-specific anti-patterns. |
| [`patterns/patterns.md`](./patterns/patterns.md) | Cross-cutting positive patterns (how infrastructure ages well). | Same — chapter-specific patterns stay in the chapter. | New-pattern proposals without primary-source backing. |
| [`patterns/references.md`](./patterns/references.md) | Curated primary-source list shared across chapters. | Per-claim citations stay inline in chapters. | Aggregated link-farm "awesome-X" lists. |
| [`glossary.md`](./glossary.md) | Canonical, opinionated definitions for every normatively used term, with primary-source citations where the industry disagrees. | None — glossary is leaf. | Synonym dictionaries, marketing-coined terms without a primary source. |

---

## 2. Cross-cutting concerns and their owner chapter

When a topic surfaces in more than one chapter, exactly one chapter
owns it. The others reference; they do not re-decide.

| Concern | Owner | Why it lives there |
|---|---|---|
| Cost (visibility, allocation, commitments, anomalies) | **ch10** | FinOps Foundation framework; needs telemetry (ch05) and topology (ch01) but the *decisions* are FinOps. |
| Security controls and supply chain | **ch06** | Threat-model-first; SLSA/SBOM/sign/verify is one chain. |
| Workload identity (machine → cloud / machine → machine) | **ch06 §3** | It is a security control with a cryptographic substrate. |
| Workforce / human identity (employees, contractors, admins) | **ch12** | Different threat model, different lifecycle (JML, SCIM, IGA). |
| Reliability (SLI/SLO/error budget, incident response) | **ch09** | Google SRE canon; everything else either feeds it (ch05) or is gated by it (ch03 §7). |
| Observability (instrumentation, OTel, wide events, sampling) | **ch05** | Substrate for SLIs (ch09), audit (ch06 §15), and unit cost (ch10). |
| Network topology, IP plan, private connectivity, DNS | **ch07** | The blast-radius and cost boundary; mesh L7 is ch04 §9. |
| Tenancy / multi-tenant isolation (data) | **ch08 §7** | Decided at the data tier; per-tenant cost allocation is referenced from ch10 §11. |
| DORA / delivery metrics | **ch11 §14** | Platform engineering owns the canonical count; ch03 records, ch11 reports. |
| Service mesh adoption | **ch04 §9** | It is a Kubernetes-runtime decision, not a network-topology one. |
| GitOps definition (OpenGitOps four principles) | **ch01** | Foundation; ch03 and ch04 reference. |
| Schema migrations as deploys | **ch08 §15** | Stateful change is a data-tier concern that ships through ch03's pipeline. |
| Audit log transport | **ch06 §15** | Security control; storage tier is ch05 §7. |
| Account / subscription / project topology | **ch01 §6** | First-class IaC-foundations decision; ch10 §11 binds cost allocation to it. |
| Zero-trust network access (BeyondCorp-style proxies, IAP) | **ch06 §13** | Identity-bound network access is a security control; the *identity* side is ch12. |

---

## 3. Topics intentionally excluded from the whole guide

These are not "we forgot"; they are out of scope. Including them would
either dilute the opinions or require expertise the maintainers do not
claim. Each line names the rationale.

- **Data engineering / lakehouse operations** (dbt, Airflow, Spark
  tuning, Iceberg/Delta/Hudi table-format choice, ELT pipeline
  design). Out: ch08 owns *operating* stateful infrastructure as a
  workload; data-engineering practice is a sibling discipline with
  its own canon.
- **AI/ML training infrastructure** (GPU scheduling at scale, model
  registries, feature stores, training-pipeline orchestrators,
  inference autoscaling). Out: GPU pools appear only as a K8s
  adoption trigger (ch04 §1) and a cost line (ch10); the rest is a
  separate guide that doesn't exist yet.
- **Mobile and frontend backend specifics** (BFF patterns, GraphQL
  federation, push-notification fanout, app-store release
  pipelines). Out: ch03's pipeline patterns generalise, but the
  defaults are server-side.
- **Edge / CDN architecture details** (cache-key design, edge-compute
  programming models, anycast routing internals). Out: ch07 covers
  edge as a *transport* concern; content strategy is product, not
  infra.
- **On-prem-heavy stacks** (VMware, OpenStack, bare-metal lifecycle,
  OS imaging at scale, HCI). Out: SCOPE.md §3 is explicit — defaults
  assume hyperscaler-managed primitives.
- **Regulated-industry control mapping** (FedRAMP Mod/High line items,
  HIPAA Security Rule mapping, PCI-DSS scoping, EU DORA, SOC 2 audit
  evidence collection). Out: ch06 names the underlying control
  families; mapping is downstream and customer-specific.
- **End-user accessibility (a11y) and internationalization (i18n).**
  Out: product concerns, not infrastructure. Mentioned only where
  they touch CDN/edge.
- **Marketing / business analytics stacks** (CDPs, attribution
  pipelines, reverse-ETL into SaaS). Out: data-platform sibling
  discipline.
- **Cost negotiation and procurement tactics.** Out: ch10 covers the
  engineering levers (commitments, rightsizing, scale-to-zero); the
  contract-negotiation craft is finance-and-procurement, not
  engineering.
- **Vendor-specific UI tutorials.** Out by policy
  (CONTRIBUTING.md § What we usually decline).

---

## 4. Opinionated vs balanced

Where the guide picks a side, it says so. Where the industry is
genuinely split and the answer is context-dependent, the chapters
present the trade and refuse to pick. This map names which is which.

**Opinionated (the guide picks a side, deviation needs a written
reason):**

- IaC: one tool per repo, one CLI per state file (ch02 §2).
- Workload identity: no static credentials, OIDC federation by
  default (ch06 §3, ch03 §4.2).
- Supply chain: SBOM + SLSA provenance + signed-and-verified
  promotion by digest (ch03 §3, ch06 §9–11).
- Observability: OpenTelemetry as the instrumentation contract; wide
  structured events as the unit; "three pillars" rejected as a
  design framework (ch05 §1–2).
- Reliability: SLO + error-budget policy as a signed contract; 100%
  is the wrong target (ch09 §1).
- Containers: default to *no* Kubernetes; default to *no* service
  mesh (ch04 §1, §9).
- Data: default to PostgreSQL until a documented reason pushes you
  off it; managed before self-hosted (ch08 §1, §5;
  decision-trees §10).
- FinOps: Inform → Optimize → Operate in that order; commitments
  cover baseline never peak (ch10 §1, §13).
- Identity: one workforce IdP even across M&A; CIAM is a separate
  tenant (ch12 §2, §5).
- Platform engineering: charter on criteria not headcount; "if you
  build it they will come" rejected (ch11 §16, §13).

**Balanced (the guide presents the trade, you pick):**

- Monorepo vs polyrepo (ch03 §6) — sized to org, not dogma.
- Push vs pull deploys (ch03 §1) — pull for mature multi-team K8s,
  push via OIDC for serverless / IaC.
- Helm vs Kustomize vs raw manifests (ch04) — one is preferred per
  context; alternatives kept on the table.
- Spanner / CockroachDB / YugabyteDB inside the distributed-SQL
  family (decision-trees §10) — family is opinionated, vendor inside
  it is not.
- Linkerd vs Istio when a mesh *is* justified (ch04 §9) —
  feature-surface trade.
- Backstage vs CLI-first golden paths (ch11 §9) — discovery problem
  has to be real before Backstage is justified.

If a future PR tries to flip an "opinionated" line to "balanced" — or
vice versa — that change needs the same primary-source rigor as a new
`must` (CONTRIBUTING.md § Sourcing).
