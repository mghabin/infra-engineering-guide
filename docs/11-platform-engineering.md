# Infrastructure Engineering — Chapter 11: Platform Engineering

This chapter is about the *team and product shape* of infrastructure work, not
the tools. Tooling (Backstage, Crossplane, Argo, Score, Humanitec, SaaS IDPs)
appears only as evidence that a pattern has crystallised. The plumbing
chapters (02 IaC, 04 Kubernetes, 07 GitOps) describe the wires; this chapter
describes who owns them, who consumes them, and how to know whether the
investment is paying off.

The thesis is uncomfortable: **a platform team that does not treat its
platform as a product, with internal customers and a roadmap, is just a
renamed ops team — and the renaming is the most expensive thing it will ever
do.**

> Conventions, mirroring sibling chapters:
> **must** = load-bearing, ≥2 independent sources required.
> **should** = strong default; deviate only with a written reason.
> **prefer** = taste call backed by evidence.
> **avoid** = explicitly rejected; replacement named.

---

## 1. WHAT IT IS

- **[WHAT IT IS] [must] Platform engineering is the discipline of building internal, opinionated, self-service products that reduce the cognitive load on stream-aligned (product) teams.**
- It is not "DevOps with a new logo" and it is not "SRE for everyone". DevOps is a culture (you-build-it-you-run-it, ch01 §6); SRE is a role with explicit reliability budgets; platform engineering is the *team shape* that productises the shared paths those cultures depend on.
- Sources: Skelton & Pais, *Team Topologies* (IT Revolution, 2019), Ch. 4 & 5; CNCF Platforms WG, *Platforms White Paper*, https://tag-app-delivery.cncf.io/whitepapers/platforms/ ; Camille Fournier (ed.), *Platform Engineering* (O'Reilly, 2024), Ch. 1.
- Concrete check: ask three random stream-aligned engineers "what does the platform team produce?". If answers are role-shaped ("they run Kubernetes") rather than product-shaped ("the paved-road service template, the deploy CLI"), the team is not yet doing platform engineering.

- **[WHAT IT IS] [must] A platform is the *thinnest viable* layer over the underlying primitives, not a re-invention of them.**
- Platforms aggregate; they do not rewrite the cloud. A platform that hides Kubernetes (or its non-K8s equivalent) so completely that consumers cannot read its manifests will fail the first time the abstraction leaks.
- Sources: CNCF Platforms WG whitepaper, §"What is a Platform"; Pais & Skelton, key concepts, https://teamtopologies.com/key-concepts/ ; ThoughtWorks Tech Radar, *Platform engineering product teams*, https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams .
- Concrete check: a stream-aligned engineer can trace any platform abstraction one layer down to the primitive it wraps (Helm chart, Terraform module, Kubernetes object, SaaS API call). If not, the abstraction is opaque.

---

## 2. AS A PRODUCT

- **[AS A PRODUCT] [must] The platform has named internal customers, a roadmap, and a feedback loop. Treat it exactly like an external product.**
- "Platform as a product" is the load-bearing idea. It implies a product owner, a backlog driven by user research not tickets, documented SLOs *to* consumers, a deprecation policy, and a roadmap visible to the rest of engineering.
- Sources: Skelton & Pais, *Team Topologies*, Ch. 5 ("Platform teams: treat the platform as a product"); Fournier, *Platform Engineering*, Part I; ThoughtWorks Tech Radar, *Platform engineering product teams*.
- Concrete check: the platform team has (a) a written product vision, (b) a public roadmap (a README is enough), (c) a recurring channel for consumer feedback, (d) named owners per capability. Absence of any is a finding.

