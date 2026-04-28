# Chapter 06 — Security & Supply Chain

Opinionated, cloud-agnostic defaults for securing infrastructure and the
software supply chain feeding it. Threat-model first, then controls. Anything
you can't tie back to a threat in your model is cargo-culted.

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint.

---

## 1. Threat model first

**Without a threat model you are cargo-culting controls.** Every system that
holds data or runs code in production gets a written threat model — STRIDE for
classic asset/spoof/tamper analysis, LINDDUN when personal data is in scope.
Microsoft's STRIDE is still the cheapest first pass; LINDDUN (KU Leuven) covers
the privacy axes STRIDE skips. Adam Shostack's *Threat Modeling: Designing for
Security* is the standard reference; the Threat Modeling Manifesto authors agree
that the artefact matters less than the four questions: *What are we working on?
What can go wrong? What are we going to do about it? Did we do a good enough
job?*

**Do** keep the threat model in the repo (`docs/threat-model.md` or an
`.tm7` exported to PDF) and review it on every architectural change. Out-of-tree
threat models are out-of-date threat models.

**Do** map each identified threat to a control and a test. NIST SP 800-218
(SSDF) PW.1 explicitly requires this traceability.

**Don't** start a security review with a tool. Start with the data flow
diagram, trust boundaries, and assets.

---

## 2. Secrets management

There is no single right answer; the right answer is *"secrets never live in
git, never live in env vars baked into images, and never live unrotated."* Pick
based on workload location and rotation needs:

| Tool                         | Best when                                                      |
|------------------------------|----------------------------------------------------------------|
| HashiCorp Vault              | Multi-cloud, dynamic DB credentials, PKI issuance, SSH CA      |
| Cloud KMS / Key Vault / Secret Manager | Single-cloud, managed HSM, IAM-native auth          |
| External Secrets Operator    | K8s clusters pulling from any of the above into `Secret` objects |
| SOPS + age / KMS             | GitOps secrets at rest; small teams without a Vault to run     |

**Do** prefer the cloud-native KMS when you live in one cloud — IAM is already
your auth plane and you avoid running another stateful service. Vault wins the
moment you're multi-cloud, need short-lived database credentials, or need
PKI/SSH CA. ESO is plumbing, not a secret store: it reconciles from your real
store into Kubernetes `Secret` objects so apps stay vanilla.

**Do** use SOPS with `age` (or KMS-backed keys) for GitOps repos when you can't
yet run a secret manager. Plain-text YAML secrets in git is a P1 incident
waiting to happen; SOPS-encrypted YAML in git is acceptable and audited via
`git log`.

**Avoid** writing secrets into container images, baked AMIs, or Helm
`values.yaml` checked into VCS. If `git log -p | grep -iE 'password|secret|key'`
returns hits, you have a rotation event, not a code review comment.

**Do** rotate. A secret that is never rotated is a credential, not a secret. CIS
controls and PCI-DSS both require defined rotation periods; pick one (90 days
for human credentials, ≤24h for workload tokens) and enforce it via the secret
store, not a calendar reminder.

---

## 3. Workload identity — the "no static credentials" rule

**Static long-lived cloud credentials in CI or on workloads are a serious
incident.** This is the single highest-leverage rule in this chapter. The
Codecov 2021 breach, the 2022 CircleCI incident, and a long tail of S3-leak
postmortems all share the same root cause: a long-lived `AKIA…` or service
account JSON sitting somewhere it shouldn't.

**Do** use workload identity everywhere it exists:

- **EKS** → IRSA (IAM Roles for Service Accounts) or EKS Pod Identity.
- **GKE** → Workload Identity (KSA ↔ GSA binding).
- **AKS** → Azure AD Workload Identity (the older "pod identity" is deprecated).
- **GitHub Actions / GitLab CI / Buildkite** → OIDC federation to cloud IAM
  (`aws-actions/configure-aws-credentials` with `role-to-assume`,
  `azure/login` with `federated`, `google-github-actions/auth` with WIF).

**Do** scope the trust policy to a specific repo, branch, environment, and (for
PR workflows) job — `token.actions.githubusercontent.com` claims include all of
these. A trust policy that accepts any workflow in the org is functionally an
access key.

**Don't** create an IAM user for a service. The right cardinality of IAM users
in a modern AWS account is "humans who break-glass, plus zero." Anything else
gets a role assumed via OIDC or instance/pod identity.

---

## 4. Policy-as-code

**Do** enforce platform invariants with admission control, not wiki pages.
Choose between OPA Gatekeeper and Kyverno based on team and policy shape:

- **OPA / Gatekeeper / Conftest.** Rego is a real language; expressive enough
  for cross-resource policies and reusable across Terraform (`conftest`), CI,
  and runtime (Gatekeeper). Pay the Rego learning tax once.
