# Infrastructure anti-patterns we reject in review

The 60-odd patterns we say "no" to in design review and PR review across IaC,
Kubernetes, CI/CD, observability, security, networking, data, reliability,
FinOps, and platform engineering — with the failure mode each one produces and
the named replacement.

> Each entry: **smell** → **why it bites** → **fix**, with a link back to the
> chapter that argues the case in depth, and at least one source.

> Companion pages: the 11 chapter drafts under [`../docs/`](../docs/) are the
> long-form argument; this file is the quick-scan reference.

---

## IaC & infrastructure foundations

### **Click-ops and out-of-band fixes against IaC-managed resources**
Hand-edited resources, console "quick fixes", and one-off scripts silently
destroy the value of IaC and create snowflake servers that take months to
reconcile. **Fix:** every change to an IaC-managed resource goes through the
IaC repo via PR; drift is reported, never auto-applied.
[See ch. 01 — Foundations](../docs/01-foundations.md) ·
<https://martinfowler.com/bliki/SnowflakeServer.html> ·
<https://infrastructure-as-code.com/book/>

### **"Self-healing" config-management agents on long-lived hosts**
Scheduled Puppet/Chef/Ansible runs against live mutable hosts are vectors for
untested change in production and reintroduce the snowflakes IaC was meant to
kill. **Fix:** use those tools to *bake images*, then deploy immutable
artifacts via blue/green or rolling replacement.
[See ch. 01 — Foundations](../docs/01-foundations.md) ·
<https://martinfowler.com/bliki/ImmutableServer.html>

### **GitOps for a single cluster with a single team**
Reconciler, CRDs, access model, and dual source-of-truth carry non-trivial
cost; the break-even is roughly more than one cluster or more than one team
sharing a cluster. **Fix:** stay on plain CI-driven `kubectl apply` /
`helm upgrade` until you have the scale that justifies Argo CD or Flux.
[See ch. 01 — Foundations](../docs/01-foundations.md) ·
<https://www.thoughtworks.com/radar/techniques/gitops>

### **Staging environments nobody actually uses**
"Prod-like" environments that engineers don't trust to validate changes are
worse than no staging — they provide false assurance and rot. **Fix:** invest
until staging is trusted (same IaC, same data shape, same deploy path), or
delete it and ship via canary/progressive delivery.
[See ch. 01 — Foundations](../docs/01-foundations.md) ·
<https://infrastructure-as-code.com/book/>

### **Adopting a second IaC tool because the first one is annoying**
Each additional IaC tool roughly doubles the maintenance surface (CLI
versions, state backends, modules, linters, scanners). **Fix:** extend the
existing tool — write a wrapper module or accept the rough edge — unless
there's a workflow it genuinely cannot do.
[See ch. 01 — Foundations](../docs/01-foundations.md) ·
<https://infrastructure-as-code.com/book/>

### **Treating the IaC repo as the ops team's repo**
When application engineers cannot read and PR the infra repo for their own
service, "you build it, you run it" is dead and drift is inevitable. **Fix:**
infra repo is org-wide, with codeowners per directory; the platform team owns
*the modules*, not every consumer's tfvars.
[See ch. 01 — Foundations](../docs/01-foundations.md) ·
<https://martinfowler.com/bliki/InfrastructureAsCode.html>

### **AWS CDK chosen for "future multi-cloud portability"**
CDK synthesizes CloudFormation and is AWS-shaped end-to-end; the abstraction
is over CloudFormation, not over clouds. **Fix:** if multi-cloud is the goal,
pick Terraform/OpenTofu or Pulumi. CDK is acceptable only when the team has
explicitly accepted AWS lock-in.
[See ch. 02 — IaC tooling](../docs/02-iac-tooling.md) ·
<https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html>

### **Inline `provider` blocks inside reusable Terraform modules**
A module that declares its own `provider "aws" { ... }` configuration cannot
be composed (multiple regions, multiple accounts) and hides credential flow.
**Fix:** modules declare `required_providers` only; the root module
configures and passes providers in.
[See ch. 02 — IaC tooling](../docs/02-iac-tooling.md) ·
<https://developer.hashicorp.com/terraform/language/modules/develop/providers>

