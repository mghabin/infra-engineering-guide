# Glossary

This glossary defines the terms used across the Infrastructure Engineering Guide. Definitions are opinionated: where the industry uses a term inconsistently, the entry calls out the disambiguation and pins this guide to a specific primary source. Cross-referenced glossary terms are *italicized*. Citations point at primary sources (specs, RFCs, project docs, original papers, named originators) — not at blog summaries.

**Industry-contested terms with strong opinions in this guide** (read these entries carefully): *GitOps*, *Service Mesh*, *Platform Engineering*, *SRE*, *DevOps*, *Infrastructure as Code*, *Observability*, *Three Pillars*, *Exactly-Once*, *Multi-Cloud*, *Shift-Left*, *Blameless Postmortem*, *Cattle vs Pets*, *RPO* / *RTO*, *MTBF*.

**Deprecated or legacy** (do not use in new work): *Puppet Master*, *Cortex*, *AAD Pod Identity*, *dockershim*, *PodSecurityPolicy*, *Spot Blocks*, *Classic Branch Protection*, *Classic PATs*, *DynamoDB Terraform State Locking*, *Three Pillars of Observability* (as a design framework).

## A

### ACME (Automatic Certificate Management Environment)
IETF protocol for automated issuance and renewal of X.509 certificates. Underpins Let's Encrypt and most public CAs; *cert-manager* speaks ACME for in-cluster TLS.
- **Used in**: ch07 §6
- **Sources**: RFC 8555 — https://www.rfc-editor.org/rfc/rfc8555 ; Let's Encrypt docs — https://letsencrypt.org/docs/

### ADR (Architecture Decision Record)
Short, dated, immutable document capturing a single architectural decision, its context, and its consequences. New ADRs supersede prior ones; you never edit history.
- **Used in**: ch11 §4
- **Sources**: Michael Nygard, *Documenting Architecture Decisions* (2011) — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions

### Allocation (FinOps)
Mapping cloud cost to the business unit, product, team, or service that incurred it, via tags, labels, hierarchy, or *Kubernetes* namespace mapping. Precondition for *showback* and *chargeback*.
- **Used in**: ch10 §3
- **Sources**: FinOps Framework — *Allocation* capability — https://www.finops.org/framework/capabilities/allocation/ ; FOCUS spec — https://focus.finops.org/

### Anycast
Routing technique where the same IP prefix is advertised from multiple locations and BGP delivers traffic to the topologically nearest one. Used by public DNS, *CDN* edges, and global load balancers.
- **Used in**: ch07 §5
- **Sources**: RFC 4786 — https://www.rfc-editor.org/rfc/rfc4786 ; Cloudflare *How Anycast Works* — https://www.cloudflare.com/learning/cdn/glossary/anycast-network/

### Argo CD
Kubernetes-native *GitOps* controller that reconciles a cluster against manifests in a git repository. Pull-based; supports app-of-apps and ApplicationSets.
- **Used in**: ch03 §6, ch04
- **Sources**: Argo CD docs — https://argo-cd.readthedocs.io/

### Argo Rollouts
Progressive-delivery controller for *Kubernetes* implementing *canary*, *blue/green*, and analysis-driven promotions on top of *Deployment* semantics.
- **Used in**: ch03 §5
- **Sources**: Argo Rollouts docs — https://argoproj.github.io/argo-rollouts/

### Atlantis
Self-hosted *Terraform* / *OpenTofu* PR automation that runs `plan` on PRs and `apply` on merge with locking per workspace.
- **Used in**: ch02 §4
- **Sources**: Atlantis docs — https://www.runatlantis.io/docs/

### Attestation (in-toto)
Signed statement describing a step in a software supply chain (build, test, scan), expressed as an *in-toto* Statement wrapped in a *DSSE* envelope.
- **Used in**: ch03 §3, ch06 §4
- **Sources**: in-toto Attestation Spec — https://github.com/in-toto/attestation ; SLSA Provenance — https://slsa.dev/spec/v1.0/provenance

### Audit Log
Append-only record of who did what, when, against which resource. Distinct from application logs; required for compliance and incident forensics.
- **Used in**: ch05 §6, ch06 §5
- **Sources**: Kubernetes Auditing — https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/ ; AWS CloudTrail concepts — https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-concepts.html

### Autovacuum
PostgreSQL background process that reclaims dead-tuple space and updates planner statistics. Tuning it (per-table thresholds, cost limits) is a standard hot-path for high-write databases.
- **Used in**: ch08 §6
- **Sources**: PostgreSQL docs — *Routine Vacuuming* — https://www.postgresql.org/docs/current/routine-vacuuming.html

## B

### Backstage
Open-source *developer portal* framework from Spotify. A portal — not a *platform* — that catalogs services (*Software Catalog*), publishes docs (*TechDocs*), and runs scaffolders (*Software Templates*).
- **Used in**: ch11 §3
- **Sources**: Backstage docs — https://backstage.io/docs/overview/what-is-backstage ; Spotify Engineering — *Backstage announcement* — https://engineering.atspotify.com/2020/03/16/announcing-backstage/

### Baggage (W3C)
Header for propagating user-defined key/value pairs across service boundaries alongside *W3C Trace Context*. Use sparingly: every hop pays for it.
- **Used in**: ch05 §2
- **Sources**: W3C Baggage — https://www.w3.org/TR/baggage/

### BCP 38
IETF best current practice mandating ingress filtering of spoofed source addresses at network edges. Foundational anti-spoofing hygiene for any AS.
- **Used in**: ch07 §5
- **Sources**: BCP 38 / RFC 2827 — https://www.rfc-editor.org/info/bcp38

### BFD (Bidirectional Forwarding Detection)
Sub-second link-failure detection protocol used with *BGP* and other routing protocols to fail over before the routing protocol's own timers fire.
- **Used in**: ch07 §5
- **Sources**: RFC 5880 — https://www.rfc-editor.org/rfc/rfc5880

### BGP (Border Gateway Protocol)
The path-vector protocol that glues the internet together. *eBGP* peers between ASes; *iBGP* within an AS. Used in cloud networking for *Direct Connect* / *ExpressRoute* and for *anycast*.
- **Used in**: ch07 §5
- **Sources**: RFC 4271 — https://www.rfc-editor.org/rfc/rfc4271

### Binlog
MySQL/MariaDB binary log of row or statement changes; the basis for replication, *PITR*, and *CDC* (e.g., *Debezium*).
- **Used in**: ch08 §2
- **Sources**: MySQL docs — *The Binary Log* — https://dev.mysql.com/doc/refman/8.0/en/binary-log.html

### Blameless Postmortem
Incident review that focuses on the system and decision-making conditions, not on punishing individuals. **Disambiguation**: blameless ≠ accountability-free; this guide follows Allspaw and Dekker — engineers describe what they saw and decided, *and* the org owns the systemic fix.
- **Used in**: ch09 §5
- **Sources**: John Allspaw, *Blameless PostMortems and a Just Culture* (Etsy, 2012) — https://www.etsy.com/codeascraft/blameless-postmortems ; Sidney Dekker, *The Field Guide to Understanding 'Human Error'* (3rd ed.)

### Blast Radius
The set of resources, tenants, or users affected when a change goes wrong. Designs minimize it via cell architecture, account/project boundaries, and small *stacks*.
- **Used in**: ch01 §6, ch04, ch07 §1
- **Sources**: AWS Builders' Library — *Static stability and Availability Zones* — https://aws.amazon.com/builders-library/static-stability-using-availability-zones/

### Blue/Green Deployment
Deployment strategy with two production environments where the inactive one is upgraded and validated before traffic flips. Cheap rollback, expensive double capacity.
- **Used in**: ch03 §5
- **Sources**: Martin Fowler — *BlueGreenDeployment* — https://martinfowler.com/bliki/BlueGreenDeployment.html

### Bottlerocket
AWS-maintained, container-optimized Linux distribution with an immutable root filesystem and API-driven configuration.
- **Used in**: ch04
- **Sources**: Bottlerocket docs — https://bottlerocket.dev/

### BuildKit
Modern Docker/Moby build engine with cache mounts, secrets, and frontend plugins. Required for reproducible and *hermetic* builds.
- **Used in**: ch03 §4, ch04
- **Sources**: BuildKit docs — https://docs.docker.com/build/buildkit/

### Bulkhead
Resilience pattern that isolates resources (thread pools, connection pools, queues) so a failure in one workload cannot drain capacity from others.
- **Used in**: ch09 §3
- **Sources**: Michael Nygard, *Release It!* (2nd ed.), ch. on Stability Patterns

### Burn Rate
Rate at which an *error budget* is being consumed relative to its *SLO* window. Used in *multi-window multi-burn-rate* alerting.
- **Used in**: ch09 §2
- **Sources**: Google SRE Workbook, ch. *Alerting on SLOs* — https://sre.google/workbook/alerting-on-slos/

## C

### CAP Theorem
Brewer's result that a distributed data store cannot simultaneously provide *Consistency*, *Availability*, and *Partition tolerance*; partitions are a fact, so the real choice is C-vs-A under partition.
- **Used in**: ch08 §1
- **Sources**: Eric Brewer, *Towards Robust Distributed Systems* (PODC 2000 keynote) ; Brewer, *CAP Twelve Years Later* (IEEE Computer, 2012) — https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/

