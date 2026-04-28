# Infrastructure Engineering — Chapter 02: IaC Tooling

Opinionated, cross-cutting defaults for choosing and running infrastructure-as-code
tools in 2026. The *philosophy* of declarative infra (idempotency, desired state,
GitOps, blast radius) lives in chapter 01; this chapter is the hands-on tooling
layer. Kubernetes-as-target is touched on at the end and covered in depth in
chapter 04.

> Conventions: **Must** = required default, ≥2 independent canonical sources.
> **Should** = strong default, deviation needs written justification.
> **Prefer** = pick this unless you have measurement or context that says otherwise.
> **Avoid** = reject in review unless an exception is filed.

---

## 1. The 2026 landscape

- **Terraform (HashiCorp, BUSL 1.1 since Aug 2023)** — largest ecosystem and
  registry. IBM-owned since Feb 2025. Non-OSI; TF Cloud / TFE is the upsell.
- **OpenTofu (Linux Foundation, MPL-2.0)** — fork of Terraform 1.5.7 (Sept
  2023). Drop-in CLI, same HCL, same provider protocol. Ships features
  Terraform doesn't have: client-side state encryption, early-eval in
  `backend` blocks, provider `for_each`, `-exclude`. Real fork.
- **Pulumi** — TS/Python/Go/.NET/Java over the same provider model. Right
  call when you need real abstraction (classes, loops, unit tests) that HCL
  fights you on, and your team is fluent in one of those languages.
- **AWS CDK** — TS/Python/Java/.NET that synthesizes CloudFormation. AWS-only.
- **Bicep** — Microsoft DSL over ARM. Azure-only; right Azure default. ARM is
  legacy. Raw CloudFormation only for things CDK can't reach.
- **Crossplane** — Kubernetes control plane that reconciles cloud resources
  via CRDs. Composition Functions (GA v1.17, 2024) made compositions writable
  in real languages. Pick when Kubernetes *is* your platform API.
- **Ansible** — *configuration management*, not IaC. Procedural, push-based,
  mutable-target. Use for OS-level config of long-lived VMs and network gear.
  **Chef / Puppet** in decline (ThoughtWorks Radar: Hold) — migration target
  for existing estates is immutable images + cloud-init, not Ansible.
- **Nix / NixOS** — ascendant for reproducible build/dev environments and OS
  images. Replaces `Dockerfile` + `apt-get` + Packer, not Terraform.

### The OpenTofu fork — position

On 10 Aug 2023 HashiCorp relicensed Terraform from MPL-2.0 to BUSL-1.1. The
fork was adopted by the Linux Foundation in Sept 2023 and shipped 1.6 GA in
Jan 2024. State and modules are bidirectionally compatible with Terraform
1.5.x. **Default new HCL work to OpenTofu** — OSI license (no
vendor/consultancy legal review), faster cadence on features people want,
compatible with the Terraform Registry today. Stay on Terraform only if
you're deep in TFC/TFE, you depend on a HashiCorp-only feature (Stacks), or
compliance requires a single commercial vendor. Do **not** run both CLIs
against the same state.

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

**Must — pick one IaC tool per repo and one CLI per state file.** Mixing
Terraform and OpenTofu against the same state, or CDK and raw CloudFormation
against the same stack, produces drift you cannot diff.

**Should — do not pick a multi-cloud tool to enable a multi-cloud strategy
you don't have.** Brikman (*TUR*, 3rd ed., ch. 1) and the AWS CDK Best
Practices guide both land here: the abstraction tax of "cloud-agnostic" code
is real, and most "multi-cloud" orgs are actually "primary cloud + a few SaaS
providers." Choose for the cloud you actually run in.

**Avoid — CDK on the basis of "we might leave AWS one day."** CDK
synthesizes CloudFormation; output is AWS-shaped down to the IAM model.
Escape hatch is "rewrite," not "swap a backend."

**Avoid — Pulumi *because* you don't like HCL.** HCL fluency is a one-week
investment. Pulumi's payoff is real abstraction — shared component classes,
unit tests, loops that don't fight `count`/`for_each`. If your modules are
200 lines of resources with a few `for_each`, HCL is the right tool.

---

## 3. Module / component design

Modules are the unit of reuse. The failure mode is the 2,000-line "platform"
module that takes 47 variables and conditionally creates everything.

