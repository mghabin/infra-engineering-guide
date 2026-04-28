# 08 — Data & State

Opinionated, cloud-agnostic defaults for running stateful infrastructure in 2026.
This chapter is about **operating** databases, caches, brokers, and object stores
— not about schema design, ORMs, or query tuning. Application-side data modelling
belongs in a different document; here we treat data as a *workload* with an SLO,
a blast radius, and a recovery clock.

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint.

The single most expensive lesson in this domain: **the only durability metric
that matters is restore time on real data.** Everything else — replication lag,
backup success rate, RPO targets in a slide deck — is a leading indicator at
best and a comforting lie at worst (Google SRE Book, ch. 26, "Data Integrity:
What You Read Is What You Wrote").
## 1. Managed vs self-hosted

**Default to managed.** In 2026, for Postgres, MySQL, Redis, Kafka, object
storage, and search, the managed offering from your primary cloud (or a
reputable specialist: Aiven, Confluent Cloud, MongoDB Atlas, Neon, PlanetScale)
is the right starting point for ≥95% of teams. The operational surface —
patching, minor-version upgrades, backup orchestration, failover, TLS rotation,
OS-level CVEs — is a *full-time team* you do not have ("Database Reliability
Engineering", Campbell & Majors, ch. 1; AWS RDS / Azure Database / Cloud SQL
shared-responsibility docs).

**Self-host only when at least one is true:** (a) regulatory — the data cannot
leave your tenancy or a region your provider doesn't serve; (b) cost at scale —
past ~$50k/month managed spend on one engine, with a real TCO model that
includes the 2+ SREs you'll hire ("DBRE", ch. 2); (c) feature gap — you need
an extension, version, or tuning knob the managed service refuses; (d) latency
floor — co-tenant placement the provider won't expose. "We're special" and "we
want control" are not on the list.

**Don't operate two of the same engine.** One Postgres fleet, one Kafka, one
Redis flavor. Multi-engine sprawl is how on-call dies.
## 2. Backups: the 3-2-1 rule, restated

The 3-2-1 rule (US-CERT; Krogh, *The DAM Book*) is still the floor: **3** copies,
**2** media, **1** off-site. In cloud terms: (1) the live DB, (2) automated
managed snapshots in the same account, (3) **cross-account, cross-region** export
pushed by a *different* identity than the one that can write production —
object-locked / WORM where the engine supports it (S3 Object Lock, Azure Blob
immutability, GCS Bucket Lock).

**Replicas are not backups.** A read replica or multi-AZ standby replays every
`DELETE` and `DROP TABLE` your primary commits. They protect against *hardware*
and *AZ* failure, not against operator error, ransomware, application bugs, or
`DELETE FROM users WHERE id = $1` with a NULL bind. Treat replicas as an
**availability** mechanism, backups as a **durability** mechanism (Google SRE
ch. 26 "Defense in depth"; AWS Well-Architected REL09).

**Point-in-time recovery (PITR) is mandatory** for any system of record.
Snapshots alone give you RPO = snapshot interval (often 24h). PITR with
WAL/binlog/oplog shipping gives RPO measured in seconds. Every managed
relational and document store ships PITR; turn it on, set retention to ≥14
days, and verify the *oldest restorable timestamp* in monitoring (AWS RDS
"Backup and restore"; Azure SQL "Automated backups"; Google Cloud SQL "PITR").

**Encrypt backups with a key in a different trust domain** than the source DB.
If the same KMS key is used and it's revoked or destroyed, both the DB and the
"backup" become bricks (NIST SP 800-57 Pt. 1, key-segregation guidance).
## 3. Restore drills: the only metric that matters

> "An untested backup is not a backup. It's a hope."

**Schedule restore drills as recurring infrastructure work**, not as a DR
exercise once a year. Quarterly is the floor; monthly is right for tier-0.

A restore drill is not complete until: (1) a full PITR restore to a *new*
instance from the off-site copy succeeded; (2) schema and row-count checksums
match a known-good fixture; (3) the restore *time* was recorded and is within
RTO; (4) an application instance was pointed at the restored DB and a smoke
test passed. Anything less is a "the snapshot exists" check.

Every team should answer two questions from a dashboard: **"When did we last
successfully restore from cold backup?"** and **"How long did it take?"**.
Google SRE ch. 26 calls this the *recoverability SLO*; treat it as a
first-class indicator.
## 4. RPO and RTO: derive from the SLO, then test

- **RPO** — acceptable data loss measured in time.
- **RTO** — acceptable downtime.

Both must be **derived from the customer-facing SLO**, not picked. If your
service has a 99.9% monthly availability SLO, that's ~43 minutes of error
budget; an RTO of 4 hours is incoherent and someone will be blamed for an
outage that math already promised.

**Tests, not declarations.** A documented 5-minute RPO that has never been
measured has a 100% confidence interval of "unknown". Quarterly game-days
where you literally fail over and time it are the only acceptable evidence
(Google SRE Workbook ch. 9; "DBRE" ch. 4).

| Tier | RPO     | RTO     | Topology                                       |
|------|---------|---------|------------------------------------------------|
| 0    | < 30 s  | < 5 min | Multi-region active/active or sync standby     |
| 1    | < 5 min | < 1 h   | Multi-AZ HA + cross-region async replica       |
| 2    | < 1 h   | < 4 h   | Multi-AZ HA + cross-region snapshot            |
| 3    | < 24 h  | < 24 h  | Daily backup, single region                    |

Most workloads are tier 1 or 2. Pretending you're tier 0 is the most common
and most expensive mistake in this chapter.
## 5. HA topologies

Three reference models:

- **Primary + sync/async standby** (Postgres, MySQL, SQL Server). Failover
  seconds-to-minutes; data loss bounded by replication lag for async. This is
  what "Multi-AZ" means on RDS, Azure Flexible Server, Cloud SQL HA.
- **Distributed SQL with consensus** (Spanner, CockroachDB, YugabyteDB,
  Aurora DSQL, TiDB). Quorum writes (Paxos/Raft), no failover ceremony, pay
  in write latency. Aurora classic is a hybrid: shared distributed storage,
  single writer (AWS Aurora paper, SIGMOD 2017).
- **Globally-distributed multi-master with tunable consistency** (Cosmos DB,
  DynamoDB Global Tables, Cassandra, Scylla). Always-writable; conflict
  resolution is *your* problem (LWW, CRDTs, app-level merge). Read Vogels,
  "Eventually Consistent" (ACM Queue 2008) before adopting one.

**Multi-primary RDBMS is hard.** Galera, MySQL Group Replication, BDR — all
work in demos, all have sharp edges Jepsen has documented (Aphyr, Jepsen
analyses of Group Replication and Galera; Kleppmann, *DDIA* ch. 5). If you
think you need multi-primary write-anywhere SQL, you almost certainly want a
distributed-SQL engine, or a single-writer design with regional read replicas.

**Failover must be automated and tested.** A runbook that begins "page the
DBA" has an RTO that is "however long it takes to find the DBA." Use managed
failover (RDS, Patroni, repmgr, orchestrator, MySQL InnoDB Cluster) and
chaos-test it.

**The CAP theorem is not a menu, it's a constraint.** Brewer himself walked
back the popular reading: in practice you trade latency for consistency
during partitions, and the real choice is *latency under load* (Brewer,
"CAP Twelve Years Later", IEEE Computer, 2012).
## 6. Sharding and partitioning

**Start later than you think.** A vertically-scaled Postgres / MySQL on
modern hardware (96 vCPU, 768 GB RAM, NVMe) handles tens of thousands of TPS
and multi-TB datasets. Sharding adds an order of magnitude to operational
complexity; defer until you have measured a wall.

When you do shard: **MySQL → Vitess** (battle-tested at YouTube, Slack,
GitHub; Vitess docs, "VReplication" and "Resharding workflows"). **Postgres
→ Citus**, or a distributed-SQL engine for greenfield with global write
needs. **Cloud-native sharded services** (DynamoDB, Cosmos DB, Bigtable,
Spanner, Aurora DSQL) when available — sharding becomes a $/RU line, not
headcount.

**Pick a shard key once.** Resharding a poorly-keyed system is the most
painful migration in this chapter. The key must be (a) high-cardinality,
(b) present on every hot query, (c) aligned with your tenancy boundary if
you have one. Write it down in an ADR.
## 7. Blast radius

> One database per blast-radius unit. No shared databases across services
> that don't share a deployment lifecycle.

This is the operational restatement of the Twelve-Factor "backing services"
factor (factor IV) and the bounded-context rule from DDD. The shared-DB
anti-pattern — "service A and service B both write to `orders`" — is how a
schema migration in service A pages service B's on-call at 03:00.

Concrete checks: each service owns its database (or a logical DB + dedicated
user) with credentials issued via the service's identity, no cross-service
DB user reuse; cross-service reads go through APIs or events, not the other
team's tables; ETL into a warehouse for analytics; schema migrations are
owned by one service, nobody else has DDL grants. A monolith DB with thirty
services attached is a single blast radius the size of the company.
## 8. Caches

**Patterns:** *cache-aside* (default — read miss fills, writes invalidate;
failure mode is thundering herd, fix with single-flight); *write-through*
(strong freshness, doubles write latency); *write-behind* (fastest, most
dangerous; data loss on cache failure unless the cache is durable);
*refresh-ahead* for predictable hot keys. Pick cache-aside unless you've
measured a reason to do otherwise.

**"There are only two hard things in CS: cache invalidation and naming
things."** — Phil Karlton. Invalidation is fundamentally distributed
coordination; the cheapest correct answer is a short TTL plus event-driven
busts on writes. Avoid relying on TTL alone for correctness; avoid relying
on busts alone for availability.

**Engine choice:** **Redis** for almost everything (data structures,
pub/sub, streams, Lua atomicity, persistence) — managed (ElastiCache,
Memorystore, Azure Cache for Redis); cluster mode for horizontal scale.
**Memcached** when you need *only* a memory-bounded LRU and zero features.
**CDN/HTTP caches** (CloudFront, Front Door, Cloud CDN, Fastly) for
cacheable HTTP — almost always cheaper and faster than an origin-side cache.

**Caches are not databases.** Anything you cannot regenerate from the
system of record does not belong in a cache. Period.
## 9. Message brokers

**Kafka is overkill for most teams.** It's right when you need durable,
replayable, partitioned event logs at high throughput with multiple
independent consumer groups — log aggregation, CDC fan-out, analytics
pipelines, event sourcing at scale. It's wrong when you need a work queue
with retries and a DLQ.

| Need                                            | Pick                                            |
|-------------------------------------------------|-------------------------------------------------|
| Work queue, per-message ack, DLQs               | SQS, Service Bus Queues, Cloud Tasks, RabbitMQ  |
| Pub/sub fan-out, low ops                        | SNS+SQS, Service Bus Topics, Pub/Sub            |
| Durable replayable log, high throughput         | Kafka (MSK/Confluent/Aiven), Pulsar, Kinesis    |
| In-cluster, lightweight, request/reply          | NATS, NATS JetStream                            |
| Database CDC                                    | Debezium → Kafka, or DMS / Datastream           |

**Prefer cloud-native managed brokers** unless you've measured a reason not
to. Self-hosting Kafka well takes a dedicated team — KRaft upgrades,
partition rebalancing, broker tuning — and is the most common "managed
would have been cheaper" story (Kafka docs "Operations"; ThoughtWorks Tech
Radar 2023–2025 entries on Confluent Cloud and self-hosted Kafka caution).