### **Committing `*.tfstate` (or `.terraform/`) to git**
State contains secrets in plaintext, and concurrent applies without locking
will corrupt it. **Fix:** remote backend with locking (S3+DynamoDB, GCS, AzureRM
with blob lease, Terraform/HCP, OpenTofu state encryption); add the standard
`Terraform.gitignore`.
[See ch. 02 — IaC tooling](../docs/02-iac-tooling.md) ·
<https://developer.hashicorp.com/terraform/language/state> ·
<https://github.com/github/gitignore/blob/main/Terraform.gitignore>

### **Auto-remediation of detected drift**
Auto-applying the result of a `terraform plan` against drift turns the
reconciler into an unreviewed deploy pipeline — every console click becomes a
production change. **Fix:** drift detection alerts a human; remediation is a
reviewed PR.
[See ch. 02 — IaC tooling](../docs/02-iac-tooling.md) ·
<https://spacelift.io/blog/terraform-drift-detection>

### **Using Infracost as budget enforcement**
Infracost estimates from the plan, not actuals; treating its output as the
authoritative budget number hides real spend caused by usage, egress, and
reservations. **Fix:** Infracost in PRs as a *signal*; budget enforcement in
the cloud's native budgets service (AWS Budgets / Azure Cost Management / GCP
Budgets).
[See ch. 02 — IaC tooling](../docs/02-iac-tooling.md) ·
<https://www.infracost.io/docs/> ·
<https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html>

### **`helm template | kubectl apply` as your long-term workflow**
You lose Helm release tracking (history, rollback, hook ordering) and gain
none of Kustomize's clarity. **Fix:** either `helm upgrade --install` (own the
release) or pure Kustomize overlays — pick one per app.
[See ch. 02 — IaC tooling](../docs/02-iac-tooling.md) ·
<https://helm.sh/docs/helm/helm_template/> ·
<https://kubectl.docs.kubernetes.io/references/kustomize/>

---

## CI/CD anti-patterns

### **Starting new pipelines on Jenkins in 2025**
Pipeline-as-code, plugin trust surface, ephemeral runners, and OIDC-based
secret federation all lost to GitHub Actions / GitLab CI / Tekton / Buildkite.
**Fix:** for greenfield, start on a hosted YAML-pipeline product; migrate
Jenkins by attrition, not big-bang.
[See ch. 03 — CI/CD](../docs/03-ci-cd.md) ·
<https://www.thoughtworks.com/radar>

### **Treating `package.json` / `*.csproj` / `requirements.txt` as an SBOM**
Dependency manifests miss transitives, native deps, OS packages, and base-image
content — exactly the surface Log4Shell, xz-utils, and friends lived in.
**Fix:** generate a real SBOM with Syft or `cyclonedx-*` from the *built
artifact* (image, jar, binary), publish it as a build attestation.
[See ch. 03 — CI/CD](../docs/03-ci-cd.md) ·
<https://spdx.dev/> ·
<https://github.com/anchore/syft>

### **"Canary = deploy to one pod and watch logs"**
That's a smoke test, not a canary — it doesn't measure user impact, has no
abort criterion, and "watch logs" doesn't scale past one engineer. **Fix:**
Argo Rollouts / Flagger with an analysis template that compares SLI metrics
against baseline and auto-aborts.
[See ch. 03 — CI/CD](../docs/03-ci-cd.md) ·
<https://flagger.app/> ·
<https://argoproj.github.io/argo-rollouts/>

### **Vanity developer-productivity metrics (PR count, story points, "DX scores")**
DORA research repeatedly shows these don't predict delivery performance or
business outcomes; they create gameable targets. **Fix:** the four DORA
metrics (lead time, deploy frequency, change-fail rate, MTTR) measured from
real telemetry, plus team-reported reliability.
[See ch. 03 — CI/CD](../docs/03-ci-cd.md) ·
<https://dora.dev/>

### **A central platform team that writes every other team's pipeline**
You build a queue and break the autonomy DORA correlates with high delivery
performance. **Fix:** platform team owns reusable workflows / composite
actions / Tekton tasks as a *product*; stream-aligned teams compose them.
[See ch. 03 — CI/CD](../docs/03-ci-cd.md) ·
<https://teamtopologies.com/key-concepts/> ·
<https://dora.dev/>

---

## Containers & Kubernetes anti-patterns

### **Installing a service mesh "for observability"**
Don't install Istio/Linkerd/Consul unless mTLS, L7 traffic shaping, *and*
per-service authz are all required. The operational cost (sidecar lifecycle,
upgrades, debugging extra hops) is real. **Fix:** OTel SDKs + a CNI-level
network policy until you can name the three mesh features you actually need.
[See ch. 04 — Containers & K8s](../docs/04-containers-k8s.md) ·
<https://buoyant.io/service-mesh-manifesto>

