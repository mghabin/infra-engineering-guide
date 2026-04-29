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
do.** DORA's 2024 Accelerate State of DevOps Report tightens this further:
even a *correctly framed* platform investment can measurably reduce change
stability and throughput unless the implementation actively preserves
developer independence (no mandatory human mediation, no opaque abstractions).
Platform engineering is net-positive only when it is also net-unblocking.

> Conventions, mirroring sibling chapters:
> **must** = load-bearing, ≥2 independent sources required.
> **should** = strong default; deviate only with a written reason.
> **prefer** = taste call backed by evidence.
> **avoid** = explicitly rejected; replacement named.

---

## 1. WHAT IT IS

- **[WHAT IT IS] [must] Platform engineering is the discipline of building internal, opinionated, self-service products that reduce the cognitive load on stream-aligned (product) teams.**
  - It is not "DevOps with a new logo" and it is not "SRE for everyone". DevOps is a culture (you-build-it-you-run-it, ch01 §6); SRE is a role with explicit reliability budgets; platform engineering is the *team shape* that productises the shared paths those cultures depend on.
  - The CNCF TAG App Delivery *Platforms* whitepaper is the canonical cross-vendor definition: "an integrated collection of capabilities defined and presented according to the needs of the platform's users." It enumerates five benefits — (1) reduced cognitive load, (2) improved reliability, (3) faster delivery via reuse, (4) lower security/regulatory risk via paved governance, (5) cost-effective use of managed services — which map cleanly onto this chapter's thesis.
  - Sources: Skelton & Pais, *Team Topologies*, 2nd ed. (IT Revolution, 2025), Ch. 4 & 5; CNCF TAG App Delivery, *Platforms White Paper*, https://tag-app-delivery.cncf.io/whitepapers/platforms/ ; Camille Fournier (ed.), *Platform Engineering* (O'Reilly, 2024), Ch. 1.
  - Concrete check: ask three random stream-aligned engineers "what does the platform team produce?". If answers are role-shaped ("they run Kubernetes") rather than product-shaped ("the paved-road service template, the deploy CLI"), the team is not yet doing platform engineering.

- **[WHAT IT IS] [must] A platform is the *thinnest viable* layer over the underlying primitives, not a re-invention of them.**
  - Platforms aggregate; they do not rewrite the cloud. A platform that hides Kubernetes (or its non-K8s equivalent) so completely that consumers cannot read its manifests will fail the first time the abstraction leaks.
  - Sources: CNCF Platforms WG whitepaper, §"What is a Platform"; Pais & Skelton, key concepts, https://teamtopologies.com/key-concepts/ ; ThoughtWorks Tech Radar, *Platform engineering product teams*, https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams .
  - Concrete check: a stream-aligned engineer can trace any platform abstraction one layer down to the primitive it wraps (Helm chart, Terraform module, Kubernetes object, SaaS API call). If not, the abstraction is opaque.

- **[WHAT IT IS] [prefer] Treat the operating model as extensible: data, security, and AI platforms now follow the same product shape.**
  - The PlatformCon community (platformengineering.org) has begun naming sub-disciplines — Infrastructure PE, DevEx PE, Data PE, Security PE, AI PE, Platform Leadership, Platform Product Management — to capture the fact that one stream-aligned team may consume *several* internal platforms, each with its own product owner. The taxonomy is not yet canonical (medium-confidence finding) but the pattern is real: the framing in this chapter applies whether the platform exposes Kubernetes, a feature store, an SBOM service, or an LLM gateway.
  - Sources: platformengineering.org/blog (sub-discipline taxonomy), https://platformengineering.org/blog ; CNCF Platforms WG whitepaper, §"Capabilities".
  - Cross-chapter ownership: this chapter (ch11) owns the IDP definition, paved-road canonical naming, the DORA five-metric list, the platform-team operating model, and the platform-as-product framing. Service-mesh implementation belongs to ch04; supply-chain attestation (SLSA) to ch06; mTLS enforcement to ch07; restore-drill split between ch08 (backups) and ch09 (incident response); SLO baselines to ch05; on-call and incident command to ch09. When this chapter cites those topics, treat the citation as a pointer, not a redefinition.

---

## 2. AS A PRODUCT

- **[AS A PRODUCT] [must] The platform has named internal customers, a roadmap, and a feedback loop. Treat it exactly like an external product.**
  - "Platform as a product" is the load-bearing idea. It implies a product owner, a backlog driven by user research not tickets, documented SLOs *to* consumers, a deprecation policy, and a roadmap visible to the rest of engineering.
  - Sources: Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5 ("Platform teams: treat the platform as a product"); Fournier, *Platform Engineering*, Part I; ThoughtWorks Tech Radar, *Platform engineering product teams* (Adopt, Oct 2021, unchanged through 2026): "a sensible default … just another product team, albeit one focused on internal platform customers."
  - Concrete check: the platform team has (a) a written product vision, (b) a public roadmap (a README is enough), (c) a recurring channel for consumer feedback, (d) named owners per capability. Absence of any is a finding.

- **[AS A PRODUCT] [must] Validate demand before building. No platform feature ships without an identified consumer team that has agreed to adopt it.**
  - The most common failure mode is "we know what they need" — a beautiful abstraction nobody adopts. This is the standard product-management failure (no problem-validation, no MVP) re-skinned. DORA 2024 puts numbers on the failure mode: platforms that consumers cannot route around are the ones that depress delivery stability.
  - Sources: Fournier, *Platform Engineering*, Ch. 2 ("Discovery and Demand"); Will Larson, *An Elegant Puzzle* (Stripe Press, 2019), §"Migrations" and §"Internal tools"; DORA, *2024 Accelerate State of DevOps Report*, https://dora.dev/research/2024/dora-report/ ; CNCF Platforms WG whitepaper, §"Capabilities".
  - Concrete check: every roadmap item links to (a) the consumer team that asked for it, (b) the user-research artefact, (c) the success metric. Items missing any of these are speculative — defer.

