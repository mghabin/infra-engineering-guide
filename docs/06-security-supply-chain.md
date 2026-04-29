# Chapter 06 — Security & Supply Chain

Opinionated, cloud-agnostic defaults for securing infrastructure and the software supply chain feeding it. Threat-model first, then controls. Anything you can't tie back to a threat in your model is cargo-culted.

> Conventions: **Do** = required default. **Don't** = reject in review unless a written exception exists. **Prefer** = strong default; deviate only with measurement or a documented constraint.

---

## 1. Threat model first

- **Without a threat model you are cargo-culting controls.** STRIDE for classic asset/tamper analysis, LINDDUN when personal data is in scope. The four questions from the *Threat Modeling Manifesto* are the irreducible core: *what are we working on, what can go wrong, what are we going to do about it, did we do a good enough job?*
- **Do** keep the threat model in the repo and review it on every architectural change. Out-of-tree threat models are out-of-date.
- **Do** map each threat to a control and a test. NIST SP 800-218 (SSDF) PW.1 requires this traceability (https://csrc.nist.gov/publications/detail/sp/800-218/final).
- **Don't** start a security review with a tool. Start with the data-flow diagram, trust boundaries, and assets.

---

## 2. Secrets management

- The right answer is *"secrets never live in git, never live in env vars baked into images, and never live unrotated."* Pick the store based on workload location and rotation needs:

| Tool                                   | Best when                                                       |
|----------------------------------------|-----------------------------------------------------------------|
| HashiCorp Vault                        | Multi-cloud, dynamic DB credentials, PKI issuance, SSH CA       |
| Cloud KMS / Key Vault / Secret Manager | Single-cloud, managed HSM, IAM-native auth                      |
| External Secrets Operator              | K8s pulling from any of the above into `Secret` objects         |
| SOPS + age / KMS                       | GitOps secrets at rest; small teams without a Vault to run      |

- **Do** prefer cloud-native KMS in single-cloud — IAM is already your auth plane. Vault wins for multi-cloud, dynamic DB credentials, or PKI/SSH CA. ESO is plumbing, not a secret store. SOPS+age is acceptable for GitOps when you can't yet run a secret manager.
- **Avoid** writing secrets into container images, baked AMIs, or `values.yaml` checked into VCS. If `git log -p | grep -iE 'password|secret|key'` returns hits, you have a rotation event, not a code review comment.
- **Do** rotate. CIS Controls v8 (Control 5) and PCI-DSS v4.0 §8 require defined rotation periods; pick one (≤90 days human, ≤24h workload) and enforce via the store, not a calendar. (Human-credential rotation cadence and authenticator strength are owned by [ch12 §3 Authentication strength](./12-identity.md#3-authentication-strength); this rule is the workload half.)

---

## 3. Workload identity — the "no static credentials" rule

> Workforce (human) identity — engineers, admins, contractors, JML, MFA, break-glass — is owned by [ch12](./12-identity.md). This section is exclusively about non-human principals (pods, CI jobs, batch workers).

- **Static long-lived cloud credentials in CI or on workloads are a serious incident.** The Codecov 2021, CircleCI 2022, and a long tail of S3-leak postmortems all share the same root cause: a long-lived `AKIA…` or service-account JSON sitting somewhere it shouldn't.
- **Do** use platform workload identity. The mechanisms differ — they are not interchangeable:
  - **EKS IRSA** — cluster's OIDC discovery URL registered as an IAM OIDC provider; kubelet projects an SA JWT into the pod, AWS SDK calls `sts:AssumeRoleWithWebIdentity`. Trust: per-SA role with `sub = system:serviceaccount:<ns>:<sa>`.
  - **EKS Pod Identity** (2023) — on-node `eks-pod-identity-agent` DaemonSet brokers credentials over a link-local endpoint; trust is an EKS *Pod Identity Association*, not an IAM OIDC trust policy. Avoids per-cluster OIDC provider sprawl.
  - **GKE Workload Identity** — KSA annotated with a GSA; GKE's `gke-metadata-server` intercepts metadata calls and exchanges the projected SA token for a Google access token via STS. Trust: the KSA↔GSA binding.
  - **AKS Workload Identity** — Azure AD *federated identity credentials* on a user-assigned managed identity reference the AKS OIDC issuer + KSA `sub`; the projected SA token is exchanged at `login.microsoftonline.com` for an AAD token. Replaces deprecated AAD Pod Identity.
  - **SPIFFE/SPIRE** — workload identities (SVIDs, X.509 or JWT) issued by a SPIRE server based on attested workload selectors (k8s SA, host UID, AWS instance identity). Right answer when the estate is **multi-cloud or mixed cloud + on-prem** and a single uniform identity primitive is needed. NIST SP 800-207A (Sept 2023, *A ZTA Model for Cloud-Native Multi-Cloud*) explicitly names SPIFFE as the recommended cross-cloud workload-identity primitive.
- **Do** use OIDC federation from CI to cloud IAM. Scope with provider-specific claims (GitHub Actions docs *Security hardening with OpenID Connect*):
  - **AWS IAM** — trust policy conditions on `aud = sts.amazonaws.com` and `token.actions.githubusercontent.com:sub` of the form `repo:ORG/REPO:ref:refs/heads/main` or `repo:ORG/REPO:environment:prod`. For reusable workflows, condition on `job_workflow_ref`.
  - **GCP Workload Identity Federation** — Workload Identity Pool + Provider with `attribute.repository`, `attribute.ref`, `attribute.workflow_ref`; bind the SA with an IAM condition on those attributes.
  - **Azure** — federated credential on the app registration / managed identity scoped by `subject` (e.g., `repo:ORG/REPO:environment:prod`) and `issuer` = the GitHub OIDC issuer. Azure federated credentials accept **any OIDC issuer** (up to 20 per identity) — federation is not GitHub-only.
- **Note:** GitHub OIDC claims include no stable per-job identifier you can pin in a cloud trust policy. Scope on `repository`, `ref`, `environment`, `workflow_ref`; use protected `environments` to gate prod role assumption.
- **Do** federate non-cloud workloads too. **AWS IAM Roles Anywhere** (GA 2022) lets on-prem or third-cloud workloads exchange an X.509 cert from your private CA for short-lived AWS credentials — there is no longer a reason for an `AKIA…` to exist on a workload anywhere. GCP's recommended posture is the org policy `iam.disableServiceAccountKeyCreation` (default-on for new orgs); SA JSON keys are an exception that needs justification, not a default.
- **Don't** create an IAM user for a service. The right cardinality of IAM users — and SA keys, and SP client secrets — in a modern cloud account is "humans who break-glass, plus zero." (Break-glass identity lifecycle, sealing, and use-monitoring are owned by [ch12 §8 Break-glass](./12-identity.md#8-break-glass).)

---

## 4. Policy-as-code

- **Do** enforce platform invariants with admission control, not wiki pages. As of K8s 1.30 (April 2024) the choice is three-way, not binary:
  - **CEL ValidatingAdmissionPolicy** (GA in K8s 1.30, `admissionregistration.k8s.io/v1`). In-process, no webhook to operate, no CRD bundle to install. Right tool for simple per-resource invariants — required labels, image-registry allowlists, replicas ≤ N, banned fields. Gaps: **no mutation, no image-signature verification, no cross-resource queries, no external data**. Use it where it fits and you remove an admission-webhook availability dependency.
  - **Kyverno.** YAML-native, no new language, *mutates* as well as validates, `verifyImages` ships in-tree (cosign + notary, keyless or key-based, attestation predicate matching). Best fit for "block hostPath, require resource limits, mirror images to internal registry, verify signatures" workloads.
  - **OPA / Gatekeeper / Conftest.** Rego is expressive enough for cross-resource policies and reusable across Terraform (`conftest`), CI, and runtime (Gatekeeper). Pay the Rego learning tax once. Image-signature verification is **not** in-tree — bolt on Sigstore Policy Controller or **Ratify** (CNCF Sandbox, accepted Sept 2023) as a Gatekeeper external-data provider. The 2024 maturation of Ratify and Policy Controller turns the verification gap from a *capability* gap into an *integration tax* — Gatekeeper-on-Ratify is comparable to Kyverno-`verifyImages` if Gatekeeper is already in production. Migrate purely for signature verification only when no admission stack exists yet.
- **Cedar** (AWS, open-sourced May 2023; formal-methods-verified policy language behind Amazon Verified Permissions) is for **application-level authorization** decisions ("which user can do what to which resource"), not cluster admission. **Prefer** Cedar over reaching for Rego inside an application's authz path; keep Rego/Kyverno/CEL on the platform side. This prevents Rego sprawl into product code.
- Use **Conftest** to apply Rego in CI against Terraform plans, Dockerfiles, Kubernetes manifests, and Helm charts *before* they reach the cluster. **Polaris** is a useful resource-request lint pass; **kube-bench** runs the CIS Kubernetes Benchmark against a live cluster.
- **Canonical:** ch06 §4 owns the policy-as-code decision for the guide:
  pick **one** engine per estate (CEL VAP, Kyverno, or OPA/Gatekeeper),
  **write policies once** and **enforce in CI *and* at admission** from
  the **same versioned bundle** (OCI artifact or git submodule). Drift
  between layers is how exceptions become permanent. ch02 §6 (Conftest
  in IaC), ch04 §7 (Kubernetes admission), ch07 §16
  (firewall-as-code), and ch10 (tag enforcement) cite this rule rather
  than re-deciding it. Two engines on one estate is a finding — pick
  one and migrate.

---

## 5. Benchmarks — what actually matters

- The CIS Benchmarks (Kubernetes, Docker, Linux, AWS/Azure/GCP — https://www.cisecurity.org/cis-benchmarks) are the closest thing to a vendor-neutral baseline — and full of low-value checks shipped with high severities. Internalise the controls that move the needle:
  - **K8s:** API server `--anonymous-auth=false`, audit logging on, RBAC on, etcd encryption at rest, PSA `restricted` (or equivalent Kyverno/Gatekeeper/CEL policies) in app namespaces, default-deny NetworkPolicy.
  - **Containers:** non-root `USER`, read-only root FS, no `--privileged`, no `hostNetwork`/`hostPID`, drop all capabilities and re-add only what's needed, seccomp `RuntimeDefault`.
  - **Cloud:** root account MFA + no access keys, CloudTrail / Activity Log / Cloud Audit Logs enabled in *every* region/subscription/project, default encryption on object storage, no public-by-default buckets/blobs. (MFA strength for human/root access — phishing-resistant FIDO2/WebAuthn at AAL3 — is owned by [ch12 §3 Authentication strength](./12-identity.md#3-authentication-strength).)
- **Avoid** treating "100% CIS pass" as the goal. The goal is a low-noise report where every remaining finding has a written exception.

---

## 6. Package ingestion controls

- Most supply-chain incidents enter before any scanner runs. Ingestion hygiene is the cheapest defense against dependency confusion (Birsan, 2021), typosquats (`event-stream`, `ua-parser-js`, `colors.js`), and namespace hijacks. Per OpenSSF *Source Code Management Best Practices* and OWASP *Dependency Management Cheat Sheet*:
- **Do** front every public registry with an internal proxy/mirror — Artifactory, Nexus, GitHub Packages, AWS CodeArtifact, GCP Artifact Registry. Cache-on-first-use, retention beyond upstream yanks, single chokepoint to allowlist sources.
- **Do** reserve your org's namespace on every public registry you publish to (`@yourorg/*` on npm, `com.yourorg.*` on Maven Central, `yourorg-*` prefix on PyPI). Unclaimed namespaces are how dependency-confusion attacks succeed.
- **Do** pin and lock. Lockfiles committed and verified in CI (`npm ci`, `pip install --require-hashes`, `go mod verify`). Floating ranges in production manifests are a finding.
- **Do** allowlist sources, not deny known-bad. `npm config set registry` pointing only at the proxy; `pip --index-url` constrained; Renovate/Dependabot gated on approved registries. Block direct installs from arbitrary VCS URLs.
- **Do** require human review on first introduction of a new external package. Socket, Snyk Advisor, `deps.dev` surface package age, maintainer count, install scripts, and capability signals before merge.
- **Do** apply the same ingestion discipline to **runtime-fetched assets** — third-party JS, fonts, CSS, container base images pulled at runtime. Self-host or cache; if you can't, pin with **Subresource Integrity (SRI)** hashes and a strict `Content-Security-Policy`. The polyfill.io 2024 incident (§18) exists because a CDN-served `<script src=…>` was implicitly trusted forever.
- **Don't** `curl | bash` third-party install scripts in CI. Vendor or hash-pin them — see §18 (Codecov).

---

## 7. SBOMs

- **Do** generate an SBOM for every artifact you publish (containers, binaries, Helm charts). Acceptable formats: **SPDX 2.3** (ISO/IEC 5962:2021) or **3.0** (April 2024, profile model — Core/Software/Security/Build/AI/Dataset/Licensing); **CycloneDX 1.5** or **1.6** (April 2024, adds CBOM, Attestations, lifecycle phases). CycloneDX has richer VEX support; SPDX has the broader regulatory footprint (US EO 14028, EU CRA).
- **Prefer** the newest minor your toolchain ingests cleanly. As of mid-2025, scanner support for SPDX 3.0 still lags the spec — verify Trivy/Grype/Dependency-Track support before mandating 3.0 across the estate; SPDX 2.3 + CycloneDX 1.5 remain a safe floor.
- **Do** match format to workload: **SPDX 3.0 AI/Dataset profiles** or **CycloneDX 1.6 ML-BOM** for AI workloads; **CycloneDX 1.6 CBOM** for cryptographic-supply-chain inventory (post-quantum readiness, FIPS module tracking).
- **Do** generate at *build* time from source (Syft, `cyclonedx-gomod`, `cyclonedx-npm`). Scanning a finished image is a fallback — the build-time graph sees test/build deps and exact resolved versions the binary won't reveal.
- **Do** attach the SBOM as a signed in-toto attestation: `cosign attest --predicate sbom.spdx.json --type spdx <image>`. Storage semantics matter:
  - The attestation is stored as an **OCI-associated artifact** discovered via the *Referrers API* (OCI Image Spec 1.1) — i.e., another blob in the registry tagged/referred to from the subject image's digest.
  - **Rekor** records the *signing event* (signature + cert + hash of the DSSE envelope) in an append-only Merkle log. Rekor is **not** the storage backend for the SBOM payload itself; the registry is.
- **Do** publish **VEX** statements (Vulnerability Exploitability eXchange) alongside the SBOM. **OpenVEX** (OpenSSF, JSON-LD, CISA-aligned, SBOM-agnostic) is the interoperable default and is consumed by Trivy, Grype, OSV-Scanner, and Dependency-Track. CycloneDX VEX or CSAF VEX are acceptable when a downstream pipeline already standardises on them. Per CISA *Minimum Requirements for VEX* (April 2023), the four required statuses are `not_affected`, `affected`, `fixed`, `under_investigation`. A scanner finding without a corresponding supplier VEX statement of `affected` or `under_investigation` is operating on incomplete data.
- **Do** ingest SBOMs you ship. Pipe them into a vulnerability database so "is log4j 2.14 anywhere in prod?" is a query, not a fire drill. The open default is **OSV.dev** (Google/OpenSSF) — broad ecosystem coverage (PyPI, npm, Go, Maven, NuGet, RubyGems, crates.io, Linux distros, OSS-Fuzz) consumed by **OSV-Scanner** (1.0 in 2024, accepts SPDX/CycloneDX, emits OpenVEX-aware results). Self-host the aggregation layer in **Dependency-Track** or **GUAC**; reserve Snyk/Anchore Enterprise for teams that want vendor-supported reachability and policy.
- **EU Cyber Resilience Act (Regulation (EU) 2024/2847).** Entered into force **10 December 2024**. Reporting obligations apply from **11 September 2026**; main obligations from **11 December 2027**. Scope is "products with digital elements" placed on the EU market — interpreted broadly: commercial software, firmware, and many SaaS-adjacent components; explicit carve-outs for FOSS not provided in the course of a commercial activity, medical devices (MDR), and cars (UN R155). Three obligations directly touch infra:
  - SBOM in machine-readable format must be available to market-surveillance authorities on request — operationally, SBOM generation and retention must be a property of the build pipeline, not a one-off, and must remain queryable for the declared support period.
  - Free security updates for the declared product support period — version policy and EOL dates become a regulated commitment, not a marketing decision; pin them in the product's CRA conformance documentation, not just in a release-notes blog.
  - 24-hour reporting of actively-exploited vulnerabilities (and severe incidents) to ENISA via the single reporting platform — incident-response runbooks need an ENISA-notification lane alongside CISA/NCSC, and the 24-hour clock starts from awareness, not from triage completion.
- **Don't** confuse SBOM with VEX or with provenance. SBOM lists ingredients; VEX states whether a known CVE applies to the way you used them; SLSA provenance describes how the artifact was built. All three are required and none substitutes for the other two.

---

## 8. Container & dependency scanning

- The market has converged: **Trivy**, **Grype**, **Snyk**, **Anchore**, **OSV-Scanner** pull NVD/GHSA/OSV/vendor advisories and match against package metadata. Trivy = broadest scope; Grype+Syft = cleanest SBOM-in/findings-out; Snyk = best dev-loop UX + reachability; Anchore Enterprise = policy engine at FedRAMP scale; OSV-Scanner = cleanest open pipeline when SBOMs and OpenVEX are first-class inputs.
- **Do** scan in *three* places: pre-merge (PR diff), post-build (full image), and continuously in the registry — vulnerabilities published *after* you build still matter (that's how Log4Shell hit you).
- **Don't** gate purely on CVSS severity. Severity is the worst predictor of "fix this now" we have:
  - **CVSS version matters.** FIRST published **CVSS v4.0** on 1 Nov 2023, with explicit nomenclature: **CVSS-B** (Base), **CVSS-BT** (Base+Threat), **CVSS-BE** (Base+Environmental), **CVSS-BTE** (Base+Threat+Environmental). Where vendors publish v4.0 vectors, **gate on CVSS-BTE, not CVSS-B**. Treat the v4.0 combination *Automatable=Yes* + *Exploit Maturity=Attacked* as functionally equivalent to a CISA KEV listing for SLA purposes.
  - **Reachability.** Does any code path actually call the vulnerable function? (Snyk Reachable, Endor Labs, Semgrep Supply Chain, OSV call-graph.)
  - **Exploitability.** Is the CVE on **CISA KEV**? Is **EPSS** above your threshold? EPSS is best applied as a **percentile**, not a flat probability — most CVEs sit at EPSS ≪ 0.1 and a flat 0.5 cut excludes most of the population. A defensible policy: *KEV ∪ (EPSS-percentile ≥ 95) = must-fix-this-week; EPSS-percentile ≥ 88 OR EPSS ≥ 0.1 = high-priority*. Calibrate to remediation capacity; whatever number you pick, write it down.
  - **Exposure.** Internet-facing service vs. internal batch vs. base-image noise that never executes.
  - **VEX status.** A supplier `not_affected` VEX with a justification suppresses the finding; an `under_investigation` keeps it visible but off the SLA clock.
  - **Fix availability.** No fixed version = exception with expiry, not a build break.
- A reasonable default policy: **fail** on (KEV ∪ (CVSS-BTE ≥ High ∧ reachable ∧ fix-available ∧ no `not_affected` VEX)) for internet-facing services; **alert** with SLA on the rest; **track** base-image CVEs separately and fix by bumping the base image.

---

## 9. SLSA & build provenance

- (Cross-ref Chapter 03.) Target the right **SLSA v1.2 level** for the artifact's blast radius. The spec at `slsa.dev/spec` is now Version 1.2 and defines **two tracks**: a Build Track (was the entirety of v1.0) and a **Source Track** (added in v1.1, refined in v1.2) covering producing source, verifying source, and assessing source-control systems.
- **Build Track levels** (per `slsa.dev/spec/v1.2/requirements`):
  - **Build L1** — provenance exists, describes how the artifact was built. No integrity guarantees.
  - **Build L2** — *signed* provenance generated by a *hosted* build platform. Tampering after the build is detectable; platform identity verifiable. Forgery by a malicious build *step* still in scope.
  - **Build L3** — adds **hardened** platform properties: build runs are isolated from each other, and **signing material is not accessible to user-defined build steps** (a malicious step cannot forge provenance for a different artifact). This is the bar that defends against in-build tampering.
- **Source Track levels** (the direct response to xz-class attacks):
  - **Source L1** — version control with retained history.
  - **Source L2** — change history is authenticated to a verified contributor identity (signed commits or platform-verified author identity).
  - **Source L3** — every change is **reviewed by a second authorised party** before merge, branch-protection prevents history rewrites, **release tags are protected and signed**, **binary artifacts in source are policy-controlled** (no opaque blobs land in the tree without justification), and the source-control system itself is assessed against the spec's organisational requirements (retained logs, two-person admin actions, separation between source-control admin and signing-key custody).
- **Canonical (ch06):** ch06 owns the SLSA floor for production artifacts.
  **Build L3 + Source L3 is the target for high-risk / regulated production
  workloads** (per `slsa.dev/spec/v1.2/levels`); **Build L2 + Source L2** is
  a defensible interim for lower-stakes contexts (internal-only,
  pre-revenue, experimental) on a documented ramp to L3. Build L1 / Source
  L1 are the permitted entry point for new repos and pre-platform work.
  ch03 (CI/CD) implements the build-side mechanics and §3 of that chapter
  cites this floor rather than re-deciding it.
- **Do** target **Build L3** for production artifacts. Three production paths meet the L3 properties on hosted platforms:
  - **GitHub Artifact Attestations** (GA 25 June 2024) — `actions/attest-build-provenance` and `actions/attest-sbom` produce SLSA v1 provenance signed via Sigstore (public-good for public repos, GitHub's private Sigstore for private repos), verifiable with `gh attestation verify -R <org>/<repo>` (works offline / air-gap once the bundle is downloaded). Lowest-friction default for most teams: one workflow YAML block, no reusable-workflow plumbing.
  - **`slsa-github-generator`** reusable workflows — predates Artifact Attestations; still the right answer when you need predicate types or builder behaviours not yet covered by the first-party action.
  - **GitLab keyless signing + provenance workflow** or **Tekton Chains** on self-hosted Tekton — meet L3 properties on those platforms.
- Rolling your own L3 builder is a multi-quarter project. Don't.
- **Do** target **Source L3** on every repo whose artifacts ship to production: branch protection requiring review by a non-author, signed commits or signed tags, no force-push on protected branches, binary-artifacts-in-source policy enforced in CI, tag-protection rules. This is the SLSA control that maps directly to the xz-utils class of attack (§18).
- **Do** verify provenance at deploy with `slsa-verifier`, `cosign verify-attestation --type slsaprovenance`, or `gh attestation verify`, asserting expected `builder.id`, source repo, and ref. Provenance without verification is a JSON file.
- **Do** record the **in-toto** layout/predicate types you accept. SLSA provenance is one in-toto v1.0 (`in-toto.io/Statement/v1`) predicate; an admission policy that accepts any `predicateType` is not actually pinning anything. Allow-list the predicate types your platform consumes (`https://slsa.dev/provenance/v1`, `https://spdx.dev/Document`, `https://cyclonedx.org/bom`, `https://openvex.dev/ns`) and reject the rest.

---

## 10. Sigstore / signing

- **Do** sign every container image, Helm chart, and release artifact. Signing model is not one-size-fits-all:

| Model                                      | When it fits                                                                                             |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------|
| **Public Sigstore keyless** (Fulcio/Rekor) | OSS, public registries, most enterprise CI signing. No signer-side keys to rotate; identity in the cert. |
| **Private/self-hosted Sigstore**           | Air-gapped, regulated, sovereign-cloud; want keyless UX but cannot depend on the public good.            |
| **KMS/HSM key-based** (cosign `--key`)     | FIPS 140-2/3 boundary required; offline release signing; long-lived publisher identity (e.g., distro).   |

- **Public keyless** (default): signer authenticates to **Fulcio** with an OIDC token; Fulcio issues a 10-minute X.509 cert binding the OIDC identity to an ephemeral key; signature + cert bundle is logged to **Rekor**'s public Merkle log. Refs: `docs.sigstore.dev`, Fulcio and Rekor specs.
- **Keyless removes signer-side rotation, not verifier-side trust maintenance.** Public Sigstore verifiers root in (a) the **Fulcio CA chain** and (b) the **Sigstore TUF root** (`sigstore/root-signing`), both of which rotate (the production Fulcio root rotation completed in 2022; TUF root re-signings are routine). Verifiers must refresh TUF metadata on a schedule (`cosign initialize`, equivalent admission-controller refresh, or vendor auto-update). A verifier that never refreshes either silently breaks on rotation or accepts a stale root — both are audit findings. Treat TUF refresh as you'd treat CRL/OCSP refresh on a traditional PKI.
- **Self-hosted Sigstore** runs the same components (Fulcio, Rekor, optional private TUF root) inside your trust boundary. Choose when public transparency is a *risk* (you can't reveal artifact names) rather than a benefit. The private-TUF deployment playbook is now documented at `docs.sigstore.dev`; the verifier-refresh discipline above applies identically against your private root.
- **KMS/HSM key-based** is right when an auditor requires a named key in an HSM, when you must sign offline, or when the publisher is a long-lived non-human identity (Linux distros, OS vendors). Operational burden — rotation, custody, revocation — is real and must be planned.
- **Do** pin a verification policy in admission (Kyverno `verifyImages`, Sigstore Policy Controller, Ratify). An unverified signature is a decoration.
- **Private Sigstore is now production-viable.** Reference deployments documented at `docs.sigstore.dev/system_config/installation` cover Helm-installed Fulcio + Rekor + private CT log + private TUF root, with rotation runbooks. Choose this when (a) artifact names are sensitive (regulated, classified), (b) you need a signed-build path in an air-gapped environment, or (c) public Rekor's availability profile does not meet your SLO. Do not roll your own — the trust-root rotation and TUF metadata management are the hard parts and they are now upstream-supported.

---

## 11. Admission-time verification

- Verification has three independent properties; check all three:
  - **Signature validity** — artifact digest signed by *some* trusted signer; for keyless, the Rekor inclusion proof verifies and the Fulcio cert chain ties to a trusted root (refreshed per §10).
  - **Identity pinning** — the signer identity is one you expect. Keyless: `--certificate-identity-regexp '^https://github\.com/myorg/myrepo/\.github/workflows/release\.yml@refs/tags/.*$'` and `--certificate-oidc-issuer https://token.actions.githubusercontent.com`. Key-based: a specific public key. Same `certificate-identity` discipline applies to GitHub Artifact Attestations verified with `gh attestation verify --signer-workflow`.
  - **Attestation predicate** — beyond "this was signed", what was *attested*? cosign/Sigstore attestations are **in-toto DSSE envelopes**; the envelope wraps a *predicate* whose `predicateType` distinguishes payloads (`https://slsa.dev/provenance/v1`, `https://spdx.dev/Document`, `https://cyclonedx.org/bom`, `https://openvex.dev/ns`). Verify the SLSA provenance predicate's `builder.id`, `buildType`, and source `repository`+`ref` match policy.
- **Kyverno example:**

  ```yaml
  verifyImages:
    - imageReferences: ["registry.example.com/*"]
      attestors:
        - entries:
          - keyless:
              subject: "https://github.com/myorg/.+/\\.github/workflows/release\\.yml@refs/tags/.*"
              issuer: "https://token.actions.githubusercontent.com"
      attestations:
        - type: "https://slsa.dev/provenance/v1"
          conditions:
            - all:
              - key: "{{ predicate.buildDefinition.externalParameters.workflow.repository }}"
                operator: Equals
                value: "https://github.com/myorg/myrepo"
  ```

- **Do** monitor Rekor for your identities. A Rekor entry signed by `release.yml@refs/tags/...` from your repo that you didn't trigger is a detection signal. `rekor-monitor` (Sigstore project) and `cosign tree`/`cosign verify --rekor-url` are the building blocks. For private Sigstore, point the monitor at your private Rekor.

---

## 12. Network — zero-trust by default

- **Do** adopt the **BeyondCorp** posture: no network is trusted, every request authenticated and authorized at the application layer with device + user identity. Primary sources: the six Google BeyondCorp papers (2014–2017), **NIST SP 800-207** (*Zero Trust Architecture*), and **NIST SP 800-207A** (Sept 2023, *A ZTA Model for Cloud-Native Multi-Cloud*) which extends 800-207 with explicit guidance on identity-based microsegmentation, API gateways, sidecar proxies, and SPIFFE-style workload identity (§3).
- **No public IPs by default.** Workloads on private subnets; ingress via load balancers or IAPs; egress through NAT/egress gateway with allow-listed destinations. NIST SP 800-204D codifies this for microservices.
- **Microsegmentation, not perimeter.** 800-207A's policy model is "identity × identity" at the workload pair, not "subnet × subnet". A flat `10.0.0.0/8` allow-rule between two VPCs is the opposite of zero trust. Express segmentation as identity-scoped network policy (NetworkPolicy + workload identity, or service-mesh authorization policies) so that a compromised workload's blast radius is its identity's scope, not its subnet's. SPIFFE SVIDs (§3) are one concrete identity primitive that survives a workload moving across clusters or clouds without re-issuing the policy.
- **Egress is half of zero trust.** Default-deny *outbound* destinations from production workloads, allow-list per service. Most data-exfil paths in published incidents (Codecov, polyfill, multiple npm-postinstall worms) ran out through unrestricted egress, not in through ingress.
- **mTLS between services** is required in production for service-to-service
  traffic (a `must`). The mechanics — cert issuance, rotation, identity
  binding (SPIFFE SVID, mesh identity, or app-issued cert) — and the
  transport-layer plumbing belong in **ch07 §7**; this chapter only states
  that mTLS is non-optional for prod. The mesh adoption decision belongs in
  **ch04 §9**. Non-prod and pre-platform clusters may stage mTLS as part of
  the maturity ramp in ch07 §7.
- **Default-deny network policies** in every namespace — canonical rule and
  YAML in **ch04 §8**. `kubectl get networkpolicy -A` returning "No resources
  found" in a multi-tenant cluster is a finding.
- **Don't** rely on VPC/subnet boundaries as a security control — they are blast-radius, not authentication.

---

## 13. Access — IAP > VPN

- **Do** front internal apps with an **identity-aware proxy** instead of a flat VPN. Cloudflare Access, Google IAP, Pomerium, and Tailscale enforce per-request identity + device posture; a VPN gives the laptop the network, then trusts everything on it. **Teleport** is the right answer for SSH/Kubernetes/DB with session recording and short-lived certs — `~/.ssh/authorized_keys` on prod hosts is a finding.
- **Do** require phishing-resistant MFA (FIDO2 / WebAuthn at AAL3) for all human access to production. Authenticator selection, AAL mapping, and the full NIST SP 800-63B rationale are owned by [ch12 §3 Authentication strength](./12-identity.md#3-authentication-strength); this chapter only requires that the IAP in front of internal apps enforces it.

---

## 14. Vulnerability management

- **Do** publish a written SLA and enforce it via tooling:

| Severity | Patch SLA (internet-facing) | Patch SLA (internal) |
|----------|-----------------------------|----------------------|
| Critical | 24–72h                      | 7d                   |
| High     | 7d                          | 30d                  |
| Medium   | 30d                         | 90d                  |
| Low      | best-effort / next release  | best-effort          |

- Numbers are illustrative — pick yours, write them down, dashboard them. FedRAMP Rev 5 RA-5 and PCI-DSS v4.0 §6.3.3 require documented timelines. EU CRA's 24-hour reporting clock for actively-exploited vulnerabilities (effective 11 Sept 2026, §7) constrains the *Critical / internet-facing* row downward for products in scope.
- **Do** weight the SLA clock by the §8 reachability/KEV/EPSS-percentile/VEX context, not raw CVSS-B. A `Critical` in an unreachable test dependency on an internal batch job with a supplier `not_affected` VEX is not the same fire as a KEV-listed RCE on the public ingress, and treating them identically is how alert fatigue starts.
- **Do** enforce the rule: *an alert without a fix path is noise.* If the only remediation is "wait for upstream," the finding belongs in a tracked exception with an expiry, not in the daily Slack noise.

---

## 15. Compliance — what infra actually owns

- Most of ISO 27001 / SOC 2 / FedRAMP / PCI / EU CRA is paperwork the security team drives. The infra deliverables are narrow:
  - **Audit logging** on for the control plane: CloudTrail (all regions, log file validation, immutable bucket), Azure Activity Log + Diagnostic Settings, GCP Cloud Audit Logs (Admin Activity + Data Access where scoped). **ch06 owns the audit-log transport requirement**: centralise to a write-once / object-lock store outside the producing account, with a documented retention period meeting the strictest applicable framework (PCI 1y hot + 1y cold floor; FedRAMP 3y; CRA aligns with vendor support period). Detection wiring (SIEM ingestion, query workload) is ch05 §X; this chapter requires only that the bytes reach an immutable destination. Identity events (sign-in, MFA, PIM elevation, SCIM, break-glass use) feed this same pipeline — the catalogue of required identity signals lives in [ch12 §11 Audit & telemetry](./12-identity.md#11-audit--telemetry).
  - **Encryption at rest** with customer-managed keys where the framework requires it (PCI, FedRAMP Moderate+).
  - **Access reviews** — quarterly export of IAM/RBAC, signed off.
  - **Change management trail** — every prod change traceable to a merged PR and a deployment record. GitOps gives you this nearly free.
  - **Backups + tested restore** — untested backups are not backups. Two cadences (defined in ch08 §3): an automated restore-verification loop (weekly, scripted, owned by **ch09 §12**) and a timed end-to-end DR drill (quarterly floor, monthly tier-0, owned by **ch08 §3**) that records actual RTO/RPO. ch06 cites the rule; the drill mechanics live in ch08/ch09.
  - **EU CRA milestone tracker** — for products in scope, surface the two regulatory dates (11 Sept 2026 reporting; 11 Dec 2027 main obligations) on the same compliance dashboard as SOC 2 audit windows. Treat them as deadlines that drive SBOM-pipeline, support-policy, and ENISA-notification work, not as background prose.
- **Avoid** building a separate "compliance pipeline." If your normal pipeline can't produce the evidence (signed builds, SBOMs, VEX, audit logs, change tickets), fix the pipeline.

---

## 16. Runtime security & audit

- **Do** run a runtime threat-detection agent on production nodes — **Falco** (CNCF graduated) is the cloud-agnostic default, with rules for shell-in-container, unexpected outbound connections, sensitive mount access, and `kubectl exec`. Tetragon (eBPF, Cilium) is the modern alternative.
- **Do** wire audit logs into detections, not just storage. CloudTrail in S3 that no one queries detected zero breaches in 2024.
- **Do** keep a runtime detection for *unexpected outbound network connections from a build agent*. SUNSPOT, Codecov, and several npm-postinstall worms are easier to catch on the egress anomaly than on any pre-deploy scan. Falco rule `Outbound Connection to C2 Servers` and Tetragon `tracingpolicy` egress watchers are the building blocks; tune to your CI's known destinations (registries, OIDC issuers, telemetry).

---

## 17. Shift left, shift everywhere

- "Shift left" is half the story. The honest reframe is **shift everywhere**:

| Stage   | Controls                                                                              |
|---------|---------------------------------------------------------------------------------------|
| Dev     | pre-commit (gitleaks, detect-secrets), IDE scanners, threat model, SRI on third-party |
| Source  | SLSA Source L3: branch protection, two-party review, signed tags, no force-push       |
| Build   | SBOM, dep scan, image scan, SLSA Build L3 provenance, cosign sign, VEX publish        |
| Deploy  | admission policy (Kyverno/Gatekeeper/CEL), signature + provenance + identity verify   |
| Runtime | Falco/Tetragon, audit logs, anomaly detection, IAP enforcement                        |

- Each stage catches what the previous one missed. Skipping runtime because "we shifted left" is how the xz backdoor stayed quiet for months — its payload only activated in specific build/runtime contexts.

---

## 18. Lessons from real supply-chain incidents

- Read the postmortems. Four are mandatory — and each humbles a different control.
- **xz-utils backdoor (CVE-2024-3094, March 2024).** Andres Freund's initial report (`openwall.com/lists/oss-security/2024/03/29/4`) and the OpenSSF post-mortem describe a multi-year social-engineering takeover of a sole maintainer; the dropper was injected via **release-tarball-only modifications** — a malicious `m4/build-to-host.m4` substitution and binary test fixtures — that **did not appear in the git tree**. Activation was scoped to specific sshd link contexts.
  - *What SLSA Build L3 would have done:* nothing direct. Build L3 hardens the **builder**, not the **inputs**. If the malicious tarball is the input, a hardened builder happily produces signed provenance for a backdoored binary.
  - *What SLSA Source L3 (v1.1+) would have done:* the two-party review requirement and the binary-artifacts-in-source policy directly target the takeover and the binary test fixtures; reproducible builds comparing the release tarball against a tree built from git would have surfaced the m4 substitution.
  - *What no technical control fixes:* maintainer burnout and social-engineering takeovers of single-maintainer projects. That's the OpenSSF Alpha-Omega / maintainer-support discussion, not a pipeline configuration.
  - *Detection that did work:* runtime behavioral observation — Andres Freund noticed a ~500ms sshd latency regression. Keep behavioral runtime detection (§16) on the table even when every shift-left control is green.
- **SolarWinds SUNBURST (Dec 2020).** Build-system compromise (SUNSPOT implant on the Orion build server) injected backdoor code into signed Orion releases. Primary sources: CISA Alert AA20-352A, Mandiant *Highly Evasive Attacker Leverages SolarWinds Supply Chain*.
  - *What SLSA Build L3 would have done:* a hardened build platform that isolates build steps and protects signing material would have raised the cost of mid-build injection and made foreign builder fingerprints visible in attestations consumers verify.
  - *What it would not have done:* if the attacker controls the build environment that issues the provenance, the provenance is signed by them too. Defenses also needed: hermetic/reproducible builds with independent re-builders, behavioral runtime detection on the deployed agent, segmentation so a build-host compromise cannot reach prod signing keys.
- **Codecov bash-uploader (April 2021).** Attacker modified `codecov-bash` to exfiltrate the CI environment, harvesting whatever was in env vars across thousands of pipelines. Primary source: Codecov post-incident report (Apr 2021).
  - *What OIDC federation removes:* one major class of secret — the long-lived cloud access key that no longer needs to be in the env at all (§3).
  - *What it does not remove:* every other token still in CI env — package-registry publish tokens, signing material if mounted, SaaS API keys, GitHub PATs. The Codecov attacker took *whatever was there.*
  - *Additional defenses:* hash-pin or vendor third-party CI helpers (don't `curl | bash`); split jobs so signing/publishing credentials only exist in the minimal job that needs them; prefer ephemeral OIDC exchanges for *all* services that support it (npm, PyPI, GitHub Packages, Docker Hub, HCP Vault).
- **polyfill.io (June 2024).** Sansec disclosed (`sansec.io/research/polyfill-supply-chain-attack`) that the new owner of `cdn.polyfill.io` was injecting mobile-targeted malware via the dynamically-generated polyfill script embedded by 100K+ sites including JSTOR, Intuit, and the World Economic Forum; redirects went to a fake `googie-anaiytics.com` sports-betting domain. The original maintainer (Andrew Betts) had publicly recommended deprecation since Feb 2024. Cloudflare and Fastly published mirrored alternatives; Namecheap eventually placed the domain on hold.
  - *What none of SLSA, cosign, SBOMs would have caught:* the supplier substituted what they **served at runtime**, not what you **built**. Your SBOM correctly listed `polyfill.io` and your provenance was perfectly signed.
  - *What does work:* **Subresource Integrity (SRI)** hashes on every external `<script>` and `<link>` (a hash mismatch fails the load); a strict **Content-Security-Policy** with explicit `script-src` allow-lists; or **self-host** third-party JS behind your own CDN so the supplier cannot serve different bytes to different users. Treat any externally-served runtime asset as an unsigned dependency unless one of those three controls is in place.
  - *Generalisation:* the "trusted supplier whose ownership changed" class includes domain expirations, npm package transfers, Docker Hub account takeovers, browser-extension sales. §6 ingestion controls handle the build-time variant; §18 polyfill teaches the runtime variant.
- **Do** add a "what would have caught xz / SUNBURST / Codecov / polyfill here?" question to architecture reviews of any system that ingests source, binaries, or runtime assets from outside the org.

---

## 19. Checklist (PR / repo gate)

- [ ] Threat model in-repo, updated this quarter.
- [ ] No long-lived cloud credentials in CI; OIDC trust scoped per provider (AWS `sub`/`workflow_ref`, GCP WIF attributes, Azure federated credential `subject`). No SA JSON keys; GCP `iam.disableServiceAccountKeyCreation` on. Non-cloud workloads federate via Roles Anywhere / equivalent X.509.
- [ ] Workload identity used on every cluster (IRSA / Pod Identity / GKE WI / AKS WI; SPIFFE/SPIRE for cross-cloud) — no node-role credential reuse.
- [ ] Secrets in a managed store or SOPS-encrypted; `gitleaks` clean.
- [ ] Internal package proxy fronts every public registry; namespaces reserved; lockfiles required; new external packages reviewed; runtime-fetched assets hash-pinned (SRI) or self-hosted.
- [ ] One admission engine per estate (CEL VAP, Kyverno, or Gatekeeper); policies shipped from a single versioned bundle that runs in CI and at admission. Signature verification path documented (Kyverno native vs. Gatekeeper + Policy Controller/Ratify vs. CEL-not-applicable).
- [ ] SBOM (SPDX 2.3/3.0 or CycloneDX 1.5/1.6) generated at build, attached as cosign in-toto attestation, ingested by a vuln DB (OSV.dev / Dependency-Track / GUAC).
- [ ] OpenVEX (or CycloneDX/CSAF VEX) statements published alongside SBOM; scanner pipeline consumes them.
- [ ] Image scan in PR + post-build + registry; gating policy uses reachability + KEV + EPSS-percentile + VEX + exposure, not raw CVSS-B; CVSS v4.0 BTE used where vendors publish it.
- [ ] Cosign / GitHub Artifact Attestations signature + SLSA v1.2 **Build L3** provenance produced and verified at deploy with identity pinning and predicate-type assertion.
- [ ] SLSA v1.2 **Source L3** posture on production repos: branch protection, two-party review, signed tags, no force-push, binary-artifact-in-source policy enforced.
- [ ] Sigstore TUF / Fulcio root refresh scheduled on every verifier (admission, CI, `cosign initialize`).
- [ ] Rekor monitored for unexpected entries under your signing identities (private Rekor for self-hosted Sigstore).
- [ ] Audit logs centralised, immutable (object-lock), retained per the strictest applicable framework, queried by detections.
- [ ] Default-deny NetworkPolicy per app namespace; identity-based microsegmentation per NIST 800-207A; no public IPs by default; egress allow-listed.
- [ ] mTLS in production for service-to-service traffic (mechanics in ch07 §7).
- [ ] IAP for human access; WebAuthn MFA; no flat VPN. (Authenticator strength and IdP wiring: [ch12 §3](./12-identity.md#3-authentication-strength), [ch12 §4](./12-identity.md#4-federation--sso).)
- [ ] Falco (or equivalent) on every prod node; rules tuned, alerts routed.
- [ ] Vuln SLA written, dashboarded, exceptions tracked with expiry; CRA reporting lane (ENISA, 24h) wired for in-scope products.
- [ ] EU CRA milestones (11 Sept 2026 reporting; 11 Dec 2027 main obligations) tracked on the compliance dashboard for products placed on the EU market.

---

*Cross-references: Chapter 03 (CI/CD — implements SLSA Build L3, signing in pipeline; cites this chapter's floor), Chapter 04 (Kubernetes — admission, NetworkPolicy, mesh decision), Chapter 05 (Observability — audit log SIEM ingestion and detections), Chapter 07 (Networking — mTLS transport mechanics, firewall-as-code), Chapter 08/09 (DR drill cadence and restore-verification loop), Chapter 11 (DORA-5, paved road).*
