# 03 — CI/CD

Opinionated, cloud-agnostic defaults for build, release, and deploy.
Everything here is do-this-by-default; deviations should be justified
in code review.

> Conventions: **must** = load-bearing; **should** = strong default;
> **prefer** = taste call backed by evidence; **avoid** = rejected,
> with a named replacement. See [`CONTRIBUTING.md`](../CONTRIBUTING.md).

> **One-line position.** Pipelines are *code* in the same repo as
> the thing they build, runners are *ephemeral*, deploys are
> *pull-based* from Git for anything Kubernetes-shaped and
> *push-based via OIDC* for serverless and IaC, every artifact is
> **SBOM'd, signed, and attested** to at least SLSA Build L3, and
> the only health metrics that matter are the four DORA ones.

---

## 1. Pipeline-as-code: pick one, commit hard

The pipeline is source code. It lives in the same repo as the
application, is reviewed in PRs, and never edited in a vendor UI.
"Click-ops on Jenkins" is the same anti-pattern as "click-ops in the
cloud portal."

The honest landscape, ranked by where greenfield work should start in
2025–2026:

- **GitHub Actions** — default for most teams. Largest action
  ecosystem, native OIDC to all three major clouds, free public
  runners, attestations and artifact signing built in. Pay the
  ecosystem-trust tax (see §4).
- **GitLab CI** — best when the SCM is already GitLab; tight
  integration with the rest of the GitLab platform (Container
  Registry, Dependency Scanning, Secret Detection). YAML model is
  cleaner than Actions for fan-out matrices.
- **Azure DevOps Pipelines** — still excellent for .NET / Windows
  shops and for environments that need YAML *and* classic approval
  gates. Microsoft's own roadmap clearly favors GitHub Actions; treat
  ADO as steady-state, not strategic.
- **Tekton** — the right answer when the *pipeline itself* must run
  on Kubernetes (regulated platforms, internal developer platforms).
  CDEvents-aware, Sigstore-native, but an order of magnitude more
  operational work than a SaaS runner.
- **CircleCI / Buildkite / Drone** — niche but legitimate: Buildkite
  for hybrid/self-hosted at scale (the build plane is yours, the
  control plane is theirs); CircleCI for fast macOS/iOS matrices;
  Drone for tiny self-hosted footprints.
- **Jenkins** — **avoid for greenfield.** Jenkins lost on three
  axes: pipeline-as-code is bolt-on (Jenkinsfile + plugin sprawl
  vs. native YAML), the plugin trust surface is enormous, and SaaS
  runners removed its only real moat (self-hosting). Maintain
  existing Jenkins; do not start new pipelines on it.

Whatever you pick, **must**: pipelines live next to code, are
reviewed in the same PR, and have no manual UI configuration that
isn't also in the repo.

---

## 2. Push vs pull: the deploy split

Two deploy models, and the choice is *per workload class*, not per
company:

**Push (CI deploys).** CI runs `kubectl apply` / `terraform apply` /
`az deploy` / `aws cloudformation deploy` with short-lived OIDC
credentials. Right answer for: IaC (the cluster doesn't yet exist),
serverless functions, edge/CDN configuration, anything outside a
Kubernetes control loop, and one-shot data migrations.

**Pull (GitOps).** A controller in the target cluster
(**Argo CD** or **Flux**) watches a Git repo of manifests and
reconciles continuously. Right answer for: every Kubernetes workload
once the cluster exists. Drift is detected and either auto-healed or
surfaced; secrets and cluster credentials never leave the cluster;
the audit log is `git log`.

Hard rules:

- **must:** Kubernetes application deploys are pull-based via Argo CD
  or Flux, not `kubectl apply` from CI. CI's job ends at "image
  pushed and digest written to the env repo." (Argo CD docs;
  ThoughtWorks Tech Radar has had GitOps in *Adopt* since 2022.)
- **must:** the env repo references images by **digest**
  (`@sha256:…`), not by mutable tag. Mutable tags defeat both Argo's
  diff and Sigstore verification.
- **should:** one *config repo* per environment family (or one repo
  with directory-per-env), separate from the application source repo.
  This is the boundary that lets platform teams promote without
  application-team write access.
- **avoid:** mixing models for the same workload — "GitOps for
  manifests but CI also kubectl-applies for hotfixes." That is how
  you get state divergence at 3am.

---

## 3. Supply chain: SLSA, SBOM, signing — the new minimum bar

Software supply-chain attacks are the #1 way teams got compromised in
2023–2024 (xz-utils backdoor, March 2024, is the canonical recent
example: a multi-year social-engineering campaign against an upstream
maintainer made it into Debian/Fedora pre-release before being caught
by a Postgres performance regression). The defensive baseline is
**SLSA + SBOM + signing**, all three.