- **[AS A PRODUCT] [must] Validate demand before building. No platform feature ships without an identified consumer team that has agreed to adopt it.**
- The most common failure mode is "we know what they need" — a beautiful abstraction nobody adopts. This is the standard product-management failure (no problem-validation, no MVP) re-skinned.
- Sources: Fournier, *Platform Engineering*, Ch. 2 ("Discovery and Demand"); Will Larson, *An Elegant Puzzle* (Stripe Press, 2019), §"Migrations" and §"Internal tools"; CNCF Platforms WG whitepaper, §"Capabilities".
- Concrete check: every roadmap item links to (a) the consumer team that asked for it, (b) the user-research artefact, (c) the success metric. Items missing any of these are speculative — defer.

---

## 3. OPERATING MODEL

- **[OPERATING MODEL] [must] Name the product manager (or PM-equivalent), the funding model, and the tenant-onboarding flow before you ship the first capability.**
- Platform-as-a-product is empty without an explicit operating model. Decide who owns the roadmap (a dedicated PM, a tech lead playing PM, or a rotating product owner from senior engineers); how the team is funded (central engineering budget, internal showback, or chargeback to consumer cost centres); and what a new tenant must do on day one (request namespace/account, register service in catalog, accept SLOs, wire CI).
- Sources: Fournier, *Platform Engineering*, Ch. 2–3; Skelton & Pais, *Team Topologies*, Ch. 5; CNCF Platforms WG whitepaper, §"Platform team enablement"; Humanitec, *Reference Architectures for IDPs*, https://humanitec.com/blog/reference-architectures-for-internal-developer-platforms .
- Concrete check: a one-page operating-model doc names PM, funding source, onboarding checklist, and the prioritisation forum where consumer requests are triaged. Absence is a finding.

- **[OPERATING MODEL] [should] Prioritise demand with a transparent intake — a public backlog plus a recurring consumer-council — not Slack DMs to the loudest team.**
- Without a structured intake, the platform serves whoever shouts. With one, the team can defend "no" decisions and surface duplication ("three teams want the same Postgres recipe — fund it once").
- Sources: Fournier, *Platform Engineering*, Ch. 2; Larson, *An Elegant Puzzle*, §"Internal tools".
- Concrete check: every shipped feature traces to a backlog item created by a consumer or council; ad-hoc DM-driven work is <20% of throughput.

---

## 4. PAVED ROADS

> Terminology: this guide uses **paved road** as the canonical term across
> all chapters. *Golden path* is treated as a synonym (Spotify's original
> phrasing) — see the glossary entry "Paved Road / Golden Path". New
> normative bullets in this and other chapters use "paved road".

- **[PAVED ROAD] [must] Provide paved roads with documented off-ramps. Mandatory *controls* are acceptable; mandatory *human mediation* is the anti-pattern.**
- Spotify's "golden path" is the supported, batteries-included way to do a thing — provision a service, ship a deploy pipeline, wire observability. In a regulated environment a control may be non-negotiable (every prod deploy must pass image signing, every database must be encrypted) and the paved road may be the only sanctioned route. That is fine *if* the path itself is self-service. The anti-pattern is when the only way to satisfy the control is "file a ticket and wait for a human": then it is not a road, it is a queue, and you are back to the bottleneck in ch01 §6.
- Sources: Spotify Engineering, *How We Use Golden Paths to Solve Fragmentation* (2020), https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/ ; Skelton & Pais, *Team Topologies*, Ch. 5 (X-as-a-Service); CNCF Platforms WG whitepaper, §"Capabilities".
- Concrete check: pick the flagship paved road. (a) Are the *controls* it enforces written down and justified? (b) Can a team satisfy those controls another way without ticket-and-wait? (c) Is there a documented exception/off-ramp procedure with an SLA? Three "yes" = healthy.

- **[PAVED ROAD] [should] The paved road must also be the lowest-friction *and* lowest-risk option. Make the right thing the easy thing.**
- If the paved road is more painful than rolling your own, engineers route around it (or game the controls) and the platform team ends up policing rather than enabling. The forcing function is ergonomics, not policy.
- Sources: Spotify "Golden Paths" post (above); Fournier, *Platform Engineering*, Ch. 4 ("Ergonomics and Adoption").
- Concrete check: time-to-first-deploy via the paved road vs off-road, including the time to satisfy required controls. If off-road is faster, the path is broken.