**Exactly-once is a marketing term.** What you get is *effectively-once*
through idempotent consumers + dedupe keys, or transactional outbox
patterns. Design every consumer to be idempotent (Helland, "Life Beyond
Distributed Transactions", CIDR 2007).
## 10. Event sourcing & CQRS — operationally

Event sourcing solves real problems (auditability, temporal queries,
replayable read models) and creates real ones: schema evolution of events
forever, snapshotting, projection rebuild times, GDPR Art. 17 right-to-
erasure against an immutable log. CQRS adds a second store you must monitor
and reconcile.

**Adopt only when** audit/regulatory requirements demand the full event
history, *or* multiple read models materially diverge from the write model,
*or* you have an operational appetite for two stores and projection-lag
SLOs. For most CRUD systems, a normal database with an outbox table and CDC
into Kafka gives you 80% of the benefits at 20% of the cost. (Helland,
"Immutability Changes Everything", CACM 2016, is the honest pitch.)
## 11. Object storage as the universal substrate

S3 (and the S3 API as implemented by GCS, Azure Blob, Cloudflare R2, MinIO,
Wasabi, B2, Ceph RGW) is the closest thing infrastructure has to a
universal primitive. Default for blobs, backups, build artifacts, logs,
analytics input/output, ML datasets, anything large and cold-ish.

**Defaults:**

- **Versioning on** for any bucket holding state you'd cry over. Pair with
  a lifecycle policy that expires noncurrent versions after N days.
- **Lifecycle policies** to tier objects to IA / Cold / Archive by age.
  Most buckets pay 5–10× more than necessary because nobody set this.
- **Object Lock / immutability** on backup, audit, and compliance buckets
  — Compliance-mode WORM where law allows, governance-mode otherwise (S3
  Object Lock; Azure Blob immutability; GCS Bucket Lock).
- **Cross-region replication ≠ versioning.** CRR replicates deletes (unless
  excluded); versioning protects against deletes. You usually want both.
- **Block public access at the account level**, not per-bucket. Every
  major data-leak postmortem of the last decade includes "the bucket was
  public."
- **Server-side encryption with a CMK** for anything sensitive. Default
  encryption is table stakes; key custody is the control.
## 12. Data sovereignty & residency

GDPR, Schrems II, India's DPDP Act, China's PIPL, and sectoral regimes
(HIPAA, PCI-DSS, FedRAMP, FINMA) make region a *first-class* deployment
property, not an afterthought.

- **Region pinning by tenant.** Each tenant has a home region recorded at
  provisioning. Primary stores, backups, and *logs* live in or replicate
  only to permitted regions.
- **Logs and traces are personal data too.** A request log with an IP, a
  user ID, or a header is in scope for GDPR. Pipe regional traffic to
  regional log stores.
- **Encryption keys in-region**, with a key per residency zone. CMKs in
  EU KMS for EU data; same for other regimes. Key material does not cross
  borders.
- **Data subject access / erasure** must be possible in the storage layer:
  document where every copy lives (warehouse, search index, cache, broker,
  backup) and how erasure propagates. Event-sourced systems need a
  crypto-shredding plan (delete the per-subject key, ciphertext becomes
  noise).

Right of erasure against an immutable event log + 35-day Kafka retention +
7-year backup + analytics warehouse is a *design problem you solve up
front*, not a ticket you file when the regulator asks.
## 13. Schema migrations as deploys

Schema migrations are the highest-blast-radius deploys in your system.
Treat them as such.

**Zero-downtime is a property of the migration, not the tool.** The pattern
is **expand / contract** (a.k.a. parallel change, Fowler):

1. **Expand:** add new columns/tables/indexes, write to both, read from old.
2. **Backfill:** migrate existing rows in batches, throttled, idempotent,
   resumable.
3. **Switch reads:** flip read path behind a flag; verify parity.
4. **Contract:** stop writing the old; drop it in a *separate* deploy after
   one full release window.

Never ship a migration that requires the application and the schema to
deploy atomically. That's a maintenance window by another name.

**Tooling:** **Atlas** (Ariga) for declarative schemas, diff-based
migrations, drift detection, CI integration — strong default for new repos
(ThoughtWorks Tech Radar Atlas entries 2023–2024). **Flyway / Liquibase**
— mature, imperative, language-agnostic; Liquibase has structured
changelogs and rollback metadata. **Sqitch** — VCS-style, dependency-graph-
aware, the serious DBA's choice. **ORM-native migrations** (EF Core,
Alembic, Rails, Prisma) — fine for app-owned schemas; do *not* let multiple
services migrate the same DB.

