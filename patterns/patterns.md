# Patterns to follow (infrastructure & platform)

Positive, named patterns distilled from the 11 chapter drafts. Each one is
something the literature, the major cloud providers, and the SRE / platform
community converge on — not just "rules", but architectural and operational
shapes that have a name, a reason, and a known failure mode when you skip
them.

> Companion pages: the chapter drafts (`01-foundations.md` … `11-platform-engineering.md`),
> shared `references.md` per chapter (`*.sources.json`), and the
> `anti-patterns.md` ledger.

Each pattern follows the same shape:

- **What it is** — one or two sentences, including *why it works*.
- **When to use** — the trigger that says "now".
- **Pitfalls** — the way it gets quietly turned into theatre.
- **See** — the chapter to read for the full treatment, plus at least one primary source URL.

---

## Infrastructure as Code

### Reconciliation loop (control-loop infrastructure)

A controller continuously observes desired state (in git or a CRD) and the
actual state of the world, and reduces the diff. Works because convergence is
*continuous* and *idempotent* — the same code applied twice is a no-op, and a
freshly-deleted environment heals itself.
- **When to use:** any system where humans, automation, or providers can all mutate the same resources (clusters, cloud accounts, DNS, IAM).
- **Pitfalls:** auto-remediating drift without a human-reviewed PR — that collapses "what should be running" with "what is running" and loses the audit trail. Reconcilers that only converge from a known-good starting state are toys.
- **See:** ch. 1 *Foundations*; <https://kubernetes.io/docs/concepts/architecture/controller/>, <https://opengitops.dev/>, <https://fluxcd.io/flux/concepts/>.

### GitOps (the four OpenGitOps principles)

Declarative, versioned & immutable, pulled automatically, continuously
reconciled. The git repo is the single source of intent; an in-cluster agent
pulls and applies. Works because the cluster credentials never leave the
cluster and the audit trail is `git log`.
- **When to use:** more than one cluster, more than one team, more than one environment, or any compliance regime that asks "who changed what when".
- **Pitfalls:** calling CI-driven `kubectl apply` "GitOps" (it's push, not pull, and it isn't reconciled). Using long-lived branches per environment instead of directories/overlays.
- **See:** ch. 1, ch. 3 *CI/CD*; <https://opengitops.dev/>, <https://argo-cd.readthedocs.io/>, <https://fluxcd.io/flux/>.

### Immutable server / artifact

Don't mutate running hosts or containers — replace them from a versioned
image. Eliminates configuration drift between machines and turns rollback
into "redeploy the previous digest".
- **When to use:** stateless tiers, all container workloads, AMI-based fleets, edge nodes.
- **Pitfalls:** treating "cattle, not pets" as universal — stateful systems remain pets and need named runbooks and tested restores. Long-running config-management agents that "self-heal" mutable hosts defeat the pattern; rebuild instead.
- **See:** ch. 1; <https://martinfowler.com/bliki/ImmutableServer.html>.

### Blast-radius-sized stack

One state file (or one stack/workspace) per *unit you can afford to lose at
once*. Sized by the worst-case `apply -destroy`, not by lines of HCL.
- **When to use:** every Terraform/OpenTofu/CDK/Bicep root module, every Pulumi stack, every Crossplane composition.
- **Pitfalls:** mega-stacks ("all of prod in one apply") and confetti-stacks (a state file per resource — every change becomes a cross-stack dance).
- **See:** ch. 2 *IaC tooling*; *Terraform: Up and Running* ch. 3, <https://developer.hashicorp.com/terraform/language/state/remote>.

### Versioned, single-purpose modules