---

## 5. IDP

- **[IDP] [must] An *Internal Developer Platform* (IDP) is the integrated product surface; a developer *portal* is one possible UI on top of it.**
- "We have a portal, therefore we have a platform" is wrong: a portal that fronts ten disconnected backend processes ("file a Jira", "ask in Slack") is just a prettier router. The platform is the set of *self-service capabilities*; the portal/CLI/API/SDK is the front door.
- Sources: CNCF Platforms WG whitepaper, §"Attributes of a platform"; Fournier, *Platform Engineering*, Ch. 1 & 4; Salatino, *Platform Engineering on Kubernetes* (Manning, 2023), Ch. 1–2.
- Concrete check: list the top five user journeys (provision env, deploy app, rotate secret, request resource, view logs). For each, can the developer self-serve end-to-end without human intervention? Each "no" is a gap.

---

## 6. SURFACE STRATEGY & VERSIONING

- **[SURFACE] [must] Choose surfaces by use-case: UI for discovery and one-off actions, CLI/API for automation, SDK only when scripting against the API is genuinely insufficient.**
- The cheapest mistake is to ship all four surfaces by default; each is a long-term maintenance commitment. Most platforms can ship a UI plus a thin CLI that wraps the API and defer SDKs until a concrete consumer (e.g., a code-generation pipeline) demands one. Every UI action should have a documented API equivalent so CI, IDEs and other automation can drive the platform.
- Sources: CNCF Platforms WG whitepaper, §"Provisioning interfaces"; Fournier, *Platform Engineering*, Ch. 4; Humanitec reference architectures (above).
- Concrete check: for each surface, name the primary use-case and the maintainer; surfaces with no named maintainer are sunset candidates.

- **[SURFACE] [must] Version the platform contract and publish a deprecation policy. Apply integration/contract tests to templates, modules and compositions before rollout.**
- A platform is an API to its consumers; breaking changes break their pipelines. Adopt explicit versioning on the consumer-facing schema (e.g., `v1`, `v1beta1`), publish a deprecation window (≥1 quarter is a common floor for internal APIs), and run contract tests that exercise each template/module against a representative consumer scenario before promotion. Test pyramid for the platform: unit tests on modules → integration tests against an ephemeral environment → smoke tests against a canary tenant → general rollout.
- Sources: CNCF Platforms WG whitepaper, §"Capabilities" and §"Operating the platform"; Fournier, *Platform Engineering*, Ch. 4–5; Salatino, *Platform Engineering on Kubernetes*, Ch. 6 ("Testing platform components"); Backstage software-templates docs, https://backstage.io/docs/features/software-templates/ .
- Concrete check: every consumer-facing schema has a version field; the deprecation policy is written; CI runs a contract test on each template PR. Missing any of the three is a finding.

---

## 7. PRINCIPLES vs IMPLEMENTATIONS

- **[PRINCIPLES] [must] Self-service infrastructure means the consumer team declares *what they want* in their own repo (or via an API call) and the platform reconciles it — not "raise a ticket".**
- The principle is platform-shape and cloud-agnostic: a consumer-facing spec is the API; the platform turns intent into provisioned resources; toil is paid once at the platform and amortised across every consumer. The implementation pattern is a choice, not a mandate.
- Sources: CNCF Platforms WG whitepaper, §"Self-service"; Fournier, *Platform Engineering*, Ch. 1 & 4; Skelton & Pais, *Team Topologies*, Ch. 5.
- Concrete check: pick any non-trivial resource (a bucket, a Postgres, a queue). Can a stream-aligned team get one in <30 minutes by editing a file in their own repo or calling a documented API? If they have to talk to a human, this is not yet self-service.

- **[IMPLEMENTATIONS] [should] Pick from a family of equivalent patterns; none is universally right.**