Every migration is reviewed by someone who has run a `pg_repack` at 03:00.
Lock-taking DDL on a 500 GB table is a separate skill from writing SQL.
## 14. Encryption

**At rest:** every managed engine encrypts by default in 2026; verify it's
on, verify the key, verify rotation. Default-encryption with a
provider-managed key is the floor; **customer-managed keys (CMK)** in your
KMS / Key Vault / Cloud KMS are the standard for regulated workloads
(PCI-DSS, HIPAA, FedRAMP; AWS KMS, Azure Key Vault, Google Cloud KMS docs).

**In transit:** TLS 1.2 minimum, 1.3 preferred, on every hop including
intra-VPC database connections. "It's a private network" is not a defence;
the cloud substrate is shared, and lateral-movement attacks routinely cross
VPC boundaries.

**Key custody:** CMK in a separate trust domain from the workload identity
(the account that runs the DB cannot also delete its own backup-encryption
key); per-tenant or per-residency keys for multi-tenant SaaS — enables
crypto-shredding and residency proofs; rotation automated; HSM-backed keys
(CloudHSM, Managed HSM, Cloud HSM) for FIPS 140-2/3 Level 3 requirements.

Avoid application-layer encryption of full database columns *unless* you
have a real threat model that beats KMS-at-rest; you'll lose indexing,
joins, and most query plans, and gain little against realistic attacks
(NIST SP 800-57; "DBRE" ch. 8).
## 15. Stateful workloads on Kubernetes

