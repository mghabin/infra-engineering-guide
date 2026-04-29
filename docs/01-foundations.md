# Infrastructure Engineering — Chapter 01: Foundations

This chapter sets the philosophical floor the rest of the guide stands on:
what "infrastructure as code" actually means, the control-loop mental model
that unifies Kubernetes / Terraform / GitOps, and the cultural prerequisites
without which the tooling buys nothing. Tool selection (OpenTofu vs Terraform
vs Pulumi vs CDK vs Crossplane) is deferred to chapter 02; this chapter argues
for the principles those tools must satisfy.

> Conventions, mirroring the sibling .NET guide:
> **must** = load-bearing, ≥2 independent sources required.
> **should** = strong default; deviate only with a written reason.
> **prefer** = taste call backed by evidence.
> **avoid** = explicitly rejected; replacement named.

> Vendor-neutrality note: HashiCorp became an IBM business unit in 2025
> following IBM's April 2024 acquisition announcement, and Terraform was
> relicensed from MPL-2.0 to BUSL-1.1 in August 2023. Where this chapter
> previously leaned on `hashicorp.com` as a neutral primary, it now pairs
> those citations with vendor-neutral sources (OpenTofu under the Linux
> Foundation, CNCF specs, Morris).

---

## 1. IaC PRINCIPLES

