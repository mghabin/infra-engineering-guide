# Chapter 06 — Security & Supply Chain

Opinionated, cloud-agnostic defaults for securing infrastructure and the software supply chain feeding it. Threat-model first, then controls. Anything you can't tie back to a threat in your model is cargo-culted.

> Conventions: **Do** = required default. **Don't** = reject in review unless a written exception exists. **Prefer** = strong default; deviate only with measurement or a documented constraint.

---

## 1. Threat model first

- **Without a threat model you are cargo-culting controls.** STRIDE for classic asset/tamper analysis, LINDDUN when personal data is in scope. The four questions from the *Threat Modeling Manifesto* are the irreducible core: *what are we working on, what can go wrong, what are we going to do about it, did we do a good enough job?*
- **Do** keep the threat model in the repo and review it on every architectural change. Out-of-tree threat models are out-of-date.
- **Do** map each threat to a control and a test. NIST SP 800-218 (SSDF) PW.1 requires this traceability.
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
- **Do** rotate. CIS Controls v8 (Control 5) and PCI-DSS v4.0 §8 require defined rotation periods; pick one (≤90 days human, ≤24h workload) and enforce via the store, not a calendar.

---

## 3. Workload identity — the "no static credentials" rule

- **Static long-lived cloud credentials in CI or on workloads are a serious incident.** The Codecov 2021, CircleCI 2022, and a long tail of S3-leak postmortems all share the same root cause: a long-lived `AKIA…` or service-account JSON sitting somewhere it shouldn't.
- **Do** use platform workload identity. The mechanisms differ — they are not interchangeable:
  - **EKS IRSA** — cluster's OIDC discovery URL registered as an IAM OIDC provider; kubelet projects an SA JWT into the pod, AWS SDK calls `sts:AssumeRoleWithWebIdentity`. Trust: per-SA role with `sub = system:serviceaccount:<ns>:<sa>`.
  - **EKS Pod Identity** (2023) — on-node `eks-pod-identity-agent` DaemonSet brokers credentials over a link-local endpoint; trust is an EKS *Pod Identity Association*, not an IAM OIDC trust policy. Avoids per-cluster OIDC provider sprawl.
  - **GKE Workload Identity** — KSA annotated with a GSA; GKE's `gke-metadata-server` intercepts metadata calls and exchanges the projected SA token for a Google access token via STS. Trust: the KSA↔GSA binding.
  - **AKS Workload Identity** — Azure AD *federated identity credentials* on a user-assigned managed identity reference the AKS OIDC issuer + KSA `sub`; the projected SA token is exchanged at `login.microsoftonline.com` for an AAD token. Replaces deprecated AAD Pod Identity.
- **Do** use OIDC federation from CI to cloud IAM. Scope with provider-specific claims (GitHub Actions docs *Security hardening with OpenID Connect*):
  - **AWS IAM** — trust policy conditions on `aud = sts.amazonaws.com` and `token.actions.githubusercontent.com:sub` of the form `repo:ORG/REPO:ref:refs/heads/main` or `repo:ORG/REPO:environment:prod`. For reusable workflows, condition on `job_workflow_ref`.
  - **GCP Workload Identity Federation** — Workload Identity Pool + Provider with `attribute.repository`, `attribute.ref`, `attribute.workflow_ref`; bind the SA with an IAM condition on those attributes.
  - **Azure** — federated credential on the app registration / managed identity scoped by `subject` (e.g., `repo:ORG/REPO:environment:prod`) and `issuer` = the GitHub OIDC issuer.
- **Note:** GitHub OIDC claims include no stable per-job identifier you can pin in a cloud trust policy. Scope on `repository`, `ref`, `environment`, `workflow_ref`; use protected `environments` to gate prod role assumption.
- **Don't** create an IAM user for a service. The right cardinality of IAM users in a modern AWS account is "humans who break-glass, plus zero."

---

## 4. Policy-as-code