### Canary Deployment
Progressive rollout that sends a small fraction of traffic to a new version while measuring health signals; promote on success, roll back on regression.
- **Used in**: ch03 §5
- **Sources**: Martin Fowler — *CanaryRelease* — https://martinfowler.com/bliki/CanaryRelease.html

### Cardinality
Number of distinct label/tag combinations a metric can take. High cardinality kills *Prometheus*-class TSDBs; *wide events* address the same need without exploding cardinality.
- **Used in**: ch05 §3
- **Sources**: Prometheus docs — *Naming and labels* — https://prometheus.io/docs/practices/naming/ ; Honeycomb — *High cardinality* — https://www.honeycomb.io/blog/observability-cardinality

### Cattle vs Pets
Operations metaphor: pets are individually named and nursed back to health; cattle are interchangeable and replaced on failure. **Disambiguation**: applies to *operational treatment*, not "stateless only" — a sharded database fleet can be cattle.
- **Used in**: ch01 §3
- **Sources**: Bill Baker, *Scaling SQL Server 2012* (origin of the metaphor); Randy Bias popularized it — see Randy Bias, *Pets vs Cattle* — https://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/

### CDC (Change Data Capture)
Pattern that streams row-level changes out of a database (commonly via *binlog* / *WAL*) into a log or topic for downstream consumers. *Debezium* is the canonical implementation.
- **Used in**: ch08 §4
- **Sources**: Debezium docs — *What is CDC?* — https://debezium.io/documentation/reference/stable/architecture.html

### CDK (Cloud Development Kit)
AWS framework for synthesizing CloudFormation from imperative code (TypeScript, Python, etc.). Counts as *IaC* when used declaratively against a reconciler.
- **Used in**: ch02 §1
- **Sources**: AWS CDK docs — https://docs.aws.amazon.com/cdk/v2/guide/

### Cell Architecture
Design that partitions a system into independent, identically-shaped cells so failures and load are bounded to one cell. The base unit of *blast-radius* control at scale.
- **Used in**: ch04, ch07 §1
- **Sources**: AWS Builders' Library — *Reducing the Scope of Impact with Cell-based Architecture* — https://aws.amazon.com/builders-library/reducing-the-scope-of-impact-with-cell-based-architecture/

### cert-manager
Kubernetes controller that issues and renews X.509 certificates from *ACME*, Vault, and private CAs.
- **Used in**: ch07 §6
- **Sources**: cert-manager docs — https://cert-manager.io/docs/

### CFS Throttling
Linux Completely Fair Scheduler behavior where a cgroup that exceeds its CPU quota in a 100 ms period is paused until the next period. The reason *Kubernetes* CPU *limits* often hurt latency.
- **Used in**: ch04
- **Sources**: kernel docs — *CFS Bandwidth Control* — https://www.kernel.org/doc/html/latest/scheduler/sched-bwc.html ; Tim Hockin et al. — Kubernetes issue #67577

### Chaos Engineering
Discipline of running controlled failure experiments in production-like environments to surface latent weakness before customers do.
- **Used in**: ch09 §4
- **Sources**: *Principles of Chaos Engineering* — https://principlesofchaos.org/ ; Casey Rosenthal & Nora Jones, *Chaos Engineering* (O'Reilly, 2020)

### Chargeback
Billing internal teams for the cloud cost they consumed. Stronger accountability than *showback*; harder to operate because allocation must be auditable.
- **Used in**: ch10 §3
- **Sources**: FinOps Framework — *Chargeback & Showback* — https://www.finops.org/framework/capabilities/chargeback-showback/

### CIDR
Classless Inter-Domain Routing notation for IP prefixes (e.g., `10.0.0.0/16`). Foundation of all VPC/subnet design.
- **Used in**: ch07 §2
- **Sources**: RFC 4632 — https://www.rfc-editor.org/rfc/rfc4632

### CIS Benchmarks
Configuration baselines for OSes, *Kubernetes*, and cloud accounts maintained by the Center for Internet Security. *kube-bench* and *cloud-bench* check against them.
- **Used in**: ch04, ch06 §3
- **Sources**: CIS Benchmarks — https://www.cisecurity.org/cis-benchmarks

### Circuit Breaker
Resilience pattern that stops calling a failing dependency after an error threshold and periodically probes for recovery, preventing cascading failure.
- **Used in**: ch09 §3
- **Sources**: Martin Fowler — *CircuitBreaker* — https://martinfowler.com/bliki/CircuitBreaker.html ; Michael Nygard, *Release It!*

### Cloud Run / Container Apps / App Runner / Fargate
Managed serverless container runtimes from GCP, Azure, AWS, and AWS again respectively. Use when you don't need a *Kubernetes* control plane.
- **Used in**: ch04
- **Sources**: Cloud Run docs — https://cloud.google.com/run/docs ; Azure Container Apps docs — https://learn.microsoft.com/azure/container-apps/ ; AWS App Runner docs — https://docs.aws.amazon.com/apprunner/ ; AWS Fargate docs — https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html

### Cluster API
Kubernetes-native API for declaratively managing *Kubernetes* clusters across providers.
- **Used in**: ch04
- **Sources**: Cluster API book — https://cluster-api.sigs.k8s.io/

### Cluster Autoscaler
Adds or removes nodes when pending pods cannot schedule or nodes are underused. Older and slower than *Karpenter* on AWS, still standard on GKE/AKS.
- **Used in**: ch04
- **Sources**: Kubernetes Autoscaler — https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler

### CNI (Container Network Interface)
Spec and plugin model for pod networking on *Kubernetes*. Choices include AWS VPC CNI, Azure CNI, Calico, Cilium, GKE Dataplane v2.
- **Used in**: ch04
- **Sources**: CNI spec — https://github.com/containernetworking/cni/blob/main/SPEC.md

### CockroachDB
Distributed SQL database using Raft for replication, PostgreSQL wire protocol, and serializable isolation. See also *Spanner*, *YugabyteDB*.
- **Used in**: ch08 §1
- **Sources**: CockroachDB docs — https://www.cockroachlabs.com/docs/

### Cognitive Load
Total mental effort a team must hold to do its work; in *Team Topologies*, you size team scope to keep extraneous load low. Originates with John Sweller (intrinsic / extraneous / germane).
- **Used in**: ch11 §2
- **Sources**: John Sweller, *Cognitive Load During Problem Solving* (1988); Skelton & Pais, *Team Topologies* (2019)

### Collector (OpenTelemetry)
Vendor-neutral process that receives, processes, and exports telemetry. Deploy as agent, gateway, or sidecar.
- **Used in**: ch05 §2
- **Sources**: OpenTelemetry Collector docs — https://opentelemetry.io/docs/collector/

### Configuration as Data
Approach to platform UX where workload intent is expressed in a small declarative schema and a controller renders it into runtime resources. *Kelsey Hightower*'s framing.
- **Used in**: ch11 §3
- **Sources**: Kelsey Hightower, *Configuration as Data* (KubeCon NA 2019 keynote / GitHub gist) — https://github.com/kelseyhightower/config

### Conftest
CLI that runs *Rego* policies (*OPA*) against arbitrary structured config (Kubernetes YAML, *Terraform* plans, Dockerfiles).
- **Used in**: ch02 §5, ch06 §3
- **Sources**: Conftest docs — https://www.conftest.dev/

### Conway's Law
"Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure." Drives the *Inverse Conway Maneuver*.
- **Used in**: ch11 §2
- **Sources**: Melvin Conway, *How Do Committees Invent?* (Datamation, 1968) — http://www.melconway.com/Home/Committees_Paper.html

### Cortex
Older horizontally-scalable Prometheus backend. **Deprecated** in this guide; superseded by *Mimir* (a Grafana fork that absorbed most active development).
- **Used in**: ch05 §4
- **Sources**: Grafana Mimir announcement — https://grafana.com/blog/2022/03/30/announcing-grafana-mimir/

### cosign
*Sigstore* CLI for signing and verifying OCI artifacts and *attestations*. Supports keyless signing via *OIDC* and *Fulcio*.
- **Used in**: ch03 §3, ch06 §4
- **Sources**: cosign docs — https://docs.sigstore.dev/cosign/

### CQRS (Command Query Responsibility Segregation)
Pattern that separates write and read models, often paired with *event sourcing* or read-side projections.
- **Used in**: ch08 §4
- **Sources**: Martin Fowler — *CQRS* — https://martinfowler.com/bliki/CQRS.html

### CRD (Custom Resource Definition)
Mechanism for extending the *Kubernetes* API with new resource types served from etcd and reconciled by *operators*.
- **Used in**: ch04
- **Sources**: Kubernetes docs — *Custom Resources* — https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/

### Crossplane
Kubernetes-native IaC controller that manages cloud resources via *CRDs* and exposes platform-curated abstractions through *Compositions*.
- **Used in**: ch02 §1, ch11 §3
- **Sources**: Crossplane docs — https://docs.crossplane.io/

### Crypto-Shredding
Erasure technique where you destroy the encryption key instead of overwriting ciphertext; sufficient for many "right to erasure" obligations and for cold-storage media.
- **Used in**: ch06 §6, ch08 §5
- **Sources**: NIST SP 800-88 Rev. 1, *Guidelines for Media Sanitization* — https://csrc.nist.gov/publications/detail/sp/800-88/rev-1/final

### CSI (Container Storage Interface)
Spec and plugin model for storage on *Kubernetes*. *StatefulSets* + CSI is the standard pattern for stateful workloads.
- **Used in**: ch04, ch08 §6
- **Sources**: CSI spec — https://github.com/container-storage-interface/spec