**Must — every module is single-purpose and composable.** Anton Babenko's
terraform-best-practices.com and the official Terraform "Standard Module
Structure" both define this: one module = one logical thing (a VPC, an RDS
instance with its parameter group, an EKS node group). Composition happens in
a *root* module (or in Terragrunt / Pulumi program), not inside a leaf.

**Should — three layers, not two.** *Resource modules* (~50–300 lines) are
thin wrappers over a provider resource set, versioned and registry-publishable.
*Service modules* are opinionated compositions ("a standard web service":
ALB + target group + ECS service + log group + alarms), internal to the org.
*Root / live config* per-environment instantiations contain no logic, only
wiring and `tfvars`.

**Should — version every shared module and pin the consumer.** Use semver
tags: `source = "git::ssh://...//modules/vpc?ref=v1.4.2"` or registry
`version = "~> 1.4"`. Floating refs (`ref=main`) in a root module are an
outage waiting to happen.

**Prefer — registry over in-repo monorepo for modules consumed by ≥3 teams.**
Below that threshold, the operational cost of a registry exceeds the benefit.

**Avoid — `count` for resource toggles when `for_each` will do.** `count`
re-indexes on removal and forces destroy/create churn; `for_each` keys by a
stable string. Most common cause of "why did my apply want to recreate
everything." Reserve `count` for the `enabled = true/false` boolean-gate
pattern, preferably at the module level.

**Avoid — inline `provider` blocks in shared modules.** Modules declare
`required_providers` with version constraints; the root passes configured
providers in. Most common reason a module "works locally but not in CI."

---

## 4. State management

State is the most dangerous artifact in your IaC system: plaintext secrets,
the source of truth the tool trusts over reality, multi-hour incident when
corrupted.

**Must — remote state with locking, always.** S3+DynamoDB, GCS native lock,
Azure Blob lease, TFC, OpenTofu HTTP backend with a lock server. Local state
in CI guarantees overwrite races.

**Must — encrypt state at rest and treat it as a secret.** Bucket-level
encryption is necessary, not sufficient — restrict bucket access to the apply
role, enable object versioning, enable access logs.

**Should — turn on OpenTofu's client-side state encryption** (or KMS-envelope
encryption on Terraform). Bucket-level protects against a stolen disk;
client-side protects against a stolen IAM credential with `s3:GetObject`.

**Must — one state file per blast radius.** Monolithic "all of prod" state
means a bad refactor can plan to destroy your VPC. Split by environment, then
by lifecycle (network, data, compute, edge). Cross-state references go
through `terraform_remote_state` — or better, via SSM/Parameter Store/Key
Vault where the producer publishes outputs and consumers read by name.

**Prefer — declarative `import` blocks (Terraform 1.5+/OpenTofu 1.6+) over
`terraform import` CLI or hand-edited state.** Reviewable in a PR, removable
after the next apply. `state rm`/`state mv` are incident escape hatches.

**Avoid — `terraform state push` against a shared backend.** If you must:
take the lock manually, snapshot to an audited location, second engineer
reviews the diff. Single command most likely to cause a multi-hour incident.

**Avoid — committing `terraform.tfstate` (or any `*.tfstate*`) to git.** Will
leak secrets; will be wrong within a day. `.gitignore` at repo root.

---

## 5. Environment separation

This is contentious. The position:

**Should — separate state files in separate directories per environment, not
workspaces.** Brikman (*Terraform: Up and Running*, 3rd ed., ch. 3) and
Babenko both land here; the Terraform docs themselves warn that CLI
workspaces "are not a suitable tool for system decomposition or
environments." Workspaces share one backend config (so prod and dev live in
the same bucket and the same blast radius), the active workspace is implicit
CLI state (the wrong-workspace `apply` is a real incident pattern), and code
differences degrade into `count = terraform.workspace == "prod" ? 1 : 0`
sprinkled through modules.

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

Each leaf directory is a root module with its own backend, its own state, its
own provider config, and its own `*.tfvars`. Terragrunt is the mainstream way
to keep this DRY without re-introducing workspace-style implicit state;
OpenTofu's `early-eval` of variables in `backend` blocks (1.8+) reduces the
need.

**Workspaces are still useful for** ephemeral per-PR or per-feature stacks of
the *same* environment (preview environments). That's their original intent.

---

## 6. Drift detection

Drift is reality diverging from desired state — someone clicked in the
console, an autoscaler edited a tag, another tool wrote a resource you also
manage.