- **Kyverno.** YAML-native, no new language, *mutates* as well as validates.
  For "block hostPath, require resource limits, mirror images to internal
  registry" workloads it is dramatically simpler than Rego and is what most
  teams should reach for first. ThoughtWorks' Tech Radar moved Kyverno to
  *Adopt* for K8s policy enforcement.

Use **Conftest** to apply Rego in CI against Terraform plans, Dockerfiles,
Kubernetes manifests, and Helm charts *before* they reach the cluster.
**Polaris** is a good lint pass for resource-request hygiene; **kube-bench**
runs the CIS Kubernetes Benchmark against a live cluster; **kube-hunter** is
useful pre-prod, noisy in prod.

**Do** ship policies as a versioned bundle (OCI artifact or git submodule) so
the same rules run locally (`conftest test`), in CI, and in admission. Drift
between layers is how exceptions become permanent.

---

## 5. Benchmarks — what actually matters

The CIS Benchmarks (Kubernetes, Docker, Linux, AWS/Azure/GCP) are the closest
thing to a vendor-neutral baseline. They are also full of low-value checks
shipped with high severities. Internalise the controls that move the needle:

- **K8s:** API server `--anonymous-auth=false`, audit logging on, RBAC
  on, etcd encryption at rest, PSA `restricted` (or equivalent Kyverno/Gatekeeper
  policies) in app namespaces, network policies default-deny.
- **Docker / containers:** non-root `USER`, read-only root FS, no
  `--privileged`, no `hostNetwork`/`hostPID`, drop all capabilities and
  re-add only what's needed, seccomp `RuntimeDefault`.
- **Cloud:** root account MFA + no access keys, CloudTrail / Activity Log /
  Cloud Audit Logs enabled in *every* region/subscription/project, default
  encryption on object storage, no public-by-default buckets/blobs.

**Avoid** treating "100% CIS pass" as the goal. The goal is a low-noise report
where every remaining finding has a written exception. Liz Rice's *Container
Security* makes this point at length.

---

## 6. SBOMs

**Do** generate an SBOM for every artifact you publish (containers, binaries,
Helm charts). Prefer **SPDX 2.3+** or **CycloneDX 1.5+** — both are ISO/ECMA
standards and consumed by every scanner that matters; pick one per org and
stick to it. CycloneDX has richer vulnerability/VEX support; SPDX has the
broader regulatory footprint (US EO 14028, EU CRA).

**Do** generate at *build* time from source (Syft, `cyclonedx-gomod`,
`cyclonedx-npm`) — scanning a finished image for an SBOM is a fallback, not
the primary source. Attach the SBOM as a signed attestation alongside the
image: `cosign attest --predicate sbom.spdx.json --type spdx <image>`. This
puts the SBOM in the OCI registry next to the artifact and in the Rekor
transparency log.

**Don't** ship an SBOM you don't ingest. An unread SBOM is a compliance prop.
Pipe them into a vulnerability database (Dependency-Track, Grype DB, GUAC) so
"is log4j 2.14 anywhere in prod?" becomes a query, not a fire drill.

---

## 7. Container & dependency scanning

The market has converged: **Trivy**, **Grype**, **Snyk**, and **Anchore** all
do roughly the same job — pull NVD/GHSA/vendor advisories, match against
package metadata. Differences that matter:

- **Trivy** (Aqua, Apache-2.0): broadest scope (images, IaC, secrets, K8s,
  SBOMs), default for OSS pipelines.
- **Grype** + **Syft** (Anchore, Apache-2.0): cleanest SBOM-in / findings-out
  separation; pairs naturally with cosign attestations.
- **Snyk** (commercial): best dev-loop UX and reachability analysis for app
  deps; license cost only justifies itself if devs actually use the IDE/PR
  integration.
- **Anchore Enterprise**: policy engine over Grype; useful at FedRAMP scale.