- **Do** enforce platform invariants with admission control, not wiki pages. Choose between OPA Gatekeeper and Kyverno based on team and policy shape:
  - **OPA / Gatekeeper / Conftest.** Rego is expressive enough for cross-resource policies and reusable across Terraform (`conftest`), CI, and runtime (Gatekeeper). Pay the Rego learning tax once.
  - **Kyverno.** YAML-native, no new language, *mutates* as well as validates. For "block hostPath, require resource limits, mirror images to internal registry" workloads it is dramatically simpler than Rego.
- **Image signature/attestation verification is not at parity between the two.** Kyverno ships `verifyImages` natively (cosign + notary, keyless or key-based, attestation predicate matching). Gatekeeper has no built-in verifier — you bolt on Sigstore Policy Controller, Connaisseur, or Ratify as a separate admission webhook. If signature verification is a hard requirement and you don't already run Gatekeeper, Kyverno is the shorter path.
- Use **Conftest** to apply Rego in CI against Terraform plans, Dockerfiles, Kubernetes manifests, and Helm charts *before* they reach the cluster. **Polaris** is a useful resource-request lint pass; **kube-bench** runs the CIS Kubernetes Benchmark against a live cluster.
- **Do** ship policies as a versioned bundle (OCI artifact or git submodule) so the same rules run locally (`conftest test`), in CI, and in admission. Drift between layers is how exceptions become permanent.

---

## 5. Benchmarks — what actually matters

- The CIS Benchmarks (Kubernetes, Docker, Linux, AWS/Azure/GCP) are the closest thing to a vendor-neutral baseline — and full of low-value checks shipped with high severities. Internalise the controls that move the needle:
  - **K8s:** API server `--anonymous-auth=false`, audit logging on, RBAC on, etcd encryption at rest, PSA `restricted` (or equivalent Kyverno/Gatekeeper policies) in app namespaces, default-deny NetworkPolicy.
  - **Containers:** non-root `USER`, read-only root FS, no `--privileged`, no `hostNetwork`/`hostPID`, drop all capabilities and re-add only what's needed, seccomp `RuntimeDefault`.
  - **Cloud:** root account MFA + no access keys, CloudTrail / Activity Log / Cloud Audit Logs enabled in *every* region/subscription/project, default encryption on object storage, no public-by-default buckets/blobs.
- **Avoid** treating "100% CIS pass" as the goal. The goal is a low-noise report where every remaining finding has a written exception.

---

## 6. Package ingestion controls

- Most supply-chain incidents enter before any scanner runs. Ingestion hygiene is the cheapest defense against dependency confusion (Birsan, 2021), typosquats (`event-stream`, `ua-parser-js`, `colors.js`), and namespace hijacks. Per OpenSSF *Source Code Management Best Practices* and OWASP *Dependency Management Cheat Sheet*:
- **Do** front every public registry with an internal proxy/mirror — Artifactory, Nexus, GitHub Packages, AWS CodeArtifact, GCP Artifact Registry. Cache-on-first-use, retention beyond upstream yanks, single chokepoint to allowlist sources.
- **Do** reserve your org's namespace on every public registry you publish to (`@yourorg/*` on npm, `com.yourorg.*` on Maven Central, `yourorg-*` prefix on PyPI). Unclaimed namespaces are how dependency-confusion attacks succeed.
- **Do** pin and lock. Lockfiles committed and verified in CI (`npm ci`, `pip install --require-hashes`, `go mod verify`). Floating ranges in production manifests are a finding.
- **Do** allowlist sources, not deny known-bad. `npm config set registry` pointing only at the proxy; `pip --index-url` constrained; Renovate/Dependabot gated on approved registries. Block direct installs from arbitrary VCS URLs.
- **Do** require human review on first introduction of a new external package. Socket, Snyk Advisor, `deps.dev` surface package age, maintainer count, install scripts, and capability signals before merge.
- **Don't** `curl | bash` third-party install scripts in CI. Vendor or hash-pin them — see §18 (Codecov).

---

## 7. SBOMs