**[IaC PRINCIPLES] [must] All infrastructure is defined as code in version control; nothing is created by clicking in a console.**
Click-ops produces snowflake servers — non-reproducible, un-auditable, and
unsafe to change. Definition files in a VCS give reproducibility, audit trail,
review, and rollback for free.
- Sources: Kief Morris, *Infrastructure as Code* (3rd ed, O'Reilly 2024), Ch. 1–2; Martin Fowler, *InfrastructureAsCode*, https://martinfowler.com/bliki/InfrastructureAsCode.html ; OpenTofu docs, *Introduction*, https://opentofu.org/docs/intro/ .
- Concrete check: pick any production resource at random and ask "where is the source of truth in git?". If the answer is "the console", the practice is failing. CI must reject changes that bypass the IaC repo.

**[IaC PRINCIPLES] [must] Definitions express *desired state* and converge to it safely on re-run.**
The load-bearing property is desired-state semantics: the code says what
should exist, a reconciler computes the diff, and applying the same code
twice converges. Most teams get this most cheaply from a declarative tool
(OpenTofu/Terraform, Kubernetes manifests). General-purpose-language tools
(Pulumi, the cloud CDKs, CDKTF, CDK8s) preserve desired-state when used in
the same way — the program emits a resource graph that a separate engine
diffs and applies. What is rejected is imperative deploy scripts whose only
artifact is the side effects they performed.
- Sources: Morris *IaC* (3rd ed), Ch. 4; Kubernetes docs, *Controllers*, https://kubernetes.io/docs/concepts/architecture/controller/ ; Pulumi docs, *How Pulumi Works*, https://www.pulumi.com/docs/concepts/how-pulumi-works/ (program → resource graph → engine diff); OpenGitOps Principles v1.0.0, https://opengitops.dev/ .
- Concrete check: a re-apply on an unchanged repo against unchanged reality must be a no-op. If it makes changes, the code is non-idempotent — fix it, regardless of the language it is written in.

**[IaC PRINCIPLES] [should, with caveat] Prefer declarative DSLs over general-purpose languages for cloud primitives — but the desired-state guarantee is an engine property, not a language property.**
Imperative automation *can* be made idempotent — it is just harder to keep
that way as the codebase grows, and it provides weaker diff and drift
semantics than a declarative reconciler. Declarative DSLs trade
expressiveness for reviewability: the diff in a PR is the diff that will be
applied. The carve-out is real and not grudging: GPL-based tools whose engine
emits a resource graph and diffs it before apply (Pulumi, CDKTF, CDK8s,
Crossplane Compositions) are not in the same category as imperative deploy
scripts. Reach for them when you have a real abstraction problem the DSL
cannot express — typed shared libraries, cross-stack invariants, regulated
estates needing language-level review tooling.
- Sources: Fowler, *InfrastructureAsCode* (declarative definition files); Morris *IaC* (3rd ed), Ch. 4; Pulumi docs, *How Pulumi Works*; Crossplane v2, *What's new*, https://docs.crossplane.io/v2.0/whats-new/ .
- Concrete check: the default tool for new stacks is the declarative one; using a general-purpose-language tool requires a one-paragraph ADR justifying the abstraction need *and* confirming the engine still produces a previewable plan.

**[IaC PRINCIPLES] [must] Every change is idempotent and re-runnable.**
You must be able to apply the same definition twice in a row, or after a
partial failure, and converge to the same state. Non-idempotent steps destroy
the ability to retry safely, which is the foundation of every recovery story.
- Sources: Morris *IaC* (3rd ed), Ch. 1; Fowler, *InfrastructureAsCode* (definition files + reproducible builds).
- Concrete check: in CI, run `tofu apply` / `terraform apply` (or equivalent) twice on a no-op PR. The second run must report zero changes.

**[IaC PRINCIPLES] [should] Prefer small, frequent changes over batched releases.**
Big-bang infra changes hide multiple errors that interact. Small diffs are
easier to review, revert, and attribute when something breaks.
- Source: Fowler, *InfrastructureAsCode* — "Small changes rather than batches… FrequencyReducesDifficulty."
- Concrete check: median PR diff in the infra repo fits on one screen. PRs that touch >3 stacks at once need an explicit reason.

**[IaC PRINCIPLES] [must] Code, state, and secrets live in separate, access-controlled stores; state is encrypted at two layers.**
Infra code is org-readable; state files contain rendered secrets and resource
IDs; secrets live in a secret manager and are referenced, never embedded.
Encryption applies at two distinct layers: (a) backend storage encryption
(S3 SSE-KMS, GCS CMEK, Azure Storage encryption) — necessary baseline, but
defeated by exfiltration of decrypted reads from the bucket; (b) state-payload
encryption with externally-held keys, so the state file remains opaque even
if the backend is compromised. OpenTofu 1.7+ provides (b) as a first-class
language feature, and 1.10 added external key providers.
- Sources: OpenTofu docs, *State encryption*, https://opentofu.org/docs/language/state/encryption/ ; OpenTofu, *1.10.0 release — External Key Providers*, https://opentofu.org/blog/opentofu-1-10-0/ ; HashiCorp, *Sensitive Data in State*, https://developer.hashicorp.com/terraform/language/state/sensitive-data ; Morris *IaC* (3rd ed), Ch. 8 & 19.
- Concrete check: `git grep` for high-entropy strings, `AKIA`, `BEGIN PRIVATE KEY` returns zero hits. State backends require encryption + a documented access list. Any state file whose plan/refresh renders production secrets uses payload-level encryption (OpenTofu state encryption or equivalent), not just bucket SSE.

**[IaC PRINCIPLES] [must] If your tool relies on shared state, that state is durable, versioned, locked, and restore-tested.**
Lost or corrupted Terraform/OpenTofu/Pulumi state is a top-tier infra
incident: without it the tool cannot map code to real resources, and recovery
means hand-importing every resource. Treat the state backend as a tier-0 data
store. Tools that do not expose shared mutable state (CloudFormation,
in-cluster GitOps controllers, Kubernetes-native reconcilers) need equivalent
guarantees on their own substrate — concurrency control, auditability, and
recoverability of the desired-state record. (Restore-drill cadence belongs
to chapters 08–09; this chapter requires only that one exists.)
- Sources: OpenTofu docs, *State*, https://opentofu.org/docs/language/state/ ; OpenTofu docs, *State locking*, https://opentofu.org/docs/language/state/locking/ ; HashiCorp, *State*, https://developer.hashicorp.com/terraform/language/state ; Morris *IaC* (3rd ed), Ch. 8.
- Concrete check: state buckets have versioning + encryption + restricted IAM + a backup-restore drill on file. A documented procedure exists for "the lock is stuck" and "the state is corrupt or lost". For non-state tools, the equivalent question is answered: where does desired state live, how is concurrent write prevented, how is it restored?

**[IaC PRINCIPLES] [must] Pre-apply policy-as-code gates every plan.**
Security and cost guardrails are code, reviewed and versioned, not human
judgement applied inconsistently in PR review. A plan that opens a public
S3 bucket, removes `prevent_destroy`, attaches `*` IAM, or provisions a
six-figure-monthly resource must fail the pipeline before it reaches an
approver, not after. Tooling: OPA/Rego + Conftest, Kyverno, Checkov,
tfsec/Trivy, HashiCorp Sentinel; pick one per pipeline, not three.
- Sources: Open Policy Agent docs, *Policy language (Rego)*, https://www.openpolicyagent.org/docs/latest/ ; CNCF, *OPA graduated project announcement (Feb 2021)*, https://www.cncf.io/announcement/2021/02/04/cloud-native-computing-foundation-announces-open-policy-agent-graduation/ ; Morris *IaC* (3rd ed), Ch. 9.
- Concrete check: every IaC pipeline runs at least one policy engine against `plan` output and blocks merge on failure. Policies live in the same repo (or a sibling) and are themselves PR-reviewed.

**[IaC PRINCIPLES] [should] Provider and module supply chain is hash-locked, signed where possible, and lockfiles are committed.**
Modules consumed by tag alone trust the registry not to retag; providers
fetched without hash verification trust whatever the network returns.
Post-XZ-utils, neither is acceptable for tier-0 tooling. Commit the
dependency lockfile (`.terraform.lock.hcl` for OpenTofu/Terraform), enforce
hash verification, and prefer registries that distribute signed digests
(OpenTofu 1.10+ supports OCI-distributed providers and modules; 1.12 hardens
the provider trust chain with `h1:` hashes alongside `zh:`).
- Sources: OpenTofu, *1.10.0 — OCI registry support*, https://opentofu.org/blog/opentofu-1-10-0/ ; OpenTofu, *1.12.0-beta1 — provider trust chain*, https://opentofu.org/blog/help-us-test-opentofu-1-12-0-beta1/ ; OpenTofu docs, *Dependency lock file*, https://opentofu.org/docs/language/files/dependency-lock/ . SLSA build-provenance floor for artifacts is owned by chapter 06.
- Concrete check: lockfile is checked in and CI fails if it would be modified by `init`. Module references resolve to a tagged version *and* a digest (or are pinned to an immutable ref). Bare `ref=main` in production is a review block.

**[IaC PRINCIPLES] [should] AI-generated IaC is reviewed against the same blast-radius standard as human-written IaC.**
Treat agent-authored plans as untrusted input: they pass syntax and often
pass review-by-tired-human, but a hallucinated `lifecycle.prevent_destroy
= false` removal or a silently widened IAM action is a production deletion
or a privilege escalation. The §1 idempotence and §3 drift bullets apply
unchanged; the policy gate above is the load-bearing control. Don't merge
agent output without the same `plan` preview, policy check, and reviewer
attention you'd require from a junior.
- Sources: Thoughtworks Technology Radar Vol 32–34 themes, *Complacency with AI-generated code* / *Codebase cognitive debt*, https://www.thoughtworks.com/radar ; OpenTofu, *1.10.0 — Registry MCP server for AI assistants* (acknowledging the workflow exists), https://opentofu.org/blog/opentofu-1-10-0/ .
- Concrete check: PR template asks "was any of this AI-generated?" and the policy pipeline runs identically regardless of the answer. A reviewer who cannot explain a plan diff blocks the merge.

**[IaC PRINCIPLES] [avoid] Hand-edited resources, console "quick fixes", and out-of-band scripts.**
Replacement: every change goes through the IaC repo, even fast-tracked. The
30-second console fix becomes a 6-month "why does prod not match the code"
investigation.
- Sources: Fowler, *SnowflakeServer*, https://martinfowler.com/bliki/SnowflakeServer.html ; Morris *IaC* (3rd ed), Ch. 1.
- Concrete check: cloud audit logs for write actions by non-CI identities should be near-zero. A non-zero number is the drift backlog.

---

## 2. IMMUTABILITY

**[IMMUTABILITY] [should] For disposable compute, replace servers and containers; do not mutate them in place.**
Phoenix / immutable server pattern: build a versioned artifact (AMI, container
image), deploy instances from it, roll forward by replacing instances rather
than `ssh`-ing in to patch. This eliminates runtime drift and makes rollback a
re-deploy of the previous artifact.
- Sources: Fowler / Ben Butler-Cole, *ImmutableServer*, https://martinfowler.com/bliki/ImmutableServer.html ; Morris *IaC* (3rd ed), Ch. 11 & 13.
- Concrete check: prod hosts in autoscaling/Kubernetes/serverless layers have no human SSH by default; deploys produce a new image tag rather than running `apt upgrade` on the existing fleet.

**[IMMUTABILITY] [prefer] "Cattle, not pets" — but understand what the metaphor does and does not mean.**
The point of the cattle reframe (attributed to Bill Baker, popularised by
Randy Bias ~2012) is that *individually identifiable, hand-tended* nodes are
an operational liability at scale. It does **not** mean "all workloads are
stateless" or "databases don't exist". Stateful systems (primary databases,
brokers, identity stores) are legitimately pets and must be managed as such —
named runbooks, careful upgrades, tested restores. The cattle pattern applies
to the *fleet* surrounding them.
- Sources: Fowler, *SnowflakeServer* (footnote attributing the metaphor); Morris *IaC* (3rd ed), Ch. 2 & 11.
- Concrete check: per system, the org can articulate "cattle or pet?", and the pets have named runbooks plus tested restores. A "cattle" system with unique on-disk state is a pet in disguise.