### **Adopting Kubernetes for a small team with no platform engineer**
K8s assumes someone owns control-plane upgrades, CNI, CSI, ingress, and a
node-image pipeline. Without that role you're paying the complexity tax twice.
**Fix:** managed PaaS (Cloud Run, App Service, Fly, Render, ECS Fargate) until
you have either the headcount or the workload shape that justifies K8s.
[See ch. 04 — Containers & K8s](../docs/04-containers-k8s.md) ·
<https://www.thoughtworks.com/radar>

### **Modelling your application's domain as Kubernetes CRDs**
You've built a bespoke control plane that only your team can run, debug, or
upgrade. **Fix:** keep CRDs for *infrastructure* abstractions (databases,
queues, certs); model app domain as plain services with a real API.
[See ch. 04 — Containers & K8s](../docs/04-containers-k8s.md) ·
<https://kubernetes.io/docs/concepts/extend-kubernetes/operator/>

---

## Observability anti-patterns

### **Logs as metrics (counting log lines for rates)**
Substring counts in alert queries are slow, expensive, fragile to log-format
changes, and lose dimensionality. **Fix:** emit a counter (Prometheus,
OpenTelemetry); log the event for forensic context, not the count.
[See ch. 05 — Observability](../docs/05-observability.md) ·
<https://charity.wtf/2019/02/05/logs-vs-structured-events/>

### **Buying a turnkey monitoring SaaS before you have an instrumentation strategy**
You'll ship cardinality bombs into a metered backend and discover the cost
when the bill arrives. **Fix:** decide on signals (RED/USE), naming, label
budget, and retention *first*; pick the vendor against that strategy.
[See ch. 05 — Observability](../docs/05-observability.md) ·
<https://www.honeycomb.io/blog/we-shipped-a-major-feature-without-it-being-done>

### **Ticket-noise pages and "auto-resolves in a few minutes" alerts**
Alert fatigue causes on-call attrition, which is the most expensive bug in the
system. **Fix:** every page must be actionable, novel, and tied to a
user-facing SLO; auto-resolving symptoms become dashboards, not pages.
[See ch. 05 — Observability](../docs/05-observability.md) ·
<https://sre.google/sre-book/practical-alerting/> ·
<https://sre.google/workbook/alerting-on-slos/>

### **eBPF as a replacement for app-level instrumentation**
Kernel telemetry has no business semantics — it can tell you a syscall was
slow, not which tenant, route, or feature flag was involved. **Fix:** OTel
SDKs in the app for business signals; eBPF for the layer below them
(network/profile).
[See ch. 05 — Observability](../docs/05-observability.md) ·
<https://www.honeycomb.io/blog/observability-engineering-book>

### **Dashboard-driven debugging**
Dashboards encode the *last* incident's questions, not the next one's; they
become the answer when the question hasn't been asked yet. **Fix:** a
high-cardinality event store you can query ad-hoc; dashboards summarise SLOs,
not investigate.
[See ch. 05 — Observability](../docs/05-observability.md) ·
<https://www.honeycomb.io/blog/observability-engineering-book>

### **High-cardinality IDs as Prometheus labels (and substring-counts as alerts)**
`user_id`, `request_id`, `trace_id` as labels blows up the TSDB; substring
counts in alerts can't survive a log-format change. **Fix:** IDs go on
*traces* and *events*, never on metrics labels; alerts read counters.
[See ch. 05 — Observability](../docs/05-observability.md) ·
<https://prometheus.io/docs/practices/naming/> ·
<https://grafana.com/blog/2022/02/15/what-are-cardinality-spikes-and-why-do-they-matter/>

### **Head-only low-rate sampling (e.g. "1% of all traces")**
You've discarded exactly the errors and slow traces you need; the cheap
happy-path traces dominate your sampled set. **Fix:** tail-based sampling that
keeps all errors and slow spans, plus a small uniform sample for baseline.
[See ch. 05 — Observability](../docs/05-observability.md) ·
<https://opentelemetry.io/docs/concepts/sampling/>

---

## Security & supply-chain anti-patterns