| Pattern family | When it fits | Representative tools |
|---|---|---|
| Kubernetes + GitOps + CRD/Composition | You already run K8s at scale; consumers are happy with YAML; ops team can run controllers | Crossplane, Argo CD/Flux, Score-on-K8s, KubeVela |
| Workload spec + multi-target compiler | Consumers should be cloud-agnostic; you target K8s *and* serverless/VMs | Score, Open Application Model, Humanitec |
| SaaS-centric IDP | Small platform team, mostly off-the-shelf SaaS infra (Vercel, Render, Heroku-style); scorecards matter more than custom controllers | Port, Cortex, OpsLevel, Atlassian Compass |
| API-gateway / service-catalog | Compliance-heavy or non-K8s estate; provisioning happens via a curated API or catalog | AWS Service Catalog, ServiceNow workflows, custom API |
| Non-Kubernetes orchestration | HPC, ML training, batch, embedded; K8s is the wrong primitive | Nomad, Slurm, vendor schedulers |

- Each row is a defensible choice; the wrong move is to copy a reference architecture without checking which row your organisation is actually in.
- Sources: CNCF Platforms WG whitepaper, §"Capabilities and tools"; ThoughtWorks Tech Radar entries on *Backstage*, *Crossplane*, *Score*, *Port*; Fournier, *Platform Engineering*, Ch. 5 ("Build vs Buy vs Adopt"); Salatino, *Platform Engineering on Kubernetes*, Ch. 3–4.
- Concrete check: the platform's architecture decision record names which row it is in, the rejected alternatives, and the criteria that decided it.

---

## 8. ABSTRACTION LEVEL

- **[ABSTRACTION LEVEL] [must] The right level is *configuration as data* over a small declarative spec — not raw IaC exposed to every team, and not a fully-opinionated PaaS that hides primitives.**
- Too low (every team writes Terraform from scratch): no leverage, infinite drift, every team re-discovers the same security mistakes. Too high (a black-box PaaS): teams cannot debug, cannot extend, cannot escape; the platform becomes a single point of organisational failure. The sweet spot is a small declarative schema (a CRD, a Score file, a Crossplane Claim, a catalog entry, an API request body) that the platform compiles into the underlying primitives.
- Sources: Kelsey Hightower, *Configuration as Data*, https://github.com/kelseyhightower/config-as-data ; CNCF Platforms WG whitepaper, §"Capabilities"; Salatino, *Platform Engineering on Kubernetes*, Ch. 3–4.
- Concrete check: examine the consumer-facing spec for the platform's main abstraction. Is it < ~30 lines of declarative YAML/JSON for a typical service? Is the compilation to primitives readable? Both yes = good.

---

## 9. BACKSTAGE