- **Do** generate an SBOM for every artifact you publish (containers, binaries, Helm charts). Prefer **SPDX 2.3+** (ISO/IEC 5962:2021) or **CycloneDX 1.5+** (ECMA-424). CycloneDX has richer VEX support; SPDX has the broader regulatory footprint (US EO 14028, EU CRA).
- **Do** generate at *build* time from source (Syft, `cyclonedx-gomod`, `cyclonedx-npm`). Scanning a finished image is a fallback — the build-time graph sees test/build deps and exact resolved versions the binary won't reveal.
- **Do** attach the SBOM as a signed in-toto attestation: `cosign attest --predicate sbom.spdx.json --type spdx <image>`. Storage semantics matter:
  - The attestation is stored as an **OCI-associated artifact** discovered via the *Referrers API* (OCI Image Spec 1.1) — i.e., another blob in the registry tagged/referred to from the subject image's digest.
  - **Rekor** records the *signing event* (signature + cert + hash of the DSSE envelope) in an append-only Merkle log. Rekor is **not** the storage backend for the SBOM payload itself; the registry is.
- **Don't** ship an SBOM you don't ingest. Pipe them into a vulnerability database (Dependency-Track, GUAC, Grype DB) so "is log4j 2.14 anywhere in prod?" is a query, not a fire drill.

---

## 8. Container & dependency scanning