### **Starting security review with a tool**
Buying a scanner before you have a data-flow diagram and trust boundaries
gives you a list of CVEs without the context to triage them. **Fix:** STRIDE
or equivalent threat model first; the tool tells you whether the controls you
chose are present.
[See ch. 06 — Security & supply chain](../docs/06-security-supply-chain.md) ·
<https://shostack.org/books/threat-modeling-book> ·
<https://github.com/cncf/tag-security>

### **Failing builds on Low/Medium CVEs with no fix path**
Alert without a fix path is noise; engineers learn to bypass the gate.
**Fix:** gate on High/Critical *with* available fixes; track unfixed findings
as exceptions with an expiry date and a named owner.
[See ch. 06 — Security & supply chain](../docs/06-security-supply-chain.md) ·
<https://best.openssf.org/>

### **No-fix vuln findings surfaced as alerts forever**
Persistent unactionable findings train the team to ignore the channel.
**Fix:** an exception register with expiry, owner, and compensating control;
alerts only fire when something *changes* (new CVE, expiry approaching, fix
released).
[See ch. 06 — Security & supply chain](../docs/06-security-supply-chain.md) ·
<https://www.oreilly.com/library/view/securing-devops/9781617294136/>

### **A separate "compliance pipeline"**
Two pipelines means the audit evidence doesn't match what shipped. **Fix:**
the normal pipeline produces signed builds, SBOMs, provenance attestations
(SLSA), and audit logs; compliance reads from those, not its own copy.
[See ch. 06 — Security & supply chain](../docs/06-security-supply-chain.md) ·
<https://slsa.dev/spec/> ·
<https://openssf.org/projects/s2c2f/>

---

## Networking anti-patterns

### **Deploying anything real into the cloud provider's default VPC**
Default VPCs have permissive defaults, no IP-plan, and shared blast radius
across accounts. **Fix:** delete or quarantine the default VPC; provision
named VPCs from a Terraform module with documented CIDRs and tags.
[See ch. 07 — Networking](../docs/07-networking.md) ·
<https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html>

### **Packing subnets tighter than /24 to "save space"**
There is no scarcity in RFC 1918, and renumbering a live VPC is a career
event — every peering, firewall rule, and allowlist downstream pins your
CIDRs. **Fix:** /24 minimum per subnet; /16 per VPC; document the IPAM
allocation in a single source of truth.
[See ch. 07 — Networking](../docs/07-networking.md) ·
<https://datatracker.ietf.org/doc/html/rfc1918> ·
<https://docs.aws.amazon.com/vpc/latest/userguide/vpc-subnets-commands-example.html>

### **Corporate user VPN as your security perimeter**
Once an attacker is on the VPN they have lateral access to everything; the
VPN concentrator is also a single point of failure for productivity. **Fix:**
zero-trust access (BeyondCorp / Cloudflare Access / Tailscale) with
per-service identity-aware proxies.
[See ch. 07 — Networking](../docs/07-networking.md) ·
<https://research.google/pubs/beyondcorp-a-new-approach-to-enterprise-security/> ·
<https://www.cloudflare.com/learning/access-management/what-is-zero-trust/>

### **NAT66 in IPv6 networks**
IPv6's model is global addresses + stateful edge firewall; NAT66 imports IPv4's
worst design constraint into a network that didn't have it. **Fix:** unique
global addresses (or ULA for genuinely-internal segments) + edge firewall.
[See ch. 07 — Networking](../docs/07-networking.md) ·
<https://datatracker.ietf.org/doc/html/rfc8200> ·
<https://datatracker.ietf.org/doc/html/rfc4193>

### **Treating a private subnet as a security boundary**
Without IAM and mTLS it's just an obscured boundary — anyone who lands inside
the VPC owns everything. **Fix:** zero-trust between services (mTLS via mesh
or SPIFFE/SPIRE), IAM-scoped data access, network segmentation as defence in
depth not as the only defence.
[See ch. 07 — Networking](../docs/07-networking.md) ·
<https://csrc.nist.gov/publications/detail/sp/800-207/final>

### **Shipping internet-facing HTTP without a WAF "to add later"**
"Later" becomes "never", and the first credential-stuffing wave or layer-7
DDoS proves it. **Fix:** managed WAF (CloudFront/AWS WAF, Azure Front Door,
Cloudflare, GCP Cloud Armor) at the edge with the OWASP CRS turned on from
day one; tune rules per app.
[See ch. 07 — Networking](../docs/07-networking.md) ·
<https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html>

