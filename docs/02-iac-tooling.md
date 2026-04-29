# Infrastructure Engineering — Chapter 02: IaC Tooling

Opinionated, cross-cutting defaults for choosing and running IaC tools in
2026. Declarative-infra philosophy lives in chapter 01; Kubernetes-as-target
is covered in depth in chapter 04.

> Conventions: **Must** = required default, ≥2 independent canonical sources.
> **Should** = strong default, deviation needs written justification.
> **Prefer** = pick this unless you have measurement or context that says otherwise.
> **Avoid** = reject in review unless an exception is filed.

---

## 1. The 2026 landscape

- **Terraform (HashiCorp, BUSL 1.1 since Aug 2023)** — largest ecosystem and
  registry. IBM-owned since 27 Feb 2025 (IBM Newsroom, *IBM Completes
  Acquisition of HashiCorp*,
  https://newsroom.ibm.com/2025-02-27-IBM-Completes-Acquisition-of-HashiCorp,-Creating-a-Comprehensive-End-to-End-Hybrid-Cloud-Platform).
  Non-OSI; TF Cloud / TFE is the upsell.
- **OpenTofu (Linux Foundation, MPL-2.0)** — fork of Terraform 1.5.7 (Sept
  2023). Drop-in CLI, same HCL, same provider protocol. Ships features
  Terraform doesn't have: client-side state encryption, early-eval in
  `backend` blocks, provider `for_each`, `-exclude`. Real fork.
- **Pulumi** — TS/Python/Go/.NET/Java over the same provider model. Right
  call when you need real abstraction (classes, loops, unit tests) HCL fights
  you on, and your team is fluent in one of those languages.
- **AWS CDK** — TS/Python/Java/.NET that synthesizes CloudFormation. AWS-only.
- **Bicep** — Microsoft DSL over ARM. Azure-only; right Azure default. ARM is
  legacy. Raw CloudFormation only for things CDK can't reach.
- **Crossplane** — Kubernetes control plane that reconciles cloud resources
  via CRDs. **v2 GA in 2025** re-architected the project: XRs and MRs are
  namespaced by default, claims are removed, Compositions can wrap *any*
  Kubernetes resource (Deployments, third-party CRDs, Cluster API objects),
  Composition Functions remain the writable layer (Go / Python / KCL). v1
  cluster-scoped XRs survive as `LegacyCluster` for back-compat. Pick when
  Kubernetes *is* your platform API.
- **Ansible** — *configuration management*, not IaC. Procedural, push-based,
  mutable-target. Use for OS-level config of long-lived VMs and network gear.
  **Chef / Puppet** in decline (ThoughtWorks Radar: Hold).
- **Nix / NixOS** — ascendant for reproducible build/dev environments and OS
  images. Replaces `Dockerfile` + `apt-get` + Packer, not Terraform.

### The OpenTofu fork — position

HashiCorp relicensed Terraform from MPL-2.0 to BUSL-1.1 in Aug 2023; the
Linux Foundation adopted the fork in Sept 2023 and shipped 1.6 GA in Jan
2024. State and modules are bidirectionally compatible with Terraform 1.5.x.

**Prefer — OpenTofu for new HCL work when you don't depend on Terraform-first
commercial paths.** OSI license, faster cadence on user-driven features
(state encryption, provider `for_each`, `-exclude`), Registry remains usable.
Stay on Terraform when you depend on TFC/TFE, Sentinel, Stacks, run tasks,
the HCP-integrated provider catalogue, or vendor support contracts that
validate Terraform first (Spacelift / env0 support both, but a non-trivial
set of compliance and audit tooling still lists Terraform explicitly). The
ecosystem-maturity gap is small and shrinking; the support-contract gap is
real and slow. Do **not** run both CLIs against the same state.

---

## 2. Tool choice — the decision framework

```
  Single cloud, "just give me resources"
    AWS only        → CDK (TypeScript or Python)
    Azure only      → Bicep
    GCP only        → Terraform/OpenTofu

  Multi-cloud or polyglot platform team
    HCL is fine     → OpenTofu (default) or Terraform
    Need real code  → Pulumi (TS or Go)

  Kubernetes is your platform, not a workload runtime
    Crossplane (with Composition Functions in Go/Python)

  OS-level config on long-lived hosts → Ansible — and only for that
  Reproducible images / dev shells    → Nix (+ Packer / nixos-generators)
```

**Must — pick one IaC tool per repo and one CLI per state file.** Mixing Terraform and OpenTofu against the same state, or CDK and raw CloudFormation against the same stack, produces drift you cannot diff.
- Sources: HashiCorp, *State*, https://developer.hashicorp.com/terraform/language/state ; OpenTofu, *Migrating from Terraform*, https://opentofu.org/docs/intro/migration/ .

**Should — do not pick a multi-cloud tool to enable a multi-cloud strategy you don't have.** Brikman (*TUR*, 3rd ed., ch. 1) and the AWS CDK Best Practices guide both land here: the abstraction tax of "cloud-agnostic" code is real, and most "multi-cloud" orgs are actually "primary cloud + a few SaaS providers." Choose for the cloud you actually run in.

**Avoid — CDK on the basis of "we might leave AWS one day."** CDK synthesizes CloudFormation; output is AWS-shaped down to the IAM model. Escape hatch is "rewrite," not "swap a backend."

**Avoid — Pulumi *because* you don't like HCL.** HCL fluency is a one-week investment. Pulumi's payoff is real abstraction — shared component classes, unit tests, loops that don't fight `count`/`for_each`. If your modules are 200 lines of resources with a few `for_each`, HCL is the right tool.

---

## 3. Module / component design

Modules are the unit of reuse. The failure mode is the 2,000-line "platform"
module that takes 47 variables and conditionally creates everything.

**Must — every module is single-purpose and composable.** Anton Babenko's terraform-best-practices.com and the official Terraform "Standard Module Structure" both define this: one module = one logical thing. Composition happens in a *root* module (or in Terragrunt / Pulumi program), not inside a leaf.

**Should — start with two layers; add a third only when modules cross teams.** Default is *resource modules* (~50–300 lines, thin wrappers, versioned) plus *root / live config* (per-environment instantiations, no logic, only wiring and `tfvars`). Once two or more teams consume the same composition (ALB + target group + ECS service + log group + alarms), promote it to a *service module* layer. Forcing the three-layer split on a single team adds indirection without improving safety.

**Should — version every shared module and pin the consumer.** Use semver tags: `source = "git::ssh://...//modules/vpc?ref=v1.4.2"` or registry `version = "~> 1.4"`. Floating refs (`ref=main`) in a root module are an outage waiting to happen. Once shared modules are consumed by multiple teams they become a *platform surface* and should be governed under the platform-tooling rules in ch11 §15 (versioning, deprecation, contribution model).

**Prefer — registry over in-repo monorepo for modules consumed by ≥3 teams.** Below that threshold, the operational cost of a registry exceeds the benefit.

**Avoid — `count` for resource toggles when `for_each` will do.** `count` re-indexes on removal and forces destroy/create churn; `for_each` keys by a stable string. Reserve `count` for the `enabled = true/false` boolean-gate pattern, preferably at the module level.

**Avoid — inline `provider` blocks in shared modules.** Modules declare `required_providers` with version constraints; the root passes configured providers in.

**Prefer — provider-defined functions over hand-rolled `regex` / `split` / `format` for parsing provider-shaped strings** (ARNs, resource IDs, KRM names). Available since Terraform 1.8 / OpenTofu 1.7 (e.g. `provider::aws::arn_parse(arn)`); covered by the provider lockfile, so they version with the provider, not the module.
- Sources: HashiCorp, *Functions — provider-defined functions*, https://developer.hashicorp.com/terraform/language/functions ; Terraform 1.10 release notes (cross-references provider functions added in 1.8), https://github.com/hashicorp/terraform/releases/tag/v1.10.0 .

**Should — on Azure, prefer Azure Verified Modules (`br/public:avm/res/...`) over hand-rolled Bicep wrappers for any resource AVM covers.** AVM is Microsoft's official cross-language (Bicep + Terraform) module standard, with WAF guidance baked into the module specs; for Terraform-on-Azure, prefer the AVM Terraform variants over the older `Azure/*` registry modules. This is the Azure analogue of the "registry over in-repo monorepo" rule above. Local Bicep wrapping of AVM resource modules into pattern modules is the right composition path.
- Source: Microsoft, *Azure Verified Modules*, https://azure.github.io/Azure-Verified-Modules/ .

---

## 4. State management

State is the most dangerous artifact in your IaC system: plaintext secrets,
the source of truth the tool trusts over reality, multi-hour incident when
corrupted.

**Must — remote state with native locking, always.** Use the backend's native lock: S3 with `use_lockfile = true` (Terraform 1.10 / OpenTofu 1.10 added native S3 conditional-write locks; **DynamoDB-based locking is deprecated** — both upstreams say the arguments will be removed in a future minor release, but it still works today, so migrate during a normal cut-over rather than treating inherited stacks as broken. Both `use_lockfile = true` and `dynamodb_table` can be set during the migration window; 1.10+ acquires from both), GCS native lock, Azure Blob lease, the TFC/TFE backend, or OpenTofu's HTTP backend with a lock server. Local state in CI guarantees overwrite races.
- Sources: HashiCorp, *S3 Backend — State Locking*, https://developer.hashicorp.com/terraform/language/backend/s3#state-locking ; OpenTofu, *S3 backend*, https://opentofu.org/docs/language/settings/backends/s3/ ; Terraform 1.10.0 release notes, https://github.com/hashicorp/terraform/releases/tag/v1.10.0 .

**Must — encrypt state at rest and treat it as a secret.** Server-side bucket/blob encryption (SSE-KMS on S3, CMK on Azure Blob, CMEK on GCS) is necessary, not sufficient — also restrict bucket access to the apply role, enable object versioning, enable access logging.
- Sources: AWS, *SSE-KMS for S3*, https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html ; HashiCorp, *Sensitive data in state*, https://developer.hashicorp.com/terraform/language/state/sensitive-data .

**Should — turn on OpenTofu's client-side state encryption when state holds real secrets.** Server-side / KMS-envelope encryption protects against a stolen disk or unauthenticated access; it does **not** protect against a credential allowed to read the object (`s3:GetObject`, `Storage Blob Data Reader`) — KMS will decrypt for that caller. OpenTofu's state encryption (1.7+) wraps the state body with a passphrase or KMS key *before* upload, so a stolen object-read credential yields ciphertext. **Pulumi** achieves equivalent protection per-value via its configurable secrets-provider model (`passphrase` default, `awskms`, `azurekeyvault`, `gcpkms`, `hashivault`) — sensitive inputs/outputs are encrypted client-side in the stack file while non-secret state remains plaintext (more granular than OpenTofu's whole-state envelope, but doesn't hide non-secret metadata). Terraform OSS has no equivalent native client-side state encryption; the Terraform-side answer is "use TFC/TFE, restrict the apply role, accept the residual risk." CDK and Bicep have no client-side equivalent — rely on KMS-on-the-bucket plus IAM scoping.
- Sources: OpenTofu, *State encryption*, https://opentofu.org/docs/language/state/encryption/ ; Pulumi, *Secrets and configurable secrets providers*, https://www.pulumi.com/docs/iac/concepts/secrets/ ; HashiCorp, *Sensitive data in state* (above) — acknowledges OSS state is stored unencrypted client-side.

**Must — one state file per blast radius.** Monolithic "all of prod" state means a bad refactor can plan to destroy your VPC. Split by environment, then by lifecycle (network, data, compute, edge). Cross-state references go through `terraform_remote_state` — or better, via SSM/Parameter Store/Key Vault where the producer publishes outputs and consumers read by name.
- Sources: HashiCorp, *Remote state*, https://developer.hashicorp.com/terraform/language/state/remote ; Brikman, *Terraform: Up & Running* (3rd ed.), chs. 3 & 5.

**Prefer — declare cross-state dependencies explicitly once you have more than ~10 states per environment.** Terraform Stacks `component`/`deployment` graph (HCP/TFE), Terragrunt `dependency` blocks, or a hand-rolled topology diagram backed by `terraform_remote_state` — pick one and keep it reviewable. Use OpenTofu 1.10's `-target-file` / `-exclude-file` for surgical replans during incident response, **not** as a routine workflow; targeted plans hide drift in the unselected resources.
- Sources: HashiCorp, *Terraform Stacks*, https://developer.hashicorp.com/terraform/language/stacks ; OpenTofu 1.10.0 release notes, https://github.com/opentofu/opentofu/releases/tag/v1.10.0 .

**Prefer — declarative `import` blocks (Terraform 1.5+/OpenTofu 1.6+) over `terraform import` CLI or hand-edited state.** Reviewable in a PR, removable after the next apply. `state rm`/`state mv` are incident escape hatches.

**Avoid — `terraform state push` against a shared backend.** If you must: take the lock manually, snapshot to an audited location, second engineer reviews the diff. Single command most likely to cause a multi-hour incident.

**Avoid — committing `terraform.tfstate` (or any `*.tfstate*`) to git.** Will leak secrets; will be wrong within a day. `.gitignore` at repo root.

### 4a. State migration

State migrations (changing backend, splitting one state into many, re-keying resources, swapping a provider) are the highest-risk class of routine IaC work. Treat them like a database migration.

- **Must — snapshot the state and lock metadata before any migration command.** Download an out-of-bucket copy (S3 versioning is not enough), record the lock ID, record provider and CLI versions. Restore = "put the snapshot back and re-take the lock."
  - Sources: HashiCorp, *Manipulating Terraform state*, https://developer.hashicorp.com/terraform/cli/state ; OpenTofu, *Backend configuration / migration*, https://opentofu.org/docs/language/settings/backends/configuration/ .
- **Should — change one variable per migration PR.** Either `tofu init -migrate-state`, or `state mv`, or `state replace-provider` — never two at once.
- **Should — drive structural moves with `moved {}` blocks over `terraform state mv`.** Reviewable in the PR and re-runnable; `state mv` is a one-shot side-effect on a shared object.
- **Should — for backend migrations, run `init -migrate-state` from a clean workspace** with the new backend configured and the old still reachable; verify `plan` returns no changes before releasing the lock.
- **Avoid — `state push` / hand-edited state JSON as a migration tool.** If snapshot + `moved` + `state mv`/`rm`/`replace-provider` cannot express the change, the design is wrong, not the tooling.
- Rollback expectation: every migration PR includes a written rollback step ("restore snapshot X to s3://…/path, release lock Y, re-run plan") and the on-call is paged before apply.

---

## 5. Environment separation

**Should — separate state files in separate directories per environment, not workspaces.** Brikman (*TUR*, 3rd ed., ch. 3) and Babenko both land here; the Terraform docs themselves warn that CLI workspaces "are not a suitable tool for system decomposition or environments." Workspaces share one backend config (so prod and dev live in the same bucket and the same blast radius), the active workspace is implicit CLI state (the wrong-workspace `apply` is a real incident pattern), and code differences degrade into `count = terraform.workspace == "prod" ? 1 : 0` sprinkled through modules.

Directory layout:

```
live/
  dev/      us-east-1/ network/  data/  compute/
  stage/    us-east-1/ network/  data/  compute/
  prod/     us-east-1/ network/  data/  compute/
            eu-west-1/ ...
modules/
  vpc/  rds/  ecs-service/  ...
```

Each leaf is a root module with its own backend, state, provider config, and `*.tfvars`. Terragrunt is the mainstream way to keep this DRY without re-introducing workspace-style implicit state; OpenTofu's `early-eval` of variables in `backend` blocks (1.8+) reduces the need. **On HCP Terraform / TFE, Terraform Stacks (GA 2025)** is HashiCorp's first-party alternative — `*.tfstack.hcl` describes components and deployments and one apply orchestrates many state files behind a declarative dependency graph. Stacks is closed-source and HCP/TFE-only; OpenTofu has explicitly said it will not implement Stacks-the-syntax and is iterating on early-eval + state primitives instead. For OSS users, stay with Terragrunt or pure early-eval.
- Source: HashiCorp, *Terraform Stacks*, https://developer.hashicorp.com/terraform/language/stacks .

**Workspaces are still useful for** ephemeral per-PR or per-feature stacks of the *same* environment (preview environments). That's their original intent.

### 5a. Ephemeral environments — operational rules

Per-PR / per-feature stacks are the right use of workspaces, but they are also the most common source of $40k surprise cloud bills and forgotten public S3 buckets. Treat them as production-adjacent.

- **Must — every ephemeral environment has a TTL and an automatic destroy job.** 72 hours is a reasonable default; the destroy runs unconditionally unless an explicit "extend" label is on the PR. The job runs on a schedule, not "when the PR closes" — closed PRs and abandoned branches are the leak.
  - Sources: HashiCorp, *TFC ephemeral workspaces / auto-destroy*, https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/deletion ; AWS, *Tagging best practices for cost allocation*, https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/welcome.html .
- **Must — every ephemeral resource carries owner / PR / TTL tags** — `Owner=<github-handle>`, `PR=<number>`, `ExpiresAt=<iso8601>`, `Environment=ephemeral`. The destroy job uses these tags as the authoritative selector; resources without them are reaped on a stricter schedule.
  - Sources: AWS tagging best practices (above); Microsoft, *Cloud Adoption Framework — Resource tagging*, https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-tagging .
- **Should — cap cost per ephemeral stack with a cloud-native budget plus hard quota.** AWS Budgets + Service Quotas, Azure Cost Management + policy, GCP Budgets — alert at 50%, page at 100%, refuse provisioning beyond a per-stack ceiling (no instance class above `large`, no managed DB above the smallest tier).
- **Should — isolate ephemeral environments at the strongest boundary your org tolerates.** Separate AWS account / Azure subscription / GCP project per stack is the safe default; a shared "sandbox" account is acceptable only when network and IAM are segregated. *Never* deploy ephemeral stacks into a prod account.
- **Avoid — manual cleanup as the cleanup story.** If destroy is a human task, the leak rate trends to 100%.

---

## 6. Drift detection

Drift is reality diverging from desired state — someone clicked in the
console, an autoscaler edited a tag, another tool wrote a resource you also
manage.

**Should — run `tofu plan` on a schedule against every prod state and alert on non-empty diffs.** Daily is the floor; every few hours for high-blast-radius environments. Mechanism doesn't need to be fancy: a GitHub Actions cron, a self-hosted runner posting to Slack, or the named feature on a TACOS — **HCP Terraform Health → Drift Detection** (formerly continuous validation), **Spacelift Drift Detection**, **env0 Drift Detection**, or **Scalr** drift per environment. For Bicep-only stacks, schedule `az deployment group what-if --no-pretty-print` against the last applied template. For raw CloudFormation, `DetectStackDrift` + an EventBridge rule is the native path. AWS Config / Azure Resource Graph / GCP Asset Inventory complement these by catching out-of-band changes the IaC tool can't see.
- Sources: HashiCorp, *HCP Terraform health checks / drift detection*, https://developer.hashicorp.com/terraform/cloud-docs/workspaces/health ; Spacelift, *Drift detection*, https://docs.spacelift.io/concepts/stack/drift-detection .

**Prefer — fail loudly, then triage.** A drift diff is either an emergency change to reconcile into code, another tool fighting you for ownership, or a provider bug. None are "ignore." Persistent accepted drift (e.g. autoscaler-managed desired-capacity) belongs in `lifecycle { ignore_changes = [...] }` with a comment.

**Avoid — auto-remediation.** Drift detection that auto-applies will eventually destroy something. The remediation is a human opening a PR.

---

## 7. IaC testing

Three layers, in cost order: (1) **Static** — `validate`, `fmt -check`, `tflint`, `bicep lint`. Free, every PR. (2) **Policy / security scan** — Checkov, Trivy (tfsec is folded in), KICS, `cdk-nag`, `psrule` for Bicep, OPA Conftest for org policy. (3) **Integration** — `terraform test` (1.6+), Terratest, `pytest-terraform`, Pulumi's harness. Real `apply` into a sandbox; nightly or per-release.

**Must — at least one static gate (`validate` + `fmt -check` + `tflint` / `bicep lint`) on every PR.** Source-only, no credentials, fast feedback.
- Sources: HashiCorp, *`terraform validate`*, https://developer.hashicorp.com/terraform/cli/commands/validate ; tflint, README, https://github.com/terraform-linters/tflint .

**Should — gate every PR on a source-level policy/security scan, and gate prod-bound or internet-exposed stacks on a plan-level scan.** Source scans (Checkov / Trivy / cdk-nag / psrule reading `.tf`/`.bicep`) need no cloud credentials and run in seconds. Plan-level scans (the same tools reading `terraform show -json plan.out`) catch issues that depend on resolved variable values, but require a `plan` — privileged credentials, slower feedback, harder PR ergonomics. Reserve plan-level scans for stacks where the marginal coverage matters: prod, regulated, internet-exposed.

**Should — codify org-specific policy as OPA/Rego or Sentinel and gate apply on it.** "No untagged resources," "no instance type bigger than `xlarge` without a label," "S3 buckets must have lifecycle rules." Conftest + Rego is the open option; Sentinel is the TFC/TFE option. Either way, policy lives in version control and is reviewable.

**Prefer — `tofu test` (1.8+) / `terraform test` (1.7+) with `mock_provider` and `override_resource` / `override_data` / `override_module` for module contract tests; Terratest only for the few cross-resource end-to-end paths where you want a real cloud round-trip.** Mocks let the native framework run in `command = apply` mode without cloud credentials — that was the main reason teams stayed on Terratest, and it's gone.
- Sources: Terraform 1.7.0 release notes (`mock_provider`, `override_*`), https://github.com/hashicorp/terraform/releases/tag/v1.7.0 ; OpenTofu 1.8.0 release notes (provider mocking, resource overrides), https://github.com/opentofu/opentofu/releases/tag/v1.8.0 .

**Avoid — testing your provider.** "I created an `aws_s3_bucket`, let me assert that an `aws_s3_bucket` exists" proves nothing. Test your *composition*: that your service module wires alarms to the right SNS topic, that private subnets actually have no IGW route.

### 7a. AI-assisted authoring

Copilot, Cursor, Pulumi Copilot, and AVM's Spec Kit are now default-on for IaC. The failure modes are concrete: hallucinated provider attribute names that pass `validate` but fail `plan`; out-of-date syntax (e.g. `aws_s3_bucket_acl` semantics the model learned before the split); silent regression of `for_each` to `count`; silent removal of `lifecycle { ignore_changes }` or `moved {}` blocks during a "refactor."

- **Must — every AI-authored change goes through the same gates as a human PR**: `validate` + `fmt -check` + `tflint` / `bicep lint`, `tofu test` with mocks, the source/plan-level policy scan from §7, the apply gate from §10. No "trusted-author" fast path for an LLM.
- **Should — pin the LLM's context to the actual provider lockfile** (the `.terraform.lock.hcl` provider versions, the AVM module version, the Pulumi provider pin). The model's training cutoff is not a substitute for `provider docs @ v5.62`.
- **Avoid — accepting LLM diffs that change `for_each` to `count`, drop `lifecycle { ignore_changes = [...]}`, remove `moved {}` blocks, or rewrite `import {}` blocks** without an explicit reviewer note explaining why. These are the silent-destroy class of changes.
- Source: Microsoft, *Azure Verified Modules — AI-Assisted IaC Solution Development*, https://azure.github.io/Azure-Verified-Modules/experimental/ai-assisted-sol-dev/ .

---

## 8. Secrets in IaC

The unbreakable rules. Every one has caused a public incident.

**Must — no plaintext secrets in `.tf`, `.tfvars`, Bicep parameters, Pulumi config, or any file in version control** — including "dev" secrets, throwaway tokens, "I'll rotate it later" placeholders. Once it hits the repo, rotate.
- Sources: HashiCorp, *Sensitive data in state and config*, https://developer.hashicorp.com/terraform/language/state/sensitive-data ; OWASP, *Secrets Management Cheat Sheet*, https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html .

**Must — no plaintext secrets in `terraform plan` output committed to a PR or posted to a CI log.** Plan output prints variable values; CI logs are read by more people than your state bucket. Mark variables `sensitive = true` (Terraform/OpenTofu) or `@secure()` (Bicep) and ensure CI redacts.
- Sources: HashiCorp, *Sensitive input variables*, https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output ; Microsoft, *Bicep secure parameters*, https://learn.microsoft.com/azure/azure-resource-manager/bicep/parameters#secure-parameters .

**Must — assume legacy state contains secrets.** Generated RDS passwords, random tokens, anything from `random_password`, anything fetched from a data source that returns a secret — if it was authored before Terraform 1.10, it is in state. State permissions and encryption are part of your secret-handling story, not separate. New code can do strictly better; see the next rule.

**Prefer — ephemeral resources and write-only arguments (Terraform 1.10+ / equivalent OpenTofu) for credentials a provider only needs at apply time.** `password_wo` + `password_wo_version` on `aws_db_instance`, `secret_string_wo` on `aws_secretsmanager_secret_version`, `ephemeral` blocks for short-lived inputs and outputs. The value is consumed by the provider during apply and **never written to state or to the saved plan file** — strictly stronger than `sensitive = true`, which only suppresses CLI output. Combine with `ephemeralasnull` for safe pass-through to non-ephemeral consumers.
- Sources: Terraform 1.10.0 release notes (ephemeral resources, ephemeral values, `ephemeralasnull`), https://github.com/hashicorp/terraform/releases/tag/v1.10.0 ; HashiCorp, *Ephemeral resources and write-only arguments*, https://developer.hashicorp.com/terraform/language/resources/ephemeral .

**Should — fetch secrets at apply time via data sources from a secret manager** (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager). For older providers without write-only support, the value still lands in state, but the source of truth is the secret manager and rotation works. Migrate to the `*_wo` form whenever the provider gains it.

**Avoid — `TF_VAR_db_password` env vars in CI without a backing secret store.** One careless `printenv` or `set -x` from a leak. Pull from the secret manager inside the job, not staged in the runner's environment.

---

## 9. Provider versioning, lock files, and supply chain

Reproducibility starts with pinning, and supply chain starts with knowing
*where* a provider comes from.

**Must — commit the lock file for both IaC providers and the host language.** For Terraform/OpenTofu that is `.terraform.lock.hcl` (pins provider versions and per-platform checksums). For Pulumi and CDK there is **no IaC-specific lock file**; the lock is the host language's lock — `package-lock.json` / `pnpm-lock.yaml` / `yarn.lock` for TS, `poetry.lock` or `requirements.txt` (with hashes via `pip-compile --generate-hashes`) for Python, `go.sum` for Go, `packages.lock.json` for .NET. Without it, `init` / `npm ci` / `pip install` on a new machine can pick up a different provider build and a different plan.
- Sources: HashiCorp, *Dependency Lock File*, https://developer.hashicorp.com/terraform/language/files/dependency-lock ; OpenTofu, *Dependency lock file*, https://opentofu.org/docs/language/files/dependency-lock/ ; AWS, *AWS CDK and dependencies*, https://docs.aws.amazon.com/cdk/v2/guide/work-with.html ; Pulumi, *Provider versioning*, https://www.pulumi.com/docs/iac/concepts/resources/providers/#provider-versioning .

CDK additionally has **`cdk.context.json`** — the cached results of context lookups (AZ lists, AMI IDs, VPC lookups). It is **not** a lock file but it *is* a reproducibility artifact: commit it, and treat changes as review-worthy (a different AMI ID is a different deployment).
- Source: AWS, *CDK Context — `cdk.context.json`*, https://docs.aws.amazon.com/cdk/v2/guide/context.html .

**Must — pin provider versions in root modules / applications with an upper bound; in shared/reusable modules, declare a *minimum* compatible version and let the root constrain final selection.** In a root module or a CDK/Pulumi app, `version = "~> 5.40"` (pessimistic constraint) or an exact `npm`/`pip` pin keeps `init` reproducible. In a *reusable* module, upper bounds force every consumer to upgrade in lockstep with the module author and break composition; declare the lowest version you actually support (`version = ">= 5.0"`) and let the root reconcile.
- Sources: HashiCorp, *Version constraints — best practices*, https://developer.hashicorp.com/terraform/language/expressions/version-constraints#best-practices ; HashiCorp, *Module providers*, https://developer.hashicorp.com/terraform/language/modules/develop/providers .

**Should — pin the IaC CLI version too.** `required_version = "~> 1.8"`. CI installs the matching version; local dev uses `tfenv`/`tofuenv`/`mise`/`asdf`.

**Should — refresh lock files in a dedicated PR.** Provider upgrades change plans subtly (new computed attributes, new defaults). A PR that does only `init -upgrade` and shows the resulting plan diff is reviewable; a PR that mixes a feature with a provider bump is not.

### 9a. Provider source addresses, mirrors, and provenance

A pinned version is half the story. The other half is *which copy* of that provider you ran.

- **Must — declare an explicit `source` for every provider.** `source = "hashicorp/aws"` (public registry) or `source = "registry.acme.corp/acme/aws"` (private mirror). The implicit-namespace shortcut was removed in Terraform 0.13 and never existed in OpenTofu; relying on it is a supply-chain hole.
  - Sources: HashiCorp, *Provider Requirements*, https://developer.hashicorp.com/terraform/language/providers/requirements ; OpenTofu, *Provider Requirements*, https://opentofu.org/docs/language/providers/requirements/ .
- **Should — run a provider mirror or private registry** for any team that builds on a release schedule. Terraform and OpenTofu both support filesystem and network mirrors via CLI configuration (`provider_installation { network_mirror { url = "https://…" } }`); Artifactory, Nexus, and the cloud-vendor artifact stores all front the registry protocol. The mirror gives you an audit trail of which provider binaries shipped, decouples CI from upstream registry availability, and is the precondition for air-gapped builds.
  - Sources: HashiCorp, *CLI Configuration — provider installation*, https://developer.hashicorp.com/terraform/cli/config/config-file#provider-installation ; OpenTofu, *Provider installation*, https://opentofu.org/docs/cli/config/config-file/#provider-installation .
- **Prefer — for OpenTofu shops, an OCI registry as the mirror.** OpenTofu 1.10 (2025) shipped first-class OCI support for **both** providers and modules: `oci_mirror { repository_template = "…" include = ["registry.opentofu.org/*/*"] }` in CLI config, and `module "vpc" { source = "oci://example.com/modules/vpc/aws" }` in HCL. Any standard OCI registry works — ACR, ECR, GHCR, Harbor, Zot, Artifactory's OCI front-end — which collapses IaC and container supply-chain tooling onto one signing/scanning pipeline (cosign, Sigstore attestations, OCI policy). Terraform OSS still requires the legacy provider-mirror protocol; this is one of the larger CLI feature deltas as of 2025.
  - Source: OpenTofu 1.10.0 release notes (OCI Registry Support), https://github.com/opentofu/opentofu/releases/tag/v1.10.0 .
- **Should — turn on plugin caching in CI** (`plugin_cache_dir`) so every job doesn't re-download every provider; combined with the lock file's `h1:`/`zh:` checksums, a cache hit is also a provenance check.
- **Should — verify provider signatures where the registry publishes them** and fail `init` on checksum mismatch; never auto-update the lock from CI.
- **Avoid — `terraform init -upgrade` in a job that also runs `apply`.** The upgrade rewrites the lock; the apply runs against the new providers; the diff is invisible to review. Split the steps.

---

## 10. Plan-review discipline

The PR is the apply gate. Treat it that way.

**Must — every change produces a `plan` artifact that the apply step consumes; the *summary* is posted to the PR, the full plan lives in the access-controlled CI store.** Posting raw plans into the PR conversation conflicts with §8 (sensitive values, resource IDs, IAM principals). Right pattern: redacted summary on the PR (resource counts, change types, flagged resources, policy results); full `plan.out` + `terraform show -json` artifact in the CI run, accessible only to the apply role and reviewers with the same scope. Atlantis, Spacelift, env0, and TFC all support this split; roll-your-own GitHub Actions need to opt in. The apply step must consume the *exact* plan artifact from the same run, never re-plan.
- Sources: HashiCorp, *Saved plans (`-out`)*, https://developer.hashicorp.com/terraform/cli/commands/plan#out-filename ; HashiCorp, *Sensitive input variables* (above).

**Should — apply runs from CI against the merged commit, not from a laptop.** Local apply means local credentials, local CLI versions, local lock contention, no audit trail. Break-glass local apply must be loud (logged, paged) and rare. (Break-glass identity, multi-party check-out, and post-use rotation are owned by [ch12 §8 Break-glass](./12-identity.md#8-break-glass).)

**Should — two-person rule on prod apply.** PR approval + apply approval, by different people. TFC/Spacelift/env0 support this natively; with Atlantis or custom actions, gate on a second `/approve` from a non-author.

**Prefer — OPA / Sentinel policy gates between plan and apply.** A reviewer should not be the last line of defense against "this plan deletes a prod RDS." `deny` rules on `delete` actions targeting `Environment=prod` outside a change window are enforceable; reviewer attention is not.

---

## 11. Cost preview

**Prefer — Infracost in the PR pipeline for repos with meaningful run-rate.** Posts a delta comment ("this PR adds $1,240/mo, mostly an `r6i.4xlarge` and 2 TB of `gp3`"). Signal-to-noise is high enough that engineers read it.

**Avoid — wiring it into low-cost dev/sandbox repos.** "$0.40/mo more" comments dilute the signal where cost matters. Gate on a threshold (~$50/mo).

**Avoid — treating Infracost as budget enforcement.** It's a *preview* tool; budget enforcement belongs in the cloud's native budgets service.

---

## 12. Kubernetes-as-target IaC

Full Kubernetes coverage is chapter 04; the IaC slice:

**Prefer — Helm for third-party charts, Kustomize for your own manifests.** Helm's templating is powerful and unsafe (string-templated YAML); right tool when *consuming* someone else's chart. Kustomize's overlay model is safer for your own workloads — typed patches, no Go templating surprises.

**Avoid — `helm template | kubectl apply` for your own charts long-term.** You lose Helm's release tracking and gain none of Kustomize's clarity.

**Prefer — Crossplane when the cloud control plane *is* your platform API.** "App team submits a namespaced `PostgresInstance` XR in their team namespace, gets an RDS instance" is the sweet spot. Crossplane v2 (GA 2025) made XRs and MRs namespaced by default and dropped claims; new platforms should start there, and v1 cluster-scoped XRs survive only as `LegacyCluster` for migration. Composition Functions (Go / Python / KCL) remain the writable layer.

**Avoid — Crossplane as a Terraform replacement for shops that don't already run Kubernetes as a platform.** You're trading one state-and-reconciliation system for two (etcd + provider caches). Operational cost is real.

---

## 13. Repo topology

The decision lives between "one repo for everything IaC" and "one repo per team / per stack." Pick deliberately.

- **Prefer — a single monorepo (`live/` + `modules/`) for small orgs (≤ ~3 platform-adjacent teams) and any team starting from scratch.** One CODEOWNERS, one CI config, atomic refactors across modules and consumers, one place to grep. Cost is coarser access control and CI scope filtering.
- **Should — split into `infra-live` (per-environment root configs) and `infra-modules` (versioned reusable modules) once modules are consumed by more than one team.** The module repo gets semver tags and a CHANGELOG; the live repo pins versions and is the apply-gated repo. CODEOWNERS on `infra-live/prod/**` enforces the two-person rule; the module repo has open contribution with a stricter review bar on breaking changes.
- **Should — keep application code out of both.** App repos consume infra outputs (via SSM/Key Vault/remote state); they don't live alongside `*.tf`. Mixing them couples release cadence and dilutes ownership.
- **Avoid — one repo per stack ("infra-vpc-prod", "infra-eks-prod", …).** The "atomic refactor across N stacks" cost compounds. Directory boundaries inside a single repo give you the same blast-radius split without the cross-repo PR dance.

### 13a. Multi-engine estates

Real platform orgs run several IaC engines simultaneously — Bicep for Azure landing zones (AVM is Bicep-first), OpenTofu for AWS, CDK for serverless apps, Crossplane v2 for namespaced app-team self-service. The "one tool per state file" rule from §2 is right at the *state* level; the *org* level needs its own discipline.

- **Should — one engine owns each resource class.** Record ownership in a `ManagedBy=<engine>:<repo>:<state>` tag on cloud resources (or a label on K8s-shaped resources). Reviewers can answer "who owns this SG?" without grepping every repo.
- **Should — use the cloud-native inventory as the cross-engine source of truth** — AWS Config, Azure Resource Graph, GCP Asset Inventory. No single engine's state can see what a different engine is managing; the inventory can. Reconcile inventory against the union of engine states on a schedule and alert on resources with no `ManagedBy` tag or with two.
  - Sources: AWS, *AWS Config*, https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html ; Microsoft, *Azure Resource Graph*, https://learn.microsoft.com/azure/governance/resource-graph/overview .
- **Avoid — two engines managing the same resource, even transiently.** Second-writer-wins; the loser silently drifts and the §6 schedule will eventually catch it, but only after damage.

---

## 14. Anti-patterns checklist

Reject in review unless an exception is on file: `provider` blocks inside reusable modules; floating `ref=main` in module sources; `*.tfstate` in git; `count = length(var.subnets)` (use `for_each`); `terraform.workspace`-based environment branching; one state file covering more than one environment; unredacted plan output in a PR; provider block without an explicit `source` or version constraint; upper-bounded provider pins inside a *reusable* module; `apply -auto-approve` against a non-sandbox account without a policy gate; a "platform" module with >~15 input variables; ephemeral environments without TTL or owner tags; Ansible used to provision cloud resources that have a real provider; CDK chosen on the basis of "we'll port it to GCP later."

---

## 15. Recommended starting stack (2026)

For a new platform team with no existing IaC: **OpenTofu 1.10+** (with `tfenv`/`mise` pinning); **monorepo** with `live/<env>/<region>/<stack>/` plus `modules/`, splitting to a separate `infra-modules` repo once a module is consumed by ≥3 teams; **state on S3 with native `use_lockfile`** (or GCS / Azure Blob equivalent) with object versioning, KMS-CMK server-side encryption *and* OpenTofu client-side state encryption on top; **CI** through Atlantis (self-hosted) or Spacelift / env0 / TFC (managed) with redacted PR plan summary (full plan in access-controlled CI artifact store), two-person prod apply, OPA Conftest gate, Checkov + tflint source scan every PR and plan-level scan on prod-bound stacks; **provider mirror** via OCI registry (ACR / ECR / GHCR / Harbor) with `oci_mirror` for providers and `oci://` module sources, plus `plugin_cache_dir` in CI; **Infracost** on PRs above $50/mo delta; **scheduled `tofu plan`** every 6h per prod stack with non-empty alerting; **ephemeral PR stacks** in a separate sandbox account with 72h TTL, owner + PR + ExpiresAt tags, per-stack budget alerts; **secrets** via ephemeral resources / `*_wo` write-only arguments where the provider supports them, otherwise pulled from Vault or the cloud-native secret manager — never in repo or staged in CI envs; **testing** via `tofu validate` + `tflint` + `checkov` per PR, `tofu test` with `mock_provider` for module contracts in CI, Terratest only for the few cross-resource end-to-end paths that matter. **Alternative for HCP/TFE shops:** swap Terragrunt-style DRY for **Terraform Stacks** as the multi-state orchestrator.

Boring on purpose. Boring infrastructure tooling is the goal.

---

## Sources

Primary canon for this chapter:

- HashiCorp Terraform docs — *State*, *S3 Backend (`use_lockfile`)*, *Remote state*, *Sensitive data in state*, *Sensitive input variables*, *Ephemeral resources and write-only arguments*, *Dependency lock file*, *Provider Requirements*, *Version constraints — best practices*, *Module providers*, *Functions (provider-defined)*, *Terraform Stacks*, *CLI configuration / provider installation*, *Saved plans (`-out`)*, *Manipulating state*, *`terraform validate`*, *HCP Terraform health checks / drift detection*, *TFC ephemeral workspaces / auto-destroy*. (developer.hashicorp.com)
- Terraform release notes — 1.7.0 (`mock_provider`, `override_*`), 1.10.0 (ephemeral resources, native S3 locking, DynamoDB deprecation). (github.com/hashicorp/terraform)
- OpenTofu docs — *Migrating from Terraform*, *S3 backend*, *State encryption*, *Dependency lock file*, *Provider Requirements*, *Provider installation / mirrors*, *Backend configuration / migration*. (opentofu.org)
- OpenTofu release notes — 1.8.0 (provider mocking), 1.10.0 (OCI registry, native S3 locking, `-target-file`/`-exclude-file`). (github.com/opentofu/opentofu)
- AWS — *S3 SSE-KMS*, *Tagging best practices for cost allocation*, *AWS CDK Developer Guide* (`work-with`, `context`), *AWS Config*. (docs.aws.amazon.com)
- Microsoft — *Bicep secure parameters*, *Cloud Adoption Framework — Resource tagging*, *Azure Verified Modules*, *AVM AI-Assisted IaC Solution Development*, *Azure Resource Graph*. (learn.microsoft.com / azure.github.io)
- Pulumi — *Provider versioning*, *Secrets and configurable secrets providers*. (pulumi.com)
- Crossplane — *What's new in Crossplane v2*, GitHub releases. (docs.crossplane.io / github.com/crossplane/crossplane)
- Spacelift — *Drift detection*. (docs.spacelift.io)
- IBM Newsroom — *IBM Completes Acquisition of HashiCorp* (27 Feb 2025). (newsroom.ibm.com)
- OWASP — *Secrets Management Cheat Sheet*. (cheatsheetseries.owasp.org)
- tflint project README. (github.com/terraform-linters/tflint)
- Yevgeniy (Jim) Brikman, *Terraform: Up & Running* (3rd ed., O'Reilly, 2022) — chs. 1, 3, 5.
- Anton Babenko, *terraform-best-practices.com* — module structure, environment separation.
- ThoughtWorks Technology Radar — entries for OpenTofu, Crossplane, Chef / Puppet (Hold), Nix.