**Default: don't.** For 90% of teams in 2026, running Postgres / MySQL /
Kafka / Redis / Elasticsearch on Kubernetes is operational masochism that
buys you nothing your cloud provider's managed service doesn't already
provide better and cheaper after headcount.

StatefulSets, CSI drivers, volume snapshot classes, storage-aware
schedulers, and the operator ecosystem (CloudNativePG, Strimzi, Zalando
Postgres Operator) have all matured — but the *failure modes* (PV stuck
detaching, CSI controller wedged, operator reconciler bug eats your
cluster) are exotic in a way managed services don't expose you to.

**Self-host on K8s only when** you're already running a multi-team K8s
platform with a dedicated storage SRE function; *and* you have a hard
requirement (air-gap, on-prem, regulated cloud-region unavailability) that
rules out managed; *and* the workload is itself K8s-native and well-
supported by a mature operator (CloudNativePG for Postgres, Strimzi for
Kafka are the strongest examples), and you've read the Jepsen reports for
the engine.

If you do: dedicated node pools for stateful workloads with taints and
PDBs; local NVMe via CSI for performance-sensitive engines, replicate at
the *application* layer not the storage layer; backups go to **object
storage outside the cluster**, written by an identity the cluster cannot
revoke; restore drills (§3) include "the entire K8s cluster is gone" as a
scenario.