### **Undocumented VPCs and subnets**
An unnumbered subnet is a future incident — nobody knows what runs there or
who owns it. **Fix:** every VPC/subnet has an owner, a purpose, and a CIDR
recorded in the IPAM source of truth; orphaned ones are deleted on a cadence.
[See ch. 07 — Networking](../docs/07-networking.md) ·
<https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html>

---

## Data & state anti-patterns

### **Two flavours of the same engine in the same org**
RDS Postgres + Aurora Postgres + self-hosted Postgres triples the operational
surface (backups, upgrades, extensions, monitoring, on-call expertise) for
zero functional gain. **Fix:** one flavour per engine in the platform
catalog; new flavours require a deprecation plan for the old one.
[See ch. 08 — Data & state](../docs/08-data-state.md) ·
<https://www.oreilly.com/library/view/database-reliability-engineering/9781491925935/> ·
<https://12factor.net/backing-services>

### **Treating replicas as backups**
Read replicas, multi-AZ standbys, and cross-region replicas replicate
*corruption* and *deletes* faithfully. **Fix:** point-in-time recovery +
periodic restore tests on a schedule; "availability" and "durability" are
documented as separate properties with separate proofs.
[See ch. 08 — Data & state](../docs/08-data-state.md) ·
<https://sre.google/sre-book/data-integrity/> ·
<https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html>

### **Multi-primary RDBMS (Galera, MySQL Group Replication, BDR)**
Jepsen has documented sharp edges in these for years; the failure modes
appear under partitions you'll see in production. **Fix:** single-writer
Postgres/MySQL with sync replicas and a documented failover; if you need
multi-writer, pick a system actually designed for it (CockroachDB, Spanner,
YugabyteDB) and budget for the Jepsen review.
[See ch. 08 — Data & state](../docs/08-data-state.md) ·
<https://jepsen.io/analyses>

### **Self-hosted Kafka without a dedicated streaming-platform team**
Kafka operations (broker upgrades, rebalancing, ZK/KRaft, ACLs, MirrorMaker,
schema registry) is a full-time job. **Fix:** managed Kafka (MSK, Confluent
Cloud, Aiven, Event Hubs) until you have an org-level platform team and an
on-call rotation that owns it.
[See ch. 08 — Data & state](../docs/08-data-state.md) ·
<https://www.confluent.io/blog/kafka-fastest-messaging-system/>

### **Application-layer encryption of full database columns "for security"**
Column-level app-side encryption breaks indexability, joins, query planning,
and analytics — usually with no improvement over KMS-at-rest against a
realistic threat model. **Fix:** KMS envelope encryption at rest by default;
field-level encryption only for genuinely sensitive fields with a documented
threat model (PII, payment data).
[See ch. 08 — Data & state](../docs/08-data-state.md) ·
<https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final>

### **Postgres / MySQL / Kafka / Redis / Elasticsearch on Kubernetes for convenience**
StatefulSets, CSI, PV resizing, backup orchestration, and version upgrades
are all subtly broken modes that the managed services have already solved.
**Fix:** managed services unless an ADR cites a hard requirement (air-gap,
regulated region, on-prem) *and* you have named storage-SRE owners *and* a
mature operator (CloudNativePG, Strimzi).
[See ch. 08 — Data & state](../docs/08-data-state.md) ·
<https://cloudnative-pg.io/> ·
<https://strimzi.io/>

---

## Reliability anti-patterns

### **SLO target of 100%**
100% means every dependency must also be 100%, which it isn't, which means
your "SLO" is just a wish. It also leaves no error budget for change. **Fix:**
pick a target the dependency chain and the user can actually feel
(99.9%/99.95%); use the gap as a release-velocity budget.
[See ch. 09 — Reliability](../docs/09-reliability.md) ·
<https://sre.google/sre-book/service-level-objectives/> ·
<https://sre.google/workbook/implementing-slos/>

### **"Operator error" / "human error" as a stated root cause**
You've stopped investigating at the first human in the chain and lost every
systemic lesson the incident had to teach. **Fix:** blameless postmortems
that treat the operator's actions as a *symptom* of the system's affordances;
fix the system that made the wrong action easy.
[See ch. 09 — Reliability](../docs/09-reliability.md) ·
<https://www.etsy.com/codeascraft/blameless-postmortems/> ·
<https://response.pagerduty.com/>