**Do** scan in *three* places, not one: pre-merge (PR check on the diff),
post-build (full image scan, blocks promotion on `Critical` / `High`), and
continuously in the registry (vulnerabilities published *after* you build still
matter — that's how Log4Shell hit you).

**Don't** fail builds on `Low`/`Medium` without a fix path. Alert without a
fix path is noise; noise trains engineers to bypass the gate. Fail on
`Critical` always, `High` when a fixed version exists.

---

## 8. SLSA & build provenance

(Cross-ref Chapter 03 — CI/CD.) **Do** target **SLSA Build Level 3** for
anything that ships to production: hosted, hardened builder; signed
provenance; non-falsifiable. The provenance attestation answers *who built
this, from what source, with what builder, at what time* — and a verifier
(`cosign verify-attestation --type slsaprovenance`, `slsa-verifier`) checks
those claims against a policy before deploy. A signature on a binary with no
provenance only tells you *someone* signed it.

**Do** use the GitHub `slsa-github-generator` reusable workflows (or
equivalents on GitLab / Buildkite) — rolling your own L3 builder is a
multi-quarter project you don't need to do.

---

## 9. Sigstore / signing

**Do** sign every container image, Helm chart, and release artifact with
**cosign**. Prefer **keyless (identity-based) signing** via Fulcio + Rekor over
long-lived KMS keys: the signer's OIDC identity (CI workload, developer SSO)
is bound into the certificate, the certificate is short-lived (10 minutes), and
the signature is logged to the Rekor public transparency log. There is no key
to rotate, leak, or HSM-manage.

How it works in one paragraph: the signer authenticates to **Fulcio** with an
OIDC token, Fulcio issues an X.509 cert binding the OIDC identity to an
ephemeral key, the signer signs, the signature + cert go to **Rekor** which
returns an inclusion proof. Verification re-checks the Rekor entry and the
Fulcio cert chain; a verifier policy asserts *which identity* (e.g.
`https://github.com/myorg/myrepo/.github/workflows/release.yml@refs/tags/*`) is
allowed to sign which artifact. Dan Lorenc's Sigstore talks and the Sigstore
docs are the canonical references.

**Do** pin the verification policy in the deploy-time admission controller
(cosign policy-controller, Kyverno `verifyImages`). An unverified signature is
a decoration.

**Avoid** key-based cosign signatures unless you have an air-gapped reason —
you've reinvented the rotation problem you adopted Sigstore to escape.

---

## 10. Network — zero-trust by default

**Do** adopt the **BeyondCorp** posture: no network is trusted, every request
is authenticated and authorized at the application layer with device + user
identity. The original Google papers (2014–2017, six in total) are still the
clearest articulation; Cloudflare's and Google's Zero Trust whitepapers are the
modern productised form.

Practical defaults:

- **No public IPs by default.** Workloads sit on private subnets; ingress is
  via load balancers or identity-aware proxies; egress goes through a NAT or
  egress gateway with allow-listed destinations. NIST SP 800-204D codifies
  this for microservices.
- **mTLS between services.** A service mesh (Istio, Linkerd, Cilium) gives
  you this for free; without a mesh, do it in the application or accept the
  risk in writing. The "do you need a mesh?" debate (cross-ref Chapter 04)
  resurfaces here — if mTLS-everywhere is a hard requirement, the mesh
  becomes much easier to justify.
- **Default-deny network policies** in every namespace. `kubectl get
  networkpolicy -A` returning "No resources found" in a multi-tenant cluster
  is a finding.

**Don't** rely on VPC/subnet boundaries as a security control. They are a
blast-radius control, not an authentication control.

---

## 11. Access — IAP > VPN

**Do** front internal apps with an **identity-aware proxy** instead of a flat
VPN. Cloudflare Access, Google IAP, Pomerium, and Tailscale (with
serve/funnel) all enforce per-request identity + device posture; a VPN gives
the laptop the network, then trusts everything on it. **Teleport** is the
right answer for SSH/Kubernetes/DB access with session recording and short-
lived certs — `~/.ssh/authorized_keys` on prod hosts is a finding.

**Do** require hardware-backed WebAuthn (YubiKey, platform authenticator) for
all human access to production. TOTP is phishable; SMS is worse.

---

## 12. Vulnerability management

**Do** publish a written SLA and enforce it via tooling:

| Severity | Patch SLA (internet-facing) | Patch SLA (internal) |
|----------|-----------------------------|----------------------|
| Critical | 24–72h                      | 7d                   |
| High     | 7d                          | 30d                  |
| Medium   | 30d                         | 90d                  |
| Low      | best-effort / next release  | best-effort          |

Numbers are illustrative — pick yours, write them down, dashboard them.
FedRAMP and PCI both require documented timelines; SOC 2 auditors will ask.

**Do** enforce the rule: *an alert without a fix path is noise.* If the only
remediation is "wait for upstream," the finding belongs in a tracked
exception with an expiry, not in the daily Slack noise.

---

## 13. Compliance — what infra actually owns

Most of ISO 27001 / SOC 2 / FedRAMP / PCI is paperwork the security team
drives. The infra-engineering deliverables are narrow and predictable:

- **Audit logging** on for the control plane: CloudTrail (all regions, log
  file validation on, immutable bucket), Azure Activity Log + Diagnostic
  Settings, GCP Cloud Audit Logs (Admin Activity + Data Access where scoped).
  Centralise to a write-once store outside the producing account.
- **Encryption at rest** with customer-managed keys where the framework
  requires it (PCI, FedRAMP Moderate+); default cloud-managed keys are fine
  for SOC 2.
- **Access reviews** — quarterly export of IAM/RBAC, signed off.
- **Change management trail** — every prod change traceable to a merged PR
  and a deployment record. GitOps gives you this almost free.
- **Backups + tested restore** — untested backups are not backups. Run a
  restore drill quarterly and record the RTO/RPO actually achieved.

**Avoid** building a separate "compliance pipeline." If your normal pipeline
can't produce the evidence (signed builds, SBOMs, audit logs, change tickets),
fix the pipeline.

---

## 14. Runtime security & audit

**Do** run a runtime threat-detection agent on production nodes — **Falco**
(CNCF graduated) is the cloud-agnostic default, with rules for shell-in-
container, unexpected outbound connections, sensitive mount access, and
kubectl exec. Tetragon (eBPF, Cilium) is the modern alternative. Pipe events
to the same SIEM as your audit logs.

**Do** wire audit logs into detections, not just storage. CloudTrail in S3
that no one queries detected zero breaches in 2024. The CNCF TAG-Security
runtime papers and Falco's own "Falco rules in production" guidance are good
starting libraries.

---

## 15. Shift left, shift everywhere

"Shift left" is half the story. The honest reframe is **shift everywhere**:

| Stage   | Controls                                                            |
|---------|---------------------------------------------------------------------|
| Dev     | pre-commit (gitleaks, detect-secrets), IDE scanners, threat model   |
| Build   | SBOM, dep scan, image scan, SLSA provenance, cosign sign            |
| Deploy  | admission policy (Kyverno/Gatekeeper), signature + provenance verify |
| Runtime | Falco/Tetragon, audit logs, anomaly detection, IAP enforcement      |

Each stage catches what the previous one missed. Skipping runtime because
"we shifted left" is how the xz backdoor would have shipped — the malicious
code passed code review, passed scanners, and only ran in specific build
contexts.

---

## 16. Lessons from real supply chain incidents

Read the postmortems. Three are mandatory:

- **xz-utils backdoor (CVE-2024-3094, March 2024).** Andres Freund noticed a
  500ms ssh login regression and pulled the thread to a multi-year social-
  engineering operation against a sole maintainer. Lessons: solo-maintainer
  dependencies are supply-chain risk; build-time autotools/m4 magic is a
  blind spot for source-level review; reproducible builds + SLSA L3 would
  have made the injection visible.
- **SolarWinds Sunburst (Dec 2020).** Build-system compromise injected
  malicious code into signed Orion releases. Mandiant and CISA reports are
  the primary sources. Lessons: signing a compromised build is worse than
  not signing — provenance about *what the build did* (SLSA) is what you
  needed, not just a signature.
- **Codecov bash-uploader (April 2021).** Attacker modified the `bash`
  uploader to exfiltrate environment variables (CI secrets) from thousands
  of pipelines. Lessons: never `curl | bash` a third-party uploader in CI
  with secrets in the environment; OIDC federation removes the secret to
  steal; SRI-style hash pinning of CI scripts is cheap insurance.

**Do** add a "what would have caught xz here?" question to architecture
reviews of any system that ingests source from outside the org.

---

## 17. Checklist (PR / repo gate)

- [ ] Threat model in-repo, updated this quarter.
- [ ] No long-lived cloud credentials in CI; OIDC trust scoped to repo+branch+environment.
- [ ] Secrets in a managed store or SOPS-encrypted; `gitleaks` clean.
- [ ] Admission policy (Kyverno/Gatekeeper) blocks privileged pods, hostPath, untrusted registries, unsigned images.
- [ ] SBOM (SPDX/CycloneDX) generated at build, attached as cosign attestation, ingested by a vuln DB.
- [ ] Image scan in PR + post-build + registry; fail policy documented.
- [ ] Cosign signature + SLSA L3 provenance produced and verified at deploy, pinned to expected signer identities.
- [ ] Audit logs centralised, immutable, queried by detections.
- [ ] Default-deny NetworkPolicy per app namespace; no public IPs by default; egress allow-listed.
- [ ] IAP for human access; WebAuthn MFA; no flat VPN.
- [ ] Falco (or equivalent) on every prod node; rules tuned, alerts routed.
- [ ] Vuln SLA written, dashboarded, exceptions tracked with expiry.

---

*Cross-references: Chapter 03 (CI/CD — SLSA, signing in pipeline), Chapter 04 (Kubernetes — admission, NetworkPolicy, mesh), Chapter 05 (Observability — audit logs, SIEM ingestion).*