### 3.1 SLSA (Supply-chain Levels for Software Artifacts)

SLSA defines four levels for **build integrity**. Targets:

- **L1** — provenance exists. Trivial; do it day one.
- **L2** — provenance is generated by a hosted build service and
  signed. GitHub Actions' built-in `attest-build-provenance` and
  GitLab's SLSA provenance hit L2 with one workflow step.
- **L3** — build is run on a **hardened, isolated** builder; the
  provenance is non-forgeable by the workflow itself. This is the
  *real* target for production: it requires reusable workflows on
  hosted runners (GitHub's `slsa-framework/slsa-github-generator`,
  GitLab's hosted SLSA L3 runners) so the build script can't tamper
  with the attestation.
- **L4** — two-party review of every change to the build process and
  hermetic builds. Worth it for distros, base images, and
  high-blast-radius shared libraries; overkill for a leaf service.

**must:** every release artifact (container image, binary, package)
ships with a signed SLSA provenance attestation, minimum L2, target
L3 for anything user-facing or internet-exposed.
**Concrete check:** `cosign verify-attestation --type slsaprovenance
<image@digest>` succeeds in a non-CI environment.

### 3.2 SBOMs

Two formats matter, both ISO/standard-track:

- **SPDX** (ISO/IEC 5962:2021) — Linux Foundation, the lingua franca
  for legal/compliance reviews and US federal procurement (EO 14028).
- **CycloneDX** (OWASP, ECMA-424) — richer security metadata
  (VEX, vulnerabilities, services). Better for runtime risk tooling.

You will end up generating both. Pick **CycloneDX as primary** for
security tooling and emit SPDX alongside for compliance.

- **must:** every container image and release binary has an SBOM
  generated at build time (Syft, `docker buildx … --sbom`,
  `Microsoft.Sbom.Targets`, Anchore Enterprise, Trivy).
- **must:** SBOMs are stored *with the artifact* — as an OCI
  referrer (`cosign attach sbom`, OCI 1.1 referrers API), not in a
  side bucket that drifts.
- **should:** SBOMs are scanned in CI **and continuously**
  post-release (Grype, Trivy, Dependency-Track). New CVEs published
  against an old image must page someone, not just sit in a tab.
- **avoid:** generating an SBOM by listing `package.json` /
  `*.csproj`. That is a manifest, not an SBOM — it misses transitive
  pins, native deps, and base-image content.

### 3.3 Sigstore: keyless signing as the default

The old model — long-lived signing keys in HSMs, distributed via PKI
— is no longer the default for OSS or for most enterprises. **Sigstore**
(Fulcio + Rekor + cosign, CNCF Graduated 2024) issues short-lived
signing certs bound to an OIDC identity (your GitHub Actions workflow,
your GitLab job, your corporate IdP), records every signature in a
public transparency log (Rekor), and gives you `cosign verify` against
the *workflow that built the artifact*, not a key blob.

- **must:** every container image and release artifact is signed
  with cosign keyless (`cosign sign --yes <image@digest>`) using the
  CI's OIDC identity. Verification policy in the deploy target
  (admission controller, Ratify, Kyverno, Connaisseur, Sigstore
  policy-controller) requires a signature whose *certificate identity*
  matches the expected workflow.
- **should:** Rekor inclusion proofs are checked offline at deploy
  time, not just trusted because cosign returned 0.
- **prefer:** keyless over long-lived KMS keys *unless* you have a
  regulatory requirement (FIPS, air-gapped) that forbids the
  transparency log. In that case, cosign supports KMS too — same
  verification UX.

---

## 4. Pinned actions, OIDC, and the `permissions:` block

The three GitHub-Actions-specific (and equally-applicable-to-GitLab)
hardening rules. Each is independently load-bearing.

**must:** every third-party action is pinned to a **full commit
SHA**, not a tag. Tags are mutable; a compromised maintainer can
re-point `v3` overnight (this is the `tj-actions/changed-files`
incident pattern, March 2025). Use `pinact` or StepSecurity's
`secure-workflows` as a CI guard, and Renovate / Dependabot to bump
SHAs with the human-readable tag in the PR body.
*Concrete check:* `grep -E 'uses: [^@]+@v?[0-9]' .github/workflows/`
returns zero matches for non-`actions/*` actions.

**must:** no static cloud credentials in CI. Use **OIDC federation**
to AWS (`configure-aws-credentials` with `role-to-assume`), Azure
(Workload Identity Federation / Federated Identity Credentials), or
GCP (Workload Identity Federation). The federated trust is scoped to
*this repo, this workflow, this branch/environment*, not "anything
in the org." Long-lived `AWS_SECRET_ACCESS_KEY` / `AZURE_CLIENT_SECRET`
in repo secrets is a finding.

**must:** every workflow declares an explicit top-level
`permissions:` block, default `contents: read`, and elevates only
per-job. The org/repo default for `GITHUB_TOKEN` should be **read**.
The default of "write-all" is the single most common cause of
PR-triggered token theft.

**should:** workflows triggered by `pull_request_target` and
`workflow_run` are reviewed with extra scrutiny — they run with
write tokens and access to secrets against PR code. Treat them as
privileged.

**should:** **OpenSSF Scorecard** runs on a schedule and the score is visible in the README. Aim for ≥7 on every public repo and ≥8 on anything that ships an artifact others consume.

**The xz lesson.** The 2024 xz-utils backdoor (CVE-2024-3094) was not an action-pinning failure; it was a *trust-scope* failure. A sole maintainer was socially engineered into handing release authority to an attacker who spent two years building reputation. The take-away isn't "pin SHAs harder" — it's: **assume any single dependency can go hostile**, and design so one malicious upstream cannot reach prod without being seen. That means SBOM diff on every release, signing identity bound to a workflow not a person, two-party review on the build pipeline itself, and Rekor entries that make a silent re-tag detectable after the fact.

---

## 5. Branch protection / Rulesets / required checks

Branch protection is the enforcement layer that turns the rules
above from "nice idea" into "merge button is grey." On GitHub, use
**Repository Rulesets** (the modern replacement for classic branch
protection — they apply across branches by pattern, are
org-manageable, and have a bypass-actor audit trail).

- **must:** `main` (and any release branch) is protected with:
  required PR review, required status checks (build, test, SBOM,
  signing, Scorecard, action-pinning lint), required signed commits
  *or* signed merge commits, linear history (or merge-queue), no
  force-push, no deletion. Admins do **not** bypass — there is no
  `admin: true` PAT in CI.
- **must:** required status checks include the supply-chain jobs
  (signing succeeded, SBOM generated, no critical CVEs introduced
  by this PR via `dependency-review-action` or equivalent).
- **should:** use a **merge queue** (GitHub Merge Queue, GitLab
  Merge Trains, Bors-NG) for repos with >5 active contributors. It
  eliminates the "green PR + green main → red main after merge"
  race and lets you batch CI cost.
- **should:** **Dependabot** *or* **Renovate** is enabled, grouped
  by ecosystem, with auto-merge for patch updates that pass CI.
  Renovate is the more flexible of the two; Dependabot is zero-config
  and fine for most repos. Pick one.
- **avoid:** "required approval from CODEOWNERS" without actually
  populating `CODEOWNERS`. Empty CODEOWNERS is worse than none — it
  silently passes.

---

## 6. Progressive delivery: flags, canary, blue/green, shadow

The point of progressive delivery is to **decouple deploy from
release**. Deploy = the bits are on the server. Release = users see
them. Conflating the two is why deploys feel scary.

The toolkit, ordered by how much complexity each adds:

1. **Feature flags** (LaunchDarkly, Unleash, OpenFeature-compliant
   provider, or a homegrown table). Cheapest progressive-delivery
   tool by far. **Should be the default** for any user-visible
   change. The OpenFeature spec (CNCF) is the right abstraction —
   write code against `OpenFeature.GetClient()`, not a vendor SDK.
2. **Blue/green.** Two identical environments, switch the router.
   Right answer for stateful apps where canary is hard, and for
   anything where rollback must be instant. Cost: 2× steady-state
   infra during cutover.
3. **Canary.** Shift a small percentage of *production traffic* to
   the new version, watch SLOs, ramp or roll back. Right answer for
   stateless HTTP services with good metrics. **Argo Rollouts** or
   **Flagger** automate this against Prometheus / Datadog SLOs;
   either is a sensible default. Spinnaker is heavyweight and only
   pays off at multi-cluster, multi-cloud scale.
4. **Shadow / mirrored traffic.** Send a copy of prod traffic to the
   new version, *don't* return its responses. Highest-fidelity test
   for performance and correctness regressions on read paths.
   Mandatory for risky migrations (new query engine, new storage
   backend). Don't shadow write paths without idempotency keys.

- **must:** every production deploy has an automated rollback path
  (re-deploy previous digest, flip blue/green, `argo rollouts undo`).
  Manual `kubectl rollout undo` typed into a terminal at 3am is not
  a rollback path.
- **should:** canary analysis is **SLO-driven** (error rate,
  latency, saturation), not "wait 10 minutes and hope." Flagger and
  Argo Rollouts both wire to Prometheus/Datadog/CloudWatch for this.
- **prefer:** Argo Rollouts when you're already on Argo CD; Flagger
  when you're on Flux or have a service mesh (Istio, Linkerd) doing
  the traffic split. Roll-your-own only if you can name the SLO that
  triggers rollback and the metric pipeline is already there.
- **avoid:** "canary" that is actually just "deploy to one pod and
  see if it crashes." That is a smoke test, not a canary.

---

## 7. DORA — the only four metrics that matter

From *Accelerate* (Forsgren / Humble / Kim) and the annual DORA
reports: four metrics predict both delivery performance and
organizational performance, and almost nothing else does.

- **Deployment frequency** — how often you ship to prod.
- **Lead time for changes** — commit → prod.
- **Change-failure rate** — % of deploys that cause a user-visible
  incident.
- **Mean time to restore** (MTTR) — incident → resolved.

- **must:** these four are instrumented and visible to the team that
  owns the service. Not "the platform team has a dashboard
  somewhere" — the *delivery* team sees them.
- **should:** trend, not snapshot. A weekly average over a 4-week
  window is the smallest unit that's signal, not noise.
- **avoid:** vanity metrics that aren't on this list (PR count,
  story points, "developer productivity" composite scores). They
  optimize the wrong thing and have been repeatedly shown not to
  predict outcomes (DORA 2023, 2024 reports).