### **Chaos experiments against services with no SLO (or while on-call is over budget)**
You can't tell signal from noise without a baseline, and you punish the
on-call who's already drowning. **Fix:** SLO + error budget *first*; chaos
runs only when budget is healthy and there's a hypothesis.
[See ch. 09 — Reliability](../docs/09-reliability.md) ·
<https://principlesofchaos.org/> ·
<https://www.gremlin.com/community/tutorials/chaos-engineering-the-history-principles-and-practice/>

### **Treating safety as a slogan**
Posters about "safety culture" without changing how you run reviews,
on-calls, and postmortems are theatre. **Fix:** Dekker's "new view" — model
incidents as normal-work-under-pressure, not as deviance; invest in slack,
tools, and training that change the affordances.
[See ch. 09 — Reliability](../docs/09-reliability.md) ·
<https://sre.google/sre-book/table-of-contents/> ·
<https://www.adaptivecapacitylabs.com/blog/>

---

## FinOps anti-patterns

### **Automated kill-switches on production budgets**
Auto-stopping production on a billing threshold causes the outage the
threshold was meant to prevent. **Fix:** prod accounts get *alerts only*; the
runbook documents the human decision tree; non-prod accounts can have
automated stops.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html> ·
<https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-acm-create-budgets>

### **Targeting 100% CPU utilization "for efficiency"**
Above ~80% utilization, queueing latency explodes — you've optimised cost by
breaking the SLO. **Fix:** SLO-driven utilization band (typically 60–75% for
latency-sensitive services); HPA/KEDA scale on the SLI, not raw CPU.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://aws.amazon.com/builders-library/> ·
<https://keda.sh/>

### **Databases, primary brokers, or stateful workloads on spot/preemptible**
Spot interruptions on a stateful node mean a leader election, a re-replication
storm, or data loss. **Fix:** on-demand or reserved capacity for stateful
nodes (with a `nodeSelector` / taint enforcing it); spot for stateless,
restart-tolerant workloads only.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html>