**[IMMUTABILITY] [should] Bake non-varying configuration into the artifact; inject only what does vary.**
Per-instance config (region, environment, secret references) is read at boot
from the platform; everything else (packages, daemons, kernel tunings) is in
the image. This is what makes the artifact actually immutable.
- Source: Fowler, *ImmutableServer* — "minimize the number and scope of per-instance configuration items".
- Concrete check: two hosts launched from the same image in the same env produce byte-identical `dpkg -l` output.

**[IMMUTABILITY] [avoid] Long-running config-management agents that "self-heal" mutable hosts *in disposable-compute fleets*.**
Replacement: rebuild from a versioned image. For ASGs, Kubernetes nodes, and
container fleets, scheduled Puppet/Chef/Ansible runs against long-lived hosts
are vectors for untested change in production. Use those tools to *build
images*, not to maintain live cloud fleets.
- Source: Fowler, *ImmutableServer* — "you shouldn't run configuration management tools, since they create opportunities for untested changes".
- Concrete check: no scheduled `puppet agent` / `chef-client` / cron-driven Ansible against autoscaled cloud hosts. Image-build pipelines are where these run.

**[IMMUTABILITY] [should] For non-disposable estates (bare metal, network gear, legacy VMs, regulated stateful hosts), use config management with strict change control and continuous drift visibility.**
Some hosts cannot be replaced cheaply: switches, hypervisors, on-prem DB
clusters, hardware appliances, specialised regulated boxes. Categorical
"never run an agent" is wrong here. The discipline shifts to: every change
flows through the same VCS + review pipeline that builds images elsewhere;
the agent's purpose is to *enforce reviewed state*, not to ad-hoc-patch; and
drift between code and host is reported continuously (see §3).
- Sources: Morris *IaC* (3rd ed), Ch. 11 & 13 (treatment of unreplaceable hosts); Puppet docs, *Continuous delivery & enforcement*, https://www.puppet.com/docs/puppet/8/puppet_overview .
- Concrete check: every config-managed host's runs are sourced from a tagged commit, agents run in enforce-not-apply-ad-hoc mode, and a drift report is reviewed at least weekly.