- **[BACKSTAGE] [should] Consider Backstage as the IDP framework once you have sustained pain from missing software catalog and discoverable docs — typically when a single engineer can no longer enumerate every service from memory.**
- Backstage (Spotify origin, accepted to CNCF Sandbox September 2020, promoted to **Incubating** in March 2022 — verify current status at https://www.cncf.io/projects/backstage/ before publication) gives three primitives out of the box: Software Catalog (truth for "what services exist and who owns them"), Software Templates (a concrete encoding of a paved road), and TechDocs (Markdown + MkDocs colocated with code, rendered in the catalog). At the right scale these three deliver more value than any homegrown alternative.
- Sources: Backstage docs, https://backstage.io/docs/overview/what-is-backstage ; CNCF project page, https://www.cncf.io/projects/backstage/ ; ThoughtWorks Tech Radar, *Backstage*, https://www.thoughtworks.com/radar/platforms/backstage .
- Concrete check: every production service appears in the catalog with an owner (a team, not a person), a TechDocs entry, and at least one Software Template that could re-create a similar service.

- **[BACKSTAGE] [avoid] Adopting Backstage when the run-cost dominates the benefit, or treating it as the platform itself rather than a frontend to a platform.**
- Backstage is a TypeScript application your team will own (auth, plugin upgrades, catalog ingestion, database, deployment). If you cannot fund that ownership, a managed alternative (Roadie, Port, Cortex, OpsLevel, Compass) or a single README per service plus a shared module repo is the right answer.
- Sources: ThoughtWorks Tech Radar, *Backstage* (cautions on adoption cost); Fournier, *Platform Engineering*, Ch. 5; Roadie, *Should you adopt Backstage?*, https://roadie.io/blog/ .

### When *not* to build or adopt — heuristic decision table

| Signal | Lean toward… |
|---|---|
| <3 stream-aligned teams suffering the same toil | Defer; share a README / Helm chart / CI template instead |
| No platform-team capacity for steady-state ownership of a TS app | Managed IDP (Roadie/Port/Cortex/OpsLevel/Compass) over self-hosted Backstage |
| Service count low enough to fit in one engineer's head | Defer software catalog; revisit when discovery becomes painful |
| Mostly SaaS-managed infra, little custom provisioning | SaaS-centric IDP over Backstage + Crossplane stack |
| Heavy regulated/compliance estate, non-K8s | API gateway / service catalog pattern over CRD-based platform |
| Sustained funding for ≥1 dedicated platform PM-plus-engineers | Build/adopt is on the table |

- These are heuristics, not thresholds. Earlier guidance ("~50 engineers, ~30 services") is a *signal* that complexity is approaching the inflection point — not a gate. Some 20-engineer fintechs need a platform on day one because of compliance; some 200-engineer agencies never do because their workloads are uniform.
- Sources: Larson, *An Elegant Puzzle*, §"Sizing engineering teams" and §"Internal tools"; Fournier, *Platform Engineering*, Ch. 1 ("When you don't need a platform team") and Ch. 5; CNCF Platforms WG whitepaper, §"Maturity model".

---

## 10. TEAM TOPOLOGIES

- **[TEAM TOPOLOGIES] [must] Organise infra work around the four team types and three interaction modes from *Team Topologies*; do not invent a fifth.**
- The four types are *stream-aligned* (owns a value stream end-to-end), *platform* (provides internal services to stream-aligned teams), *enabling* (helps adopt new capabilities, then steps back), and *complicated-subsystem* (deep specialism — ML inference, video pipeline). The three interaction modes are *collaboration* (short-term, high-bandwidth), *X-as-a-Service* (consumer treats provider as black box), and *facilitating* (enabling team helps another). Platform work is overwhelmingly X-as-a-Service; enabling work is short-lived and skill-transfer shaped.
- Sources: Skelton & Pais, *Team Topologies*, Ch. 4 & 7; key concepts, https://teamtopologies.com/key-concepts/ ; Fournier, *Platform Engineering*, Ch. 3.
- Concrete check: every infra-adjacent team is labelled with one of the four types and its dominant interaction mode is documented per consumer. Teams that interact with everyone, in collaboration mode, all the time, are misclassified bottlenecks.

---

## 11. CONWAY

- **[CONWAY] [must] The architecture you ship will mirror the communication structure of the org that built it. Plan accordingly (the Inverse Conway Maneuver).**
- Conway's 1968 paper *How Do Committees Invent?* argued, in his words, that "organizations which design systems … are constrained to produce designs which are copies of the communication structures of these organizations." The familiar punchier wording — "any organization that designs a system will inevitably produce a design whose structure is a copy of the organization's communication structure" — is Conway's own *later informal restatement* of that thesis (see his site at melconway.com), not a verbatim quote from the 1968 paper. Either form supports the same point. The inverse maneuver is to deliberately restructure teams so the desired architecture is the *easy* one to build.
- Sources: Melvin Conway, *How Do Committees Invent?*, Datamation, April 1968 — paper and author commentary at http://www.melconway.com/Home/Committees_Paper.html ; Skelton & Pais, *Team Topologies*, Ch. 2; ThoughtWorks Tech Radar, *Inverse Conway Maneuver*, https://www.thoughtworks.com/radar/techniques/inverse-conway-maneuver .
- Concrete check: the platform-team boundary matches the API boundary of the platform. If the platform exposes one product but is split across three teams that don't share a backlog, expect the API to fragment to match.

---

## 12. COGNITIVE LOAD

- **[COGNITIVE LOAD] [must] The platform exists, fundamentally, to keep the *intrinsic* cognitive load on stream-aligned teams within human limits.**
- Cognitive load (Sweller, educational psychology, 1988) decomposes into intrinsic (inherent task complexity), extraneous (poor tools, accidental complexity), and germane (cost of building useful mental models). A stream-aligned team that must also operate Kubernetes, Terraform, secrets, CI runners, and observability *plus* its own product domain is overloaded — the platform's job is to make those shared concerns X-as-a-Service so the team's load fits.
- Sources: John Sweller, *Cognitive Load During Problem Solving*, Cognitive Science 12(2), 1988, https://onlinelibrary.wiley.com/doi/10.1207/s15516709cog1202_4 ; Skelton & Pais, *Team Topologies*, Ch. 3 ("Team-First Thinking, Cognitive Load"); Fournier, *Platform Engineering*, Ch. 1.
- Concrete check: a stream-aligned team can list, in one breath, the *concepts* it owns (its domain, its service, the platform spec it writes). If the list also includes "Helm internals", "Argo CD reconciler loops", "VPC CIDRs", "IAM trust policies", the platform is leaking.

---

## 13. ANTI-PATTERNS

- **[ANTI-PATTERNS] [avoid] Ivory-tower platform — design without consumer contact.**
- Replacement: weekly office hours, an "embedded engineer" rotation, or shadowing one stream-aligned team per quarter.
- Sources: Fournier, *Platform Engineering*, Ch. 2; Skelton & Pais, *Team Topologies*, Ch. 5.
- Concrete check: count the platform-team commits in a quarter whose linked issue cites a stream-aligned consumer. <50% is a flag.

- **[ANTI-PATTERNS] [avoid] Mandatory human mediation on the paved road. The moment a human must approve every traversal, the road becomes a queue.**
- Replacement: enforce *controls* in the path itself (signing, policy-as-code, automated review) and reserve human review for genuine exceptions with a documented SLA. Compete on quality, not policy.
- Sources: Spotify "Golden Paths" post; Skelton & Pais, *Team Topologies*, Ch. 5; CNCF Platforms WG whitepaper, §"Capabilities".
- Concrete check: at least one production team can satisfy required controls and ship via the paved road with zero synchronous human approvals.

- **[ANTI-PATTERNS] [avoid] "We know what they need" — building before validating.**
- Replacement: an MVP scoped to one consumer team that has signed up to adopt within a fixed window. Kill the project if adoption does not happen.
- Sources: Larson, *An Elegant Puzzle*, §"Migrations" and §"Internal tools"; Fournier, *Platform Engineering*, Ch. 2.
- Concrete check: every shipped feature has a named first adopter who used it within 30 days of release.

- **[ANTI-PATTERNS] [avoid] Treating the platform team as an SRE-ticket queue.**
- Replacement: SRE work has its own home; platform work is *productised* shared services. The two are different shapes; do not collapse them.
- Sources: Google SRE book, Ch. 1, https://sre.google/sre-book/introduction/ ; Skelton & Pais, *Team Topologies*, Ch. 4.
- Concrete check: the platform team's tracker has <~20% break-fix tickets; the rest is roadmap product work.

---

## 14. METRICS

- **[METRICS] [must] Measure platform success on two axes: (a) consumer outcomes via DORA's current five software-delivery metrics, and (b) direct platform KPIs that the platform team controls.**
- DORA's *current* model (verify at https://dora.dev/ before publication) is **five** metrics, evolved from the original four-key model: change lead time, deployment frequency, **failed deployment recovery time** (replacing MTTR), change fail rate, and **deployment rework rate**. These measure the throughput and instability of the delivery system the platform is part of. Pair them with **SPACE** (Forsgren, Storey et al., 2021) for developer-experience signals — satisfaction, performance, activity, communication, efficiency — that DORA does not capture.
- Direct platform KPIs the team owns: **platform SLOs** (e.g., 99.9% successful provisioning of a Postgres in <10 min), **self-service success rate** (% of journeys completed without a human ticket), **journey latency** (p50/p95 time from intent to working resource), **onboarding time** (new tenant from request to first deploy), **adoption** per shipped capability, and a recurring **developer survey** (NPS-style or SPACE-aligned, ≥quarterly).
- Sources: DORA research program and current metrics, https://dora.dev/ and https://dora.dev/guides/dora-metrics-four-keys/ ; Forsgren, Humble & Kim, *Accelerate* (IT Revolution, 2018), Ch. 2; Forsgren, Storey, Maddila, Zimmermann, Houck & Butler, *The SPACE of Developer Productivity*, ACM Queue 2021, https://queue.acm.org/detail.cfm?id=3454124 ; Fournier, *Platform Engineering*, Ch. 6; CNCF Platforms WG whitepaper, §"Measuring success".
- Concrete check: the platform team publishes, at least quarterly, (a) the five DORA metrics for top consumer teams, (b) platform SLO attainment, (c) self-service success rate and journey latency for the top five journeys, (d) adoption per capability, (e) a developer-experience survey result. Absence of any line is a finding.

---

## 15. TOOLING

- **[TOOLING] [should] Build the platform from established CNCF/OSS or managed building blocks; integrate, don't reinvent. Pick the family that matches §7.**
- Reference stacks, late 2025, by family (sources: CNCF landscape, https://landscape.cncf.io/ ; CNCF Platforms WG whitepaper, *Capabilities and tools*; ThoughtWorks Tech Radar entries on each project; Salatino, *Platform Engineering on Kubernetes*, Ch. 6–10):
  - K8s + GitOps + CRD: *Backstage* (catalog/portal), *Crossplane* (provisioning), *Argo CD* / *Flux* (deploy), *Score* / *Humanitec* (workload spec), *OpenTelemetry*, *cert-manager*, *external-secrets*.
  - SaaS-centric IDP: *Port* / *Cortex* / *OpsLevel* / *Compass* + cloud-native managed services + Terraform Cloud / Spacelift.
  - API-gateway / catalog: *AWS Service Catalog* or equivalent, fronted by a thin internal API and a portal of choice.
  - Non-K8s: *Nomad* / *Slurm* / vendor schedulers, with the same catalog/portal/observability seams.
- Concrete check: the architecture diagram names the OSS or SaaS project behind each capability and shows where each customisation was added. Diagrams that show only product names hide lock-in.

---

## 16. READINESS

- **[READINESS] [must] Charter a platform team only when the criteria are met, not at a fixed headcount.**
- Criteria (all four, not any one):
  - Repeated toil: at least three stream-aligned teams suffer the same shared-infra pain.
  - Demonstrated demand: those teams have signed up to adopt the first capability.
  - Sustained funding: ≥2 engineers plus PM-equivalent capacity for at least a year, not borrowed time.
  - Complexity worth amortising: the abstraction saves more team-hours per quarter than it costs to maintain.
- Engineer-count or service-count thresholds (the often-cited "~30 engineers / ~10 services") are *signals* that these criteria are likely to be met — not gates. Some compliance-heavy 20-engineer orgs need a platform on day one; some 200-engineer orgs with uniform workloads never do.
- Sources: Larson, *An Elegant Puzzle*, §"Sizing engineering teams"; Fournier, *Platform Engineering*, Ch. 1 ("When you don't need a platform team"); Skelton & Pais, *Team Topologies*, Ch. 5.
- Concrete check: before chartering, write the "what duplication is this avoiding, who has signed up, how is it funded?" memo. If any of the four criteria is missing, defer.

---

## 17. DOCS

- **[DOCS] [must] Documentation is a first-class platform feature: colocated with the code, searchable, versioned. "If it isn't searchable, it doesn't exist."**
- Backstage TechDocs is the canonical implementation in the K8s family: Markdown lives next to code in each repo, MkDocs renders it, the portal aggregates. SaaS IDPs typically provide an equivalent (Port docs widgets, Cortex catalog docs). ADRs (Nygard, 2011) capture *why* a decision was made and are themselves discoverable docs.
- Sources: Backstage TechDocs, https://backstage.io/docs/features/techdocs/ ; Michael Nygard, *Documenting Architecture Decisions*, https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions ; Write the Docs, *Docs as Code*, https://www.writethedocs.org/guide/docs-as-code/ ; Fournier, *Platform Engineering*, Ch. 4.
- Concrete check: every platform capability has a doc reachable in ≤2 clicks from the catalog/portal; every load-bearing decision has an ADR in the platform repo. Search "deploy a service" in the portal — if the answer is not in the first three results, docs are failing.

---

## Sources

### Books

- Matthew Skelton & Manuel Pais, *Team Topologies*, IT Revolution, 2019 — https://teamtopologies.com/book
- Camille Fournier (ed.), *Platform Engineering*, O'Reilly, 2024
- Mauricio Salatino, *Platform Engineering on Kubernetes*, Manning, 2023
- Nicole Forsgren, Jez Humble & Gene Kim, *Accelerate*, IT Revolution, 2018
- Will Larson, *An Elegant Puzzle*, Stripe Press, 2019

### Foundational papers

- Melvin E. Conway, *How Do Committees Invent?*, Datamation, April 1968 — http://www.melconway.com/Home/Committees_Paper.html
- John Sweller, *Cognitive Load During Problem Solving*, Cognitive Science 12(2), 1988 — https://onlinelibrary.wiley.com/doi/10.1207/s15516709cog1202_4
- Forsgren, Storey, Maddila, Zimmermann, Houck & Butler, *The SPACE of Developer Productivity*, ACM Queue 19(1), 2021 — https://queue.acm.org/detail.cfm?id=3454124
- Michael Nygard, *Documenting Architecture Decisions*, 2011 — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions

### Standards & working groups

- CNCF Platforms Working Group, *Platforms White Paper* — https://tag-app-delivery.cncf.io/whitepapers/platforms/
- CNCF TAG App Delivery — https://github.com/cncf/tag-app-delivery
- CNCF project page, *Backstage* (incubating; verify status at publication) — https://www.cncf.io/projects/backstage/
- DORA research program (current five-metric model) — https://dora.dev/ and https://dora.dev/guides/dora-metrics-four-keys/
- Score workload specification — https://score.dev/docs/

### Vendor / project documentation (canonical statement, not endorsement)

- Backstage — https://backstage.io/docs/overview/what-is-backstage
- Backstage TechDocs — https://backstage.io/docs/features/techdocs/
- Backstage Software Templates — https://backstage.io/docs/features/software-templates/
- Crossplane Compositions — https://docs.crossplane.io/latest/concepts/compositions/
- Humanitec reference architectures — https://humanitec.com/blog/reference-architectures-for-internal-developer-platforms
- Roadie (managed Backstage) — https://roadie.io/
- Argo CD — https://argo-cd.readthedocs.io/
- Flux — https://fluxcd.io/

### Named industry voices

- Pais & Skelton, Team Topologies key concepts — https://teamtopologies.com/key-concepts/
- Kelsey Hightower, *Configuration as Data* — https://github.com/kelseyhightower/config-as-data
- Spotify Engineering, *How We Use Golden Paths* — https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/

### Industry analysis

- ThoughtWorks Tech Radar, *Platform engineering product teams* — https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams
- ThoughtWorks Tech Radar, *Backstage* — https://www.thoughtworks.com/radar/platforms/backstage
- ThoughtWorks Tech Radar, *Inverse Conway Maneuver* — https://www.thoughtworks.com/radar/techniques/inverse-conway-maneuver
- ThoughtWorks Tech Radar, *Score* — https://www.thoughtworks.com/radar/languages-and-frameworks/score
- internaldeveloperplatform.org community — https://internaldeveloperplatform.org/

---