### **Pulling large datasets across regions/clouds "so the team can process them where they work"**
Egress is the most expensive line on the bill and the slowest path; the data
gravity wins. **Fix:** move the *compute* to the data (run the job in the
data's region/cloud); cross-region transfers above a documented threshold
require architecture-review sign-off.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://www.finops.org/framework/> ·
<https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html>

### **Platform team buys reservations/Savings Plans without engineering forecasts**
Commitments are a 1–3 year bet on usage shape; making it without the owning
team's forecast leaves the org overcommitted *and* the engineers unaccountable
for utilization. **Fix:** every commitment purchase has a linked forecast
signed by the owning service team; FinOps team facilitates, doesn't decide
unilaterally.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://www.finops.org/framework/principles/> ·
<https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html>

### **Lift-and-shift to a "cheaper" cloud (or back to on-prem) without a real TCO**
The sticker price ignores engineering FTEs, retraining, lost managed
services, and the migration year itself. **Fix:** TCO model with a 3-year
horizon, engineering-FTE line items, and the cost of the managed services
you're rebuilding; revisit annually.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://a16z.com/the-cost-of-cloud-a-trillion-dollar-paradox/> ·
<https://www.oreilly.com/library/view/cloud-finops-2nd/9781492098348/>

### **Cost dashboards with no owner and no review cadence**
They're decoration — looked at once, never acted on, and stale within a
quarter. **Fix:** every dashboard has an owner field and a documented review
cadence; orphaned dashboards are pruned each quarter.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://www.finops.org/framework/>

### **Weekly cost meetings with no authority to act**
A review forum that can only "raise concerns" is theatre — the spend keeps
growing while everyone files a ticket. **Fix:** every review forum's charter
lists the actions it can take this week (rightsizing budget, kill-switch on
non-prod, SP/RI purchase up to $X); minutes record the action.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://www.finops.org/framework/principles/>

### **Tagging "policy" without enforcement**
Optional tags = no tags; six months in your cost reports can't attribute
30% of spend. **Fix:** SCP / Azure Policy / GCP Org Policy that *denies
create* on missing required tags; CI fails IaC PRs missing them; backfill
sweeps are scheduled.
[See ch. 10 — FinOps](../docs/10-finops.md) ·
<https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html> ·
<https://learn.microsoft.com/azure/cost-management-billing/costs/cost-mgt-best-practices>

---

## Platform-engineering anti-patterns

### **Adopting Backstage in a <30-engineer org (or treating Backstage *as* the platform)**
Backstage is a frontend; it costs platform-team-weeks in plugin upgrades,
auth, and catalog ingestion before it shows value. Below the break-even
headcount it consumes the budget that should have funded the actual paved
roads. **Fix:** ship the paved roads as a CLI / templates / docs first; add
Backstage when there are enough services that a catalog beats a `README`.
[See ch. 11 — Platform engineering](../docs/11-platform-engineering.md) ·
<https://backstage.io/docs/overview/what-is-backstage> ·
<https://www.thoughtworks.com/radar/platforms/backstage>

### **Ivory-tower platform — design without consumer contact**
A platform team that ships features no stream-aligned team asked for accretes
unused surface area and loses budget renewal. **Fix:** weekly office hours,
embedded-engineer rotation, or shadow one stream team per quarter; <50% of
platform commits tied to a consumer issue is the warning sign.
[See ch. 11 — Platform engineering](../docs/11-platform-engineering.md) ·
<https://teamtopologies.com/key-concepts/> ·
<https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams>

### **Mandatory paved road**
Mandating the golden path converts a road into a gatekeeping queue and kills
the only honest signal you have ("are people choosing it?"). **Fix:** compete
on ergonomics, not policy; at least one production team uses a documented
off-ramp; treat that as healthy market feedback.
[See ch. 11 — Platform engineering](../docs/11-platform-engineering.md) ·
<https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/>

### **"We know what they need" — building before validating**
Shipping with no MVP and no signed-up first adopter produces orphaned
features that nobody runs and nobody can deprecate. **Fix:** every shipped
platform feature has a named first adopter who used it within 30 days of
release; otherwise kill or re-scope.
[See ch. 11 — Platform engineering](../docs/11-platform-engineering.md) ·
<https://tag-app-delivery.cncf.io/whitepapers/platforms/>

### **Platform team as an SRE-ticket queue**
SRE break-fix work and productised-platform work have different shapes
(reactive vs. roadmap, per-incident vs. per-release); collapsing them turns
the platform team into an ops team in a platform t-shirt. **Fix:** track
break-fix as a separate workstream capped at ~20% of capacity; the rest is
roadmap product work.
[See ch. 11 — Platform engineering](../docs/11-platform-engineering.md) ·
<https://teamtopologies.com/book> ·
<https://sre.google/sre-book/introduction/>

---

## Cross-cutting

The following themes show up in more than one chapter and are worth naming
explicitly.

### **"We'll add it later" for any control with org-wide blast radius**
WAFs, tagging policies, SBOM/provenance, restore tests, IPAM, and OIDC
federation share a failure mode: every quarter you defer them, the cost of
adoption grows super-linearly because more workloads need retrofitting.
**Fix:** these go in from day one of each new account/cluster/region; the
platform paved road provides them by default so individual teams don't choose.
See chapters [02](../docs/02-iac-tooling.md), [06](../docs/06-security-supply-chain.md),
[07](../docs/07-networking.md), [08](../docs/08-data-state.md),
[10](../docs/10-finops.md).

### **Two of anything where one would do**
Two IaC tools, two pipeline systems, two observability vendors, two flavours
of Postgres, two service meshes, two secrets managers — each "and also" doubles
the surface for upgrades, training, on-call, and audit. **Fix:** explicit
deprecation plan on every "we're adding X alongside Y"; ADR required.
See chapters [01](../docs/01-foundations.md), [02](../docs/02-iac-tooling.md),
[08](../docs/08-data-state.md).

### **Tools chosen before strategy**
Buying a SaaS to "get observability", "get security", "get FinOps", or "get a
platform" before you've decided what signals/controls/decisions you want
produces dashboards nobody reads and bills nobody can defend. **Fix:** write
the one-page strategy first (what we measure, what we gate on, what we
deprecate); pick tools against it.
See chapters [05](../docs/05-observability.md),
[06](../docs/06-security-supply-chain.md), [10](../docs/10-finops.md),
[11](../docs/11-platform-engineering.md).

### **Mandatory anything**
Mandatory pipelines, mandatory paved roads, mandatory dashboards, mandatory
review forums — mandates skip the validation step that would have caught the
fact that the thing being mandated doesn't work. **Fix:** ship as a default;
measure adoption; investigate every off-ramp.
See chapters [03](../docs/03-ci-cd.md), [11](../docs/11-platform-engineering.md).

---

See the chapter drafts under [`../docs/`](../docs/) for the full argument and
the per-chapter `sources.json` files for the curated reading list.