### CUD (Committed Use Discount)
GCP commitment construct (resource-based or flexible) that trades a 1- or 3-year spend pledge for a discount. Analogous to AWS *Savings Plans* and Azure Reservations.
- **Used in**: ch10 §2
- **Sources**: Google Cloud — *Committed use discounts* — https://cloud.google.com/docs/cuds

### CVE / CVSS / KEV / EPSS
Vulnerability identifiers (*CVE*) and scoring/triage signals: *CVSS* base score, *KEV* (CISA's exploited list), *EPSS* (exploit-probability score). Use *KEV* and *EPSS* over CVSS alone for prioritization.
- **Used in**: ch06 §4
- **Sources**: CISA KEV — https://www.cisa.gov/known-exploited-vulnerabilities-catalog ; FIRST EPSS — https://www.first.org/epss/ ; FIRST CVSS — https://www.first.org/cvss/

## D

### Dataplane v2 (GKE)
GKE's eBPF-based dataplane built on *Cilium*, replacing iptables-based kube-proxy.
- **Used in**: ch04
- **Sources**: GKE docs — *Dataplane V2* — https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2

### Dependabot / Renovate
Automated dependency-update bots. *Renovate* is more configurable; *Dependabot* is bundled with GitHub.
- **Used in**: ch03 §7
- **Sources**: Dependabot docs — https://docs.github.com/en/code-security/dependabot ; Renovate docs — https://docs.renovatebot.com/

### Dependency Confusion
Supply-chain attack where a malicious public package shadows an internal one because the resolver prefers the higher version. Discovered and named by Alex Birsan.
- **Used in**: ch06 §4
- **Sources**: Alex Birsan, *Dependency Confusion* (2021) — https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610 (originator-of-record cite per CONTRIBUTING.md exception)

### DevOps
Cultural movement aimed at collapsing the dev/ops wall via shared ownership, automation, and feedback. **Disambiguation**: this guide treats DevOps as a *culture*; *SRE* is a *role/practice* applying it; *Platform Engineering* is a *team shape* enabling it.
- **Used in**: ch01 §1, ch11 §1
- **Sources**: Nicole Forsgren, Jez Humble, Gene Kim, *Accelerate* (2018); Patrick Debois (origin of the term, *DevOps Days* Ghent, 2009)

### Direct Connect / ExpressRoute / Cloud Interconnect
Dedicated private circuits from on-prem to AWS / Azure / GCP. Use for predictable bandwidth and low jitter; transport via *BGP*.
- **Used in**: ch07 §4
- **Sources**: AWS Direct Connect docs — https://docs.aws.amazon.com/directconnect/ ; Azure ExpressRoute docs — https://learn.microsoft.com/azure/expressroute/ ; Google Cloud Interconnect docs — https://cloud.google.com/network-connectivity/docs/interconnect

### Distroless
Container base images containing only the application and its runtime, no shell or package manager. Smaller attack surface, harder to debug — pair with *ephemeral container* for debugging.
- **Used in**: ch04, ch06 §3
- **Sources**: Google distroless — https://github.com/GoogleContainerTools/distroless

### dockershim
Removed shim that let *kubelet* talk to Docker. **Deprecated**: removed in Kubernetes 1.24; use *containerd* or CRI-O.
- **Used in**: ch04
- **Sources**: Kubernetes blog — *Don't Panic: Kubernetes and Docker* — https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/

### DORA Four Keys
Deployment Frequency, Lead Time for Changes, Change Failure Rate, and Mean Time to Restore — research-validated delivery performance metrics. The 2023 report added a fifth (Reliability).
- **Used in**: ch03 §1, ch11 §4
- **Sources**: Forsgren, Humble, Kim, *Accelerate* (2018); Google Cloud DORA — https://dora.dev/

### Drift
Divergence between declared *IaC* desired state and actual cloud state. Closing it requires *reconciliation*; *GitOps* makes the loop continuous.
- **Used in**: ch01 §4, ch02 §3
- **Sources**: HashiCorp — *Drift detection* — https://developer.hashicorp.com/terraform/cloud-docs/workspaces/health

### DSSE (Dead Simple Signing Envelope)
Signing envelope format used by *Sigstore* and *in-toto* attestations.
- **Used in**: ch03 §3, ch06 §4
- **Sources**: DSSE spec — https://github.com/secure-systems-lab/dsse

### Dual Writes
Anti-pattern where an app writes to two systems (DB + queue) without atomicity. Use a *transactional outbox* or *CDC* instead.
- **Used in**: ch08 §4
- **Sources**: Pat Helland, *Life Beyond Distributed Transactions* (CIDR 2007) — https://www.cidrdb.org/cidr2007/papers/cidr07p15.pdf ; Confluent — *Dual writes* — https://www.confluent.io/blog/dual-write-problem/

## E

### eBPF
In-kernel programmable VM enabling safe attach points for tracing, networking, and security (*Cilium*, *Hubble*, *Falco*, *Tetragon*, *Pixie*, *Parca*).
- **Used in**: ch04, ch05 §5, ch07 §7
- **Sources**: ebpf.io — https://ebpf.io/

### ECH (Encrypted Client Hello)
TLS extension that encrypts the SNI so the destination hostname isn't visible to on-path observers.
- **Used in**: ch07 §5
- **Sources**: IETF draft — *TLS Encrypted Client Hello* — https://datatracker.ietf.org/doc/draft-ietf-tls-esni/

### Effectively-Once
What "*exactly-once*" really means in distributed systems: at-least-once delivery plus *idempotency keys* or transactional dedup. The honest term to use in design docs.
- **Used in**: ch08 §4
- **Sources**: Tyler Akidau, *Streaming 101 / 102* — https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/ ; Pat Helland, *Idempotence Is Not a Medical Condition* (ACM Queue, 2012) — https://queue.acm.org/detail.cfm?id=2187821

### Ephemeral Container
Kubernetes container injected into a running pod for debugging; doesn't restart, has no resources.
- **Used in**: ch04
- **Sources**: Kubernetes docs — *Ephemeral Containers* — https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/

### Ephemeral Environment
Short-lived, per-PR or per-feature environment provisioned from *IaC*. Cheap, isolated, garbage-collected.
- **Used in**: ch02 §3, ch03 §6
- **Sources**: HashiCorp — *Workspaces* — https://developer.hashicorp.com/terraform/cloud-docs/workspaces

### EPSS (Exploit Prediction Scoring System)
Probability that a given *CVE* will be exploited in the wild within 30 days. Use alongside *KEV* for triage.
- **Used in**: ch06 §4
- **Sources**: FIRST EPSS — https://www.first.org/epss/

### Error Budget
The amount of unreliability allowed by an *SLO* over a window — `1 − SLO`. Spent by incidents and risky changes; informs the *error-budget policy*.
- **Used in**: ch09 §2
- **Sources**: Google SRE Book, ch. *Embracing Risk* — https://sre.google/sre-book/embracing-risk/

### Error-Budget Policy
Pre-agreed actions a team takes when *error budget* is exhausted (e.g., freeze feature work, focus on reliability).
- **Used in**: ch09 §2
- **Sources**: Google SRE Workbook — *Implementing SLOs* — https://sre.google/workbook/implementing-slos/

### ESO (External Secrets Operator)
Kubernetes operator that syncs secrets from external stores (*Vault*, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) into *Secrets*.
- **Used in**: ch06 §2
- **Sources**: ESO docs — https://external-secrets.io/

### etcd
Strongly-consistent distributed KV store using Raft; the *Kubernetes* control-plane datastore.
- **Used in**: ch04
- **Sources**: etcd docs — https://etcd.io/docs/

### Event Sourcing
Persistence pattern where state is derived from an append-only log of domain events.
- **Used in**: ch08 §4
- **Sources**: Martin Fowler — *Event Sourcing* — https://martinfowler.com/eaaDev/EventSourcing.html

### Exactly-Once
Marketing term for message delivery. **Disambiguation**: with independent failures you cannot have it; design for *effectively-once* via *idempotency keys* or transactions.
- **Used in**: ch08 §4
- **Sources**: Mathias Verraes / Pat Helland (above); Kafka docs — *Exactly Once Semantics* — https://kafka.apache.org/documentation/#semantics (qualified by transactional scope)

### Exemplar
A specific trace ID attached to a metric data point so you can jump from a bad histogram bucket to a representative trace.
- **Used in**: ch05 §3
- **Sources**: Prometheus — *Exemplars* — https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage ; OpenMetrics spec — https://github.com/OpenObservability/OpenMetrics

### Expand and Contract (Parallel Change)
Schema-change pattern: add new shape additively, dual-write/read, migrate clients, then remove the old shape. Pairs with *BACKWARD*-compatible schema evolution.
- **Used in**: ch08 §3
- **Sources**: Danilo Sato — *ParallelChange* — https://martinfowler.com/bliki/ParallelChange.html

## F

### Falco
Runtime-security tool that detects suspicious syscalls via *eBPF*.
- **Used in**: ch04, ch06 §3
- **Sources**: Falco docs — https://falco.org/docs/

### Feature Flag
Runtime toggle separating deploy from release. Enables *canary*, kill-switches, and per-tenant rollout. Use a managed service or *OpenFeature*-conformant SDK.
- **Used in**: ch03 §5
- **Sources**: Pete Hodgson — *Feature Toggles* — https://martinfowler.com/articles/feature-toggles.html ; OpenFeature — https://openfeature.dev/

### FinOps
Cultural and operational practice for cloud financial management; iterates Inform → Optimize → Operate phases.
- **Used in**: ch10 §1
- **Sources**: FinOps Foundation — *Framework* — https://www.finops.org/framework/ ; J.R. Storment & Mike Fuller, *Cloud FinOps* (2nd ed., O'Reilly)

### FIPS 140-2 / 140-3
US/Canadian cryptographic-module validation programs. Required for federal workloads; influences HSM and library choices.
- **Used in**: ch06 §6
- **Sources**: NIST CMVP — https://csrc.nist.gov/projects/cryptographic-module-validation-program

### Flagger
Progressive-delivery operator originally from Weaveworks; alternative to *Argo Rollouts*, integrates with multiple service meshes.
- **Used in**: ch03 §5
- **Sources**: Flagger docs — https://docs.flagger.app/

### Flux
CNCF *GitOps* toolkit (Kustomize/Helm controllers, image automation). See also *Argo CD*.
- **Used in**: ch03 §6
- **Sources**: Flux docs — https://fluxcd.io/flux/

### FOCUS (FinOps Open Cost & Usage Specification)
Open spec for normalized cost/usage data across providers. Lets you write one cost model for AWS, Azure, GCP, Oracle, etc.
- **Used in**: ch10 §3
- **Sources**: FOCUS spec — https://focus.finops.org/

### Four Golden Signals
Latency, traffic, errors, saturation — Google SRE's recommended starting metrics for any user-facing service.
- **Used in**: ch05 §3
- **Sources**: Google SRE Book, ch. *Monitoring Distributed Systems* — https://sre.google/sre-book/monitoring-distributed-systems/

### Fulcio / Rekor
*Sigstore* components: Fulcio is the keyless code-signing CA; Rekor is the transparency log.
- **Used in**: ch03 §3
- **Sources**: Sigstore docs — https://docs.sigstore.dev/

## G

### Gateway API
Kubernetes successor to *Ingress* (GatewayClass / Gateway / HTTPRoute / TLSRoute / GRPCRoute). Role-oriented and extensible.
- **Used in**: ch04
- **Sources**: Gateway API docs — https://gateway-api.sigs.k8s.io/

### GitOps
Operating model where the desired state of a system lives in git and an in-cluster *reconciliation loop* continuously pulls and applies it. **Disambiguation**: this guide adopts the OpenGitOps v1.0 definition — declarative, versioned & immutable, pulled automatically, continuously reconciled. "We use git for our YAML" alone does **not** qualify.
- **Used in**: ch01 §5, ch03 §6, ch04, ch11
- **Sources**: OpenGitOps v1.0.0 (CNCF GitOps WG) — https://opengitops.dev/ ; Flux — *Core Concepts* — https://fluxcd.io/flux/concepts/

### Goodhart's Law
"When a measure becomes a target, it ceases to be a good measure." Why naive *DORA*/SLO targeting backfires.
- **Used in**: ch09 §6, ch11 §4
- **Sources**: Marilyn Strathern's formulation (1997) of Charles Goodhart's 1975 paper — https://en.wikipedia.org/wiki/Goodhart%27s_law (used as pointer; primary cite: Goodhart, *Monetary Relationships: A View from Threadneedle Street*, 1975)

### Graceful Degradation
Designing a system so that under stress it sheds non-essential functionality first (e.g., disable recommendations, keep checkout).
- **Used in**: ch09 §3
- **Sources**: Google SRE Workbook, ch. *Managing Load* — https://sre.google/workbook/managing-load/

## H

### HCL (HashiCorp Configuration Language)
Declarative config language used by *Terraform*, *OpenTofu*, *Packer*, *Nomad*.
- **Used in**: ch02 §1
- **Sources**: HCL spec — https://github.com/hashicorp/hcl

### Hermetic Build
Build that depends only on declared, pinned inputs and produces bit-identical outputs. Required for *SLSA* L3+.
- **Used in**: ch03 §4
- **Sources**: Bazel docs — *Hermeticity* — https://bazel.build/basics/hermeticity ; SLSA — https://slsa.dev/

### HPA (Horizontal Pod Autoscaler)
Kubernetes controller that scales replicas based on CPU, memory, or custom/external metrics. Pair with *KEDA* for event-driven autoscaling.
- **Used in**: ch04
- **Sources**: Kubernetes docs — *HPA* — https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

### HSM (Hardware Security Module)
Tamper-resistant device that stores keys and performs crypto. Cloud-managed flavors: AWS CloudHSM, Azure Dedicated HSM, GCP Cloud HSM.
- **Used in**: ch06 §6
- **Sources**: NIST SP 800-57 part 1 rev. 5 — https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final

### Hubble
Cilium's observability layer for L3/L4/L7 network flows via *eBPF*.
- **Used in**: ch07 §7
- **Sources**: Hubble docs — https://docs.cilium.io/en/stable/gettingstarted/hubble/

## I

### IaC (Infrastructure as Code)
Practice of expressing infrastructure as version-controlled, machine-applied definitions, executed by a reconciler that converges actual to desired state. **Disambiguation**: *Pulumi* / *CDK* qualify when applied declaratively; ad-hoc scripts do not.
- **Used in**: ch01 §2, ch02 (entire)
- **Sources**: Kief Morris, *Infrastructure as Code* (2nd ed., O'Reilly, 2020); HashiCorp — *What is IaC* — https://developer.hashicorp.com/terraform/intro

### IAP (Identity-Aware Proxy)
Per-request user/device identity check at the proxy boundary; basis of *BeyondCorp*-style access. Cloudflare Access, Google IAP, *Teleport* implement variants.
- **Used in**: ch06 §5
- **Sources**: Google IAP docs — https://cloud.google.com/iap/docs/concepts-overview ; Rory Ward & Betsy Beyer, *BeyondCorp: A New Approach to Enterprise Security* (;login:, 2014) — https://research.google/pubs/pub43231/

### IDP (Internal Developer Platform)
Curated set of self-service tools and golden paths a *platform team* offers to its internal customers. Distinct from *Backstage* (a *developer portal*).
- **Used in**: ch11 §3
- **Sources**: PlatformEngineering.org glossary — https://platformengineering.org/blog/what-is-an-internal-developer-platform ; Humanitec, *Platform Engineering: A Guide* — https://humanitec.com/platform-engineering

### Idempotency Key
Client-supplied unique ID a server uses to dedup retried operations; the foundation of *effectively-once* APIs.
- **Used in**: ch08 §4
- **Sources**: Stripe API — *Idempotent Requests* — https://stripe.com/docs/api/idempotent_requests ; IETF draft *The Idempotency-Key HTTP Header* — https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/

### Idempotent
A change that produces the same result whether applied once or many times. Required of *IaC* providers and reconcilers.
- **Used in**: ch01 §2
- **Sources**: Pat Helland, *Idempotence Is Not a Medical Condition* (ACM Queue, 2012) — https://queue.acm.org/detail.cfm?id=2187821

### Immutable Server / Immutable Infrastructure
Servers never modified after deploy; changes are made by replacement.
- **Used in**: ch01 §3
- **Sources**: Martin Fowler — *ImmutableServer* — https://martinfowler.com/bliki/ImmutableServer.html ; Kief Morris, *Immutable Infrastructure* (2013) — https://martinfowler.com/articles/immutable-infrastructure.html

### Ingress (legacy)
Original Kubernetes north-south HTTP API. Largely superseded by *Gateway API* for new work.
- **Used in**: ch04
- **Sources**: Kubernetes docs — *Ingress* — https://kubernetes.io/docs/concepts/services-networking/ingress/

### in-toto
Framework for cryptographically attesting software-supply-chain steps. *attestations* + *DSSE* envelopes + a layout describing the chain.
- **Used in**: ch03 §3
- **Sources**: in-toto spec — https://github.com/in-toto/docs

### Inverse Conway Maneuver
Deliberately reshape teams to produce the architecture you want, given *Conway's Law*.
- **Used in**: ch11 §2
- **Sources**: Skelton & Pais, *Team Topologies* (2019); Jonny LeRoy / James Lewis at ThoughtWorks (origin of phrase)

### IPAM (IP Address Management)
The practice and tooling for allocating and tracking IP space across *VPCs*, on-prem, and partner networks. Cloud-native: AWS VPC IPAM, Azure VNet Manager, GCP Network Topology.
- **Used in**: ch07 §2
- **Sources**: AWS VPC IPAM docs — https://docs.aws.amazon.com/vpc/latest/ipam/

### IRSA / Pod Identity / Workload Identity
Mechanisms binding cloud IAM identities to *Kubernetes* service accounts via *OIDC* federation: *IRSA* (legacy AWS), *EKS Pod Identity* (current AWS), *GKE Workload Identity*, *AKS Workload Identity*. **Deprecated**: *AAD Pod Identity* — superseded by AKS Workload Identity.
- **Used in**: ch06 §2
- **Sources**: AWS docs — *EKS Pod Identity* — https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html ; GKE Workload Identity — https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity ; AKS Workload Identity — https://learn.microsoft.com/azure/aks/workload-identity-overview

## J

### Jepsen
Kyle Kingsbury's testing framework and report series that empirically probes distributed-database consistency claims. The standard reference for *CAP* claims.
- **Used in**: ch08 §1
- **Sources**: Jepsen analyses — https://jepsen.io/analyses

### Jitter
Random delay added to retries and scheduled jobs to prevent synchronized stampedes. Pair with capped exponential backoff.
- **Used in**: ch09 §3
- **Sources**: Marc Brooker (AWS), *Exponential Backoff and Jitter* — https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/

## K

### Karpenter
AWS-originated *Kubernetes* node autoscaler that provisions right-sized nodes from a flexible instance pool, much faster than *Cluster Autoscaler*.
- **Used in**: ch04
- **Sources**: Karpenter docs — https://karpenter.sh/

### KEDA (Kubernetes Event-Driven Autoscaling)
HPA add-on driving scale from external event sources (queues, Kafka, Prometheus). Standard way to do scale-to-zero workers.
- **Used in**: ch04
- **Sources**: KEDA docs — https://keda.sh/

### KEV (Known Exploited Vulnerabilities)
CISA's catalog of CVEs known to be exploited in the wild. Patch these first.
- **Used in**: ch06 §4
- **Sources**: CISA KEV — https://www.cisa.gov/known-exploited-vulnerabilities-catalog

### Kubebench / kube-bench
Tool that runs the *CIS Kubernetes Benchmark* against a live cluster.
- **Used in**: ch04, ch06 §3
- **Sources**: kube-bench — https://github.com/aquasecurity/kube-bench

### Kyverno / Gatekeeper
Kubernetes admission and policy engines. *Kyverno* uses YAML policies; *Gatekeeper* runs *OPA* / *Rego*.
- **Used in**: ch04, ch06 §3
- **Sources**: Kyverno docs — https://kyverno.io/docs/ ; Gatekeeper docs — https://open-policy-agent.github.io/gatekeeper/

## L

### Landing Zone
Pre-baked, multi-account/multi-subscription/multi-project foundation (identity, network, logging, guardrails) onto which workloads land. AWS Control Tower, Azure Landing Zones, GCP Foundation Blueprint.
- **Used in**: ch07 §1
- **Sources**: AWS — *Landing Zone* — https://docs.aws.amazon.com/prescriptive-guidance/latest/migration-aws-environment/welcome.html ; Azure — *Cloud Adoption Framework Landing Zones* — https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/

### Liveness / Readiness / Startup Probe
Three Kubernetes probes with very different semantics: liveness restarts containers, readiness gates traffic, startup gives slow apps time to initialize. Misconfigured liveness probes cause more outages than they prevent.
- **Used in**: ch04
- **Sources**: Kubernetes docs — *Configure Liveness, Readiness and Startup Probes* — https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

### Load Shedding
Server-side capacity protection that drops or rejects requests above a budget so the service degrades predictably instead of collapsing.
- **Used in**: ch09 §3
- **Sources**: Marc Brooker, *Using Load Shedding to Avoid Overload* (AWS Builders' Library) — https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/

### Lock File
Pinned transitive-dependency manifest (e.g., `.terraform.lock.hcl`, `package-lock.json`). Required for reproducible builds.
- **Used in**: ch02 §1, ch03 §4
- **Sources**: Terraform — *Dependency Lock File* — https://developer.hashicorp.com/terraform/language/files/dependency-lock

### Loki
Grafana Labs log aggregator that indexes labels rather than full text; cheap at scale, weaker for ad-hoc text search.
- **Used in**: ch05 §4
- **Sources**: Loki docs — https://grafana.com/docs/loki/latest/

## M

### Mimir
Grafana Labs' horizontally-scalable Prometheus backend; supersedes *Cortex*.
- **Used in**: ch05 §4
- **Sources**: Mimir docs — https://grafana.com/docs/mimir/latest/

### Module (Terraform/OpenTofu)
Reusable, versioned bundle of resources with inputs and outputs. The unit of platform-curated abstraction in *HCL*.
- **Used in**: ch02 §1
- **Sources**: Terraform — *Modules* — https://developer.hashicorp.com/terraform/language/modules

### MSS Clamping
Adjusting TCP MSS at a router/firewall to prevent fragmentation when the path *MTU* shrinks (e.g., over IPsec or VXLAN).
- **Used in**: ch07 §5
- **Sources**: RFC 4459 — https://www.rfc-editor.org/rfc/rfc4459

### MTBF (Mean Time Between Failures)
Hardware reliability metric. **Deprecated** for software in this guide — software fails for systemic reasons that violate the i.i.d. assumption MTBF requires; use *SLO* burn rate and incident frequency instead.
- **Used in**: ch09 §6
- **Sources**: Google SRE Workbook, ch. *Implementing SLOs* — https://sre.google/workbook/implementing-slos/

### MTU
Maximum transmission unit on a link. Tunneling protocols (VXLAN, GENEVE, IPsec) and *PrivateLink* paths often shrink the effective MTU; set or *MSS-clamp* accordingly.
- **Used in**: ch07 §5
- **Sources**: RFC 791 / RFC 1191 (PMTUD)

### Multi-Cloud
Deliberately running production workloads across more than one hyperscaler. **Disambiguation**: most "multi-cloud" orgs are *primary cloud + SaaS in another cloud*. True multi-cloud has a high tax (people, data movement, abstractions); justify it with a concrete requirement (regulatory, M&A, redundancy you've measured the value of).
- **Used in**: ch07 §1, ch11 §1
- **Sources**: Gregor Hohpe, *Multi-Cloud is the Worst Practice* — https://architectelevator.com/cloud/multicloud-worst-practice/ ; AWS Well-Architected — *Multi-account strategy* — https://docs.aws.amazon.com/wellarchitected/latest/management-and-governance-guide/multi-account-strategy.html

### Multi-Window Multi-Burn-Rate Alerting
SLO alerting strategy that fires on combinations of fast-burn (short window, high rate) and slow-burn (long window, low rate) to balance precision and detection time.
- **Used in**: ch05 §3, ch09 §2
- **Sources**: Google SRE Workbook — *Alerting on SLOs* — https://sre.google/workbook/alerting-on-slos/

### MVCC (Multi-Version Concurrency Control)
Database technique where readers see a consistent snapshot without blocking writers (PostgreSQL, MySQL InnoDB, Spanner).
- **Used in**: ch08 §6
- **Sources**: Hellerstein, Stonebraker, Hamilton, *Architecture of a Database System* (2007) — http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf

## N

### NAT64 / DNS64
Translation mechanisms that let IPv6-only clients reach IPv4-only services. Standard ingredient of v6-only networks.
- **Used in**: ch07 §3
- **Sources**: RFC 6146 (NAT64) — https://www.rfc-editor.org/rfc/rfc6146 ; RFC 6147 (DNS64) — https://www.rfc-editor.org/rfc/rfc6147

### NetworkPolicy
Kubernetes namespaced firewall resource for L3/L4 pod-to-pod rules; enforced by the *CNI*.
- **Used in**: ch04, ch07 §7
- **Sources**: Kubernetes docs — *Network Policies* — https://kubernetes.io/docs/concepts/services-networking/network-policies/

### NIST SSDF (SP 800-218)
NIST Secure Software Development Framework — practices that organizations should adopt across the SDLC.
- **Used in**: ch06 §1
- **Sources**: NIST SP 800-218 — https://csrc.nist.gov/publications/detail/sp/800-218/final

### NIST 800-207 (Zero Trust)
NIST's reference architecture for zero-trust access. Per-request authn/z, no implicit trust based on network location.
- **Used in**: ch06 §5
- **Sources**: NIST SP 800-207 — https://csrc.nist.gov/publications/detail/sp/800-207/final

## O

### Object Lock / Bucket Lock / Immutable Blob
Provider features that make object-storage objects immutable for a retention period, defending against ransomware and bad actors. AWS Object Lock, GCS Bucket Lock, Azure immutable blob storage.
- **Used in**: ch08 §2
- **Sources**: AWS S3 Object Lock — https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html ; GCS Bucket Lock — https://cloud.google.com/storage/docs/bucket-lock ; Azure immutable storage — https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview

### OCI Distribution Spec
Standard registry API used to push, pull, and reference container images and *OCI artifacts*. The *Referrers API* enables attaching *attestations*.
- **Used in**: ch03 §3
- **Sources**: OCI Distribution Spec — https://github.com/opencontainers/distribution-spec

### Observability
Property of a system: how well you can answer *novel* questions about it from its outputs without shipping new code. **Disambiguation**: not "logs + metrics + traces"; per Charity Majors, the unit is high-cardinality, high-dimensional *wide events* you can slice arbitrarily.
- **Used in**: ch05 §1
- **Sources**: Charity Majors, *Observability — A 3-Year Retrospective* (2019) — https://thenewstack.io/observability-a-3-year-retrospective/ ; Majors, Fong-Jones, Miranda, *Observability Engineering* (O'Reilly, 2022)

### OIDC Federation (Workload Identity Federation)
Trusting an external *OIDC* provider (GitHub Actions, GitLab, *Kubernetes* SA tokens) to mint short-lived cloud credentials — replaces long-lived static keys.
- **Used in**: ch03 §3, ch06 §2
- **Sources**: GitHub OIDC — https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect ; AWS — *OIDC Federation* — https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html

### OpenCost / Kubecost
CNCF spec and product for *Kubernetes* cost allocation. *OpenCost* is the open-source standard; Kubecost is its commercial origin.
- **Used in**: ch10 §3
- **Sources**: OpenCost — https://www.opencost.io/

### OpenFeature
CNCF vendor-neutral *feature flag* SDK and spec.
- **Used in**: ch03 §5
- **Sources**: OpenFeature — https://openfeature.dev/specification/

### OpenGitOps
CNCF GitOps Working Group's formal definition of *GitOps* in four principles.
- **Used in**: ch01 §5
- **Sources**: OpenGitOps v1.0.0 — https://opengitops.dev/

### OpenTelemetry (OTel)
CNCF project unifying telemetry SDKs, *OTLP* protocol, and the *Collector*. The default instrumentation choice.
- **Used in**: ch05 §2
- **Sources**: OpenTelemetry docs — https://opentelemetry.io/docs/

### OpenTofu
Open-source fork of *Terraform* under the Linux Foundation; protocol-compatible with the same providers and *HCL*.
- **Used in**: ch02 §1
- **Sources**: OpenTofu docs — https://opentofu.org/docs/

### Operator (Kubernetes)
Controller that encodes operational knowledge for an application as a *CRD* + reconcile loop (*CloudNativePG*, *Strimzi*, etc.).
- **Used in**: ch04
- **Sources**: CoreOS, *Introducing Operators* (2016) — https://web.archive.org/web/20170129131616/https://coreos.com/blog/introducing-operators.html ; *Operator Pattern* docs — https://kubernetes.io/docs/concepts/extend-kubernetes/operator/

### OPA / Rego
Open Policy Agent and its *Rego* policy language. Used by *Gatekeeper*, *Conftest*, and many CI gates.
- **Used in**: ch02 §5, ch04, ch06 §3
- **Sources**: OPA docs — https://www.openpolicyagent.org/docs/

### OTLP
OpenTelemetry's wire protocol for traces, metrics, logs.
- **Used in**: ch05 §2
- **Sources**: OTLP spec — https://opentelemetry.io/docs/specs/otlp/

## P

### Paved Road / Golden Path
The platform-blessed, opinionated way to build and run a service: best supported, lowest friction, deviation allowed but unsupported. Spotify popularized "*Golden Paths*"; Netflix uses "Paved Road."
- **Used in**: ch01 §6, ch11 §3
- **Sources**: Spotify Engineering, *How We Use Golden Paths* — https://engineering.atspotify.com/2020/08/17/how-we-use-golden-paths-to-solve-fragmentation-in-our-software-ecosystem/ ; Netflix Tech Blog, *Full Cycle Developers* — https://netflixtechblog.com/full-cycle-developers-at-netflix-a08c31f83249

### PDB (PodDisruptionBudget)
Kubernetes object capping voluntary disruptions for a workload (e.g., during node drain).
- **Used in**: ch04
- **Sources**: Kubernetes docs — *PDB* — https://kubernetes.io/docs/concepts/workloads/pods/disruptions/

### PgBouncer
Lightweight PostgreSQL connection pooler; standard fix for connection-storm problems.
- **Used in**: ch08 §6
- **Sources**: PgBouncer docs — https://www.pgbouncer.org/usage.html

### Phoenix Server
Server destroyed and rebuilt from scratch on a regular cadence, eliminating *snowflake* drift.
- **Used in**: ch01 §3
- **Sources**: Martin Fowler — *PhoenixServer* — https://martinfowler.com/bliki/PhoenixServer.html

### PITR (Point-in-Time Recovery)
Restoring a database to any prior moment by replaying *WAL* / *binlog* against a base backup.
- **Used in**: ch08 §2
- **Sources**: PostgreSQL docs — *Continuous Archiving and PITR* — https://www.postgresql.org/docs/current/continuous-archiving.html

### Platform Engineering
Discipline of building an *Internal Developer Platform* as a product. **Disambiguation**: the platform is the *thinnest viable layer* that takes cognitive load off product teams; it is **not** an ops queue. Success measured by adoption and time-to-first-deploy, not ticket throughput.
- **Used in**: ch11 §1
- **Sources**: PlatformEngineering.org — https://platformengineering.org/blog/what-is-platform-engineering ; ThoughtWorks Tech Radar — *Platform engineering teams* — https://www.thoughtworks.com/radar ; CNCF Platforms WG, *Platforms White Paper* — https://tag-app-delivery.cncf.io/whitepapers/platforms/

### PMTUD (Path MTU Discovery)
Mechanism by which endpoints learn the smallest MTU on a path. Often broken by ICMP filtering; pair with *MSS clamping*.
- **Used in**: ch07 §5
- **Sources**: RFC 1191 — https://www.rfc-editor.org/rfc/rfc1191

### Pod QoS Class
Kubernetes pod scheduling/eviction class derived from requests/limits: Guaranteed, Burstable, BestEffort. Drives kubelet eviction order.
- **Used in**: ch04
- **Sources**: Kubernetes docs — *Pod Quality of Service Classes* — https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/

### PodSecurityPolicy
Removed admission controller. **Deprecated**; replaced by *Pod Security Admission* and policy engines (*Kyverno*, *Gatekeeper*).
- **Used in**: ch04, ch06 §3
- **Sources**: Kubernetes blog — *PodSecurityPolicy Deprecation* — https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/

### PrivateLink / Private Endpoint / PSC
Provider-specific mechanisms exposing a service inside a customer *VPC* without traversing the internet (AWS PrivateLink, Azure Private Endpoint, GCP Private Service Connect).
- **Used in**: ch07 §4
- **Sources**: AWS PrivateLink — https://docs.aws.amazon.com/vpc/latest/privatelink/ ; Azure Private Link — https://learn.microsoft.com/azure/private-link/ ; GCP PSC — https://cloud.google.com/vpc/docs/private-service-connect

### Provider Mirror
Local cache or proxy of *Terraform*/*OpenTofu* providers used for offline or pinned environments.
- **Used in**: ch02 §1
- **Sources**: Terraform docs — *Provider Network Mirror Protocol* — https://developer.hashicorp.com/terraform/internals/provider-network-mirror-protocol

### Promote-by-Digest
Promotion discipline where the same OCI image referenced by `image@sha256:…` is what flows from staging to production — **no rebuild on promote**. A rebuild produces a new digest, invalidating prior signatures, attestations, scans, and approvals. Distinct from *pin by digest* (consumer-side reference), though they reinforce each other: promote-by-digest is the registry/CD discipline that makes pin-by-digest meaningful end-to-end.
- **Used in**: ch03 §3.4, ch04 §2, ch06 §11
- **Sources**: OCI Distribution Spec — https://github.com/opencontainers/distribution-spec ; SLSA v1.0 — https://slsa.dev/spec/v1.0/

### Pin by Digest
Consumer-side reference discipline: workloads, manifests, base images, and GitHub Actions are pinned to an immutable content hash (`@sha256:…` or full commit SHA) rather than a mutable tag (`:latest`, `@v3`). Required for reproducibility and for *Promote-by-Digest* to be meaningful in production.
- **Used in**: ch03 §4.1, ch04 §2
- **Sources**: OCI Distribution Spec — https://github.com/opencontainers/distribution-spec ; GitHub Actions security hardening — https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions

### PSA (Pod Security Admission)
Kubernetes built-in admission controller enforcing *privileged*, *baseline*, or *restricted* profiles per namespace. Successor to *PodSecurityPolicy*.
- **Used in**: ch04, ch06 §3
- **Sources**: Kubernetes docs — *Pod Security Admission* — https://kubernetes.io/docs/concepts/security/pod-security-admission/

### Pulumi
*IaC* tool that uses general-purpose languages to describe desired state and reconciles via providers. Counts as IaC when used declaratively.
- **Used in**: ch02 §1
- **Sources**: Pulumi docs — https://www.pulumi.com/docs/

### Puppet Master
Centralized Puppet control server. **Deprecated** terminology; the project now uses "primary server."
- **Used in**: ch01 §3
- **Sources**: Puppet docs — *Server architecture* — https://puppet.com/docs/puppet/latest/architecture.html

## Q

### QUIC / HTTP/3
UDP-based encrypted transport (QUIC, RFC 9000) and the HTTP binding over it (HTTP/3, RFC 9114). Lower head-of-line blocking; key for mobile and lossy paths.
- **Used in**: ch07 §5
- **Sources**: RFC 9000 — https://www.rfc-editor.org/rfc/rfc9000 ; RFC 9114 — https://www.rfc-editor.org/rfc/rfc9114

## R

### RBAC (Kubernetes)
Role-based access control: Roles/ClusterRoles + RoleBindings/ClusterRoleBindings.
- **Used in**: ch04, ch06 §2
- **Sources**: Kubernetes docs — *Using RBAC Authorization* — https://kubernetes.io/docs/reference/access-authn-authz/rbac/

### RED Method
Rate, Errors, Duration — Tom Wilkie's request-oriented service metrics. Pair with *USE* for resources.
- **Used in**: ch05 §3
- **Sources**: Tom Wilkie, *The RED Method* — https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/

### Reconciliation Loop / Control Loop
Continuous "observe → diff → act" loop that drives actual state to declared state. Foundation of *Kubernetes*, *GitOps*, and modern *IaC*.
- **Used in**: ch01 §4
- **Sources**: Kubernetes docs — *Controllers* — https://kubernetes.io/docs/concepts/architecture/controller/

### REINDEX CONCURRENTLY
PostgreSQL command that rebuilds an index without long lock; standard tool for index bloat.
- **Used in**: ch08 §6
- **Sources**: PostgreSQL docs — *REINDEX* — https://www.postgresql.org/docs/current/sql-reindex.html

### Repatriation
Moving workloads off a public cloud back to private/colo for cost or sovereignty reasons. Almost always specific workloads, not whole orgs.
- **Used in**: ch10 §4
- **Sources**: Andreessen Horowitz, Sarah Wang & Martin Casado, *The Cost of Cloud, a Trillion Dollar Paradox* — https://a16z.com/the-cost-of-cloud-a-trillion-dollar-paradox/

### ResourceQuota / LimitRange
Namespace-scoped Kubernetes objects enforcing aggregate and per-object resource caps.
- **Used in**: ch04
- **Sources**: Kubernetes docs — *Resource Quotas* — https://kubernetes.io/docs/concepts/policy/resource-quotas/ ; *Limit Ranges* — https://kubernetes.io/docs/concepts/policy/limit-range/

### Retry Budget
Cap on the *fraction* of traffic that may be retries, preventing retry-driven amplification and metastable failure.
- **Used in**: ch09 §3
- **Sources**: gRPC docs — *Retries* — https://grpc.io/docs/guides/retry/ ; Marc Brooker (above) on metastable failures

### RFC 1918 / RFC 6598 / RFC 4193
Reserved address ranges: RFC 1918 (private IPv4: 10/8, 172.16/12, 192.168/16), RFC 6598 (CGN/100.64), RFC 4193 (IPv6 ULA fc00::/7). Foundation of *VPC* design.
- **Used in**: ch07 §2-3
- **Sources**: RFC 1918 — https://www.rfc-editor.org/rfc/rfc1918 ; RFC 6598 — https://www.rfc-editor.org/rfc/rfc6598 ; RFC 4193 — https://www.rfc-editor.org/rfc/rfc4193

### RPO / RTO
**Recovery Point Objective** (max data loss tolerated, in time) and **Recovery Time Objective** (max downtime tolerated). **Disambiguation**: must be *measured* via DR drills, not declared in slides; an untested RPO/RTO is a guess.
- **Used in**: ch08 §2, ch09 §4
- **Sources**: NIST SP 800-34 Rev. 1, *Contingency Planning Guide* — https://csrc.nist.gov/publications/detail/sp/800-34/rev-1/final

### Runbook
Step-by-step operational instructions for handling a known scenario; ideally executable, otherwise versioned with the service.
- **Used in**: ch09 §5
- **Sources**: Google SRE Book, ch. *Being On-Call* — https://sre.google/sre-book/being-on-call/

## S

### Savings Plan / Reserved Instance
AWS commitment-based discounts: *RIs* are instance-shape constrained; *Savings Plans* (Compute / EC2 Instance / SageMaker) trade flexibility for slightly less discount. Use **laddered** commitments.
- **Used in**: ch10 §2
- **Sources**: AWS — *Savings Plans* — https://docs.aws.amazon.com/savingsplans/ ; AWS — *Reserved Instances* — https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html

### SBOM (Software Bill of Materials)
Machine-readable inventory of components in a build artifact. Two dominant formats: *SPDX* and *CycloneDX*.
- **Used in**: ch03 §3, ch06 §4
- **Sources**: NTIA, *Minimum Elements for a Software Bill of Materials* — https://www.ntia.gov/report/2021/minimum-elements-software-bill-materials-sbom ; SPDX spec — https://spdx.dev/ ; CycloneDX spec — https://cyclonedx.org/

### Schema Registry
Service that stores and validates *Avro* / *Protobuf* / *JSON Schema* versions and enforces compatibility (BACKWARD/FORWARD/FULL).
- **Used in**: ch08 §3
- **Sources**: Confluent Schema Registry docs — https://docs.confluent.io/platform/current/schema-registry/index.html

### Scorecard (OpenSSF)
Tool that scores OSS repos on supply-chain hygiene checks (signed releases, branch protection, dependency-update tools).
- **Used in**: ch03 §3, ch06 §4
- **Sources**: OpenSSF Scorecard — https://github.com/ossf/scorecard

### Service Mesh
L7 east-west traffic infrastructure (mTLS, retries, traffic shaping, observability) for service-to-service traffic. **Disambiguation**: distinct from an *API gateway* (north-south) and from *Ingress* / *Gateway API*; Istio ambient and Linkerd are the current production-grade choices.
- **Used in**: ch04, ch07 §7
- **Sources**: William Morgan (Linkerd), *What's a service mesh?* — https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/ ; Istio docs — https://istio.io/latest/about/service-mesh/

### Shift-Left
Moving validation (security, testing, schema review) earlier in the SDLC. **Disambiguation**: in this guide we reframe as *shift-everywhere* — left-only checks miss runtime drift, so pair pre-merge gates with admission controls, runtime telemetry (*Falco*, *Tetragon*), and incident review.
- **Used in**: ch03 §1, ch06 §1
- **Sources**: Larry Smith, *Shift-Left Testing* (Dr. Dobb's, 2001); Jessica Kerr & Charity Majors, *Shift Left and Right* (interviews/talks); Forsgren et al., *Accelerate*

### Showback
Reporting cloud cost back to the business unit that incurred it without billing them. Step before *chargeback*.
- **Used in**: ch10 §3
- **Sources**: FinOps Framework — *Chargeback & Showback* — https://www.finops.org/framework/capabilities/chargeback-showback/

### SLI (Service Level Indicator)
A *good-events / valid-events* ratio measuring something a user actually cares about (e.g., 200-OK rate, p99 latency under N ms).
- **Used in**: ch09 §2
- **Sources**: Google SRE Book — *Service Level Objectives* — https://sre.google/sre-book/service-level-objectives/

### SLO (Service Level Objective)
A target value for an *SLI* over a window. Drives the *error budget* and the *error-budget policy*.
- **Used in**: ch09 §2
- **Sources**: Google SRE Workbook — *Implementing SLOs* — https://sre.google/workbook/implementing-slos/ ; Alex Hidalgo, *Implementing Service Level Objectives* (O'Reilly, 2020)

### SLO-as-Code
Defining *SLOs* in machine-readable form so alerts, dashboards, and *burn-rate* policies are generated, not handcrafted. Sloth, Pyrra, OpenSLO.
- **Used in**: ch05 §3, ch09 §2
- **Sources**: OpenSLO spec — https://github.com/OpenSLO/OpenSLO ; Sloth — https://sloth.dev/

### SLSA (Supply-chain Levels for Software Artifacts)
Tiered framework (L1–L3+) for build integrity. L3 requires *hermetic* and isolated builds with verifiable *provenance*.
- **Used in**: ch03 §3, ch06 §4
- **Sources**: SLSA spec — https://slsa.dev/spec/v1.0/

### Snowflake Server
A server hand-tweaked over years until nobody dares rebuild it. The thing *cattle*, *immutable servers*, and *phoenix servers* exist to kill.
- **Used in**: ch01 §3
- **Sources**: Martin Fowler — *SnowflakeServer* — https://martinfowler.com/bliki/SnowflakeServer.html

### SOPS
Mozilla CLI for encrypting structured config files (YAML/JSON/INI/env) with KMS, *Vault*, age, or PGP. Common with *GitOps*.
- **Used in**: ch06 §2
- **Sources**: SOPS — https://github.com/getsops/sops

### SPACE Framework
Developer productivity framework spanning Satisfaction, Performance, Activity, Communication, Efficiency. Designed to *complement*, not replace, *DORA*.
- **Used in**: ch03 §1, ch11 §4
- **Sources**: Forsgren, Storey, Maddila, Zimmermann, Houck, Butler, *The SPACE of Developer Productivity* (ACM Queue, 2021) — https://queue.acm.org/detail.cfm?id=3454124

### SPIFFE / SPIRE
Spec and reference implementation for cryptographic workload identity (*SPIFFE ID*, SVID). Foundation of zero-trust workload-to-workload auth.
- **Used in**: ch06 §5, ch07 §6
- **Sources**: SPIFFE spec — https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/

### Spot / Preemptible
Discounted, interruptible cloud capacity. *Dual-pool spot* (on-demand fallback) is the cost-safe default. **Deprecated**: AWS Spot Blocks (1–6 h reserved spot) — discontinued.
- **Used in**: ch10 §2
- **Sources**: AWS Spot docs — https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html ; GCP Preemptible/Spot — https://cloud.google.com/compute/docs/instances/spot

### Stack
Deployable unit of *IaC* (CloudFormation stack, Terraform root module, Pulumi stack). Smaller stacks ⇒ smaller *blast radius*.
- **Used in**: ch01 §6, ch02 §2
- **Sources**: Kief Morris, *Infrastructure as Code* (2nd ed.), ch. on Stacks

### State (Terraform/OpenTofu) / State Locking / Remote State
Source-of-truth file mapping config to real resources; must be in remote backend with locking. **Deprecated**: standalone DynamoDB locking — Terraform AWS S3 backend now supports native S3 locking (Terraform 1.10+ / OpenTofu 1.10+).
- **Used in**: ch02 §2
- **Sources**: Terraform docs — *State* — https://developer.hashicorp.com/terraform/language/state ; HashiCorp blog — *S3 Native State Locking* (2024) — https://www.hashicorp.com/blog/terraform-aws-s3-backend-adds-support-for-native-state-locking

### StatefulSet
Kubernetes workload type that gives pods stable identity, ordered deployment, and stable storage via *CSI*.
- **Used in**: ch04, ch08 §6
- **Sources**: Kubernetes docs — *StatefulSets* — https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

### STRIDE / LINDDUN
Threat-modeling taxonomies. *STRIDE* (Microsoft) covers security threats; *LINDDUN* covers privacy threats. Pair with the *Threat Modeling Manifesto*.
- **Used in**: ch06 §1
- **Sources**: Microsoft, *STRIDE* — https://learn.microsoft.com/azure/security/develop/threat-modeling-tool-threats ; LINDDUN — https://linddun.org/ ; Threat Modeling Manifesto — https://www.threatmodelingmanifesto.org/

## T

### Tail / Head Sampling
Trace-sampling strategies. Head sampling decides at root; tail sampling buffers and decides after the trace completes (e.g., keep slow or error traces). Tail sampling needs trace-aware *Collector* topology.
- **Used in**: ch05 §2
- **Sources**: OpenTelemetry — *Sampling* — https://opentelemetry.io/docs/concepts/sampling/

### Talos
Kubernetes-only Linux distribution with no SSH, API-driven and *immutable*.
- **Used in**: ch04
- **Sources**: Talos Linux docs — https://www.talos.dev/latest/

### Team Topologies
Skelton & Pais's model of four team types (stream-aligned, platform, enabling, complicated-subsystem) and three interaction modes (collaboration, X-as-a-Service, facilitating). Operationalizes *Conway's Law*.
- **Used in**: ch11 §2
- **Sources**: Matthew Skelton & Manuel Pais, *Team Topologies* (IT Revolution, 2019) — https://teamtopologies.com/

### TechDocs
*Backstage* plugin that builds and serves docs-as-code from each repo using MkDocs.
- **Used in**: ch11 §3
- **Sources**: TechDocs docs — https://backstage.io/docs/features/techdocs/

### Tekton
Kubernetes-native CI primitive (*Tasks*, *Pipelines*, *PipelineRuns*) intended as a building block for higher-level CI/CD systems.
- **Used in**: ch03 §6
- **Sources**: Tekton docs — https://tekton.dev/docs/

### Teleport
Identity-aware access proxy for SSH, Kubernetes, databases, and apps. Implements zero-trust patterns from *NIST 800-207* and *BeyondCorp*.
- **Used in**: ch06 §5
- **Sources**: Teleport docs — https://goteleport.com/docs/

### Terraform
HashiCorp's *IaC* tool. License changed to BSL in 2023; community fork *OpenTofu* lives under the Linux Foundation.
- **Used in**: ch02 §1
- **Sources**: Terraform docs — https://developer.hashicorp.com/terraform/docs

### Tetragon
Cilium's *eBPF* runtime-security and observability tool; alternative/companion to *Falco*.
- **Used in**: ch04, ch06 §3
- **Sources**: Tetragon docs — https://tetragon.io/docs/

### Three Pillars (of Observability)
Logs + metrics + traces. **Deprecated as a design framework** in this guide: it implies three siloed pipelines and obscures the actual unit of *observability* — high-cardinality wide events. Use it descriptively, not prescriptively.
- **Used in**: ch05 §1
- **Sources**: Charity Majors, *The Three Pillars With Zero Answers* — https://charity.wtf/2018/09/05/the-three-pillars-with-zero-answers-towards-a-new-scorecard-for-observability/ ; *Observability Engineering* (O'Reilly, 2022)

### Toil
Kind of work that is manual, repetitive, automatable, tactical, devoid of enduring value, and scales linearly with service growth.
- **Used in**: ch09 §1
- **Sources**: Google SRE Book — *Eliminating Toil* — https://sre.google/sre-book/eliminating-toil/

### topologySpreadConstraints
Kubernetes scheduler hint to distribute pods across failure domains (zones, nodes, hostnames).
- **Used in**: ch04
- **Sources**: Kubernetes docs — *Pod Topology Spread Constraints* — https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/

### Transactional Outbox
Pattern that writes domain events to an `outbox` table in the same transaction as state changes; a relay reads it and publishes downstream — replaces *dual writes*.
- **Used in**: ch08 §4
- **Sources**: Chris Richardson — *Pattern: Transactional outbox* — https://microservices.io/patterns/data/transactional-outbox.html

### Transit Gateway / vWAN / NCC
Cloud network hubs that interconnect many *VPCs*/VNets and on-prem links. AWS Transit Gateway, Azure Virtual WAN, Google Network Connectivity Center.
- **Used in**: ch07 §4
- **Sources**: AWS TGW — https://docs.aws.amazon.com/vpc/latest/tgw/ ; Azure vWAN — https://learn.microsoft.com/azure/virtual-wan/ ; GCP NCC — https://cloud.google.com/network-connectivity/docs/network-connectivity-center

## U

### Unit Economics
Cost (and revenue) per business unit (request, tenant, MAU). The metric *FinOps* maturity points at — not raw spend.
- **Used in**: ch10 §3
- **Sources**: J.R. Storment & Mike Fuller, *Cloud FinOps* (2nd ed.) ; FinOps Foundation — *Unit Economics* — https://www.finops.org/framework/capabilities/unit-economics/

### USE Method
Utilization, Saturation, Errors — Brendan Gregg's resource-oriented metrics. Pair with *RED* for services.
- **Used in**: ch05 §3
- **Sources**: Brendan Gregg — *The USE Method* — https://www.brendangregg.com/usemethod.html

## V

### Vault
HashiCorp secrets-management system; canonical source of dynamic secrets, PKI, and encryption-as-a-service.
- **Used in**: ch06 §2
- **Sources**: Vault docs — https://developer.hashicorp.com/vault/docs

### vCluster
Virtual *Kubernetes* clusters running inside a host cluster's namespace. Stronger isolation than namespaces, cheaper than separate clusters.
- **Used in**: ch04
- **Sources**: vCluster docs — https://www.vcluster.com/docs

### Vector / Fluent Bit
Telemetry agents/forwarders. *Vector* is Datadog's Rust-based pipeline; *Fluent Bit* is the CNCF lightweight collector.
- **Used in**: ch05 §2
- **Sources**: Vector docs — https://vector.dev/docs/ ; Fluent Bit docs — https://docs.fluentbit.io/

### VPA (Vertical Pod Autoscaler)
Kubernetes controller that recommends or applies right-sized requests/limits. Don't run in `Auto` mode alongside HPA on the same metric.
- **Used in**: ch04
- **Sources**: Kubernetes Autoscaler — *VPA* — https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

### VPC / VNet
Provider-isolated virtual network (AWS VPC, Azure VNet, GCP VPC). Subnets typically AZ-scoped on AWS/Azure, region-scoped on GCP — a frequent multi-cloud footgun.
- **Used in**: ch07 §2
- **Sources**: AWS VPC docs — https://docs.aws.amazon.com/vpc/ ; Azure Virtual Network docs — https://learn.microsoft.com/azure/virtual-network/ ; GCP VPC docs — https://cloud.google.com/vpc/docs

## W

### W3C Trace Context
Standard trace propagation headers (`traceparent`, `tracestate`). Required for cross-vendor distributed tracing.
- **Used in**: ch05 §2
- **Sources**: W3C Trace Context — https://www.w3.org/TR/trace-context/

### WAL (Write-Ahead Log)
Durability mechanism: every write goes to an append-only log before mutating pages. Basis of *PITR* and replication in PostgreSQL and most modern DBs.
- **Used in**: ch08 §2
- **Sources**: PostgreSQL docs — *Write-Ahead Logging* — https://www.postgresql.org/docs/current/wal-intro.html

### WebAuthn / AAL3
W3C standard for phishing-resistant authentication; AAL3 is NIST's highest authenticator-assurance level (hardware-backed, multi-factor).
- **Used in**: ch06 §5
- **Sources**: W3C WebAuthn Level 3 — https://www.w3.org/TR/webauthn-3/ ; NIST SP 800-63B — https://pages.nist.gov/800-63-3/sp800-63b.html

### Wide Event
Single high-cardinality, high-dimensional event per unit of work, with all the context needed to slice and aggregate after the fact. The actual unit of *observability* per Majors et al.
- **Used in**: ch05 §1
- **Sources**: Charity Majors, *Observability Engineering* (O'Reilly, 2022); Honeycomb docs — *Events* — https://docs.honeycomb.io/get-started/basics/events/

### Wolfi
Chainguard's undistro Linux base for *distroless*-style images, built around current packages and signed metadata.
- **Used in**: ch04, ch06 §3
- **Sources**: Wolfi docs — https://github.com/wolfi-dev/

### Workspace (Terraform/OpenTofu)
Named state instance under one config — useful for *ephemeral environments*. Note: distinct from Terraform Cloud workspaces (a different concept).
- **Used in**: ch02 §3
- **Sources**: Terraform docs — *Workspaces* — https://developer.hashicorp.com/terraform/language/state/workspaces

## Y

### YugabyteDB
Distributed SQL DB, PostgreSQL wire protocol, Raft replication; another point in the *Spanner*/*CockroachDB* design space.
- **Used in**: ch08 §1
- **Sources**: YugabyteDB docs — https://docs.yugabyte.com/

## Z

### Zero Trust
Security model: never trust based on network location; verify identity, device posture, and authorization on every request. See *NIST 800-207*, *BeyondCorp*.
- **Used in**: ch06 §5
- **Sources**: NIST SP 800-207 — https://csrc.nist.gov/publications/detail/sp/800-207/final ; Google, *BeyondCorp* — https://www.beyondcorp.com/