**Should — run `tofu plan` on a schedule against every prod state and alert
on non-empty diffs.** Daily is the floor; every few hours for high-blast-
radius environments. Mechanism doesn't need to be fancy: a GitHub Actions
cron, a Spacelift/Atlantis/env0/TFC drift task, or a self-hosted runner
posting to Slack. The managed options' value-add is UI and routing.

**Prefer — fail loudly, then triage.** A drift diff is either an emergency
change to reconcile into code, another tool fighting you for ownership, or a
provider bug. None are "ignore." Persistent accepted drift (e.g. autoscaler-
managed desired-capacity) belongs in `lifecycle { ignore_changes = [...] }`
with a comment.

**Avoid — auto-remediation.** Drift detection that auto-applies will
eventually destroy something. The remediation is a human opening a PR.

---

## 7. IaC testing

Three layers, in cost order: (1) **Static** — `terraform validate`, `tofu fmt
-check`, `tflint`, `bicep build`/`bicep lint`. Run on every PR. Free.
(2) **Policy / security scan** — Checkov, KICS, tfsec (folded into Trivy),
`cdk-nag`, `psrule` for Bicep, OPA Conftest for org policy. Read source or
`plan` JSON; apply rules ("no public S3," "RDS encryption on," "no
0.0.0.0/0 SSH"). (3) **Integration** — Terratest (Go), `pytest-terraform`,
native `terraform test` (1.6+), Pulumi's testing harness. Real `apply` into a
sandbox; slow, run nightly or per-release.

**Must — at least one static and one policy scanner gate every PR.** Checkov
- tflint is the safe default. Scanning the *plan output*, not just source,
catches issues that depend on resolved variable values.

**Should — codify org-specific policy as OPA/Rego or Sentinel and gate apply
on it.** "No untagged resources," "no instance type bigger than `xlarge`
without a label," "S3 buckets must have lifecycle rules." Conftest + Rego is
the open option; Sentinel is the TFC/TFE option. Either way, policy lives in
version control and is reviewable.

**Prefer — `terraform test` / `tofu test` for module contract tests, Terratest
only for cross-resource end-to-end paths in a real cloud.** The native
framework covers ~80% of what teams used Terratest for, without the Go
toolchain.

**Avoid — testing your provider.** "I created an `aws_s3_bucket`, let me
assert that an `aws_s3_bucket` exists" proves nothing. Test your
*composition*: that your service module wires alarms to the right SNS topic,
that private subnets actually have no IGW route.

---

## 8. Secrets in IaC

The unbreakable rules. Every one has caused a public incident.

**Must — no plaintext secrets in `.tf`, `.tfvars`, Bicep parameters, Pulumi
config, or any file in version control** — including "dev" secrets, throwaway
tokens, "I'll rotate it later" placeholders. Once it hits the repo, rotate.

**Must — no plaintext secrets in `terraform plan` output committed to a PR or
posted to a CI log.** Plan output prints variable values; CI logs are read by
more people than your state bucket. Mark variables `sensitive = true`
(Terraform/OpenTofu) or `@secure()` (Bicep) and ensure CI redacts.

**Must — assume state contains secrets.** Generated RDS passwords, random
tokens, anything from `random_password`, anything fetched from a data source
that returns a secret. State permissions and encryption are part of your
secret-handling story, not separate.

**Should — fetch secrets at apply time via data sources from a secret
manager** (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager).
The value still ends up in state, but the source of truth is the secret
manager and rotation works.

**Avoid — `TF_VAR_db_password` env vars in CI without a backing secret
store.** One careless `printenv` or `set -x` from a leak. Pull from the
secret manager inside the job, not staged in the runner's environment.

---

## 9. Provider versioning & lock files

Reproducibility starts with pinning.

**Must — commit the lock file.** `.terraform.lock.hcl` (Terraform/OpenTofu),
`pulumi-policy-lockfile` plus `package-lock.json`/`go.sum` (Pulumi),
`cdk.json` plus `package-lock.json` (CDK). The lock file pins both version
and provider checksums; without it, `init` on a new machine can pick up a
different provider build and a different plan.

**Must — pin provider versions with an upper bound.** `version = "~> 5.40"`,
not `>= 5.0`. Pessimistic constraint operator is the right default. Pulumi:
exact versions in `package.json`/`requirements.txt`. CDK: pin `aws-cdk-lib`;
don't chase weekly releases on prod.

**Should — pin the IaC CLI version too.** `required_version = "~> 1.8"`. CI
installs the matching version; local dev uses `tfenv`/`tofuenv`/`mise`/`asdf`.

**Should — refresh lock files in a dedicated PR.** Provider upgrades change
plans subtly (new computed attributes, new defaults). A PR that does only
`init -upgrade` and shows the resulting plan diff is reviewable; a PR that
mixes a feature with a provider bump is not.

---

## 10. Plan-review discipline

The PR is the apply gate. Treat it that way.

**Must — every change produces a `plan` artifact attached to the PR.**
Atlantis, Spacelift, env0, Terraform Cloud, or a roll-your-own GitHub Action
all do this. The reviewer reviews the *plan*, not just the diff. A two-line
diff can produce a plan that destroys a database; only the plan tells you.

**Should — apply runs from CI against the merged commit, not from a laptop.**
Local apply means local credentials, local CLI versions, local lock
contention, no audit trail. Break-glass local apply must be loud (logged,
paged) and rare.

**Should — two-person rule on prod apply.** PR approval + apply approval, by
different people. TFC/Spacelift/env0 support this natively; with Atlantis or
custom actions, gate on a second `/approve` from a non-author.

**Prefer — OPA / Sentinel policy gates between plan and apply.** A reviewer
should not be the last line of defense against "this plan deletes a prod
RDS." `deny` rules on `delete` actions targeting `Environment=prod` outside a
change window are enforceable; reviewer attention is not.

---

## 11. Cost preview

**Prefer — Infracost in the PR pipeline for repos with meaningful run-rate.**
Posts a delta comment ("this PR adds $1,240/mo, mostly an `r6i.4xlarge` and
2 TB of `gp3`"). Signal-to-noise is high enough that engineers read it.

**Avoid — wiring it into low-cost dev/sandbox repos** — "$0.40/mo more"
comments dilute the signal where cost matters. Gate on a threshold (~$50/mo).

**Avoid — treating Infracost as budget enforcement.** It's a *preview* tool;
budget enforcement belongs in the cloud's native budgets service.

---

## 12. Kubernetes-as-target IaC

Full Kubernetes coverage is chapter 04; the IaC slice:

**Prefer — Helm for third-party charts, Kustomize for your own manifests.**
Helm's templating is powerful and unsafe (string-templated YAML); right tool
when *consuming* someone else's chart. Kustomize's overlay model is safer for
your own workloads — typed patches, no Go templating surprises.

**Avoid — `helm template | kubectl apply` for your own charts long-term.**
You lose Helm's release tracking and gain none of Kustomize's clarity.

**Prefer — Crossplane when the cloud control plane *is* your platform API.**
"App team submits a `PostgresInstance` CR, gets an RDS instance" is the
sweet spot. Composition Functions (GA v1.17) made it operationally tractable.

**Avoid — Crossplane as a Terraform replacement for shops that don't already
run Kubernetes as a platform.** You're trading one state-and-reconciliation
system for two (etcd + provider caches). Operational cost is real.

---

## 13. Anti-patterns checklist

Reject in review unless an exception is on file: `provider` blocks inside
reusable modules; floating `ref=main` in module sources; `*.tfstate` in git;
`count = length(var.subnets)` (use `for_each`); `terraform.workspace`-based
environment branching; one state file covering more than one environment;
unredacted plan output in a PR; provider block without a version constraint;
`apply -auto-approve` against a non-sandbox account without a policy gate;
a "platform" module with >~15 input variables; Ansible used to provision
cloud resources that have a real provider; CDK chosen on the basis of "we'll
port it to GCP later."

---

## 14. Recommended starting stack (2026)

For a new platform team with no existing IaC: **OpenTofu 1.8+** (with
`tfenv`/`mise` pinning); **`live/<env>/<region>/<stack>/` directories** plus
`modules/`, with a private module registry once a module is consumed by ≥3
teams; **state on S3+DynamoDB** (or GCS / Azure Blob equivalent) with object
versioning, KMS-CMK encryption, OpenTofu state encryption on top; **CI**
through Atlantis (self-hosted) or Spacelift / env0 / TFC (managed) with PR
plan, two-person apply, OPA Conftest gate, Checkov + tflint scan;
**Infracost** on PRs above $50/mo delta; **scheduled `tofu plan`** every 6
hours per prod stack with non-empty alerting; **secrets** in Vault or the
cloud-native secret manager, never in repo or staged in CI envs; **testing**
via `tofu validate` + `tflint` + `checkov` per PR, `tofu test` module
contracts in CI, Terratest only for the few cross-resource end-to-end paths
that matter.

Boring on purpose. Boring infrastructure tooling is the goal.