- The market has converged: **Trivy**, **Grype**, **Snyk**, **Anchore** pull NVD/GHSA/vendor advisories and match against package metadata. Trivy = broadest scope; Grype+Syft = cleanest SBOM-in/findings-out; Snyk = best dev-loop UX + reachability; Anchore Enterprise = policy engine at FedRAMP scale.
- **Do** scan in *three* places: pre-merge (PR diff), post-build (full image), and continuously in the registry — vulnerabilities published *after* you build still matter (that's how Log4Shell hit you).
- **Don't** gate purely on CVSS severity. Severity is the worst predictor of "fix this now" we have:
  - **Reachability.** Does any code path actually call the vulnerable function? (Snyk Reachable, Endor Labs, Semgrep Supply Chain, OSV call-graph.)
  - **Exploitability.** Is the CVE on **CISA KEV**? Does EPSS exceed your threshold (e.g., >0.5)?
  - **Exposure.** Internet-facing service vs. internal batch vs. base-image noise that never executes.
  - **Fix availability.** No fixed version = exception with expiry, not a build break.
- A reasonable default policy: **fail** on (KEV ∪ (severity ≥ High ∧ reachable ∧ fix-available)) for internet-facing services; **alert** with SLA on the rest; **track** base-image CVEs separately and fix by bumping the base image.

---

## 9. SLSA & build provenance

- (Cross-ref Chapter 03.) Target the right **SLSA v1.0 Build Track** level for the artifact's blast radius. Per `slsa.dev/spec/v1.0/requirements`:
  - **Build L1** — provenance exists, describes how the artifact was built. No integrity guarantees.
  - **Build L2** — *signed* provenance generated by a *hosted* build platform. Tampering after the build is detectable; platform identity verifiable. Forgery by a malicious build *step* still in scope.
  - **Build L3** — adds **hardened** platform properties: build runs are isolated from each other, and **signing material is not accessible to user-defined build steps** (a malicious step cannot forge provenance for a different artifact). This is the bar that defends against in-build tampering.
- **Do** target L3 for production artifacts. Use `slsa-github-generator` reusable workflows on GitHub Actions, GitLab's keyless signing + provenance workflow, or Tekton Chains on self-hosted Tekton — all three meet L3 properties. Rolling your own L3 builder is a multi-quarter project.
- **Do** verify provenance at deploy with `slsa-verifier` or `cosign verify-attestation --type slsaprovenance`, asserting expected `builder.id`, source repo, and ref. Provenance without verification is a JSON file.

---

## 10. Sigstore / signing

- **Do** sign every container image, Helm chart, and release artifact. Signing model is not one-size-fits-all:

| Model                                      | When it fits                                                                                             |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------|
| **Public Sigstore keyless** (Fulcio/Rekor) | OSS, public registries, most enterprise CI signing. No keys to rotate; identity in the cert.             |
| **Private/self-hosted Sigstore**           | Air-gapped, regulated, sovereign-cloud; want keyless UX but cannot depend on the public good.            |
| **KMS/HSM key-based** (cosign `--key`)     | FIPS 140-2/3 boundary required; offline release signing; long-lived publisher identity (e.g., distro).   |

- **Public keyless** (default): signer authenticates to **Fulcio** with an OIDC token; Fulcio issues a 10-minute X.509 cert binding the OIDC identity to an ephemeral key; signature + cert bundle is logged to **Rekor**'s public Merkle log. Refs: `docs.sigstore.dev`, Fulcio and Rekor specs.
- **Self-hosted Sigstore** runs the same components (Fulcio, Rekor, optional private TUF root) inside your trust boundary. Choose when public transparency is a *risk* (you can't reveal artifact names) rather than a benefit.
- **KMS/HSM key-based** is right when an auditor requires a named key in an HSM, when you must sign offline, or when the publisher is a long-lived non-human identity (Linux distros, OS vendors). Operational burden — rotation, custody, revocation — is real and must be planned.
- **Do** pin a verification policy in admission (Kyverno `verifyImages`, Sigstore Policy Controller). An unverified signature is a decoration.

---

## 11. Admission-time verification

- Verification has three independent properties; check all three:
  - **Signature validity** — artifact digest signed by *some* trusted signer; for keyless, the Rekor inclusion proof verifies and the Fulcio cert chain ties to a trusted root.
  - **Identity pinning** — the signer identity is one you expect. Keyless: `--certificate-identity-regexp '^https://github\.com/myorg/myrepo/\.github/workflows/release\.yml@refs/tags/.*$'` and `--certificate-oidc-issuer https://token.actions.githubusercontent.com`. Key-based: a specific public key.
  - **Attestation predicate** — beyond "this was signed", what was *attested*? cosign/Sigstore attestations are **in-toto DSSE envelopes**; the envelope wraps a *predicate* whose `predicateType` distinguishes payloads (`https://slsa.dev/provenance/v1`, `https://spdx.dev/Document`, `https://cyclonedx.org/bom`). Verify the SLSA provenance predicate's `builder.id`, `buildType`, and source `repository`+`ref` match policy.
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

- **Do** monitor Rekor for your identities. A Rekor entry signed by `release.yml@refs/tags/...` from your repo that you didn't trigger is a detection signal. `rekor-monitor` (Sigstore project) and `cosign tree`/`cosign verify --rekor-url` are the building blocks.

---

## 12. Network — zero-trust by default

- **Do** adopt the **BeyondCorp** posture: no network is trusted, every request authenticated and authorized at the application layer with device + user identity. Primary sources: the six Google BeyondCorp papers (2014–2017) and NIST SP 800-207 (*Zero Trust Architecture*).
- **No public IPs by default.** Workloads on private subnets; ingress via load balancers or IAPs; egress through NAT/egress gateway with allow-listed destinations. NIST SP 800-204D codifies this for microservices.
- **mTLS between services.** A mesh (Istio, Linkerd, Cilium) gives you this for free; without a mesh, do it in the app or accept the risk in writing.
- **Default-deny network policies** in every namespace. `kubectl get networkpolicy -A` returning "No resources found" in a multi-tenant cluster is a finding.
- **Don't** rely on VPC/subnet boundaries as a security control — they are blast-radius, not authentication.

---

## 13. Access — IAP > VPN

- **Do** front internal apps with an **identity-aware proxy** instead of a flat VPN. Cloudflare Access, Google IAP, Pomerium, and Tailscale enforce per-request identity + device posture; a VPN gives the laptop the network, then trusts everything on it. **Teleport** is the right answer for SSH/Kubernetes/DB with session recording and short-lived certs — `~/.ssh/authorized_keys` on prod hosts is a finding.
- **Do** require hardware-backed WebAuthn (YubiKey, platform authenticator) for all human access to production. NIST SP 800-63B AAL3 effectively mandates this. TOTP is phishable; SMS is worse.

---

## 14. Vulnerability management

- **Do** publish a written SLA and enforce it via tooling:

| Severity | Patch SLA (internet-facing) | Patch SLA (internal) |
|----------|-----------------------------|----------------------|
| Critical | 24–72h                      | 7d                   |
| High     | 7d                          | 30d                  |
| Medium   | 30d                         | 90d                  |
| Low      | best-effort / next release  | best-effort          |

- Numbers are illustrative — pick yours, write them down, dashboard them. FedRAMP Rev 5 RA-5 and PCI-DSS v4.0 §6.3.3 require documented timelines.
- **Do** weight the SLA clock by the §8 reachability/KEV/exposure context, not raw CVSS. A `Critical` in an unreachable test dependency on an internal batch job is not the same fire as a KEV-listed RCE on the public ingress, and treating them identically is how alert fatigue starts.
- **Do** enforce the rule: *an alert without a fix path is noise.* If the only remediation is "wait for upstream," the finding belongs in a tracked exception with an expiry, not in the daily Slack noise.

---

## 15. Compliance — what infra actually owns

- Most of ISO 27001 / SOC 2 / FedRAMP / PCI is paperwork the security team drives. The infra deliverables are narrow:
  - **Audit logging** on for the control plane: CloudTrail (all regions, log file validation, immutable bucket), Azure Activity Log + Diagnostic Settings, GCP Cloud Audit Logs (Admin Activity + Data Access where scoped). Centralise to a write-once store outside the producing account.
  - **Encryption at rest** with customer-managed keys where the framework requires it (PCI, FedRAMP Moderate+).
  - **Access reviews** — quarterly export of IAM/RBAC, signed off.
  - **Change management trail** — every prod change traceable to a merged PR and a deployment record. GitOps gives you this nearly free.
  - **Backups + tested restore** — untested backups are not backups. Run a restore drill quarterly and record actual RTO/RPO.
- **Avoid** building a separate "compliance pipeline." If your normal pipeline can't produce the evidence (signed builds, SBOMs, audit logs, change tickets), fix the pipeline.

---

## 16. Runtime security & audit

- **Do** run a runtime threat-detection agent on production nodes — **Falco** (CNCF graduated) is the cloud-agnostic default, with rules for shell-in-container, unexpected outbound connections, sensitive mount access, and `kubectl exec`. Tetragon (eBPF, Cilium) is the modern alternative.
- **Do** wire audit logs into detections, not just storage. CloudTrail in S3 that no one queries detected zero breaches in 2024.

---

## 17. Shift left, shift everywhere

- "Shift left" is half the story. The honest reframe is **shift everywhere**:

| Stage   | Controls                                                              |
|---------|-----------------------------------------------------------------------|
| Dev     | pre-commit (gitleaks, detect-secrets), IDE scanners, threat model     |
| Build   | SBOM, dep scan, image scan, SLSA provenance, cosign sign              |
| Deploy  | admission policy (Kyverno/Gatekeeper), signature + provenance verify  |
| Runtime | Falco/Tetragon, audit logs, anomaly detection, IAP enforcement        |

- Each stage catches what the previous one missed. Skipping runtime because "we shifted left" is how the xz backdoor stayed quiet for months — its payload only activated in specific build/runtime contexts.

---

## 18. Lessons from real supply-chain incidents

- Read the postmortems. Three are mandatory — and each humbles a different control:
- **xz-utils backdoor (CVE-2024-3094, March 2024).** Andres Freund's initial report (`openwall.com/lists/oss-security/2024/03/29/4`) and subsequent timelines describe a multi-year social-engineering operation against a sole maintainer; the malicious payload hid in autotools/m4 generated artifacts and binary test fixtures, activated only in specific sshd link contexts.
  - *What SLSA L3 would have done:* verifiable traceability from published tarball back to a specific builder and source ref; made out-of-tree binary blobs in the release tarball easier to flag against the git tree.
  - *What it would not have done:* the source itself was malicious and the maintainer was the legitimate releaser. Provenance signs *that* build, not the *intent* of the change. Real defenses: source integrity (review of generated/binary fixtures, two-person release control, reproducible builds comparing tarball vs. git), and runtime behavioral detection (the original signal was a 500ms latency regression).
- **SolarWinds SUNBURST (Dec 2020).** Build-system compromise (SUNSPOT implant on the Orion build server) injected backdoor code into signed Orion releases. Primary sources: CISA Alert AA20-352A, Mandiant *Highly Evasive Attacker Leverages SolarWinds Supply Chain*.
  - *What SLSA L3 would have done:* a hardened build platform that isolated build steps and protected signing material would have raised the cost of mid-build injection and made foreign builder fingerprints visible in attestations consumers verify.
  - *What it would not have done:* if the attacker controls the build environment that issues the provenance, the provenance is signed by them too. Defenses also needed: hermetic/reproducible builds with independent re-builders, behavioral runtime detection on the deployed agent, segmentation so a build-host compromise cannot reach prod signing keys.
- **Codecov bash-uploader (April 2021).** Attacker modified `codecov-bash` to exfiltrate the CI environment, harvesting whatever was in env vars across thousands of pipelines. Primary source: Codecov post-incident report (Apr 2021).
  - *What OIDC federation removes:* one major class of secret — the long-lived cloud access key that no longer needs to be in the env at all.
  - *What it does not remove:* every other token still in CI env — package-registry publish tokens, signing material if mounted, SaaS API keys, GitHub PATs. The Codecov attacker took *whatever was there.*
  - *Additional defenses:* hash-pin or vendor third-party CI helpers (don't `curl | bash`); split jobs so signing/publishing credentials only exist in the minimal job that needs them; prefer ephemeral OIDC exchanges for *all* services that support it (npm, PyPI, GitHub Packages, Docker Hub, HCP Vault).
- **Do** add a "what would have caught xz / SUNBURST / Codecov here?" question to architecture reviews of any system that ingests source or binaries from outside the org.

---

## 19. Checklist (PR / repo gate)

- [ ] Threat model in-repo, updated this quarter.
- [ ] No long-lived cloud credentials in CI; OIDC trust scoped per provider (AWS `sub`/`workflow_ref`, GCP WIF attributes, Azure federated credential `subject`).
- [ ] Workload identity used on every cluster (IRSA / Pod Identity / GKE WI / AKS WI) — no node-role credential reuse.
- [ ] Secrets in a managed store or SOPS-encrypted; `gitleaks` clean.
- [ ] Internal package proxy fronts every public registry; namespaces reserved; lockfiles required; new external packages reviewed.
- [ ] Admission policy (Kyverno/Gatekeeper) blocks privileged pods, hostPath, untrusted registries, unsigned images. Signature verification path documented (Kyverno native vs. Gatekeeper + Policy Controller/Ratify).
- [ ] SBOM (SPDX/CycloneDX) generated at build, attached as cosign in-toto attestation, ingested by a vuln DB.
- [ ] Image scan in PR + post-build + registry; gating policy uses reachability + KEV + exposure, not raw CVSS.
- [ ] Cosign signature + SLSA v1.0 Build L3 provenance produced and verified at deploy, with identity pinning and predicate-type assertion.
- [ ] Rekor monitored for unexpected entries under your signing identities.
- [ ] Audit logs centralised, immutable, queried by detections.
- [ ] Default-deny NetworkPolicy per app namespace; no public IPs by default; egress allow-listed.
- [ ] IAP for human access; WebAuthn MFA; no flat VPN.
- [ ] Falco (or equivalent) on every prod node; rules tuned, alerts routed.
- [ ] Vuln SLA written, dashboarded, exceptions tracked with expiry.

---

*Cross-references: Chapter 03 (CI/CD — SLSA, signing in pipeline), Chapter 04 (Kubernetes — admission, NetworkPolicy, mesh), Chapter 05 (Observability — audit logs, SIEM ingestion).*
