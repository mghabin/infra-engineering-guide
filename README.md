# infra-engineering-guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/mghabin/infra-engineering-guide/badge)](https://securityscorecards.dev/viewer/?uri=github.com/mghabin/infra-engineering-guide)
[![CI](https://github.com/mghabin/infra-engineering-guide/actions/workflows/ci.yml/badge.svg)](https://github.com/mghabin/infra-engineering-guide/actions/workflows/ci.yml)

An opinionated reference for engineering modern infrastructure:
Infrastructure-as-Code, CI/CD, container platforms, observability,
security & supply chain, networking, data, reliability, FinOps,
platform engineering, and workforce identity. Compiled from primary
sources (CNCF, Google SRE Book, the cloud Well-Architected frameworks,
NIST, FinOps Foundation, ThoughtWorks Tech Radar) and named industry
voices. Every page ends with a Sources section so you can follow the
citations.

Concepts apply across hyperscalers; the *defaults* assume managed
cloud-native primitives, OpenTelemetry, and an SLO-driven operating
model. See [`SCOPE.md`](./SCOPE.md) before adopting recommendations
into a non-cloud-native estate.

## Who this guide is for

Mid-sized cloud SaaS / product organisations with multiple
stream-aligned teams, Git-based delivery, a paid on-call rotation, and
an existing or emerging platform / SRE function. If that's you, the
defaults will work; if it isn't, individual chapters are still useful
as background reading but the *defaults* may be wrong for you.

- [`SCOPE.md`](./SCOPE.md) — who this guide is for, who it isn't, what
  organisational and technical envelope the defaults assume, and the
  glossary of disclaimers used in chapters.
- [`coverage-map.md`](./coverage-map.md) — what each chapter owns,
  what it defers to another chapter, and what is intentionally
  excluded from the whole guide.

## Read these first (the first 90 days)

If you only have an hour with the guide, start here. These two pages
are the synthesis; the numbered chapters are doctrine you go to *when
the decision tree points you there*.

- [`docs/decision-trees.md`](./docs/decision-trees.md) — the 12
  highest-leverage decisions and one-screen decision trees for the
  questions teams actually argue about (which IaC tool, adopt
  Kubernetes, service mesh y/n, multi-region, multi-cloud, ship a
  change safely, SLO model, cost guardrails, data-store family,
  tenancy, human IdP). Each node cites the chapter where the rationale
  lives.
- [`checklist.md`](./checklist.md) — one-page do/don't card for design
  review. Every line is a `must`-severity rule. Print it, paste it
  into your PR template, or use it as the cliff-notes for the rest of
  this guide.

## Doctrine — the 12 chapters

Read in depth when the decision tree or checklist points you here. If
you're new to the field or to a cloud-native stack, the numbered order
also works as a linear read.

| # | Page | What's in it |
|---|------|--------------|
| 01 | [`docs/01-foundations.md`](./docs/01-foundations.md) | What "infrastructure as code" actually means · declarative vs imperative · immutability · drift · GitOps philosophy · how to choose tooling. |
| 02 | [`docs/02-iac-tooling.md`](./docs/02-iac-tooling.md) | Terraform / OpenTofu / Pulumi / Bicep / Crossplane · module design · state management · environment separation · drift detection · IaC testing. |
| 03 | [`docs/03-ci-cd.md`](./docs/03-ci-cd.md) | Pipeline-as-code · supply chain (SLSA, Sigstore, SBOMs) · pinned actions · OIDC over secrets · progressive delivery · blue/green & canary. |
| 04 | [`docs/04-containers-k8s.md`](./docs/04-containers-k8s.md) | Docker hygiene · Kubernetes production-readiness · managed K8s vs serverless containers · service mesh (when *not* to) · ingress · autoscaling. |
| 05 | [`docs/05-observability.md`](./docs/05-observability.md) | OpenTelemetry everywhere · the three-pillars trap · SLO-driven alerting · vendor neutrality · structured logging · exemplars. |
| 06 | [`docs/06-security-supply-chain.md`](./docs/06-security-supply-chain.md) | Secret management · policy-as-code · SBOMs & scanning · SLSA levels · signing · workload identity · zero-trust networking. |
| 07 | [`docs/07-networking.md`](./docs/07-networking.md) | VPC/VNet design · peering · private endpoints · DNS · mTLS · egress control · hub-spoke vs landing-zone. |
| 08 | [`docs/08-data-state.md`](./docs/08-data-state.md) | Managed databases · backup & restore drills · PITR · replicas · blast-radius isolation · cache patterns · message brokers. |
| 09 | [`docs/09-reliability.md`](./docs/09-reliability.md) | Google SRE canon: SLI / SLO / error budgets · incident response · chaos engineering · runbooks · on-call hygiene · postmortems. |
| 10 | [`docs/10-finops.md`](./docs/10-finops.md) | Cost visibility · budgets & alerts · rightsizing · scale-to-zero · spot/preemptible · tagging discipline · reservations · the "cost of governance" trap. |
| 11 | [`docs/11-platform-engineering.md`](./docs/11-platform-engineering.md) | Internal developer platforms · Backstage · golden paths · Team Topologies framing · "platform as a product". |
| 12 | [`docs/12-identity.md`](./docs/12-identity.md) | Workforce / human identity · IdP choice · AAL & WebAuthn · SCIM JML · PAM/PIM · break-glass · IGA & access reviews. |

## Patterns & anti-patterns

Separate from the chapter docs because these are about **how
infrastructure ages well**, not about a single subsystem:

- [`patterns/anti-patterns.md`](./patterns/anti-patterns.md) — the
  things we reject in design review, each with the failure mode and
  the right replacement.
- [`patterns/patterns.md`](./patterns/patterns.md) — the positive
  patterns to reach for instead.
- [`patterns/references.md`](./patterns/references.md) — curated
  source list shared across the chapters above (primary specs, books,
  team blogs, industry voices, conference talks).

## Glossary

- [`glossary.md`](./glossary.md) — canonical, opinionated definitions
  for every term used normatively in this guide. Where the industry
  uses a term inconsistently, the glossary picks a side and cites a
  primary source.

## A worked example

Many of these practices are applied end-to-end in a small reference
service:

👉 **[mghabin/entra-auth-patterns-dotnet](https://github.com/mghabin/entra-auth-patterns-dotnet)**

Among other things, that sample demonstrates:

- GitHub Actions OIDC to Azure (no static secrets), federated identity
  credentials managed declaratively in IaC
- Every third-party Action pinned to a commit SHA with `pinact` as a
  CI guard
- OpenSSF Scorecard + SLSA build attestations
- Infrastructure as Bicep modules with composition (no monolith)
- Container Apps with hard cost caps (cpu, memory, max replicas) +
  scale-to-zero by default
- Subscription budget with multi-stage email alerts as a safety net
- GitHub Repository Rulesets for branch protection (no admin PAT in CI)

## Companion guide

- **[mghabin/dotnet-engineering-guide](https://github.com/mghabin/dotnet-engineering-guide)**
  — language-and-runtime side of the same opinionated approach
  (project layout, ASP.NET Core, EF Core, testing, performance,
  cloud-native .NET, client).

## Contributing

Issues and PRs that improve clarity, fix factual errors, or add
citations are very welcome. PRs that broaden scope toward "everything
about DevOps" are usually not — see [`CONTRIBUTING.md`](./CONTRIBUTING.md)
and [`SCOPE.md`](./SCOPE.md).

## License

[MIT](./LICENSE) © 2026 mghabin