---

## 3. DRIFT

**[DRIFT] [must] Drift is detected continuously, not "when something breaks".**
Drift is inevitable: humans break-glass during incidents, providers change
defaults, controllers in adjacent systems mutate tags, IAM gets "temporarily"
widened. The question is how long it lives undetected. A scheduled `plan` in
CI, or a GitOps controller reporting `OutOfSync`, surfaces drift in hours
instead of quarters.
- Sources: Argo CD, *Architecture* — Application Controller "continuously monitors… compares the current, live state against the desired target state… detects OutOfSync", https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/ ; Flux (CNCF Graduated, Nov 2022), *Core Concepts — Reconciliation*, https://fluxcd.io/flux/concepts/ ; OpenGitOps Principles v1.0.0, Principle 4 *Continuously Reconciled*, https://opengitops.dev/ ; Morris *IaC* (3rd ed), Ch. 20.
- Concrete check: every stack has either (a) a GitOps controller reporting `OutOfSync`, or (b) a scheduled CI `plan` posting non-empty diffs to a channel an on-call engineer reads. "Drift-detection-as-CI" is the named pattern for estates that use OpenTofu/Terraform without an in-cluster controller.

**[DRIFT] [should] When drift is detected, default to reconciling *toward* the code, not absorbing the change into the code.**
Otherwise the console becomes the source of truth and the repo becomes
documentation. The exception is when reality is correct and the code is
wrong — fix via PR, not by editing state.
- Sources: OpenGitOps Principles v1.0.0, Principle 4 *Continuously Reconciled*, https://opengitops.dev/ (and version-stable: https://github.com/open-gitops/documents/releases/tag/v1.0.0 ); Morris *IaC* (3rd ed), Ch. 20.
- Concrete check: drift-resolution playbook names "revert reality" as default; "import into code" requires a written justification.

**[DRIFT] [prefer] Treat the IaC repo as "what *should* be running" and the cloud provider as "what *is* running" — never collapse the two.**
The gap is your operational signal. Bulk-import tools are useful for
migrating off click-ops once, but routine use trains the team that the
console is acceptable.
- Source: Morris *IaC* (3rd ed), Ch. 20; OpenTofu docs, *Importing infrastructure*, https://opentofu.org/docs/cli/import/ .
- Concrete check: `tofu import` / `terraform import` appears in PRs only with an attached drift-incident reference.

---

## 4. RECONCILIATION

**[RECONCILIATION] [must] Adopt the control-loop mental model as the primary abstraction for infrastructure systems.**
A controller observes desired state (git, CRD, config), observes actual state
(via API), computes a diff, and takes the smallest action that moves actual
toward desired — then loops forever. The same idea behind a thermostat, a
Kubernetes controller, `tofu apply`, and a GitOps agent. Internalising it
dissolves a lot of accidental complexity: stop writing "deploy scripts",
start writing "desired state plus a reconciler".
- Sources: Kubernetes docs, *Controllers* (canonical statement of the pattern); OpenGitOps Principles 1 & 4 (declarative + continuously reconciled); Flux, *Reconciliation*.
- Concrete check: a new automation proposal names the desired-state representation, the observer, and the reconciliation interval. A "script on a schedule that does X" should be pushed back on.

**[RECONCILIATION] [should] Reconciliation is continuous, with bounded drift between observation and correction.**
Webhook-only reconciliation misses out-of-band mutations. Default Flux/Argo
intervals are minutes; choose based on tolerable undetected-drift, not
"however infrequently is convenient". Flux v2.7+/2.8 use server-side apply
by default, which changes how field-management interacts with the §7
single-ownership invariant — be explicit about which controller "owns" each
field on a shared resource.
- Sources: Flux docs, *Kustomization* (default reconcile interval); Flux blog, *Flux 2.8 GA — server-side apply / Helm v4*, https://fluxcd.io/blog/ ; Argo CD, *Architecture*.
- Concrete check: each reconciler has a documented interval and an alert that fires if it falls behind.

**[RECONCILIATION] [prefer] Pick reconcilers that converge from any starting state, including a freshly-deleted cluster.**
"Bootstrap from git" is the test: delete the cluster, rerun the controller
against the same repo, get the same cluster back. This is the difference
between a control plane and a deploy script that happens to be idempotent.
Crossplane (v2, 2025) is now positioned as a framework for building
*application* control planes — namespaced composite and managed resources,
composition over arbitrary Kubernetes resources, operations CRDs — not a
drop-in OpenTofu replacement; weigh it accordingly in §7.
- Sources: Flux docs, *Bootstrap*, https://fluxcd.io/flux/installation/bootstrap/ ; Crossplane v2, *What's new*, https://docs.crossplane.io/v2.0/whats-new/ .
- Concrete check: a documented "rebuild from zero" runbook exists and is exercised at least annually.

---

## 5. GITOPS

