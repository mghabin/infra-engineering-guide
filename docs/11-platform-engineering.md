# Infrastructure Engineering — Chapter 11: Platform Engineering

This chapter is about the *team and product shape* of infrastructure work, not
the tools. Tooling (Backstage, Crossplane, Argo, Score, Humanitec) appears
only as evidence that a pattern has crystallised. The plumbing chapters (02
IaC, 04 Kubernetes, 07 GitOps) describe the wires; this chapter describes who
owns them, who consumes them, and how to know whether the investment is
paying off.

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

**[WHAT IT IS] [must] Platform engineering is the discipline of building
internal, opinionated, self-service products that reduce the cognitive load on
stream-aligned (product) teams.**
It is not "DevOps with a new logo" and it is not "SRE for everyone". DevOps is
a culture (you-build-it-you-run-it, ch01 §6); SRE is a role with explicit
reliability budgets; platform engineering is the *team shape* that productises
the shared paths those cultures depend on.
- Sources: Skelton & Pais, *Team Topologies* (IT Revolution, 2019), Ch. 4 & 5; CNCF Platforms WG, *Platforms White Paper*, https://tag-app-delivery.cncf.io/whitepapers/platforms/ ; Camille Fournier (ed.), *Platform Engineering* (O'Reilly, 2024), Ch. 1.
- Concrete check: ask three random stream-aligned engineers "what does the platform team produce?". If answers are role-shaped ("they run Kubernetes") rather than product-shaped ("the paved-road service template, the deploy CLI"), the team is not yet doing platform engineering.

**[WHAT IT IS] [must] A platform is the *thinnest viable* layer over the
underlying primitives, not a re-invention of them.**
Platforms aggregate; they do not rewrite the cloud. A platform that hides
Kubernetes so completely that consumers cannot read its manifests will fail
the first time the abstraction leaks.
- Sources: CNCF Platforms WG whitepaper, §"What is a Platform"; Pais & Skelton, key concepts, https://teamtopologies.com/key-concepts/ ; ThoughtWorks Tech Radar, *Platform engineering product teams*, https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams .
- Concrete check: a stream-aligned engineer can trace any platform abstraction one layer down to the primitive it wraps (Helm chart, Terraform module, Kubernetes object). If not, the abstraction is opaque.

---

## 2. AS A PRODUCT

**[AS A PRODUCT] [must] The platform has named internal customers, a roadmap,
and a feedback loop. Treat it exactly like an external product.**
"Platform as a product" is the load-bearing idea. It implies a product owner,
a backlog driven by user research not tickets, documented SLOs *to*
consumers, a deprecation policy, and a roadmap visible to the rest of
engineering.
- Sources: Skelton & Pais, *Team Topologies*, Ch. 5 ("Platform teams: treat the platform as a product"); Fournier, *Platform Engineering*, Part I; ThoughtWorks Tech Radar, *Platform engineering product teams*.
- Concrete check: the platform team has (a) a written product vision, (b) a public roadmap (a README is enough), (c) a recurring channel for consumer feedback, (d) named owners per capability. Absence of any is a finding.

**[AS A PRODUCT] [must] Validate demand before building. No platform feature
ships without an identified consumer team that has agreed to adopt it.**
The most common failure mode is "we know what they need" — a beautiful
abstraction nobody adopts. This is the standard product-management failure
(no problem-validation, no MVP) re-skinned.
- Sources: Fournier, *Platform Engineering*, Ch. 2 ("Discovery and Demand"); Will Larson, *An Elegant Puzzle* (Stripe Press, 2019), §"Migrations" and §"Internal tools"; Pais, *Team Topologies & Platforms* talk series, https://teamtopologies.com/news .
- Concrete check: every roadmap item links to (a) the consumer team that asked for it, (b) the user-research artefact, (c) the success metric. Items missing any of these are speculative — defer.

---

## 3. GOLDEN PATHS