---

## 8. Pipeline hygiene

- **must:** runners are **ephemeral**. Every job starts on a fresh
  VM/container; nothing persists between runs except an explicit
  cache. Self-hosted persistent runners are a documented attack
  vector (GitHub's own security guide: "self-hosted runners on
  public repositories" is *the* warning section).
- **should:** if you must self-host (cost, GPU, network), use
  **ephemeral, just-in-time runners** (Actions Runner Controller on
  Kubernetes, GitLab autoscaling runners on EC2 spot,
  Buildkite-agent autoscaler). Never re-use a runner VM across
  workflows.
- **must:** caches are **per-branch with a fallback to default
  branch**, scoped by lockfile hash. Cross-PR cache poisoning is a
  real attack class — never accept a cache from an untrusted PR
  branch into a privileged workflow.
- **should:** **fail fast.** Lint and type-check before tests, tests
  before integration, integration before deploy. Matrix jobs use
  `fail-fast: true` unless you genuinely need full results.
- **should:** parallelize at the test-runner level, not by spawning
  more pipeline jobs (cheaper, simpler, fewer cache invalidations).
- **prefer:** runners placed in the same region/network as the
  systems they talk to. Cross-region pull-of-base-image is a silent
  cost and latency tax.
- **avoid:** secrets in environment variables that are echoed by
  any step. Mask, scope to the smallest job, and rotate. Better:
  fetch from a secret manager with OIDC at job start.

---

## 9. The "platform team owns the pipeline" anti-pattern

Conway's Law applies to pipelines. If the platform team writes and
owns every team's pipeline, you have built a queue: every change to
how a service is built or deployed must wait on a central team.
Lead time goes up, autonomy goes down, and the platform team becomes
the bottleneck the org was trying to avoid.

The right model (Team Topologies, *Accelerate*, ThoughtWorks Tech
Radar on "platform as a product"):

- The **platform team** owns *reusable workflows / shared templates /
  golden-path actions*, the runner fleet, the signing/SBOM/OIDC
  building blocks, and the policy that says "thou shalt sign."
- The **delivery team** owns *their* `.github/workflows/*.yml`,
  `gitlab-ci.yml`, etc., consuming the shared building blocks. They
  can fork the golden path when they need to and pay the cost of
  doing so.
- The platform team's product is *the paved road*, not *the only
  road*. If teams can't pave their own road when the paved one
  doesn't fit, you have a tollbooth, not a platform.

- **avoid:** a single monorepo of pipeline YAML maintained by
  platform, applied via include/extends to every service. That is
  the queue.
- **prefer:** versioned reusable workflows (`uses: org/.github/.../
  workflow.yml@v3`), service teams pin their version and upgrade on
  their own cadence.

---

## Sources

See [`03-ci-cd.sources.json`](./03-ci-cd.sources.json) for the full
list with URLs. Primary canon: SLSA spec, Sigstore docs, OpenSSF
Best Practices and Scorecard, GitHub Actions security hardening,
Argo CD / Flux / Argo Rollouts / Flagger docs, OpenFeature spec,
Tekton, *Accelerate* (Forsgren/Humble/Kim) and the DORA reports,
*Continuous Delivery* (Humble/Farley), Charity Majors and Liz
Fong-Jones on progressive delivery, the xz-utils CVE-2024-3094
postmortems, and ThoughtWorks Tech Radar on GitOps/Argo/Flux/
Sigstore/SLSA.