**[GITOPS] [must] If you call something GitOps, it satisfies all four OpenGitOps principles: declarative, versioned & immutable, pulled automatically, continuously reconciled.**
The term has been diluted to mean "we use git for our YAML". The
Weaveworks-originated, CNCF-stewarded definition is specific. Without *pull*
and *continuous reconcile* you have "CI that runs `kubectl apply`" — fine,
but not GitOps, and missing the drift-correction property.
- Sources: OpenGitOps Principles v1.0.0, https://opengitops.dev/ (version-stable: https://github.com/open-gitops/documents/releases/tag/v1.0.0 ); Flux, *Core Concepts*; Argo CD, *Architecture*.
- Concrete check: name the agent that pulls, the reconcile interval, and the immutable artifact (commit SHA or signed OCI digest) it pinned to. If any is missing, do not market the system as GitOps.

**[GITOPS] [should] Prefer pull-based GitOps over push-based CI/CD for cluster-resident workloads.**
In the pull model the agent runs *inside* the trust boundary and reaches
outward to read git/OCI; CI does not need cluster-admin credentials. This
shrinks CI-compromise blast radius dramatically. Push is appropriate when
the target is not a cluster (raw cloud accounts via Terraform Cloud /
Atlantis style), or when network constraints make pull impractical.
- Sources: OpenGitOps Principle 3 ("Pulled Automatically"); Flux docs (controllers run in-cluster, CNCF Graduated Nov 2022, https://fluxcd.io/blog/2022/11/flux-is-a-cncf-graduated-project/ ); Argo CD architecture (in-cluster controller).
- Concrete check: production cluster credentials do not exist as long-lived secrets in CI; the cluster's outbound git/OCI read is the only path by which manifests get applied.

**[GITOPS] [should] One environment per directory (or overlay), not per long-lived branch.**
Branch-per-environment ("dev → staging → main = prod") is the single most
common GitOps anti-pattern. Merges stop being clean and you end up with
"snowflakes as code". Use directories or Kustomize/Helm overlays so promotion
is the same manifests with different parameters.
- Sources: Morris *IaC* (3rd ed), Ch. 8 & 12; Argo CD docs, *Best practices — environments and overlays*, https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/ . See also: ThoughtWorks Radar *GitOps* (Apr 2021/Sep 2023, archived), https://www.thoughtworks.com/radar/techniques/gitops .
- Concrete check: one long-lived branch (`main`). Environments are dirs or overlays under it. PRs that introduce `staging`/`prod` branches are rejected.

**[GITOPS] [prefer] Weigh GitOps' operational tax against the drift-correction and audit benefits before adopting in small estates.**
A reconciler, CRDs, an extra access model, and a second source of truth (git
plus cluster) all have non-trivial cost. Adopt when **any one** of these
triggers holds: more than one environment, more than one cluster, more than
one team sharing a cluster, or any compliance regime that asks
"who-changed-what-when". Below all four triggers, a disciplined CI pipeline
is usually the right answer; decide explicitly rather than by default.
- Sources: OpenGitOps v1.0.0 (audit + reconciliation as primary benefits); Morris *IaC* (3rd ed), Ch. 1.
- Concrete check: before adopting Argo CD / Flux, write down (a) which of the four triggers (>1 env / >1 cluster / >1 team / compliance audit) currently applies, and (b) what fails *today* that GitOps would fix. If none of the triggers holds and (b) is weak, defer; otherwise adopt.

---

## 6. ENVIRONMENTS

**[ENVIRONMENTS] [must] Environments are produced from the *same* code with different parameters; not copy-pasted.**
If `prod/main.tf` and `staging/main.tf` are independent files, they will
diverge — silently, and you only learn during the incident the divergence
caused. Use modules (OpenTofu/Terraform), overlays (Kustomize), stacks
(Pulumi/CDK), or the equivalent. Differences between environments are
parameter values, not parallel codebases.
- Sources: Morris *IaC* (3rd ed), Ch. 8 & 12; OpenTofu docs, *Modules*, https://opentofu.org/docs/language/modules/ .
- Concrete check: `diff -r staging/ prod/` shows only parameter values, not resource definitions.

**[ENVIRONMENTS] [should] Staging mirrors production for the components and controls that determine production failure modes; size, data, and cost-driven extras may differ.**
A staging that lacks the load balancer, the auth path, the WAF, or runs a
different DB engine cannot catch the failure modes those components produce.
What must match is *topology of the critical path*: ingress, identity, data
store class and version, deployment mechanism, security controls. What may
legitimately differ is replica count, instance size, data volume, and
non-critical observability/cost-saving extras. The point is parity for
failure modes, not literal resource-by-resource cloning.
- Sources: Morris *IaC* (3rd ed), Ch. 12; Fowler, *InfrastructureAsCode* (testing infra changes through pipelines); Google SRE Workbook, *Canarying Releases*, https://sre.google/workbook/canarying-releases/ (parity required for the controls being tested).
- Concrete check: list the production-critical components (ingress, auth, DB engine + version, deploy path, security controls). Each is present in staging with the same type and version. Documented exceptions exist for cost-driven scale differences.

**[ENVIRONMENTS] [prefer] Use ephemeral, per-PR environments; pick the isolation layer that fits the workload.**
Shared dev environments accumulate everyone's broken experiments and become
nobody's responsibility, so per-PR environments are attractive in principle.
The 2024–2026 baseline isn't "full stack vs nothing" — isolation can live at
several layers: per-PR cluster/account (heavy, full fidelity); per-PR
namespace on a shared cluster (vCluster, Crossplane v2 namespaced XRs);
per-PR schema on a shared DB; per-PR branch/index on shared search/analytics.
Substrate-sharing makes "ephemeral" affordable for previously-heavyweight
stacks. A shared dev/test environment with strict ownership and nightly
rebuild is still a legitimate alternative; pick per workload and per layer.
- Sources: Morris *IaC* (3rd ed), Ch. 12 & 17; Crossplane v2, *What's new* (namespaced XRs/MRs enable per-PR isolation on shared substrate), https://docs.crossplane.io/v2.0/whats-new/ ; Flux docs, *Multi-tenancy*, https://fluxcd.io/flux/installation/configuration/multitenancy/ .
- Concrete check: for each workload, the team has decided the isolation layer (cluster / namespace / schema / branch) and can justify the choice in cost-and-iteration-speed terms. Either choice is defensible; absence of a choice is not.

**[ENVIRONMENTS] [avoid] A "prod-like" environment that engineers do not actually use to validate changes.**
Replacement: either invest until staging is trusted, or delete it and be
honest you test in prod with feature flags. A staging everyone ignores
provides false assurance and rots.
- Source: Morris *IaC* (3rd ed), Ch. 12 & 17.
- Concrete check: ask three engineers when they last caught a real bug in staging. If none can name one in the last quarter, the environment is dead.

---

## 7. TOOLING DECISION

**[TOOLING DECISION] [must] For any given resource, exactly one tool is authoritative.**
Two tools managing the same resources is the fastest way to produce drift
between competing controllers — each "fixes" what the other "broke". A
healthy stack often uses several tools (e.g. OpenTofu for cloud primitives,
Helm for app packaging, Kustomize for overlay edits, a GitOps agent for
delivery), and that is fine *as long as their boundaries are explicit and
non-overlapping*. The invariant is single ownership per resource, not single
tool per layer.
- Sources: Morris *IaC* (3rd ed), Ch. 4 & 9; Kubernetes docs, *Controllers* (controller fight semantics when ownership overlaps); Kubernetes docs, *Server-Side Apply — field management*, https://kubernetes.io/docs/reference/using-api/server-side-apply/ .
- Concrete check: every resource type in the inventory names exactly one tool as its owner. Where Helm and Kustomize coexist, document which fields each may write (server-side apply makes this enforceable). Cross-tool overlap is treated as an incident, not a workflow.

**[TOOLING DECISION] [prefer] Default to a declarative, HCL-based tool with a strong provider ecosystem; treat OpenTofu as the new-work default and Terraform as a license-encumbered alternative.**
HashiCorp relicensed Terraform from MPL-2.0 to BUSL-1.1 on 10 Aug 2023; the
Linux Foundation accepted the OpenTofu fork in Sep 2023, and OpenTofu has
shipped continuously since (1.6 Jan 2024 → 1.10 May 2025 → 1.11 Dec 2025).
Post-1.5.5 Terraform is no longer OSI-licensed; whether that matters depends
on your distribution model and your customers' procurement constraints —
but it is a first-class selection criterion, not a footnote. Runners-up
(Pulumi, the cloud CDKs, CDKTF, CDK8s) let you use a general-purpose
language; pick them per the §1 carve-out. Deep comparison is in chapter 02.
- Sources: OpenTofu, *Manifesto*, https://opentofu.org/manifesto/ ; OpenTofu, *1.10.0 release announcement*, https://opentofu.org/blog/opentofu-1-10-0/ ; IBM Newsroom, *IBM to Acquire HashiCorp* (24 Apr 2024), https://newsroom.ibm.com/2024-04-24-IBM-to-Acquire-HashiCorp-Inc-Creating-a-Comprehensive-End-to-End-Hybrid-Cloud-Platform ; Morris *IaC* (3rd ed), Ch. 4.
- Concrete check: a one-page ADR records (a) which tool was chosen, (b) the licence under which it ships, (c) why this tool over the obvious alternative for *this* org's constraints.

**[TOOLING DECISION] [should] Use a decision framework: blast radius, desired-state durability, drift detection, secrets handling, ecosystem, exit cost, *policy-as-code support*, *licence and governance*.**
Tool-shape-dependent non-negotiables: (1) if the tool relies on shared
mutable state, it must support remote state with locking, versioning, and
recovery — see §1; tools without shared state (CloudFormation, GitOps
controllers, K8s-native reconcilers) must demonstrate equivalent
concurrency-safety, audit, and recoverability on their own substrate. (2) A
`plan`-style preview showing diffs before apply. (3) Integration with a
secret manager. (4) Credible drift detection. (5) A maintained provider for
every platform you depend on. (6) Replaceable without rewriting every
module. (7) A first-class policy-as-code integration point that gates `plan`
output before `apply`. (8) Licence and governance — OSI-approved licence,
open governance (foundation-stewarded > single-vendor), and a credible exit
path if the upstream relicenses again.
- Sources: Morris *IaC* (3rd ed), Ch. 3 & 4; OpenTofu, *Manifesto* (governance + licence axis); Open Policy Agent, *Use cases — Terraform/IaC*, https://www.openpolicyagent.org/docs/latest/terraform/ ; AWS CloudFormation docs, *Stack updates and change sets*, https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html .
- Concrete check: the chosen tool is graded against the eight criteria in a short ADR, with gaps explicitly listed and a mitigation for each.

**[TOOLING DECISION] [avoid] Adopting a second IaC tool because the first is "annoying" for one workflow.**
Replacement: extend the existing tool (write a module, write a provider, push
the missing feature upstream) or accept the rough edge. Each additional tool
roughly doubles the maintenance surface. Two tools is sometimes correct;
three is almost never.
- Source: Morris *IaC* (3rd ed), Ch. 1 & 4.
- Concrete check: introducing a new IaC tool requires an ADR naming the workflow the existing tool genuinely cannot do, plus a sunset plan if the new tool's scope expands.

---

## 8. VERSIONING, MODULARITY & BLAST RADIUS

**[MODULARITY] [must] Stacks (state files / equivalents) are sized so a worst-case `apply` only destroys what you can afford to lose.**
"One giant state for the whole company" is the worst failure mode: one bad
PR can take down everything, and `apply` times grow until nobody dares run
them. Split by blast radius: per-environment, per-bounded-context, per-account.
- Sources: Morris *IaC* (3rd ed), Ch. 7; OpenTofu docs, *State*, https://opentofu.org/docs/language/state/ .
- Concrete check: name a stack and ask "if this `apply` ran with `-replace` on every resource, what would break and who would page?". The answer must be small enough to ship.

**[MODULARITY] [must] Modules are versioned and consumed by signed digest or hash-locked version, not by tag alone.**
Floating to `main` couples every consumer to every change at once — no
rollout, only big bang. Tag-only pinning trusts the registry not to retag a
release out from under you, which is exactly the post-XZ threat model.
Pin to an immutable digest (OCI registries) or to a tagged version with a
committed lockfile that records the hash; upgrade consumers one at a time,
observe.
- Sources: Morris *IaC* (3rd ed), Ch. 10 & 15; OpenTofu, *1.10.0 — OCI registry support*, https://opentofu.org/blog/opentofu-1-10-0/ ; OpenTofu docs, *Dependency lock file*, https://opentofu.org/docs/language/files/dependency-lock/ .
- Concrete check: every module reference in production stacks resolves to (a) a tagged version with the lockfile committed, or (b) an OCI digest. `ref=main` in a prod stack is a review block.

**[MODULARITY] [prefer] A small, well-documented module beats a large configurable one.**
The "do-everything" module with 80 input variables is its own DSL, badly
designed, with no type system. Three composable modules of 10 inputs each
are easier to test, upgrade, and delete.
- Source: Morris *IaC* (3rd ed), Ch. 5 & 10.
- Concrete check: modules with >20 inputs need a written reason; modules with >40 should be split.

---

## 9. CULTURE: "you build it, you run it"

**[CULTURE] [should] Service teams retain production ownership and a working feedback loop from production, even when supported by SRE/platform teams.**
The original DevOps thesis: feedback from production is a primary input to
design, and it cannot reach designers if a wall sits between them and the
pager. The point is not "delete the SRE team" — platform/SRE teams are
valuable when they provide *leverage* (paved roads, shared
observability). Ownership of correctness in production sits with the team
that wrote the code. This is a strong default, not an absolute: regulated
estates, 24×7 follow-the-sun coverage, and shared infra services
legitimately split operational duties — the test is whether designers still
receive production feedback fast and unfiltered.
- Sources: Werner Vogels (interviewed by Jim Gray), *A Conversation with Werner Vogels*, ACM Queue, vol. 4, no. 4, June 2006 — the original "you build it, you run it" citation, https://queue.acm.org/detail.cfm?id=1142065 ; Google, *Site Reliability Engineering*, Ch. 1 *Introduction*, https://sre.google/sre-book/introduction/ ; Skelton & Pais, *Team Topologies*, IT Revolution 2019.
- Concrete check: the on-call rotation for service X is staffed by, or directly co-staffed with, the team that merges to service X's repo. If a different team owns the pager with no feedback path back to authors, you have ops, not DevOps.

**[CULTURE] [should] Platform / SRE teams build paved roads, not gatekeeping queues.**
A platform team that ships modules, golden CI templates, and a supported
runtime is leverage. A platform team that approves every PR or provisions
every resource is a bottleneck. Test: can a product team go from zero to a
running service without filing a ticket? (The canonical "paved road" model
is owned by chapter 11; this chapter only requires that one exists.)
- Sources: Skelton & Pais, *Team Topologies* (platform team as enabling, not gating); Morris *IaC* (3rd ed), Ch. 21 ("Governance").
- Concrete check: measure mean time from "new team starts" to "service running in prod". Hours-to-days is healthy; weeks-to-months indicates gatekeeping.

**[CULTURE] [must] Operability is a first-class requirement of every change: ship logs, metrics, alerts tied to user-visible signals, and a runbook entry — not a follow-up ticket.**
A PR that ships a new endpoint without observability and a runbook entry is
incomplete. This is what makes "you build it, you run it" survivable for the
team building it — the alternative is on-call on top of code that cannot be
debugged at 3am.
- Sources: Google, *Site Reliability Engineering*, Ch. 6 *Monitoring Distributed Systems*, https://sre.google/sre-book/monitoring-distributed-systems/ ; Google, *The Site Reliability Workbook*, *Implementing SLOs*, https://sre.google/workbook/implementing-slos/ ; Google, *SRE Book*, Ch. 11 *Being On-Call*, https://sre.google/sre-book/being-on-call/ ; Charity Majors, *Observability — A Manifesto*, https://www.honeycomb.io/blog/observability-a-manifesto .
- Concrete check: PR template includes "how will this be observed in prod (metrics, logs, alert)" and "what's the runbook entry / link". Empty answers are review blocks. Alerts are tied to SLOs/user-visible signals, not raw resource metrics.

**[CULTURE] [avoid] Treating the IaC repo as the operations team's repo.**
Replacement: the IaC repo is a normal product repo, owned by the team(s)
whose infra it describes, with the same review/test/release standards as
application code. The moment infra code lives where application engineers
cannot read and change it, "you build it, you run it" is dead and drift is
inevitable.
- Sources: Fowler, *InfrastructureAsCode* (treat infra code "just like any software system"); Morris *IaC* (3rd ed), Ch. 1.
- Concrete check: application engineers can open a PR against the infra repo for their own service without a special request. If they cannot, fix that first.

---

## Sources

### Books
- Kief Morris, *Infrastructure as Code: Dynamic Systems for the Cloud Age*, 3rd Edition, O'Reilly, 2024 — https://infrastructure-as-code.com/book/
- Betsy Beyer et al. (eds.), *Site Reliability Engineering*, Google / O'Reilly, 2016 — https://sre.google/sre-book/table-of-contents/
- Betsy Beyer et al. (eds.), *The Site Reliability Workbook*, Google / O'Reilly, 2018 — https://sre.google/workbook/table-of-contents/
- Matthew Skelton & Manuel Pais, *Team Topologies*, IT Revolution, 2019 — https://teamtopologies.com/book

### Standards & working groups
- OpenGitOps Principles v1.0.0 (CNCF GitOps Working Group) — https://opengitops.dev/ ; version-stable: https://github.com/open-gitops/documents/releases/tag/v1.0.0
- Kubernetes documentation, *Controllers* — https://kubernetes.io/docs/concepts/architecture/controller/
- Kubernetes documentation, *Server-Side Apply* — https://kubernetes.io/docs/reference/using-api/server-side-apply/
- Open Policy Agent — https://www.openpolicyagent.org/docs/latest/ ; CNCF graduation (Feb 2021) — https://www.cncf.io/announcement/2021/02/04/cloud-native-computing-foundation-announces-open-policy-agent-graduation/

### Vendor / project documentation (canonical statement of a pattern, not product endorsement)
- OpenTofu (Linux Foundation), *Manifesto* — https://opentofu.org/manifesto/
- OpenTofu, *1.10.0 release* — https://opentofu.org/blog/opentofu-1-10-0/
- OpenTofu, *1.12.0-beta1 release (provider trust chain)* — https://opentofu.org/blog/help-us-test-opentofu-1-12-0-beta1/
- OpenTofu docs, *State*, *State encryption*, *State locking*, *Modules*, *Dependency lock file*, *Importing infrastructure* — https://opentofu.org/docs/
- HashiCorp (now an IBM business unit), *State / Backend / Sensitive Data in State* — https://developer.hashicorp.com/terraform/language/state
- IBM Newsroom, *IBM to Acquire HashiCorp* (24 Apr 2024) — https://newsroom.ibm.com/2024-04-24-IBM-to-Acquire-HashiCorp-Inc-Creating-a-Comprehensive-End-to-End-Hybrid-Cloud-Platform
- Pulumi, *How Pulumi Works* — https://www.pulumi.com/docs/concepts/how-pulumi-works/
- AWS, *CloudFormation: Updating Stacks Using Change Sets* — https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html
- Argo CD, *Architecture* / *Best practices* — https://argo-cd.readthedocs.io/
- Flux (CNCF Graduated, Nov 2022), *Core Concepts* / *Bootstrap* / *Multi-tenancy* / *Flux 2.8 GA* — https://fluxcd.io/
- Crossplane v2, *What's new* — https://docs.crossplane.io/v2.0/whats-new/
- Puppet, *Puppet overview* — https://www.puppet.com/docs/puppet/8/puppet_overview

### Named industry voices
- Martin Fowler, *InfrastructureAsCode* — https://martinfowler.com/bliki/InfrastructureAsCode.html
- Martin Fowler, *SnowflakeServer* — https://martinfowler.com/bliki/SnowflakeServer.html
- Martin Fowler & Ben Butler-Cole, *ImmutableServer* — https://martinfowler.com/bliki/ImmutableServer.html
- Werner Vogels (interviewed by Jim Gray), *A Conversation with Werner Vogels*, ACM Queue, vol. 4, no. 4, June 2006 — https://queue.acm.org/detail.cfm?id=1142065
- Charity Majors, *Observability — A Manifesto* — https://www.honeycomb.io/blog/observability-a-manifesto

### Industry analysis (historical / see-also)
- ThoughtWorks Technology Radar, *GitOps* (Apr 2021 / Sep 2023, archived) — https://www.thoughtworks.com/radar/techniques/gitops
- ThoughtWorks Technology Radar, *Infrastructure as Code* — https://www.thoughtworks.com/radar/techniques/infrastructure-as-code
- ThoughtWorks Technology Radar, *OpenTofu* — https://www.thoughtworks.com/radar/tools/opentofu
- ThoughtWorks Technology Radar Vol 32–34, themes on AI-assisted code risk — https://www.thoughtworks.com/radar