**[GOLDEN PATHS] [must] Provide *opt-in* paved roads, not mandatory walled
gardens.**
Spotify's "golden path" is the supported, batteries-included way to do a
thing — provision a service, ship a deploy pipeline, wire observability. The
opt-in property is non-negotiable: the moment the path is mandatory, it is no
longer a road, it is a gatekeeping queue, and you are back to the
anti-pattern in ch01 §6.
- Sources: Spotify Engineering, *How We Use Golden Paths to Solve Fragmentation* (2020), https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/ ; Skelton & Pais, *Team Topologies*, Ch. 5 (X-as-a-Service); CNCF Platforms WG whitepaper, §"Capabilities".
- Concrete check: pick the flagship golden path. Can a team consciously *not* use it (bring its own pipeline) without asking permission? "No" = walled garden. Document the off-ramp.

**[GOLDEN PATHS] [should] The golden path must also be the lowest-friction
*and* lowest-risk option. Make the right thing the easy thing.**
If the paved road is more painful than rolling your own, engineers route
around it and the platform team ends up policing rather than enabling. The
forcing function is ergonomics, not policy.
- Sources: Spotify "Golden Paths" post (above); Fournier, *Platform Engineering*, Ch. 4 ("Ergonomics and Adoption").
- Concrete check: time-to-first-deploy via the golden path vs off-road. If off-road is faster, the path is broken.

---

## 4. IDP

**[IDP] [must] An *Internal Developer Platform* (IDP) is the integrated
product surface; a developer *portal* is one possible UI on top of it.**
"We have a portal, therefore we have a platform" is wrong: a portal that
fronts ten disconnected backend processes ("file a Jira", "ask in Slack") is
just a prettier router. The platform is the set of *self-service
capabilities*; the portal/CLI/API is the front door.
- Sources: CNCF Platforms WG whitepaper, §"Attributes of a platform"; internaldeveloperplatform.org, *What is an IDP?*, https://internaldeveloperplatform.org/what-is-an-internal-developer-platform/ ; Humanitec, *Reference Architectures for IDPs*, https://humanitec.com/blog/reference-architectures-for-internal-developer-platforms .
- Concrete check: list the top five user journeys (provision env, deploy app, rotate secret, request resource, view logs). For each, can the developer self-serve end-to-end without human intervention? Each "no" is a gap.

**[IDP] [must] Expose the platform as an *API* first; the UI is one client of
that API.**
If the only way to use the platform is a humans-only portal, CI cannot use
it, scripting cannot use it, IDEs cannot integrate, and the platform team
becomes the only client capable of fixing edge cases.
- Sources: Humanitec reference architectures (above); CNCF Platforms WG whitepaper, §"Provisioning interfaces"; Salatino, *Platform Engineering on Kubernetes* (Manning, 2023), Ch. 1–2.
- Concrete check: every UI action has a documented CLI/API equivalent. If the UI can do something the API cannot, that capability is not yet platformised.

---

## 5. BACKSTAGE