- **[AS A PRODUCT] [must] Acknowledge the measured trade-off: platform investment can degrade delivery stability if it removes developer independence.**
  - DORA 2024 (the first major industry research to quantify a downside): "utilizing an internal developer platform improves individual productivity, team performance, and overall organizational performance. However, it can also lead to decreased change stability and throughput, requiring careful implementation focused on developer independence." The same report finds an analogous trade-off for AI assistance. Read together, the message is that opacity and mandated abstractions are the enemy — not platform investment per se.
  - Sources: DORA, *2024 Accelerate State of DevOps Report* — key findings, https://dora.dev/research/2024/dora-report/ ; DORA, *Platform engineering* capability page, https://dora.dev/capabilities/platform-engineering/ ; ThoughtWorks Tech Radar, *Platform engineering product teams*.
  - Concrete check: track the five DORA metrics (§14) for the *consumer* teams before and after onboarding to a new platform capability. A measurable drop in stability or throughput that does not recover within two quarters is a signal that the abstraction is opaque or human-mediated; revisit §4 and §13.

---

## 3. OPERATING MODEL

- **[OPERATING MODEL] [must] Name the product manager (or PM-equivalent), the funding model, and the tenant-onboarding flow before you ship the first capability.**
  - Platform-as-a-product is empty without an explicit operating model. Decide who owns the roadmap (a dedicated PM, a tech lead playing PM, or a rotating product owner from senior engineers); how the team is funded (central engineering budget, internal showback, or chargeback to consumer cost centres); and what a new tenant must do on day one (request namespace/account, register service in catalog, accept SLOs, wire CI).
  - Sources: Fournier, *Platform Engineering*, Ch. 2–3; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5; CNCF Platforms WG whitepaper, §"Platform team enablement"; Humanitec, *Reference Architectures for IDPs*, https://humanitec.com/whitepapers/reference-architecture-for-internal-developer-platforms .
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
  - Spotify's "golden path" originated as a Hack Week project in backend engineering around 2014 (six years before the 2020 blog post that publicised it) and was framed from day one as opt-in: in the original internal write-up, "if you are an adventurer you can of course leave the Golden Path and do your own thing, but then you will not have the same support." That sentence is the strongest one-line defence of the model: the path is *supported*, the off-road is *unsupported*, but neither is forbidden. In a regulated environment a *control* may be non-negotiable (every prod deploy must pass image signing, every database must be encrypted) and the paved road may be the only sanctioned route. That is fine *if* the path itself is self-service. The anti-pattern is when the only way to satisfy the control is "file a ticket and wait for a human": then it is not a road, it is a queue, and you are back to the bottleneck in ch01 §6.
  - Sources: Spotify Engineering, *How We Use Golden Paths to Solve Fragmentation* (2020), https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/ ; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5 (X-as-a-Service); CNCF Platforms WG whitepaper, §"Capabilities"; DORA 2024 (developer-independence finding).
  - Concrete check: pick the flagship paved road. (a) Are the *controls* it enforces written down and justified? (b) Can a team satisfy those controls another way without ticket-and-wait? (c) Is there a documented exception/off-ramp procedure with an SLA? Three "yes" = healthy.

- **[PAVED ROAD] [should] The paved road must also be the lowest-friction *and* lowest-risk option. Make the right thing the easy thing.**
  - If the paved road is more painful than rolling your own, engineers route around it (or game the controls) and the platform team ends up policing rather than enabling. The forcing function is ergonomics, not policy. Netflix popularised "paved road" as the operational term for the same idea — a supported default that wins on quality, not on mandate.
  - Sources: Spotify "Golden Paths" post (above); Fournier, *Platform Engineering*, Ch. 4 ("Ergonomics and Adoption").
  - Concrete check: time-to-first-deploy via the paved road vs off-road, including the time to satisfy required controls. If off-road is faster, the path is broken.

---

## 5. IDP

- **[IDP] [must] An *Internal Developer Platform* (IDP) is the integrated product surface; a developer *portal* is one possible UI on top of it.**
  - "We have a portal, therefore we have a platform" is wrong: a portal that fronts ten disconnected backend processes ("file a Jira", "ask in Slack") is just a prettier router. The platform is the set of *self-service capabilities*; the portal/CLI/API/SDK is the front door.
  - The Humanitec-stewarded internaldeveloperplatform.org community decomposes an IDP into five core components: **Infrastructure Orchestration**, **Application Configuration Management**, **Deployment Management**, **Environment Management**, and **Role-Based Access Control**. Ownership splits sensibly: Ops owns Infrastructure Orchestration and RBAC; Application Configuration is shared between platform and stream-aligned teams; Deployment and Environment Management are developer-facing. A "portal" that does not back onto these five components is a façade.
  - Sources: CNCF Platforms WG whitepaper, §"Attributes of a platform"; Fournier, *Platform Engineering*, Ch. 1 & 4; Salatino, *Platform Engineering on Kubernetes* (Manning, 2023), Ch. 1–2; internaldeveloperplatform.org, *What is an IDP?*, https://internaldeveloperplatform.org/what-is-an-internal-developer-platform/ .
  - Concrete check: list the top five user journeys (provision env, deploy app, rotate secret, request resource, view logs). For each, can the developer self-serve end-to-end without human intervention? Each "no" is a gap. Then map each journey to one of the five IDP components — gaps tend to cluster in Application Configuration or Environment Management.