Modules are small, single-purpose, declare their own provider requirements
(but don't configure them), and are consumed by *semver tag*, never by
`main`. Composition happens in a root module, not inside a leaf.
- **When to use:** anything you'd reuse across two or more environments.
- **Pitfalls:** the "kitchen-sink module" with 80 inputs that nobody reads; inline `provider` blocks inside reusable modules; floating `ref=` in module sources.
- **See:** ch. 2; <https://developer.hashicorp.com/terraform/language/modules/develop/structure>, <https://www.terraform-best-practices.com/code-structure>.

### Plan-as-the-review-artifact

The PR comment is a `terraform plan` (or equivalent) attached by CI;
reviewers approve the plan, not just the diff. Apply runs from CI against the
merged commit; prod requires a different human to approve the apply than
approved the PR.
- **When to use:** any IaC repo with more than one contributor.
- **Pitfalls:** auto-apply on merge for prod; pasting un-redacted plan output into public PR comments (plans contain secrets).
- **See:** ch. 2; <https://www.runatlantis.io/docs/>, <https://docs.env0.com/docs/plan-and-apply>.

### Policy-as-code gate

OPA/Rego, Sentinel, Kyverno, or Gatekeeper express org-specific guardrails
(no public S3, no `*` IAM, no privileged pods, blast-radius limits) that run
in CI *and* at admission. Same bundle in both places — drift is impossible.
- **When to use:** the moment you have more than one IaC repo or more than one cluster.
- **Pitfalls:** policy-only-in-CI (cluster admins bypass it) or policy-only-at-admission (PRs land green and fail at deploy).
- **See:** ch. 2, ch. 6 *Security*; <https://www.openpolicyagent.org/>, <https://kyverno.io/docs/>, <https://www.conftest.dev/>.

---

## CI/CD & Supply chain

### Pull-based deploy (env repo + reconciler)

CI builds and signs an image; a separate environment repo references images
by digest; an in-cluster reconciler (Argo CD, Flux) deploys. CI never holds
cluster credentials. Promotion = a PR in the env repo.
- **When to use:** Kubernetes workloads, especially across multiple clusters or teams.
- **Pitfalls:** referencing images by mutable tag (`:latest`, `:prod`) — you lose reproducibility and signature-binding. Letting the application repo also write to the env repo on merge defeats the separation.
- **See:** ch. 3; <https://argo-cd.readthedocs.io/>, <https://fluxcd.io/flux/>.

### OIDC federation (no long-lived cloud creds in CI)

CI workflows mint a short-lived cloud token via OIDC, with trust scoped to
`repo + branch + environment + workflow`. AWS IAM roles, Azure federated
credentials, GCP Workload Identity Federation all support it.
- **When to use:** every CI pipeline that touches a cloud account, including preview environments.
- **Pitfalls:** trust policies that match `repo:*` (any branch can assume prod). Forgetting that `pull_request_target` and `workflow_run` run with privileges from the *target* repo — review them like deploy jobs.
- **See:** ch. 3, ch. 6; <https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect>, <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html>.

### SBOM + signed-provenance chain (SLSA + Sigstore)

Every release artifact ships an SBOM (SPDX or CycloneDX) generated at build
time, a SLSA provenance attestation (target L3), and a cosign keyless
signature bound to the CI's OIDC identity. Verification at admission pins
the *signer identity*, not "has a signature".
- **When to use:** every container, helm chart, and release binary that reaches production.
- **Pitfalls:** treating `package.json` as an SBOM (misses transitives, native deps, base image); shipping SBOMs nobody scans; verifying only in CI and not at the cluster.
- **See:** ch. 3, ch. 6; <https://slsa.dev/spec/v1.0/>, <https://docs.sigstore.dev/>, <https://spdx.dev/specifications/>.

### Pin actions/images by digest

Third-party actions to a full commit SHA; container images to `@sha256:…`.
Tags are mutable — the `tj-actions/changed-files` compromise (March 2025)
weaponised exactly that. Renovate/Dependabot bumps the SHA with the
human-readable tag preserved in the PR body.
- **When to use:** every workflow, every image reference in env manifests.
- **Pitfalls:** "we'll pin later" (the supply chain attack happens before later); pinning the action but not its transitive `uses:` calls.
- **See:** ch. 3; <https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions>, <https://www.stepsecurity.io/blog/harden-runner-detection-tj-actions-changed-files-action-is-compromised>.

### Progressive delivery (canary/blue-green with automated analysis)

Argo Rollouts or Flagger ships new versions to a slice of traffic, runs
metric-based analysis (latency, error rate, custom SLI), and promotes or
rolls back automatically. "Deploy to one pod and watch logs" is a smoke
test, not a canary.
- **When to use:** user-facing services with measurable SLIs and rollback paths.
- **Pitfalls:** canarying without an automated rollback; analysis windows shorter than the failure mode you care about.
- **See:** ch. 3; <https://argo-rollouts.readthedocs.io/>, <https://docs.flagger.app/>.

### Feature flags decouple deploy from release

Ship dark, release with a flag flip. Inverts the risk profile: deploys
become routine, releases become reversible without a redeploy.
- **When to use:** any user-visible change that is risky to revert via redeploy (schema-coupled features, billing, auth flows).
- **Pitfalls:** flags that never get cleaned up become permanent dead branches; flags as a substitute for testing.
- **See:** ch. 3; <https://openfeature.dev/specification/>.

### Trunk + merge queue

Trunk-based development with required status checks, no force-push, and a
merge queue (or merge train) once the repo has more than ~5 active
contributors. Avoids the "green PR, red main" race.
- **When to use:** any repo that deploys to production from `main`.
- **Pitfalls:** long-lived feature branches with the same protections — doesn't help, it just delays the integration cost.
- **See:** ch. 3; <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue>.

---

## Containers & Kubernetes

### Distroless + non-root + read-only root FS

Runtime image is distroless or Wolfi/Chainguard with no shell or package
manager; container runs as a non-root UID with a read-only root filesystem;
all capabilities dropped, seccomp `RuntimeDefault`, no privilege escalation.
Reduces the post-exploitation surface to "what your binary can do" instead
of "what a shell with `apt` can do".
- **When to use:** every production container.
- **Pitfalls:** distroless without multi-stage build (you ship the compiler too); read-only rootfs without `emptyDir` for `/tmp` (apps that write logs locally crash).
- **See:** ch. 4, ch. 6; <https://github.com/GoogleContainerTools/distroless>, <https://kubernetes.io/docs/concepts/security/pod-security-standards/>.

### Pod Security Admission "restricted" by default

Enforce the `restricted` Pod Security Standard at the namespace level. No
`privileged`, no `hostNetwork`/`hostPID`/`hostPath`, drop ALL caps, seccomp
`RuntimeDefault`. Anything that needs more lives in a clearly-named
namespace with a documented exception.
- **When to use:** every workload namespace, day one.
- **Pitfalls:** turning it on cluster-wide *after* workloads exist — turn it on in `warn`+`audit` mode first, then `enforce`.
- **See:** ch. 4; <https://kubernetes.io/docs/concepts/security/pod-security-admission/>.

### Default-deny NetworkPolicy

Every namespace ships with a deny-all NetworkPolicy; allow-lists are
explicit. Forces the egress and ingress requirements of every service to be
*declared*, which is also the documentation.
- **When to use:** every multi-tenant or PCI/HIPAA-adjacent cluster, which in practice is every cluster.
- **Pitfalls:** writing the deny-all and forgetting the allow for kube-dns; policies that only cover ingress (egress to the internet stays open).
- **See:** ch. 4, ch. 7 *Networking*; <https://kubernetes.io/docs/concepts/services-networking/network-policies/>.

### Requests always, memory limits yes, CPU limits usually no

Set CPU and memory *requests* on every container (the scheduler needs them).
Set a memory *limit* (rely on OOM-kill rather than node exhaustion). For
latency-sensitive services, drop the CPU limit — CFS throttling at the
limit causes p99 spikes that look like "slow code".
- **When to use:** every Deployment/StatefulSet.
- **Pitfalls:** copying CPU limits from a tutorial; running VPA in auto-update mode on the same metric HPA scales on (they fight).
- **See:** ch. 4; <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>, <https://erickhun.com/posts/kubernetes-faster-services-no-cpu-limits/>.

### PDB + topology spread for every multi-replica workload

A PodDisruptionBudget plus topology-spread constraints across zones.
Without both, a node drain or a zonal failure can take all replicas at once
even though you "have three".
- **When to use:** anything with `replicas > 1`.
- **Pitfalls:** PDB `maxUnavailable: 0` (blocks node maintenance forever); spread constraints with `whenUnsatisfiable: DoNotSchedule` and not enough zones.
- **See:** ch. 4; <https://kubernetes.io/docs/concepts/workloads/pods/disruptions/>, <https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/>.

### Scale-to-zero for spiky workloads (KEDA / Cloud Run / Functions)

Event-driven scaling on real signals — queue depth, cron, HTTP RPS — down
to zero when idle. Pairs with on-demand floors for system pods (see
"dual-pool spot" below).
- **When to use:** async workers, batch, low-traffic internal services, preview environments.
- **Pitfalls:** adopting FaaS without measuring cold-start in your p99 budget; scaling stateful workloads to zero (connection storms on warm-up).
- **See:** ch. 4, ch. 10 *FinOps*; <https://keda.sh/>.

### Workload identity (no static cloud creds on pods)

Pods assume cloud roles via IRSA / EKS Pod Identity / GKE Workload Identity
/ AKS Workload Identity. Trust is bound to a Kubernetes ServiceAccount in a
specific namespace.
- **When to use:** every pod that talks to a cloud API.
- **Pitfalls:** mounting an access key as a Secret "just for now"; trust policies that match `system:serviceaccount:*:default`.
- **See:** ch. 4, ch. 6; <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html>, <https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity>.

---

## Observability

### Wide structured events as the primary signal

Instrument once with high-cardinality, structured events; treat metrics,
logs, and traces as projections of those events, not as three independent
"pillars". Lets you ask new questions of a running system without shipping
code — the working definition of observability.
- **When to use:** every new service, and any time you're rewriting an old one's instrumentation.
- **Pitfalls:** the three-pillar mindset that forces every team to instrument the same fact three times; storing events in a metrics TSDB (cardinality explodes).
- **See:** ch. 5; <https://www.oreilly.com/library/view/observability-engineering/9781492076438/>, <https://charity.wtf/2018/02/19/metrics-not-the-observability-droids-youre-looking-for/>.

### OpenTelemetry SDK + Collector

Apps emit OTLP via OpenTelemetry SDKs; a Collector between apps and backends
handles batching, redaction, sampling, and backend swapping. No vendor SDK
in application code.
- **When to use:** any greenfield service; any retro-fit where you control the build.
- **Pitfalls:** custom trace headers (use W3C `traceparent`); auto-instrument *and* hand-instrument the same boundary (duplicate spans).
- **See:** ch. 5; <https://opentelemetry.io/docs/specs/otel/>, <https://opentelemetry.io/docs/collector/>.

### RED + USE + four golden signals

RED (rate, errors, duration) per service endpoint; USE (utilization,
saturation, errors) per resource; the four golden signals (latency,
traffic, errors, saturation) at the edge. Three lenses, one mental model:
"what is this *thing* doing, and is it healthy?"
- **When to use:** every service dashboard and every infrastructure dashboard.
- **Pitfalls:** dashboards full of CPU graphs and zero RED metrics; alerting on USE saturation instead of user-visible RED errors.
- **See:** ch. 5; <https://thenewstack.io/monitoring-microservices-red-method/>, <https://www.brendangregg.com/usemethod.html>, <https://sre.google/sre-book/monitoring-distributed-systems/>.

### Multi-window, multi-burn-rate SLO alerts

Alert on user-visible SLO burn, not infrastructure metrics. Use two
windows: fast (5m+1h burn ≥ 14.4) for "we'll exhaust the monthly budget in
2 days", slow (30m+6h burn ≥ 6) for "we're slowly chewing through the
budget". Codify with sloth/pyrra/OpenSLO so docs and Prometheus rules can't
drift.
- **When to use:** every paging alert on a user-facing service.
- **Pitfalls:** pages on raw CPU; single-window burn-rate alerts that flap; SLO targets of 100% (no error budget = no room to deploy).
- **See:** ch. 5, ch. 9 *Reliability*; <https://sre.google/workbook/alerting-on-slos/>, <https://www.honeycomb.io/blog/the-math-behind-slo-burn-alerts>.

### Tail-based sampling at trace granularity

Sample whole traces or none of them, in the Collector, after the trace is
complete. Always keep 100% of errors and slow traces; probabilistic
baseline for the rest. Head-based low-rate sampling discards exactly the
errors you need.
- **When to use:** any service with more than ~100 traces/sec where 100% ingest is too expensive.
- **Pitfalls:** sampling per-span (orphans the tree); sampling at the SDK before the error is known.
- **See:** ch. 5; <https://opentelemetry.io/docs/concepts/sampling/>, <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>.

---

## Security & Supply chain

### BeyondCorp / Zero-Trust posture

The network is not the perimeter. Authenticate and authorize every request;
mTLS service-to-service; identity-aware proxy in front of internal apps
instead of a flat VPN; hardware-backed WebAuthn for human production
access. Workloads on private subnets, no public IPs by default.
- **When to use:** day one, before the VPN sprawls.
- **Pitfalls:** treating "private subnet" as a security boundary; SMS or TOTP-only MFA on prod paths.
- **See:** ch. 6, ch. 7; <https://research.google/pubs/pub43231/>, <https://csrc.nist.gov/publications/detail/sp/800-207/final>.

### Threat model in the repo (STRIDE / LINDDUN)

Every prod system has a data-flow diagram and a STRIDE (or LINDDUN for PII)
threat model committed alongside the code, reviewed on architectural
change. The review starts with the diagram, not with a tool.
- **When to use:** every new service; on any change that crosses a trust boundary.
- **Pitfalls:** threat-modelling once at design, never again; outsourcing it to a security team that doesn't ship the code.
- **See:** ch. 6; <https://shostack.org/books/threat-modeling-book>, <https://www.threatmodelingmanifesto.org/>.

### Secrets in a managed store, never in VCS

Vault / cloud KMS / Secret Manager, or SOPS-encrypted in git. Every secret
has a rotation policy enforced by the *store*, not a calendar reminder.
Workloads consume via External Secrets / CSI / SDK, not env vars baked into
images.
- **When to use:** every secret, including dev and "throwaway" ones.
- **Pitfalls:** committing `.env` "just for local"; rotating quarterly by hand and calling it a policy.
- **See:** ch. 6; <https://developer.hashicorp.com/vault/docs>, <https://external-secrets.io/>, <https://github.com/getsops/sops>.

### Runtime detection (Falco / Tetragon)

Runtime threat-detection on every prod node, with rules tuned and routed.
Catches what build-time scanners can't — xz-style trigger code,
behavioural threats, post-exploitation activity. "Shifted left" is not a
substitute.
- **When to use:** every production cluster.
- **Pitfalls:** installing it and never tuning the alerts (page fatigue); routing alerts to a security mailbox no one reads.
- **See:** ch. 6; <https://falco.org/docs/>, <https://tetragon.io/>.

### Compliance-as-a-byproduct

Don't build a separate compliance pipeline. Produce the evidence — signed
builds, SBOMs, audit logs, change tickets, restore-test results — from the
normal delivery and operation pipeline. Auditors get artifacts, engineers
keep their flow.
- **When to use:** any regulated environment (SOC 2, ISO 27001, FedRAMP, PCI, HIPAA).
- **Pitfalls:** a "compliance team" that exports CSVs from the real pipeline into a parallel system of record.
- **See:** ch. 6; <https://csrc.nist.gov/publications/detail/sp/800-218/final>, <https://slsa.dev/spec/v1.0/>.

---

## Networking

### Landing zone + hub-and-spoke

Adopt a multi-account/subscription/project landing zone (AWS Control Tower,
Azure CAF landing zones, GCP organization with folders) from day one. A
central connectivity account/subscription owns the hub; spokes attach via
Transit Gateway / Virtual WAN / Network Connectivity Center.
- **When to use:** the moment you have more than one VPC or one team — i.e. immediately.
- **Pitfalls:** a mesh-peering topology that grows to N² connections; putting workloads in the connectivity account.
- **See:** ch. 7; <https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html>, <https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/>.

### Central IPAM (no overlapping CIDRs, ever)

A written, central IP plan: /16 per VPC, /24 subnets, RFC 6598
(`100.64.0.0/10`) reserved for Kubernetes pod CIDRs and overlay networks.
Renumbering is a career event — plan once.
- **When to use:** before the first VPC.
- **Pitfalls:** packing subnets to /27 to "save space" (RFC 1918 has 17M addresses, scarcity is imaginary); using `10.0.0.0/16` as your default (it collides with everything).
- **See:** ch. 7; <https://datatracker.ietf.org/doc/html/rfc1918>, <https://datatracker.ietf.org/doc/html/rfc6598>.

### PrivateLink / Private Endpoint / PSC for PaaS

Terminate every supported managed service over PrivateLink-style endpoints;
disable the public endpoint. Unidirectional, no CIDR coupling, no public
IP.
- **When to use:** every PaaS that supports it (most of S3, RDS, Storage Account, Cloud SQL, Snowflake, Databricks, etc.).
- **Pitfalls:** keeping the public endpoint "for emergencies"; forgetting that DNS still resolves to the public IP without private DNS zones.
- **See:** ch. 7; <https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html>, <https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview>.

### Centralized egress with FQDN allow-list

Production egress flows through a hub firewall with FQDN-based policy.
NAT-only is dev-grade — it gives you "any TCP/443 to anywhere" which is
not policy.
- **When to use:** every production VPC.
- **Pitfalls:** allow-list maintained by ticket; bypass routes for "the one team that needs it".
- **See:** ch. 7; <https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/network-security>.

---

## Data & State

### Managed-by-default for stateful systems

Default to a managed service for Postgres, MySQL, Redis, Kafka, object
storage, and search. Self-hosting requires a *written exception* and a
storage-SRE function. The cost of running a database well is dominated by
the backup, restore, failover, and upgrade story — that's exactly what the
managed service sells you.
- **When to use:** every system of record.
- **Pitfalls:** running the same engine three ways in one org (RDS Postgres + Aurora + self-hosted); self-hosting Kafka without a streaming team.
- **See:** ch. 8; <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html>, <https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview>.

### 3-2-1 backups + tested restore drill

Live DB + same-region snapshots + cross-account, cross-region, object-locked
off-site copy *written by a separate identity*. Run a timed end-to-end
restore drill quarterly (monthly for tier-0). Dashboard "time since last
successful restore" and page when it exceeds policy. **A backup you have
not restored is a hope.**
- **When to use:** every system of record, every audit log, every compliance bucket.
- **Pitfalls:** treating multi-AZ standbys or read replicas as backups; backups encrypted by the same key the workload identity controls (one IAM mistake = no backup).
- **See:** ch. 8; <https://sre.google/sre-book/data-integrity/>, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html>.

### Expand-and-contract schema migrations (parallel change)

Never ship a migration that requires app and schema to deploy atomically.
Add the new column/table, dual-write, backfill, switch reads, drop the old.
Every step is independently deployable and reversible.
- **When to use:** every schema change to a system of record.
- **Pitfalls:** "small" non-backwards-compatible changes that break the previous app version mid-deploy; using a migration tool but applying DDL by hand in prod.
- **See:** ch. 8; <https://martinfowler.com/bliki/ParallelChange.html>, <https://atlasgo.io/docs>.

### Idempotent consumers (exactly-once is a fairy tale)

Every consumer of a queue/stream is idempotent; messages carry an
idempotency key; the consumer dedupes on `(key, handler)`. Don't rely on
broker "exactly-once" semantics — they are at-least-once with vendor-
specific caveats.
- **When to use:** every async handler, every webhook receiver, every retried mutation.
- **Pitfalls:** dedup state with no TTL (unbounded growth); idempotency keyed on payload hash without including the operation.
- **See:** ch. 8, ch. 9; <https://www.ics.uci.edu/~cs223/papers/cidr07p15.pdf>, <https://kafka.apache.org/documentation/#operations>.

### One database per service (no shared schemas)

A service owns its schema. Other services read via API or via a
materialized read model, never via cross-service `JOIN`. Without this,
release cadences couple and "the database is the integration" returns.
- **When to use:** every service in a microservices estate; the moment two teams want to deploy the same database independently.
- **Pitfalls:** shared "lookup" tables that quietly become the integration surface; ETLs that read the OLTP store directly and break on every migration.
- **See:** ch. 8; <https://12factor.net/backing-services>.

---

## Reliability

### SLI/SLO + error-budget policy

Every user-facing service ships with at least one SLI (a ratio of good-to-
valid events on a real user journey), an SLO target the user can feel, and
a *written* error-budget policy that says what happens when the budget is
spent (freeze risky work, prioritize reliability, etc.). Without the
policy, an SLO is decoration.
- **When to use:** every service that has users, internal or external.
- **Pitfalls:** SLOs on infrastructure metrics; targets of 100%; an SLO with no consequence when missed.
- **See:** ch. 5, ch. 9; <https://sre.google/workbook/implementing-slos/>, <https://www.oreilly.com/library/view/implementing-service-level/9781492076803/>.

### Resilience trio: timeout + jitter-backoff retry + idempotency key

Every network call has an explicit timeout *shorter than the caller's*.
Retries use exponential backoff with **full jitter** and a per-client retry
budget. Retried mutations carry an idempotency key. The three are a set —
any two without the third is a footgun.
- **When to use:** every cross-process call, every SDK wrapper, every outbound integration.
- **Pitfalls:** infinite waits ("the SDK has a default"); equal-jitter backoff that still synchronizes thundering herds; retries on POSTs without idempotency keys (double-charged customers).
- **See:** ch. 9; <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>.

### Bulkhead + load-shed at the edge

Bulkhead resources (connection pools, queues, thread pools) per
*dependency* so a misbehaving downstream can't drown the rest. Shed load
at the edge before queues build inside.
- **When to use:** any service with more than one downstream dependency.
- **Pitfalls:** one shared pool sized for the happy path; load-shedding that drops the wrong traffic class (you shed health checks first).
- **See:** ch. 9; *Release It!* (Nygard).

### Blameless postmortem with timeline + action items

Mandatory for SEV1/SEV2 and any near-miss touching auth/data/budget.
Published within 5 business days. Timeline captured in UTC *during* the
incident. Action items have an owner, a ticket, a due date, and are tagged
*prevent* vs *mitigate*. "Operator error" is never a root cause.
- **When to use:** every qualifying incident, no exceptions.
- **Pitfalls:** postmortems that re-litigate decisions instead of learning from them; action items with no owner; private postmortems that no one outside the team reads.
- **See:** ch. 9; <https://www.etsy.com/codeascraft/blameless-postmortems/>, <https://response.pagerduty.com/>.

### Game-day / chaos with hypothesis and abort

Chaos experiments declare a hypothesis ("the system stays within SLO when
AZ-A is unreachable"), a blast-radius, and an abort condition. Run in
production once safe. Never run against a service with no SLO or while
on-call is over the page budget.
- **When to use:** quarterly, per tier-1 service; before every claimed-but- untested DR capability.
- **Pitfalls:** chaos for chaos's sake; experiments that nobody analyses afterwards.
- **See:** ch. 9; <https://principlesofchaos.org/>, <https://www.oreilly.com/library/view/chaos-engineering/9781492043850/>.

---

## FinOps

### FinOps loop: Inform → Optimize → Operate

Adopt the FinOps Foundation framework explicitly. Cross-functional (eng +
finance + product), continuous, with maturity tracked per workload
(Crawl/Walk/Run) — not uniformly across the org.
- **When to use:** the moment cloud spend has a budget owner.
- **Pitfalls:** "the platform team will fix the bill"; finance-only dashboards that engineers never see.
- **See:** ch. 10; <https://www.finops.org/framework/>, <https://focus.finops.org/>.

### Tagging-as-code (FOCUS-aligned)

Required tags on every cloud resource — `cost-center`, `service`, `env`,
`owner`, `data-classification` — enforced in IaC at admission, not at
cleanup. Standardize cross-cloud cost data on the FOCUS specification.
- **When to use:** day one of cloud usage; retrofit before chargeback.
- **Pitfalls:** "optional" tags (= no tags); a tagging policy with no enforcement; bespoke ETL on raw billing exports per cloud.
- **See:** ch. 10; <https://focus.finops.org/>.

### Dual-pool spot (on-demand floor + diversified spot bulk)

In production Kubernetes: an on-demand pool sized for system + stateful
workloads, a spot pool diversified across ≥3 instance families and ≥3 AZs
for stateless. Eviction is correlated within a pool, so diversity is the
hedge.
- **When to use:** any K8s workload with a meaningful stateless tier and more than one AZ.
- **Pitfalls:** a single spot pool ("90% savings until the AZ evicts and you're at zero"); running databases or primary brokers on spot.
- **See:** ch. 10; <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html>, <https://karpenter.sh/docs/>.

### Laddered commitments (SP/CUD)

Stagger 1y/3y commitments so they expire continuously, not all at once.
Target 60–80% coverage on steady-state compute, never 100%. Prefer
flexible Compute Savings Plans / flexible CUDs over family-locked RIs
unless the family is genuinely fixed.
- **When to use:** once you have ≥6 months of usage data.
- **Pitfalls:** 100% coverage (you pay for idle when you scale down); 3-year RI on a workload you'll rewrite in 12 months.
- **See:** ch. 10; <https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html>, <https://www.vantage.sh/blog/reserved-instance-ladder-strategy>.

### Unit cost as the headline metric

The headline engineering metric is *unit cost* — $/order, $/active user,
$/inference — not the absolute monthly bill. The bill goes up as the
business grows; unit cost reveals whether you're getting more efficient.
- **When to use:** every product/service team that owns a P&L line.
- **Pitfalls:** unit-cost denominators nobody trusts (which "active user"?); cost dashboards with no owner and no review cadence (delete them).
- **See:** ch. 10; <https://www.oreilly.com/library/view/cloud-finops-2nd/9781492098348/>.

### Budget kill-switch (non-prod only)

Every account/subscription/project has a budget with thresholds (50/80/100%)
and a documented action per threshold. At 100% in non-prod, automatically
scale-to-zero or stop dev/test. **Never** put automated kill-switches on
production budgets — prod gets paged and human-decided.
- **When to use:** every non-prod account; budget alerts (not actions) on prod.
- **Pitfalls:** budget alerts with no recipient; a kill-switch on a shared dev account that takes out 30 engineers' work.
- **See:** ch. 10; <https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html>, <https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-acm-create-budgets>.

---

## Platform engineering

### Platform-as-a-product

The platform has named internal customers, a public roadmap, SLOs to
consumers, a deprecation policy, and a feedback loop. Validate demand
before building — no feature ships without an identified consumer team
that has agreed to adopt it. Treat it exactly like an external product.
- **When to use:** the moment a platform team exists.
- **Pitfalls:** ivory-tower platform built without consumer contact; "we know what they need" features with no first adopter.
- **See:** ch. 11; <https://tag-app-delivery.cncf.io/whitepapers/platforms/>, <https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams>.

### Golden paths (paved roads, opt-in)

Provide opinionated, low-friction defaults for the common cases — service
templates, CI pipelines, observability wiring, secrets, deploy. The path
is *opt-in*. The moment it becomes mandatory, it stops being a road and
becomes a queue.
- **When to use:** as soon as ≥3 teams are doing the same thing three different ways.
- **Pitfalls:** mandatory paved roads (= gatekeeping queue); paved roads that are *higher* friction than rolling your own (engineers route around).
- **See:** ch. 11; <https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/>.

### Self-service via reconciler (not tickets)

Consumers declare what they want in their own repo (a `Score` spec, a
Crossplane claim, a workload CRD), and a platform controller reconciles
it. "Raise a ticket" is not self-service.
- **When to use:** any platform capability requested more than ~once a month.
- **Pitfalls:** self-service that still requires a human approval step for the common case; bespoke abstractions that re-invent Kubernetes APIs badly.
- **See:** ch. 11; <https://score.dev/docs/>, <https://docs.crossplane.io/latest/concepts/compositions/>.

### Team Topologies + Inverse Conway Maneuver

Organise infra work around the four team types (stream-aligned, platform,
enabling, complicated-subsystem) and the three interaction modes
(collaboration, X-as-a-service, facilitating). Shape the org so the
*desired* architecture is the easy one to build — Conway's law is going
to win either way.
- **When to use:** any reorg, any new platform team, any time architecture and org chart are visibly mismatched.
- **Pitfalls:** inventing a fifth team type; calling a wall-throwing ops team a "platform team"; embedding SREs as a permanent crutch instead of an enabling team that leaves.
- **See:** ch. 1, ch. 11; <https://teamtopologies.com/key-concepts/>, <http://www.melconway.com/Home/Committees_Paper.html>.

### Measure platforms by consumers' DORA, not platform output

A platform is successful when its consuming stream-aligned teams have
better DORA metrics (deploy frequency, lead time, change-failure rate,
MTTR) and high adoption of platform features — *not* when the platform
team has shipped many features or closed many tickets.
- **When to use:** every platform-team OKR cycle.
- **Pitfalls:** dashboards of platform-team velocity; adoption rates that aren't tied to a consumer outcome.
- **See:** ch. 3, ch. 11; <https://dora.dev/>, <https://itrevolution.com/product/accelerate/>.

### Backstage (or equivalent) once you need a software catalog

Adopt Backstage as the default IDP framework once the org needs a software
catalog (~50+ engineers, or ~30+ services). The three load-bearing
primitives are Catalog + Templates + TechDocs. Below ~30 engineers, don't
bother — the catalog has nothing to catalog.
- **When to use:** at the size threshold above; pick build/buy/adopt (Backstage / Roadie / Port / Cortex) deliberately.
- **Pitfalls:** treating Backstage as the platform itself; building Backstage from scratch when a hosted option fits.
- **See:** ch. 11; <https://backstage.io/docs/overview/what-is-backstage>, <https://internaldeveloperplatform.org/>.

---

## Cross-cutting

### "You build it, you run it"

Teams that build a service operate it; on-call and operability are part of
the team that ships the code, not a separate ops group. The original
Vogels-at-Amazon framing (2006), and the prerequisite for every other
pattern on this page — without operational ownership, SLOs, postmortems,
golden paths, and FinOps all degenerate into theatre.
- **When to use:** day one. Retrofit by moving the pager, not the org chart.
- **Pitfalls:** "embedded SRE" that is permanent (it should be temporary, enabling, with an exit); on-call without compensation or a rotation of ≥6 humans.
- **See:** ch. 1, ch. 9, ch. 11; <https://queue.acm.org/detail.cfm?id=1142065>, <https://teamtopologies.com/key-concepts/>.