The honest version: most teams that ran Postgres on K8s in 2020–2023
migrated back to managed by 2025. Learn from their gas bills.
## 16. Observability of the data tier

Detail belongs in the observability chapter. The data-tier minimum:
**replication lag** as an SLI with an SLO (for async standbys this is your
real RPO); **backup age & last-successful-restore age** as SLIs with
pages; **query-level metrics** (`pg_stat_statements`, Performance Schema,
slow logs shipped) — not just host CPU/IO; **connection pool saturation**
(the most common "database is down" outage is actually pool exhaustion in
the app tier); **storage growth rate** with capacity projection (Sridharan,
"Distributed Systems Observability", O'Reilly 2018).
## 17. The short list

If you read nothing else: (1) use the managed service, justify self-hosting
in writing; (2) PITR on, ≥14 days, cross-account off-site copy, KMS key in
a separate trust domain; (3) quarterly restore drills, timed, into a fresh
instance, with smoke test, dashboarded; (4) RPO/RTO derived from SLO,
tested by game-day not by assertion; (5) one DB per service, no shared
schemas across services with different release cadences; (6) expand/
contract migrations, reviewed by a DBA-grade reviewer, with an automated
tool (Atlas / Flyway / Liquibase / Sqitch); (7) object storage with
versioning, lifecycle, and Object Lock on anything you'd cry over; (8)
don't run stateful workloads on K8s unless you have a written reason and a
storage SRE. Everything else is detail.