- **[IDP] [should] Build or adopt an IDP only when row (b) of the platform readiness matrix is crossed — see [`§16 → Platform readiness matrix (heuristic)`](#platform-readiness-matrix-heuristic). Below that row, the right answer is paved-road shell scripts plus README templates plus a shared module repo.**
  - The matrix is the canonical wording: ≥3 stream-aligned teams *and* repeated paved-road requests for the same capability, with at least one consumer team signed up to adopt the first paved road. The numbers are heuristic anchors, not standards; an org that crosses earlier or later for a defended reason records that reason in the platform charter ADR.
  - Sources: see §16 → Platform readiness matrix (heuristic).
  - Concrete check: write the row (b) trigger that fired (which teams, which paved-road requests, which signed-up consumer) into the platform charter ADR. If row (b) is not crossed, the IDP is premature — defer and revisit at the next planning cycle.

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
  - Sources: CNCF Platforms WG whitepaper, §"Self-service"; Fournier, *Platform Engineering*, Ch. 1 & 4; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5.
  - Concrete check: pick any non-trivial resource (a bucket, a Postgres, a queue). Can a stream-aligned team get one in <30 minutes by editing a file in their own repo or calling a documented API? If they have to talk to a human, this is not yet self-service.

- **[IMPLEMENTATIONS] [should] Pick from a family of equivalent patterns; none is universally right.**

| Pattern family | When it fits | Representative tools |
|---|---|---|
| Kubernetes + GitOps + CRD/Composition | You already run K8s at scale; consumers are happy with YAML; ops team can run controllers | Crossplane, Argo CD/Flux, Score-on-K8s, KubeVela |
| Workload spec + multi-target compiler | Consumers should be cloud-agnostic; you target K8s *and* serverless/VMs | Score, Open Application Model, Humanitec |
| SaaS-centric IDP | Small platform team, mostly off-the-shelf SaaS infra (Vercel, Render, Heroku-style); scorecards matter more than custom controllers | Port, Cortex, OpsLevel, Atlassian Compass, Spotify Portal |
| API-gateway / service-catalog | Compliance-heavy or non-K8s estate; provisioning happens via a curated API or catalog | AWS Service Catalog, ServiceNow workflows, custom API |
| Non-Kubernetes orchestration | HPC, ML training, batch, embedded; K8s is the wrong primitive | Nomad, Slurm, vendor schedulers |

- Each row is a defensible choice; the wrong move is to copy a reference architecture without checking which row your organisation is actually in.
- Sources: CNCF Platforms WG whitepaper, §"Capabilities and tools"; ThoughtWorks Tech Radar entries on *Backstage*, *Crossplane*, *Score*, *Port*; Fournier, *Platform Engineering*, Ch. 5 ("Build vs Buy vs Adopt"); Salatino, *Platform Engineering on Kubernetes*, Ch. 3–4.
- Concrete check: the platform's architecture decision record names which row it is in, the rejected alternatives, and the criteria that decided it.

---

## 8. ABSTRACTION LEVEL

- **[ABSTRACTION LEVEL] [must] The right level is *configuration as data* over a small declarative spec — not raw IaC exposed to every team, and not a fully-opinionated PaaS that hides primitives.**
  - Too low (every team writes Terraform from scratch): no leverage, infinite drift, every team re-discovers the same security mistakes. Too high (a black-box PaaS): teams cannot debug, cannot extend, cannot escape; the platform becomes a single point of organisational failure — and (per DORA 2024) the loss of developer independence shows up as degraded delivery stability. The sweet spot is a small declarative schema (a CRD, a Score file, a Crossplane Claim, a catalog entry, an API request body) that the platform compiles into the underlying primitives.
  - Sources: Kelsey Hightower, *Configuration as Data* (KubeCon NA 2019 keynote), https://www.youtube.com/watch?v=ZrRzVl7-S5s ; CNCF Platforms WG whitepaper, §"Capabilities"; Salatino, *Platform Engineering on Kubernetes*, Ch. 3–4; DORA 2024 report.
  - Concrete check: examine the consumer-facing spec for the platform's main abstraction. Is it < ~30 lines of declarative YAML/JSON for a typical service? Is the compilation to primitives readable? Both yes = good.

---

## 9. BACKSTAGE

- **[BACKSTAGE] [should] Consider Backstage as the IDP framework once you have sustained pain from missing software catalog and discoverable docs — typically when a single engineer can no longer enumerate every service from memory.**
  - Backstage was accepted into the CNCF Sandbox in September 2020 and promoted to **Incubating** on 15 March 2022; as of early 2026 it has *not* graduated. Maturity has come from product-surface evolution — the New Frontend System adoption track in 2025, the plugin ecosystem, scaffolder/permissions improvements through Backstage 1.30+ — not from a CNCF tier change. The framework gives three primitives out of the box: Software Catalog (truth for "what services exist and who owns them"), Software Templates (a concrete encoding of a paved road), and TechDocs (Markdown + MkDocs colocated with code, rendered in the catalog). At the right scale these three deliver more value than any homegrown alternative.
  - Sources: Backstage docs, https://backstage.io/docs/overview/what-is-backstage ; Backstage blog, *Backstage Wrapped 2025* (2025-12-30) and *BackstageCon Atlanta 2025* recap, https://backstage.io/blog/ ; CNCF project page, https://www.cncf.io/projects/backstage/ ; ThoughtWorks Tech Radar, *Backstage*, https://www.thoughtworks.com/radar/platforms/backstage .
  - Concrete check: every production service appears in the catalog with an owner (a team, not a person), a TechDocs entry, and at least one Software Template that could re-create a similar service.

- **[BACKSTAGE] [must] Enable the permission framework before exposing Backstage to production users. The default is open.**
  - The official documentation states it directly: "By default, Backstage endpoints are not protected, and all actions are available to anyone." Backstage's permission framework supports RBAC, ABAC, code-based policies, and external providers, but operators must opt in. A Backstage instance that surfaces production catalog data, scaffolds new repos, or invokes provisioning actions without permissions configured is a self-inflicted privilege-escalation vector.
  - Sources: Backstage docs — *Permissions overview*, https://backstage.io/docs/permissions/overview/ ; Backstage release-versions, https://backstage.io/docs/overview/release-versions .
  - Concrete check: in the deployed Backstage instance, attempt to invoke a scaffolder template or read a sensitive catalog entity as an unauthenticated user (or as a user with no team membership). If either succeeds, the permission framework is not configured. Cross-link with ch07 mTLS and ch06 supply-chain controls — Backstage scaffolder actions can mint new repos and CI configurations; treat them as privileged.

- **[BACKSTAGE] [should] Adopt Backstage only when row (c) of the platform readiness matrix is crossed — see [`§16 → Platform readiness matrix (heuristic)`](#platform-readiness-matrix-heuristic). If discovery is not yet the bottleneck, or if you will use only one or two of the three Backstage surfaces (catalog, TechDocs, Scaffolder), a `services.yaml` plus a docs site plus `cookiecutter`/`copier` templates is cheaper and easier to keep alive.**
  - The matrix is the canonical wording: deploy Backstage when discovery is the bottleneck (a single engineer can no longer enumerate services from memory) *and* all three surfaces will be in steady-state use *and* there is a funded owner for the TypeScript app, plugin upgrades, auth, catalog ingestion, and Postgres. The "≥30 catalog entities" smell test cited in earlier drafts is a *consequence* of those conditions, not an independent gate.
  - Sources: see §16 → Platform readiness matrix (heuristic); Backstage docs, https://backstage.io/docs/overview/what-is-backstage ; Spotify Portal, https://backstage.spotify.com/ ; ThoughtWorks Tech Radar, *Backstage*, https://www.thoughtworks.com/radar/platforms/backstage .
  - Concrete check: confirm row (c)'s gating signal in writing before deploying Backstage. If any of the three surfaces will not be in steady use, or if no team owns the run-cost, defer Backstage and ship the cheaper substitute.

- **[BACKSTAGE] [avoid] Adopting Backstage when the run-cost dominates the benefit, or treating it as the platform itself rather than a frontend to a platform.**
  - Backstage is a TypeScript application your team will own (auth, plugin upgrades, catalog ingestion, database, deployment). If you cannot fund that ownership, a managed alternative — **Spotify Portal for Backstage** (the first-party hosted offering, with optional premium plugins: Soundcheck, AiKA, RBAC, Insights, Skill Exchange), Roadie, Port, Cortex, OpsLevel, Compass, Humanitec — or a single README per service plus a shared module repo is the right answer.
  - Sources: ThoughtWorks Tech Radar, *Backstage* (cautions on adoption cost); Fournier, *Platform Engineering*, Ch. 5; Spotify Portal landing page, https://backstage.spotify.com/ ; Roadie, *Should you adopt Backstage?*, https://roadie.io/blog/ .

- **[BACKSTAGE] [should] Treat AI assistants on the platform — Backstage's Actions Registry, MCP server support, Spotify's AiKA — as code/decision generators with the same review and provenance requirements as any other code-gen tool.**
  - Backstage's roadmap as of late 2025 is explicitly "AI-Native": the Actions Registry and MCP server support let LLM agents drive scaffolder and catalog actions; Spotify's commercial AiKA plugin is an LLM "AI Knowledge Assistant" over the catalog/TechDocs corpus. These are real surfaces, not hypotheticals, and they inherit two well-documented risks. First, hallucination over stale TechDocs ("how do I deploy?" answered from a 2-year-old runbook) — every AI answer must surface the source documents it drew from and their last-modified date. Second, agent-driven scaffolder actions can mint repos, CI pipelines, cloud resources; without the permission framework (above) and audit logging, an LLM prompt injection becomes a privileged execution path. DORA 2024 separately reports that AI assistance, like platform engineering, raises individual productivity but can degrade delivery stability if introduced without guardrails.
  - Sources: Backstage blog, *Backstage Wrapped 2025* and *BackstageCon Atlanta 2025* recap, https://backstage.io/blog/ ; Spotify Portal — AiKA description, https://backstage.spotify.com/ ; DORA, *2024 Accelerate State of DevOps Report*, https://dora.dev/research/2024/dora-report/ .
  - Concrete check: every AI-assisted answer in the portal renders provenance links (source doc + last-modified). Every AI-driven scaffolder/MCP action is gated by the permission framework, logged with prompt+result, and produces a reviewable PR rather than a direct write. If any of these is missing, disable the AI surface until they are.

### When *not* to build or adopt — heuristic decision table

| Signal | Lean toward… |
|---|---|
| <3 stream-aligned teams suffering the same toil | Defer; share a README / Helm chart / CI template instead |
| No platform-team capacity for steady-state ownership of a TS app | Spotify Portal (first-party hosted) or another managed IDP (Roadie/Port/Cortex/OpsLevel/Compass) over self-hosted Backstage |
| Service count low enough to fit in one engineer's head | Defer software catalog; revisit when discovery becomes painful |
| Mostly SaaS-managed infra, little custom provisioning | SaaS-centric IDP over Backstage + Crossplane stack |
| Heavy regulated/compliance estate, non-K8s | API gateway / service catalog pattern over CRD-based platform |
| Sustained funding for ≥1 dedicated platform PM-plus-engineers | Build/adopt is on the table |

- **The platform readiness matrix in §16 is the canonical source for *when* to embed engineers, adopt an IDP, deploy Backstage, or charter a team. Override a matrix row only with a written ADR naming the regulated/compliance, M&A-roll-up, or uniform-workload reason; "we feel ready" is not such a reason.** Some 20-engineer fintechs *do* need a platform on day one because a regulator says so; some 200-engineer agencies never do because their workloads are uniform — those are the documented exceptions, not the rule.
- Sources: Larson, *An Elegant Puzzle*, §"Sizing engineering teams" and §"Internal tools"; Fournier, *Platform Engineering*, Ch. 1 ("When you don't need a platform team") and Ch. 5; CNCF Platforms WG whitepaper, §"Maturity model"; Humanitec, *State of Platform Engineering*; Puppet, *2024 State of Platform Engineering Report*.

---

## 10. TEAM TOPOLOGIES

- **[TEAM TOPOLOGIES] [must] Organise infra work around the four team types and three interaction modes from *Team Topologies*; do not invent a fifth.**
  - The four types are *stream-aligned* (owns a value stream end-to-end), *platform* (provides internal services to stream-aligned teams), *enabling* (helps adopt new capabilities, then steps back), and *complicated-subsystem* (deep specialism — ML inference, video pipeline). The three interaction modes are *collaboration* (short-term, high-bandwidth), *X-as-a-Service* (consumer treats provider as black box), and *facilitating* (enabling team helps another). Platform work is overwhelmingly X-as-a-Service; enabling work is short-lived and skill-transfer shaped. The model is unchanged in the 2nd edition (August 2025); the new material expands operating-model guidance and case studies, not the taxonomy.
  - Sources: Skelton & Pais, *Team Topologies*, 2nd ed. (IT Revolution, 2025), Ch. 4 & 7 — see https://teamtopologies.com/news for the 2nd-edition announcement; key concepts, https://teamtopologies.com/key-concepts/ ; Fournier, *Platform Engineering*, Ch. 3.
  - Concrete check: every infra-adjacent team is labelled with one of the four types and its dominant interaction mode is documented per consumer. Teams that interact with everyone, in collaboration mode, all the time, are misclassified bottlenecks.

---

## 11. CONWAY

- **[CONWAY] [must] The architecture you ship will mirror the communication structure of the org that built it. Plan accordingly (the Inverse Conway Maneuver).**
  - Conway's 1968 *Datamation* paper, *How Do Committees Invent?*, argued that "organizations which design systems … are constrained to produce designs which are copies of the communication structures of these organizations." The familiar punchier wording is Conway's *own informal restatement*, offered on his website roughly four decades later: "to save you the trouble" of reading 45 paragraphs, he writes, "I'll give an informal version of it to you now: Any organization that designs a system … will inevitably produce a design whose structure is a copy of the organization's communication structure." Either form supports the same point. The inverse maneuver is to deliberately restructure teams so the desired architecture is the *easy* one to build.
  - Sources: Melvin E. Conway, *How Do Committees Invent?*, Datamation, April 1968 — paper and author's note at http://www.melconway.com/Home/Committees_Paper.html ; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 2; ThoughtWorks Tech Radar, *Inverse Conway Maneuver*, https://www.thoughtworks.com/radar/techniques/inverse-conway-maneuver .
  - Concrete check: the platform-team boundary matches the API boundary of the platform. If the platform exposes one product but is split across three teams that don't share a backlog, expect the API to fragment to match. (ThoughtWorks Radar names this exact failure: "layered platform teams that simply preserve existing technology silos.")

---

## 12. COGNITIVE LOAD

- **[COGNITIVE LOAD] [must] The platform exists, fundamentally, to keep the *intrinsic* cognitive load on stream-aligned teams within human limits.**
  - Cognitive load (Sweller, educational psychology, 1988) decomposes into intrinsic (inherent task complexity), extraneous (poor tools, accidental complexity), and germane (cost of building useful mental models). A stream-aligned team that must also operate Kubernetes, Terraform, secrets, CI runners, and observability *plus* its own product domain is overloaded — the platform's job is to make those shared concerns X-as-a-Service so the team's load fits.
  - Sources: John Sweller, *Cognitive Load During Problem Solving*, Cognitive Science 12(2), 1988, https://onlinelibrary.wiley.com/doi/10.1207/s15516709cog1202_4 ; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 3 ("Team-First Thinking, Cognitive Load"); Fournier, *Platform Engineering*, Ch. 1.
  - Concrete check: a stream-aligned team can list, in one breath, the *concepts* it owns (its domain, its service, the platform spec it writes). If the list also includes "Helm internals", "Argo CD reconciler loops", "VPC CIDRs", "IAM trust policies", the platform is leaking.

---

## 13. ANTI-PATTERNS

- **[ANTI-PATTERNS] [avoid] Ivory-tower platform — design without consumer contact.**
  - Replacement: weekly office hours, an "embedded engineer" rotation, or shadowing one stream-aligned team per quarter. internaldeveloperplatform.org puts it bluntly: platforms must "meet developers where they are at" or fail adoption regardless of technical merit.
  - Sources: Fournier, *Platform Engineering*, Ch. 2; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5; internaldeveloperplatform.org, https://internaldeveloperplatform.org/what-is-an-internal-developer-platform/ .
  - Concrete check: count the platform-team commits in a quarter whose linked issue cites a stream-aligned consumer. <50% is a flag.

- **[ANTI-PATTERNS] [avoid] Mandatory human mediation on the paved road. The moment a human must approve every traversal, the road becomes a queue.**
  - Replacement: enforce *controls* in the path itself (signing, policy-as-code, automated review) and reserve human review for genuine exceptions with a documented SLA. Compete on quality, not policy. This is now the empirically-backed anti-pattern: DORA 2024's developer-independence finding ties exactly here — platforms that remove independence depress stability and throughput.
  - Sources: Spotify "Golden Paths" post; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5; CNCF Platforms WG whitepaper, §"Capabilities"; DORA 2024 report.
  - Concrete check: at least one production team can satisfy required controls and ship via the paved road with zero synchronous human approvals.

- **[ANTI-PATTERNS] [avoid] "We know what they need" — building before validating.**
  - Replacement: an MVP scoped to one consumer team that has signed up to adopt within a fixed window. Kill the project if adoption does not happen.
  - Sources: Larson, *An Elegant Puzzle*, §"Migrations" and §"Internal tools"; Fournier, *Platform Engineering*, Ch. 2.
  - Concrete check: every shipped feature has a named first adopter who used it within 30 days of release.

- **[ANTI-PATTERNS] [avoid] Treating the platform team as an SRE-ticket queue.**
  - Replacement: SRE work has its own home; platform work is *productised* shared services. The two are different shapes; do not collapse them. (Incident response and on-call are owned by ch09; SLO baselines by ch05.)
  - Sources: Google SRE book, Ch. 1, https://sre.google/sre-book/introduction/ ; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 4.
  - Concrete check: the platform team's tracker has <~20% break-fix tickets; the rest is roadmap product work.

- **[ANTI-PATTERNS] [avoid] Layered platform teams that preserve existing technology silos and do not share a backlog.**
  - ThoughtWorks Radar names this directly: "We caution against [layered platform teams] that simply preserve existing technology silos … and ticket-driven platform operating models." The Conway prediction (§11) follows: a platform exposed by three sibling teams that don't share a backlog will expose three sibling APIs.
  - Replacement: one platform-team boundary per platform-product boundary; if multiple sub-teams are unavoidable, force a shared backlog and a single PM.
  - Sources: ThoughtWorks Tech Radar, *Platform engineering product teams*, https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams ; Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5.
  - Concrete check: count the distinct planning forums that prioritise work landing on the platform's user-facing surface. More than one is a smell.

---

## 14. METRICS

- **[METRICS] [must] Measure platform success on two axes: (a) consumer outcomes via DORA's current five software-delivery metrics, and (b) direct platform KPIs that the platform team controls.**
  - DORA's current model is **five** metrics, evolved from the original four-key model and confirmed in the 2024 *Accelerate State of DevOps Report*: change lead time, deployment frequency, **failed deployment recovery time** (replacing MTTR), change fail rate, and **deployment rework rate**. These measure the throughput and instability of the delivery system the platform is part of. Pair them with **SPACE** (Forsgren, Storey, Maddila, Zimmermann, Houck & Butler, ACM Queue 2021) for developer-experience signals — Satisfaction, Performance, Activity, Communication, Efficiency — that DORA does not capture. SPACE is explicitly designed to be used alongside DORA, not as a replacement: DORA covers system outcomes, SPACE covers human outcomes, and the platform-as-a-product thesis is only defensible when both improve.
  - Direct platform KPIs the team owns: **platform SLOs** (e.g., 99.9% successful provisioning of a Postgres in <10 min), **self-service success rate** (% of journeys completed without a human ticket), **journey latency** (p50/p95 time from intent to working resource), **onboarding time** (new tenant from request to first deploy), **adoption** per shipped capability, and a recurring **developer survey** (NPS-style or SPACE-aligned, ≥quarterly).
  - Sources: DORA, *2024 Accelerate State of DevOps Report*, https://dora.dev/research/2024/dora-report/ ; DORA metrics guide, https://dora.dev/guides/dora-metrics/ ; Forsgren, Humble & Kim, *Accelerate* (IT Revolution, 2018), Ch. 2; Forsgren, Storey, Maddila, Zimmermann, Houck & Butler, *The SPACE of Developer Productivity*, ACM Queue 2021, https://queue.acm.org/detail.cfm?id=3454124 ; Fournier, *Platform Engineering*, Ch. 6; CNCF Platforms WG whitepaper, §"Measuring success".
  - Concrete check: the platform team publishes, at least quarterly, (a) the five DORA metrics for top consumer teams, (b) platform SLO attainment, (c) self-service success rate and journey latency for the top five journeys, (d) adoption per capability, (e) a developer-experience survey result. Absence of any line is a finding. Watch the DORA series for a stability/throughput drop after a major capability launch — per DORA 2024, that drop is the leading indicator of an opaque or human-mediated abstraction.

---

## 15. TOOLING

- **[TOOLING] [should] Build the platform from established CNCF/OSS or managed building blocks; integrate, don't reinvent. Pick the family that matches §7.**
- Reference stacks, late 2025/early 2026, by family (sources: CNCF landscape, https://landscape.cncf.io/ ; CNCF Platforms WG whitepaper, *Capabilities and tools*; ThoughtWorks Tech Radar entries on each project; Salatino, *Platform Engineering on Kubernetes*, Ch. 6–10):
  - K8s + GitOps + CRD: *Backstage* (catalog/portal, CNCF Incubating since March 2022), *Crossplane* (provisioning), *Argo CD* / *Flux* (deploy), *Score* / *Humanitec* (workload spec), *OpenTelemetry*, *cert-manager*, *external-secrets*.
  - SaaS-centric IDP: *Spotify Portal* (first-party managed Backstage) / *Roadie* / *Port* / *Cortex* / *OpsLevel* / *Compass* + cloud-native managed services + Terraform Cloud / Spacelift.
  - API-gateway / catalog: *AWS Service Catalog* or equivalent, fronted by a thin internal API and a portal of choice.
  - Non-K8s: *Nomad* / *Slurm* / vendor schedulers, with the same catalog/portal/observability seams.
- Concrete check: the architecture diagram names the OSS or SaaS project behind each capability and shows where each customisation was added. Diagrams that show only product names hide lock-in.

---

## 16. READINESS

### Platform readiness matrix (heuristic)

This is the canonical wording for *when* to take each platform-shaped step.
Sibling sections (§5 IDP, §9 Backstage), `docs/decision-trees.md` §5, and
`checklist.md` §11 should link here rather than restate thresholds. The
numeric anchors below are **heuristic anchors, not standards**: they come
from Humanitec / CNCF / Puppet platform-engineering surveys and Team
Topologies' X-as-a-Service guidance, and an org that crosses a row's
gating signal earlier or later for a defended reason should record that
reason in the platform charter ADR. Apply the rows in order — embed
before IDP, IDP before Backstage, Backstage before chartering a dedicated
team.

| Row | Trigger | Gating signal | Fallback if not met |
|---|---|---|---|
| (a) Embed part-time platform engineers into stream-aligned teams | ≥1 stream-aligned team is repeatedly blocked on shared-infra toil | Platform-shaped work appears in two consecutive quarterly plans with a named owner | Coach the loudest team in place; defer any separate platform structure |
| (b) Adopt curated IDP / golden paths | ≥3 stream-aligned teams *and* repeated paved-road requests for the same capability (recurring postmortem actions, DevEx survey regressions, paved-road bypass rate >30%, or ≥6 months of demonstrated friction) | At least one consumer team has signed up to adopt the first paved road end-to-end | Shell scripts + README templates + a shared module repo |
| (c) Deploy Backstage (or an equivalent catalog) | Discovery is the bottleneck — a single engineer can no longer enumerate services from memory — *and* all three Backstage surfaces (catalog, TechDocs, Scaffolder) will be used in steady state | A funded owner exists for the TypeScript app, plugin upgrades, auth, catalog ingestion, and the Postgres behind it | `services.yaml` + a docs site + `cookiecutter`/`copier` templates; or a managed alternative (Spotify Portal, Roadie, Port, Cortex, OpsLevel, Compass) |
| (d) Charter a dedicated platform team | All four: ≥4 stream-aligned teams suffer the same shared-infra pain; recurring funded backlog of cross-team enablement (≥1 quarter visible) with consumer teams signed up; sustained funding (≥2 engineers + PM-equivalent for ≥1 year, not borrowed time); the abstraction saves more team-hours/quarter than it costs to maintain | A written "which 4+ teams, what funded backlog, how is it staffed for ≥1 year?" memo exists and names the rows above as already-crossed | Embed part-time per row (a) and revisit at the next planning cycle |

- The "≥30 services" and "~10 services per engineer" numbers cited in earlier drafts of this chapter are *consequences* of these rows, not independent gates — use them as smell tests for whether you have actually crossed a row, not as standalone triggers.
- Premature charters (row (d) without rows (a)–(c)) regress on DORA delivery metrics within 12 months in both the Humanitec and Puppet surveys; premature Backstage adoption (row (c) for one of three surfaces) is the most-cited reason ThoughtWorks Tech Radar has held Backstage in *Trial* for several editions.
- Sources: Skelton & Pais, *Team Topologies* 2nd ed., Ch. 5 ("When to form a platform team"); Larson, *An Elegant Puzzle*, §"Sizing engineering teams"; Fournier, *Platform Engineering*, Ch. 1 ("When you don't need a platform team"); Humanitec, *State of Platform Engineering*, https://humanitec.com/state-of-platform-engineering ; Puppet, *2024 State of Platform Engineering Report*, https://www.puppet.com/resources/state-of-platform-engineering ; CNCF Platforms WG, *Platform Engineering Maturity Model*, https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/ ; Gartner, *Hype Cycle for Platform Engineering* (2024), https://www.gartner.com/en/documents/5571295 ; ThoughtWorks Tech Radar, *Backstage*, https://www.thoughtworks.com/radar/platforms/backstage .

- **[READINESS] [must] Charter on the matrix above, not on a fixed headcount.** Before standing up an IDP, Backstage, or a chartered platform team, walk the rows in order and check the gating signal. If row (d)'s four-part test is not all-true, embed part-time per row (a) and revisit at the next planning cycle. The four-part test is the canonical statement; sibling chapters and synthesis files cite this row, they do not redefine it.

---

## 17. DOCS

- **[DOCS] [must] Documentation is a first-class platform feature: colocated with the code, searchable, versioned. "If it isn't searchable, it doesn't exist."**
  - Backstage TechDocs is the canonical implementation in the K8s family: Markdown lives next to code in each repo, MkDocs renders it, the portal aggregates. SaaS IDPs typically provide an equivalent (Port docs widgets, Cortex catalog docs). ADRs (Nygard, 2011) capture *why* a decision was made and are themselves discoverable docs. If an AI assistant fronts the docs (§9), every answer must link to the source TechDocs entry and its last-modified date — stale docs answered with confidence are worse than no answer.
  - Sources: Backstage TechDocs, https://backstage.io/docs/features/techdocs/ ; Michael Nygard, *Documenting Architecture Decisions*, https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions ; Write the Docs, *Docs as Code*, https://www.writethedocs.org/guide/docs-as-code/ ; Fournier, *Platform Engineering*, Ch. 4.
  - Concrete check: every platform capability has a doc reachable in ≤2 clicks from the catalog/portal; every load-bearing decision has an ADR in the platform repo. Search "deploy a service" in the portal — if the answer is not in the first three results, docs are failing.

---

## 18. CRITIQUES AND OPEN QUESTIONS

- **[CRITIQUE] [should] Take seriously the "platform engineering is just rebadged ops" critique — including from operations veterans.**
  - The most-cited counter-argument inside the SRE/operations community is that "platform engineering" is what good operations teams already did, with a product-shaped vocabulary bolted on. Charity Majors and others have publicly warned that the rename can become an excuse to defund SRE practice (on-call, error budgets, blameless postmortems) and to reframe ticket-driven work as a "platform". The defence is empirical, not rhetorical: a platform team that publishes the metrics in §14 and meets the readiness criteria in §16 is observably different from a renamed ops team; one that does neither is the critique made flesh.
  - Sources: Charity Majors public commentary on platform vs operations (honeycomb.io blog and conference talks); ThoughtWorks Tech Radar, *Platform engineering product teams* (cautions section); DORA 2024 report (developer-independence finding).
  - Concrete check: a sceptical reviewer reads the platform team's last-quarter tracker, last-quarter metrics dashboard, and operating-model doc. If none of the three would distinguish the team from a 2015-vintage central ops team, the critique applies.

- **[CRITIQUE] [prefer] Keep one open question visible: when the platform itself is the bottleneck, who unblocks?**
  - The Inverse Conway Maneuver (§11) assumes the platform team can be re-shaped to match the desired architecture. In practice, the platform team is often the *most* constrained team in the org (specialised hires, multi-quarter roadmaps, regulated change windows). Stream-aligned teams that depend on a platform capability that is six months out have the same wait-state as the ticket-queue era — just with a nicer Jira board. The honest answer is the off-ramp (§4): the paved road is supported, the off-road is unsupported but permitted, and the platform team's job is to shorten the off-road's half-life, not to police it.
  - Sources: Larson, *An Elegant Puzzle*, §"Migrations"; Fournier, *Platform Engineering*, Ch. 5; DORA 2024 (developer-independence finding).
  - Concrete check: for the top three "blocked on platform" complaints in the last quarter, is there a documented off-ramp the consumer team could have taken? If not, write one.

---

## 19. CROSS-CHAPTER GLOSSARY (chapter-owned terms)

This section makes the terms this chapter owns reusable from sibling chapters; sibling chapters should link here rather than redefine.

- **Internal Developer Platform (IDP)** — the integrated set of self-service capabilities a platform team exposes to stream-aligned teams. Five core components per the internaldeveloperplatform.org community: Infrastructure Orchestration, Application Configuration Management, Deployment Management, Environment Management, RBAC. Distinct from a developer *portal*, which is one possible UI on top of an IDP.
- **Paved road** — the supported, batteries-included, opt-in path for a common task (provision a service, ship a deploy pipeline, wire observability). Canonical term in this guide; *golden path* (Spotify, ~2014) is treated as a synonym.
- **Off-ramp** — a documented, supported way to leave the paved road for a specific need, with an exception SLA. Distinct from "off-road" (unsupported but permitted).
- **DORA-5** — the current five-metric DORA software-delivery model: change lead time, deployment frequency, failed deployment recovery time, change fail rate, deployment rework rate. Replaces the "DORA-4" framing from *Accelerate* (2018).
- **SPACE** — Forsgren et al. (ACM Queue 2021) developer-productivity framework: Satisfaction, Performance, Activity, Communication, Efficiency. Used alongside DORA, not instead of it.
- **Platform-as-a-product** — the operating model in which the platform has a named PM, a roadmap, named consumer teams, documented SLOs *to* those consumers, and a deprecation policy. The chapter's load-bearing thesis.
- **Stream-aligned / platform / enabling / complicated-subsystem** — the four *Team Topologies* team types. **Collaboration / X-as-a-Service / facilitating** — the three interaction modes. Unchanged in the 2nd edition (2025).
- **Inverse Conway Maneuver** — deliberately restructuring teams so the desired system architecture is the easy one to build, given Conway's law (§11). Distinct from "let the org chart drift and hope for the best".
- **Configuration as data** — Hightower's framing (§8): consumer-facing schema is small, declarative, and compiled by the platform into underlying primitives. The sweet spot between raw IaC and a black-box PaaS.
- **AI-Native platform surface** — AI assistants integrated into the IDP (Backstage Actions Registry / MCP, Spotify AiKA, equivalents). Treated in this guide as a code/decision generator: requires provenance links, permission gating, and audit logging (§9, §17).

---

## Sources

### Books

- Matthew Skelton & Manuel Pais, *Team Topologies*, 2nd ed., IT Revolution, 2025 — https://teamtopologies.com/book (1st ed. 2019; 2nd-ed announcement https://teamtopologies.com/news )
- Camille Fournier (ed.), *Platform Engineering*, O'Reilly, 2024
- Mauricio Salatino, *Platform Engineering on Kubernetes*, Manning, 2023
- Nicole Forsgren, Jez Humble & Gene Kim, *Accelerate*, IT Revolution, 2018
- Will Larson, *An Elegant Puzzle*, Stripe Press, 2019

### Foundational papers

- Melvin E. Conway, *How Do Committees Invent?*, Datamation, April 1968 — http://www.melconway.com/Home/Committees_Paper.html (paper plus author's informal restatement)
- John Sweller, *Cognitive Load During Problem Solving*, Cognitive Science 12(2), 1988 — https://onlinelibrary.wiley.com/doi/10.1207/s15516709cog1202_4
- Forsgren, Storey, Maddila, Zimmermann, Houck & Butler, *The SPACE of Developer Productivity*, ACM Queue 19(1), 2021 — https://queue.acm.org/detail.cfm?id=3454124
- Michael Nygard, *Documenting Architecture Decisions*, 2011 — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions

### Standards & working groups

- CNCF TAG App Delivery, *Platforms White Paper* — https://tag-app-delivery.cncf.io/whitepapers/platforms/
- CNCF TAG App Delivery — https://github.com/cncf/tag-app-delivery
- CNCF project page, *Backstage* (Incubating since 15 March 2022; not yet Graduated) — https://www.cncf.io/projects/backstage/
- DORA, *2024 Accelerate State of DevOps Report* — https://dora.dev/research/2024/dora-report/
- DORA metrics guide (current five-metric model) — https://dora.dev/guides/dora-metrics/
- DORA, *Platform engineering* capability — https://dora.dev/capabilities/platform-engineering/
- Score workload specification — https://docs.score.dev/

### Vendor / project documentation (canonical statement, not endorsement)

- Backstage — https://backstage.io/docs/overview/what-is-backstage
- Backstage TechDocs — https://backstage.io/docs/features/techdocs/
- Backstage Software Templates — https://backstage.io/docs/features/software-templates/
- Backstage Permissions — https://backstage.io/docs/permissions/overview/
- Backstage blog (Wrapped 2025; BackstageCon recaps; AI-Native roadmap) — https://backstage.io/blog/
- Crossplane Compositions — https://docs.crossplane.io/latest/concepts/composition/
- Humanitec reference architectures — https://humanitec.com/whitepapers/reference-architecture-for-internal-developer-platforms
- Spotify Portal for Backstage (commercial managed Backstage + premium plugins, incl. AiKA) — https://backstage.spotify.com/
- Roadie (managed Backstage) — https://roadie.io/
- Argo CD — https://argo-cd.readthedocs.io/
- Flux — https://fluxcd.io/

### Named industry voices

- Pais & Skelton, Team Topologies key concepts — https://teamtopologies.com/key-concepts/
- Kelsey Hightower, *Configuration as Data* (KubeCon NA 2019 keynote) — https://www.youtube.com/watch?v=ZrRzVl7-S5s
- Spotify Engineering, *How We Use Golden Paths to Solve Fragmentation* (origin ~2014 Hack Week; published 2020) — https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/

### Industry analysis

- ThoughtWorks Tech Radar, *Platform engineering product teams* (Adopt, Oct 2021) — https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams
- ThoughtWorks Tech Radar, *Backstage* — https://www.thoughtworks.com/radar/platforms/backstage
- ThoughtWorks Tech Radar, *Inverse Conway Maneuver* — https://www.thoughtworks.com/radar/techniques/inverse-conway-maneuver
- ThoughtWorks Tech Radar, *Score* — https://www.thoughtworks.com/radar/languages-and-frameworks/score
- internaldeveloperplatform.org community (five-component IDP definition) — https://internaldeveloperplatform.org/what-is-an-internal-developer-platform/
- platformengineering.org / PlatformCon (sub-discipline taxonomy) — https://platformengineering.org/blog

---