**[BACKSTAGE] [should] Use Backstage as the default IDP framework once the
organisation needs a software catalog (~50+ engineers, or ~30+ services).**
Backstage (Spotify origin, CNCF graduated 2022) gives three primitives out of
the box: Software Catalog (truth for "what services exist and who owns
them"), Software Templates (a concrete encoding of a golden path), and
TechDocs (Markdown + MkDocs colocated with code, rendered in the catalog).
At scale these three deliver more value than any homegrown alternative.
- Sources: Backstage docs, https://backstage.io/docs/overview/what-is-backstage ; CNCF project page, https://www.cncf.io/projects/backstage/ ; ThoughtWorks Tech Radar, *Backstage*, https://www.thoughtworks.com/radar/platforms/backstage .
- Concrete check: every production service appears in the catalog with an owner (a team, not a person), a TechDocs entry, and at least one Software Template that could re-create a similar service.

**[BACKSTAGE] [avoid] Adopting Backstage in an org under ~30 engineers, or
treating it as the platform itself rather than a frontend to a platform.**
Replacement: a single README per service plus a shared Terraform/Helm
module repo. Backstage is a TypeScript application your team will own; if
that team is one or two people, ownership cost dominates the benefit.
- Sources: ThoughtWorks Tech Radar, *Backstage* (cautions on adoption cost); Fournier, *Platform Engineering*, Ch. 5 ("Build vs Buy vs Adopt"); Roadie, *Should you adopt Backstage?*, https://roadie.io/blog/ .
- Concrete check: estimate platform-team capacity in engineer-weeks; subtract Backstage steady-state cost (plugin upgrades, auth, catalog ingestion). If the remainder cannot fund the actual paved roads, defer Backstage.

---

## 6. ALTERNATIVES

**[ALTERNATIVES] [should] Pick the IDP frontend on a build/buy/adopt axis;
the answer is rarely "build from scratch".**
Honest landscape, late 2025:
- *Backstage* (OSS, self-hosted): max flexibility, highest run cost. Right when you have a team that can own a TypeScript app.
- *Roadie* (managed Backstage): trades flexibility for someone else owning upgrades.
- *Port* / *Cortex* / *OpsLevel*: SaaS portals with strong catalog and scorecards. Faster to value; lower ceiling for custom plugins.
- *Atlassian Compass*: integrates with Jira/Confluence shops; weak outside that ecosystem.
- *Homegrown*: justified only if you have a *very* unusual workflow, almost never if Backstage's primitives fit.
- Sources: ThoughtWorks Tech Radar entries on *Backstage*, *Port*, *Cortex*; CNCF Platforms WG whitepaper landscape; Fournier, Ch. 5.
- Concrete check: a one-page decision document names which IDP frontend was chosen, the two rejected alternatives, and the reason. "We just picked X" is not a reason.

---

## 7. SELF-SERVICE

**[SELF-SERVICE] [must] Self-service infrastructure means the consumer team
declares *what they want* in their own repo and the platform reconciles it —
not "raise a ticket".**
The pattern is the same across the ecosystem: a consumer-facing spec
(Kubernetes CRD, Crossplane Composition, Score workload, AWS Service
Catalog product, Humanitec resource definition) is the API; a controller
reconciles it to actual cloud resources. Toil is paid once at the platform
and amortised across every consumer.
- Sources: Crossplane, *Compositions*, https://docs.crossplane.io/latest/concepts/compositions/ ; Score specification, https://score.dev/docs/ ; Humanitec reference architectures (above); Salatino, *Platform Engineering on Kubernetes*, Ch. 4.
- Concrete check: pick any non-trivial resource (a bucket, a Postgres, a queue). Can a stream-aligned team get one in <30 minutes by editing a file in their own repo? If they have to talk to a human, this is not yet self-service.

**[SELF-SERVICE] [should] Workload specification (Score, Open Application
Model) is the right shape for "describe my app once, deploy it to many
environments".**
Score-style specs separate workload intent ("I need a Postgres, 8GiB, an
HTTP endpoint") from environment-specific resolution ("dev = a Docker
container; prod = RDS"). This is the configuration-as-data sweet spot from
§8 in concrete form.
- Sources: Score docs (above); ThoughtWorks Tech Radar, *Score*, https://www.thoughtworks.com/radar/languages-and-frameworks/score ; CNCF Platforms WG whitepaper, §"Capabilities".
- Concrete check: the same workload spec deploys to at least two environments (local Docker and a real cluster) without modification.

---

## 8. ABSTRACTION LEVEL

**[ABSTRACTION LEVEL] [must] The right level is *configuration as data* over
a small declarative spec — not raw IaC exposed to every team, and not a
fully-opinionated PaaS that hides primitives.**
Too low (every team writes Terraform from scratch): no leverage, infinite
drift, every team re-discovers the same security mistakes. Too high (a
black-box PaaS): teams cannot debug, cannot extend, cannot escape; the
platform becomes a single point of organisational failure. The sweet spot is
a small declarative schema (a CRD, a Score file, a Crossplane Claim) that
the platform compiles into the underlying primitives.
- Sources: Kelsey Hightower, *Configuration as Data*, https://github.com/kelseyhightower/config-as-data ; Crossplane Compositions (above); Salatino, *Platform Engineering on Kubernetes*, Ch. 3–4.
- Concrete check: examine the consumer-facing spec for the platform's main abstraction. Is it < ~30 lines of declarative YAML/JSON for a typical service? Is the compilation to primitives readable? Both yes = good.

---

## 9. TEAM TOPOLOGIES

**[TEAM TOPOLOGIES] [must] Organise infra work around the four team types
and three interaction modes from *Team Topologies*; do not invent a fifth.**
The four types are *stream-aligned* (owns a value stream end-to-end),
*platform* (provides internal services to stream-aligned teams),
*enabling* (helps adopt new capabilities, then steps back), and
*complicated-subsystem* (deep specialism — ML inference, video pipeline).
The three interaction modes are *collaboration* (short-term, high-bandwidth),
*X-as-a-Service* (consumer treats provider as black box), and *facilitating*
(enabling team helps another). Platform work is overwhelmingly
X-as-a-Service; enabling work is short-lived and skill-transfer shaped.
- Sources: Skelton & Pais, *Team Topologies*, Ch. 4 & 7; key concepts, https://teamtopologies.com/key-concepts/ ; Fournier, *Platform Engineering*, Ch. 3.
- Concrete check: every infra-adjacent team is labelled with one of the four types and its dominant interaction mode is documented per consumer. Teams that interact with everyone, in collaboration mode, all the time, are misclassified bottlenecks.

---

## 10. CONWAY

**[CONWAY] [must] The architecture you ship will mirror the communication
structure of the org that built it. Plan accordingly (the Inverse Conway
Maneuver).**
Conway's 1968 paper observed that "any organization that designs a system…
will inevitably produce a design whose structure is a copy of the
organization's communication structure". The inverse maneuver is to
deliberately restructure teams so the desired architecture is the *easy* one
to build.
- Sources: Melvin Conway, *How Do Committees Invent?*, Datamation, April 1968, http://www.melconway.com/Home/Committees_Paper.html ; Skelton & Pais, *Team Topologies*, Ch. 2; ThoughtWorks Tech Radar, *Inverse Conway Maneuver*, https://www.thoughtworks.com/radar/techniques/inverse-conway-maneuver .
- Concrete check: the platform-team boundary matches the API boundary of the platform. If the platform exposes one product but is split across three teams that don't share a backlog, expect the API to fragment to match.

---

## 11. COGNITIVE LOAD

**[COGNITIVE LOAD] [must] The platform exists, fundamentally, to keep the
*intrinsic* cognitive load on stream-aligned teams within human limits.**
Cognitive load (Sweller, educational psychology, 1988) decomposes into
intrinsic (inherent task complexity), extraneous (poor tools, accidental
complexity), and germane (cost of building useful mental models). A
stream-aligned team that must also operate Kubernetes, Terraform, secrets,
CI runners, and observability *plus* its own product domain is overloaded —
the platform's job is to make those shared concerns X-as-a-Service so the
team's load fits.
- Sources: John Sweller, *Cognitive Load During Problem Solving*, Cognitive Science 12(2), 1988, https://onlinelibrary.wiley.com/doi/10.1207/s15516709cog1202_4 ; Skelton & Pais, *Team Topologies*, Ch. 3 ("Team-First Thinking, Cognitive Load"); Fournier, *Platform Engineering*, Ch. 1.
- Concrete check: a stream-aligned team can list, in one breath, the *concepts* it owns (its domain, its service, the platform spec it writes). If the list also includes "Helm internals", "Argo CD reconciler loops", "VPC CIDRs", "IAM trust policies", the platform is leaking.

---

## 12. ANTI-PATTERNS

**[ANTI-PATTERNS] [avoid] Ivory-tower platform — design without consumer
contact.**
Replacement: weekly office hours, an "embedded engineer" rotation, or
shadowing one stream-aligned team per quarter.
- Sources: Fournier, *Platform Engineering*, Ch. 2; Skelton & Pais, *Team Topologies*, Ch. 5; Charity Majors, https://charity.wtf .
- Concrete check: count the platform-team commits in a quarter whose linked issue cites a stream-aligned consumer. <50% is a flag.

**[ANTI-PATTERNS] [avoid] Mandatory paved road. The moment it is mandated, it
stops being a road and becomes a queue.**
Replacement: make the paved road radically more ergonomic than off-road.
Compete on quality, not policy.
- Sources: Spotify "Golden Paths" post; Skelton & Pais, *Team Topologies*, Ch. 5; ThoughtWorks Tech Radar, *Platform engineering product teams*.
- Concrete check: at least one production team uses a documented and exercised off-ramp away from the golden path.

**[ANTI-PATTERNS] [avoid] "We know what they need" — building before
validating.**
Replacement: an MVP scoped to one consumer team that has signed up to adopt
within a fixed window. Kill the project if adoption does not happen.
- Sources: Larson, *An Elegant Puzzle*, §"Migrations" and §"Internal tools"; Fournier, *Platform Engineering*, Ch. 2.
- Concrete check: every shipped feature has a named first adopter who used it within 30 days of release.

**[ANTI-PATTERNS] [avoid] Treating the platform team as an SRE-ticket queue.**
Replacement: SRE work has its own home; platform work is *productised*
shared services. The two are different shapes; do not collapse them.
- Sources: Charity Majors, *charity.wtf* (search "platform engineering"); Google SRE book, Ch. 1, https://sre.google/sre-book/introduction/ ; Skelton & Pais, *Team Topologies*, Ch. 4.
- Concrete check: the platform team's tracker has <~20% break-fix tickets; the rest is roadmap product work.

---

## 13. METRICS

**[METRICS] [must] Measure platform success by the DORA metrics of the
*stream-aligned teams that consume it* plus an adoption metric of platform
features. Do not measure by tickets closed or projects shipped by the
platform team.**
Deploy frequency, lead time for changes, change failure rate, mean time to
restore — the empirical measure of whether the delivery system (which the
platform is part of) is working. Platform teams that improve those for their
consumers are doing their job; platform teams that ship a lot of features no
one adopts are not.
- Sources: Forsgren, Humble & Kim, *Accelerate* (IT Revolution, 2018), Ch. 2; DORA research program, https://dora.dev/ ; Fournier, *Platform Engineering*, Ch. 6; CNCF Platforms WG whitepaper, §"Measuring success".
- Concrete check: the platform team publishes, at least quarterly, (a) the four DORA metrics for top consumer teams, (b) adoption rate per shipped feature, (c) a satisfaction score (NPS-style survey). Absence is itself a finding.

---

## 14. TOOLING

**[TOOLING] [should] Build the platform from established CNCF/OSS building
blocks; integrate, don't reinvent.**
Reference stack, late 2025: *Backstage* (catalog/portal), *Crossplane* or
*Terraform + a controller* (resource provisioning), *Argo CD* / *Flux*
(GitOps deploy, ch07), *Score* or *Humanitec* (workload abstraction),
*OpenTelemetry* (observability seam), *cert-manager*, *external-secrets*.
Each is independently useful and deliberately composable.
- Sources: CNCF Platforms WG whitepaper, *Capabilities and tools*; CNCF landscape, https://landscape.cncf.io/ ; ThoughtWorks Tech Radar entries on each project; Salatino, *Platform Engineering on Kubernetes*, Ch. 6–10.
- Concrete check: the architecture diagram names the OSS project behind each capability and shows where each customisation was added. Diagrams that show only product names hide lock-in.

---

## 15. READINESS

**[READINESS] [must] Do not invest in a platform team before the
organisation's complexity justifies it. Rule of thumb: ~30+ engineers or
~10+ independent services.**
Below that threshold the cost of a dedicated platform team (people, runtime,
adoption friction) exceeds the avoided duplication. A "platform of one
README, one shared Helm chart, one CI template" is the right answer for
small orgs and is itself a form of platform.
- Sources: Larson, *An Elegant Puzzle*, §"Sizing engineering teams"; Fournier, *Platform Engineering*, Ch. 1 ("When you don't need a platform team"); Skelton & Pais, *Team Topologies*, Ch. 5.
- Concrete check: before chartering a platform team, write the "what duplication is this avoiding?" memo. If you cannot point at three stream-aligned teams suffering the same pain, defer.

---

## 16. DOCS

**[DOCS] [must] Documentation is a first-class platform feature: colocated
with the code, searchable, versioned. "If it isn't searchable, it doesn't
exist."**
Backstage TechDocs is the canonical implementation: Markdown lives next to
code in each repo, MkDocs renders it, the portal aggregates. ADRs (Nygard,
2011) capture *why* a decision was made and are themselves discoverable
docs.
- Sources: Backstage TechDocs, https://backstage.io/docs/features/techdocs/ ; Michael Nygard, *Documenting Architecture Decisions*, https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions ; Write the Docs, *Docs as Code*, https://www.writethedocs.org/guide/docs-as-code/ ; Fournier, *Platform Engineering*, Ch. 4.
- Concrete check: every platform capability has a TechDocs entry reachable in ≤2 clicks from the catalog; every load-bearing decision has an ADR in the platform repo. Search "deploy a service" in the portal — if the answer is not in the first three results, docs are failing.

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
- Michael Nygard, *Documenting Architecture Decisions*, 2011 — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions

### Standards & working groups

- CNCF Platforms Working Group, *Platforms White Paper* — https://tag-app-delivery.cncf.io/whitepapers/platforms/
- CNCF TAG App Delivery — https://github.com/cncf/tag-app-delivery
- DORA research program — https://dora.dev/
- Score workload specification — https://score.dev/docs/

### Vendor / project documentation (canonical statement, not endorsement)

- Backstage — https://backstage.io/docs/overview/what-is-backstage
- Backstage TechDocs — https://backstage.io/docs/features/techdocs/
- Crossplane Compositions — https://docs.crossplane.io/latest/concepts/compositions/
- Humanitec reference architectures — https://humanitec.com/blog/reference-architectures-for-internal-developer-platforms
- Roadie (managed Backstage) — https://roadie.io/
- Argo CD — https://argo-cd.readthedocs.io/
- Flux — https://fluxcd.io/

### Named industry voices

- Pais & Skelton, Team Topologies key concepts — https://teamtopologies.com/key-concepts/
- Charity Majors, *charity.wtf* — https://charity.wtf
- Kelsey Hightower, *Configuration as Data* — https://github.com/kelseyhightower/config-as-data
- Spotify Engineering, *How We Use Golden Paths* — https://engineering.atspotify.com/2020/08/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/

### Industry analysis

- ThoughtWorks Tech Radar, *Platform engineering product teams* — https://www.thoughtworks.com/radar/techniques/platform-engineering-product-teams
- ThoughtWorks Tech Radar, *Backstage* — https://www.thoughtworks.com/radar/platforms/backstage
- ThoughtWorks Tech Radar, *Inverse Conway Maneuver* — https://www.thoughtworks.com/radar/techniques/inverse-conway-maneuver
- ThoughtWorks Tech Radar, *Score* — https://www.thoughtworks.com/radar/languages-and-frameworks/score
- internaldeveloperplatform.org community — https://internaldeveloperplatform.org/

---
