# 08 — Data & State

Opinionated, cloud-agnostic defaults for running stateful infrastructure in 2026. This chapter is about **operating** databases, caches, brokers, and object stores — not about schema design, ORMs, or query tuning. Application-side data modelling belongs in a different document; here we treat data as a *workload* with an SLO, a blast radius, and a recovery clock.

> Conventions: **Do** = required default. **Don't** = reject in review unless a written exception exists. **Prefer** = strong default; deviate only with measurement or a documented constraint.

The single most expensive lesson in this domain: **the only durability metric that matters is restore time on real data.** Everything else — replication lag, backup success rate, RPO targets in a slide deck — is a leading indicator at best and a comforting lie at worst (Google SRE Book, ch. 26, "Data Integrity: What You Read Is What You Wrote").

## 1. Managed vs self-hosted

**Default to managed.** In 2026, for Postgres, MySQL, Redis, Kafka, object storage, and search, the managed offering from your primary cloud (or a reputable specialist: Aiven, Confluent Cloud, MongoDB Atlas, Neon, PlanetScale) is the right starting point for ≥95% of teams. The operational surface — patching, minor-version upgrades, backup orchestration, failover, TLS rotation, OS-level CVEs — is a *full-time team* you do not have (AWS RDS "Shared responsibility model"; Google Cloud SQL "Overview of high availability"; Campbell & Majors, *Database Reliability Engineering*, ch. 1).

**Self-host only when at least one is true:** (a) regulatory — the data cannot leave your tenancy or a region your provider doesn't serve; (b) cost at scale — past ~$50k/month managed spend on one engine, with a real TCO model that includes the 2+ SREs you'll hire; (c) feature gap — you need an extension, version, or tuning knob the managed service refuses; (d) latency floor — co-tenant placement the provider won't expose. "We're special" and "we want control" are not on the list.

**Don't operate two of the same engine.** One Postgres fleet, one Kafka, one Redis flavor. Multi-engine sprawl is how on-call dies.

## 2. Backups: the 3-2-1 rule, restated

The 3-2-1 rule (US-CERT TA14-353A; Krogh, *The DAM Book*) is the floor: **3** copies, **2** media, **1** off-site. In cloud terms: (1) the live DB, (2) automated managed snapshots in the same account, (3) **cross-account, cross-region** export pushed by a *different* identity than the one that can write production — object-locked / WORM where the engine supports it (S3 Object Lock; Azure Blob immutable storage; GCS Bucket Lock — see §11 for per-provider differences).

**Replicas are not backups.** A read replica or multi-AZ standby replays every `DELETE` and `DROP TABLE` your primary commits. They protect against *hardware* and *AZ* failure, not against operator error, ransomware, application bugs, or `DELETE FROM users WHERE id = $1` with a NULL bind. Treat replicas as an **availability** mechanism, backups as a **durability** mechanism (Google SRE ch. 26 "Defense in depth"; AWS Well-Architected REL09).

**Point-in-time recovery (PITR) is engine-specific; turn it on and read the fine print.** Snapshots alone give RPO = snapshot interval. PITR layers log/WAL replay on top, but mechanics differ:

- **Snapshot + WAL/binlog replay** (RDS, Aurora, Cloud SQL, Azure Flexible Server). PITR always restores to a *new* instance — you cannot rewind the existing one. Latest-restorable-time trails wall-clock by seconds-to-minutes depending on log shipping cadence (AWS RDS "Restoring a DB instance to a specified time"; Cloud SQL "Use point-in-time recovery"; Azure "Backup and restore in Flexible Server").
- **Continuous archiving** (self-hosted Postgres with `pgBackRest`/`WAL-G`, MySQL with binlog streaming). Tighter potential RPO but *you* own the archiver, retention, and integrity checks.
- **Document/wide-column** (MongoDB Atlas continuous backup, DynamoDB PITR). PITR is per-table or per-cluster and **does not give a globally consistent point in time across multiple databases or tables** unless documented. Cross-database referential restores are an application reconciliation problem.

Set retention to ≥14 days, monitor *latest restorable time* as an SLI (not just "backups succeeded"), and rehearse the restore (§3).

**Encrypt backups with a key in a different trust domain** than the source DB. If the same KMS key is used and it's revoked or destroyed, both the DB and the "backup" become bricks (NIST SP 800-57 Pt. 1 Rev. 5, §6 "Key management phases"; AWS KMS "Key policies" — separate `kms:ScheduleKeyDeletion` from workload identity).

## 3. Restore drills: the only metric that matters

> "An untested backup is not a backup. It's a hope."

**Schedule restore drills as recurring infrastructure work**, not as a DR exercise once a year. Quarterly is the floor; monthly is right for tier-0.

A restore drill is not complete until: (1) a full PITR restore to a *new* instance from the off-site copy succeeded; (2) schema and row-count checksums match a known-good fixture; (3) the restore *time* was recorded and is within RTO; (4) an application instance was pointed at the restored DB and a smoke test passed. Anything less is a "the snapshot exists" check.

Every team should answer two questions from a dashboard: **"When did we last successfully restore from cold backup?"** and **"How long did it take?"**. Google SRE ch. 26 calls this the *recoverability SLO*; treat it as a first-class indicator.

## 4. RPO and RTO: derive from impact, then test

- **RPO** — acceptable data loss measured in time.
- **RTO** — acceptable downtime.

Both are **business decisions informed by three inputs together**, not a formula derived from the availability SLO alone:

- **User-visible impact** of data loss vs unavailability (a 5-minute outage is usually cheaper than 5 minutes of lost writes for payments; the reverse is often true for content).
- **The error budget** for the customer-facing SLO — sets the ceiling on how often a slow restore can happen before you burn the quarter's budget.
- **Recoverability constraints** — what the engine, backup pipeline, and staffing model can actually deliver under load, measured (Google SRE Workbook ch. 9; *DBRE* ch. 4).

A 99.9% monthly availability SLO does not, by itself, *imply* a particular RTO; a 4-hour RTO can be entirely coherent if data-loss events are rare and the business tolerates one long outage per year over many short ones. What is *not* coherent is choosing an RTO you have never measured.

**Tests, not declarations.** A documented 5-minute RPO that has never been measured has a 100% confidence interval of "unknown". Quarterly game-days where you literally fail over and time it are the only acceptable evidence.

| Tier | RPO     | RTO     | Topology (default)                             |
|------|---------|---------|------------------------------------------------|
| 0    | < 30 s  | < 5 min | Sync standby or distributed-SQL quorum         |
| 1    | < 5 min | < 1 h   | Multi-AZ HA + cross-region async replica       |
| 2    | < 1 h   | < 4 h   | Multi-AZ HA + cross-region snapshot            |
| 3    | < 24 h  | < 24 h  | Daily backup, single region                    |

Most workloads are tier 1 or 2. Pretending you're tier 0 is the most common and most expensive mistake in this chapter.

## 5. HA topologies

Three reference models:

- **Primary + sync/async standby** (Postgres, MySQL, SQL Server). Failover seconds-to-minutes; data loss bounded by replication lag for async. This is "Multi-AZ" on RDS, Azure Flexible Server, Cloud SQL HA — and is the **correct default for the vast majority of workloads**.
- **Distributed SQL with consensus** (Spanner, CockroachDB, YugabyteDB, Aurora DSQL, TiDB). Quorum writes (Paxos/Raft), no failover ceremony, pay in write latency. Aurora classic is a hybrid: shared distributed storage, single writer (Verbitski et al., "Amazon Aurora", SIGMOD 2017).
- **Globally-distributed multi-master with tunable consistency** (Cosmos DB, DynamoDB Global Tables, Cassandra, Scylla). Always-writable; conflict resolution is *your* problem (LWW, CRDTs, app-level merge). Read Vogels, "Eventually Consistent" (ACM Queue 2008) before adopting one.

**Multi-region active/active is an exception, not a default.** Adopt it only with a quantified justification: a documented business loss per minute of regional outage, a regulator that mandates it, or measured exposure to a provider's regional incidents. The cost is real — write latency, conflict handling, double the compliance surface, and split-brain bug classes that don't exist in single-writer designs. **Multi-AZ HA + a tested cross-region restore is the right answer for the overwhelming majority of tier-1 systems.**

**Multi-primary RDBMS is hard.** Galera, MySQL Group Replication, and BDR all work in demos and have specific, documented failure modes under partition and failover. See Kyle Kingsbury's Jepsen analyses of *MySQL 8.0.34* (jepsen.io/analyses/mysql-8.0.34, 2023 — lost-update under `READ COMMITTED`/`REPEATABLE READ`) and *MariaDB Galera Cluster 10.x* (jepsen.io/analyses/galera — write-skew and stale reads under partitions); also Kleppmann, *DDIA*, ch. 5. If you think you need multi-primary write-anywhere SQL, you almost certainly want a distributed-SQL engine, or a single-writer design with regional read replicas.

**Failover must be automated and tested.** A runbook that begins "page the DBA" has an RTO that is "however long it takes to find the DBA." Use managed failover (RDS, Patroni, repmgr, orchestrator, MySQL InnoDB Cluster) and chaos-test it.

**The CAP theorem is not a menu, it's a constraint.** Brewer himself walked back the popular reading: in practice you trade latency for consistency during partitions, and the real choice is *latency under load* (Brewer, "CAP Twelve Years Later", IEEE Computer, 2012).

## 6. Sharding and partitioning

**Start later than you think.** A vertically-scaled Postgres / MySQL on modern hardware (96 vCPU, 768 GB RAM, NVMe) handles tens of thousands of TPS and multi-TB datasets. Sharding adds an order of magnitude to operational complexity; defer until you have measured a wall.

When you do shard, **gate the choice with a workload test**, not a logo:

- **MySQL → Vitess** is battle-tested at YouTube, Slack, and GitHub, but cross-shard transactions are limited (Vitess "Two-Phase Commit" is opt-in and slower; default is best-effort) and SQL compatibility is a *subset* — some joins, subqueries, and `INFORMATION_SCHEMA` patterns behave differently (Vitess docs, "MySQL Compatibility"). Run your real query mix through Vitess in staging before committing.
- **Postgres → Citus** wins when queries co-locate on a single distribution key (multi-tenant SaaS where `tenant_id` is on every hot path); it is *not* a transparent drop-in for arbitrary Postgres workloads. Cross-shard joins and DDL have caveats; foreign keys across distributed tables require the reference column to be the distribution key (Citus docs, "Determining the Application Type" and "Reference Tables"). Validate with `EXPLAIN` on top-N queries.
- **Cloud-native sharded services** (DynamoDB, Cosmos DB, Bigtable, Spanner, Aurora DSQL) when available — sharding becomes a $/RU line, not headcount.

**Pick a shard key once.** Resharding a poorly-keyed system is the most painful migration in this chapter. The key must be (a) high-cardinality, (b) present on every hot query, (c) aligned with your tenancy boundary if you have one. Write it down in an ADR.

## 7. Blast radius

> One database per blast-radius unit. No shared databases across services that don't share a deployment lifecycle.

This is the operational restatement of the Twelve-Factor "backing services" factor (factor IV) and the bounded-context rule from DDD. The shared-DB anti-pattern — "service A and service B both write to `orders`" — is how a schema migration in service A pages service B's on-call at 03:00.

Concrete checks: each service owns its database (or a logical DB + dedicated user) with credentials issued via the service's identity, no cross-service DB user reuse; cross-service reads go through APIs or events, not the other team's tables; ETL into a warehouse for analytics; schema migrations are owned by one service, nobody else has DDL grants. A monolith DB with thirty services attached is a single blast radius the size of the company.

## 8. Caches

**Patterns:** *cache-aside* (default — read miss fills, writes invalidate; thundering-herd failure mode, fix with single-flight); *write-through* (strong freshness, doubles write latency); *write-behind* (fastest, most dangerous; data loss on cache failure unless the cache is durable); *refresh-ahead* for predictable hot keys. Pick cache-aside unless you've measured a reason to do otherwise.

**"There are only two hard things in CS: cache invalidation and naming things."** — Phil Karlton. Invalidation is fundamentally distributed coordination; the cheapest correct answer is a short TTL plus event-driven busts on writes. Avoid relying on TTL alone for correctness; avoid relying on busts alone for availability.

**Engine choice:** **Redis** for almost everything (data structures, pub/sub, streams, Lua atomicity, persistence) — managed (ElastiCache, Memorystore, Azure Cache for Redis); cluster mode for horizontal scale. **Memcached** when you need *only* a memory-bounded LRU and zero features. **CDN/HTTP caches** (CloudFront, Front Door, Cloud CDN, Fastly) for cacheable HTTP — almost always cheaper and faster than an origin-side cache.

**Caches are not databases.** Anything you cannot regenerate from the system of record does not belong in a cache. Period.

## 9. Message brokers

**Kafka is overkill for most teams.** It's right when you need durable, replayable, partitioned event logs at high throughput with multiple independent consumer groups — log aggregation, CDC fan-out, analytics pipelines, event sourcing at scale. It's wrong when you need a work queue with retries and a DLQ.

| Need                                            | Pick                                            |
|-------------------------------------------------|-------------------------------------------------|
| Work queue, per-message ack, DLQs               | SQS, Service Bus Queues, Cloud Tasks, RabbitMQ  |
| Pub/sub fan-out, low ops                        | SNS+SQS, Service Bus Topics, Pub/Sub            |
| Durable replayable log, high throughput         | Kafka (MSK/Confluent/Aiven), Pulsar, Kinesis    |
| In-cluster, lightweight, request/reply          | NATS, NATS JetStream                            |
| Database CDC                                    | Debezium → Kafka, or DMS / Datastream           |

**Kafka vs Pulsar — a short note.** Kafka couples broker compute and storage on the same node (per-partition log files); Pulsar separates compute (brokers) from storage (BookKeeper bookies), making per-tenant elasticity and tiered storage cleaner but adding a second distributed system to operate (Apache Pulsar docs, "Architecture overview"; Apache Kafka docs, "KRaft" and "Tiered Storage" — KIP-405). Pulsar's geo-replication is built into the broker (per-topic via `pulsar-admin namespaces set-clusters`); Kafka's cross-cluster replication is external — **MirrorMaker 2** or **Confluent Replicator** with its own operational footprint. **Kafka usually wins on ecosystem maturity** (Connect, Streams, ksqlDB, Schema Registry, the connector catalogue). Pick Pulsar when multi-tenancy isolation, geo-replication, or storage/compute separation is a *measured* requirement; otherwise the marginal complexity isn't paid back.

**Prefer cloud-native managed brokers.** Self-hosting Kafka well takes a dedicated team — KRaft upgrades, partition rebalancing, broker tuning — and is the most common "managed would have been cheaper" story.

**Exactly-once is a marketing term.** What you get is *effectively-once* through idempotent consumers + dedupe keys, or transactional outbox patterns (§11). Design every consumer to be idempotent (Helland, "Life Beyond Distributed Transactions", CIDR 2007).

## 10. Schema evolution & compatibility

Schema evolution applies to **events on the wire** as much as to database DDL (§14). The failure mode is the same: a producer ships a change, a consumer crashes in production, and the rollback is a release window. The discipline is *contracts, registered and enforced*.

- **Use a schema registry** for any non-trivial event bus — Confluent Schema Registry, AWS Glue Schema Registry, Azure Schema Registry, Apicurio. Pick Avro, Protobuf, or JSON Schema; the registry enforces compatibility on publish (Confluent docs, "Schema Evolution and Compatibility").
- **Default to backward-compatible changes:** add optional fields with defaults, never remove or rename a field, never change a type, never change the meaning of an enum value. Field numbers/positions are permanent (Google Protocol Buffers docs, "Updating A Message Type").
- **Consumer-first vs producer-first rollouts.** For a *backward-compatible* change (new readers can read old data), roll **consumers first**: deploy the new consumer, verify it still handles old events, then ship the producer. For a *forward-compatible* change (old readers can read new data — rarer), roll **producers first**. **Breaking changes** (incompatible in either direction) require a parallel topic / new subject + dual-publish + drain old consumers + retire — the event-bus analogue of expand/contract (§14).
- **Version events explicitly** in the topic or schema name (`orders.v2`) when a breaking change is unavoidable. Never silently re-purpose a topic.
- **Compatibility mode is a registry setting, not a convention.** Configure `BACKWARD` (or `BACKWARD_TRANSITIVE` if you replay history) and let CI fail builds that violate it.

## 11. Change data capture & the outbox

CDC is how you escape **dual writes** — the anti-pattern where application code writes to a database *and* publishes an event, and the second write fails. There is no `try/catch` that fixes this; the database's write log must *be* the source of truth.

- **Log-based CDC.** Read the engine's replication log directly: **Debezium** for Postgres logical decoding (`pgoutput`), MySQL binlog (`ROW` format), MongoDB oplog, SQL Server CDC tables (Debezium docs, "Connector for PostgreSQL"; AWS DMS / GCP Datastream are managed equivalents). Provides ordered, at-least-once delivery of every committed change with no application code changes. Watch for: replication-slot bloat on Postgres if a consumer falls behind (the WAL is retained until the slot advances), schema-change handling, and initial-snapshot load on large tables.
- **Transactional outbox.** When you need to publish an event *as part of* a business transaction, write it to an `outbox` table in the same DB transaction, and have a separate process (often Debezium tailing that table) ship rows to the broker (Richardson, *Microservices Patterns*, ch. 3, "Transactional Outbox"). The right answer when "did the customer get charged AND the order-placed event get published?" must be atomic.
- **Avoid dual writes.** If you find yourself writing `db.save(x); broker.publish(x)` in app code, you have a bug waiting for the next network blip. Replace with outbox + CDC.
- **Idempotency keys on every consumer.** Every event must carry a stable ID (often source PK + change LSN), and consumers must dedupe on it. CDC is at-least-once; replays happen on every restart, failover, or slot reset.
- **Tombstones for deletes.** Compacted Kafka topics use a `null` value as a tombstone to signal deletion downstream (Apache Kafka docs, "Log Compaction"). Consumers must handle them; sinks (search indexes, caches, materialized views) must propagate the delete or you've built a GDPR Art. 17 problem (§13).

## 12. Event sourcing & CQRS — operationally

Event sourcing solves real problems (auditability, temporal queries, replayable read models) and creates real ones: schema evolution of events forever (§10), snapshotting, projection rebuild times, GDPR Art. 17 right-to-erasure against an immutable log (§13). CQRS adds a second store you must monitor and reconcile.

**Adopt only when** audit/regulatory requirements demand the full event history, *or* multiple read models materially diverge from the write model, *or* you have an operational appetite for two stores and projection-lag SLOs. For most CRUD systems, a normal database with an outbox table and CDC into Kafka (§11) gives you 80% of the benefits at 20% of the cost. (Helland, "Immutability Changes Everything", CACM 2016.)

## 13. Object storage as the universal substrate

Object storage is the closest thing infrastructure has to a universal primitive — default for blobs, backups, build artifacts, logs, analytics input/output, ML datasets, anything large and cold-ish. Two distinctions matter and the chapter used to blur them:

- **S3-compatible API ≠ feature parity.** R2, MinIO, Wasabi, B2, Ceph RGW implement most of the S3 *wire protocol* and core verbs, with varying coverage of newer features (multipart edge cases, conditional writes, SSE-KMS, request-payer). GCS and Azure Blob expose **S3-compatible endpoints** for interop (GCS "XML API"; Azure Blob "S3 API support"), but their *native* APIs and feature models are distinct. Test against the specific provider, not "S3".
- **Object Lock-equivalent retention is provider-specific.** "Immutable" means different things:
  - **AWS S3 Object Lock** is **per-object-version**; requires versioning on; supports Governance mode (bypassable with `s3:BypassGovernanceRetention`) and Compliance mode (no bypass, even by root, until retention expires) plus Legal Hold (S3 docs, "Using S3 Object Lock").
  - **GCS Bucket Lock** sets a **bucket-wide** retention period that applies to every object until it has aged past the policy; once *locked*, it cannot be reduced or removed (GCS docs, "Retention policies and Bucket Lock").
  - **Azure Blob immutable storage** supports **time-based retention** and **legal hold**, configured at container or version level, with **locked vs unlocked** policies — unlocked policies are mutable (for testing); only locked policies are WORM (Azure docs, "Immutable storage for Blob Storage").

**Defaults:**

- **Versioning on** for any bucket holding state you'd cry over. Pair with a lifecycle policy that expires noncurrent versions after N days. S3 Object Lock requires versioning; deleting a versioned object creates a delete marker, not a true deletion.
- **Lifecycle policies** to tier objects to IA / Cold / Archive by age. Most buckets pay 5–10× more than necessary because nobody set this.
- **Object Lock / immutability** on backup, audit, and compliance buckets — Compliance-mode WORM (S3) / locked policy (Azure) / locked Bucket Lock (GCS) where law allows; governance / unlocked / shorter retention otherwise.
- **Cross-region replication ≠ versioning.** CRR replicates deletes (unless excluded); versioning protects against deletes. You usually want both, and the replica bucket in a separate account with its own Object Lock.
- **Block public access at the account level**, not per-bucket. Every major data-leak postmortem of the last decade includes "the bucket was public."
- **Server-side encryption with a CMK** for anything sensitive. Default encryption is table stakes; key custody is the control.

## 14. Data sovereignty & residency

GDPR, Schrems II, India's DPDP Act, China's PIPL, and sectoral regimes (HIPAA, PCI-DSS, FedRAMP, FINMA) make region a *first-class* deployment property, not an afterthought.

- **Region pinning by tenant.** Each tenant has a home region recorded at provisioning. Primary stores, backups, and *logs* live in or replicate only to permitted regions.
- **Logs and traces are personal data too.** A request log with an IP, user ID, or header is in scope for GDPR. Pipe regional traffic to regional log stores.
- **Encryption keys in-region**, with a key per residency zone. CMKs in EU KMS for EU data; same for other regimes. Key material does not cross borders.

**GDPR Art. 17 is not a `DELETE` statement.** The "right to erasure" (Regulation (EU) 2016/679, Art. 17) has explicit exceptions (legal obligation, public interest, legitimate-interest balancing) and the European Data Protection Board has consistently treated **backups pragmatically**: data must be put **beyond use** in the live system immediately, while backup copies may persist until they age out under a documented retention schedule, provided they are not selectively restored to evade the request (EDPB Guidelines 4/2019; UK ICO, "Right to erasure" guidance — same posture).

For event-sourced systems and immutable logs, **crypto-shredding** is defensible *only when*: each subject's PII is encrypted with an isolated, per-subject (or per-tenant) key; that key is held in a KMS where revocation is auditable and irreversible; and **no plaintext replicas** of the PII exist downstream — no warehouse copy, no search index, no cache, no analytics extract — because shredding the key only renders ciphertext useless, not plaintext elsewhere (NIST SP 800-88 Rev. 1 "Cryptographic Erase"; ENISA, "Pseudonymisation Techniques and Best Practices", 2019). If you can't satisfy all three, plan for *logical erasure* (overwriting records with tombstones / hashed PII) plus retention rolloff — and document the gap to your DPO.

## 15. Schema migrations as deploys

Schema migrations are the highest-blast-radius deploys in your system. Treat them as such.

**Zero-downtime is a property of the migration, not the tool.** The pattern is **expand / contract** (a.k.a. parallel change, Fowler):

1. **Expand:** add new columns/tables/indexes, write to both, read from old.
2. **Backfill:** migrate existing rows in batches, throttled, idempotent, resumable.
3. **Switch reads:** flip read path behind a flag; verify parity.
4. **Contract:** stop writing the old; drop it in a *separate* deploy after one full release window.

Never ship a migration that requires the application and the schema to deploy atomically. That's a maintenance window by another name.

**Tooling:** **Atlas** (Ariga) for declarative schemas, diff-based migrations, drift detection, CI integration — strong default for new repos. **Flyway / Liquibase** — mature, imperative, language-agnostic; Liquibase has structured changelogs and rollback metadata. **Sqitch** — VCS-style, dependency-graph-aware, the serious DBA's choice. **ORM-native migrations** (EF Core, Alembic, Rails, Prisma) — fine for app-owned schemas; do *not* let multiple services migrate the same DB.

Every migration is reviewed by someone who has run a `pg_repack` at 03:00. Lock-taking DDL on a 500 GB table is a separate skill from writing SQL.

## 16. Encryption

**At rest:** every managed engine encrypts by default in 2026; verify it's on, verify the key, verify rotation. Default-encryption with a provider-managed key is the floor; **customer-managed keys (CMK)** in your KMS / Key Vault / Cloud KMS are the standard for regulated workloads (PCI-DSS v4.0 Req. 3; HIPAA Security Rule §164.312(a)(2)(iv); FedRAMP SC-28; AWS KMS, Azure Key Vault, Google Cloud KMS docs).

**In transit:** TLS 1.2 minimum, 1.3 preferred, on every hop including intra-VPC database connections. "It's a private network" is not a defence; the cloud substrate is shared, and lateral-movement attacks routinely cross VPC boundaries.

**Key custody:** CMK in a separate trust domain from the workload identity (the account that runs the DB cannot also delete its own backup-encryption key); per-tenant or per-residency keys for multi-tenant SaaS — enables crypto-shredding (§14) and residency proofs; rotation automated; HSM-backed keys (CloudHSM, Managed HSM, Cloud HSM) for FIPS 140-3 Level 3 requirements.

Avoid application-layer encryption of full database columns *unless* you have a real threat model that beats KMS-at-rest; you'll lose indexing, joins, and most query plans, and gain little against realistic attacks.

## 17. Day-2 operations

The boring work that determines whether the system survives year two.

- **Physical vs logical backups.** Physical (`pg_basebackup`, Percona XtraBackup, RDS snapshots) copies data files; fast to take and restore at the same major version, useless for cross-version moves or single-table extraction. Logical (`pg_dump`, `mysqldump`, `mongodump`) exports rows; portable and selective, slow on large datasets, **not consistent across databases** without explicit transactional snapshotting (PostgreSQL docs, "Backup and Restore", §26.1 vs §26.2). Run both: physical for fast DR, logical-per-tenant for partial restores and migrations.
- **Key escrow and recoverability.** A CMK that nobody can use is ransomware you wrote yourself. Document who can recover the key, where the recovery quorum lives, and rehearse it. AWS KMS, Cloud KMS, and Key Vault all support multi-principal grants and recovery windows — use them.
- **Restore drills by region.** Drills (§3) must include the **target region** explicitly: "restore Postgres-prod from cross-region S3 backup into eu-west-1 within RTO". A restore that only works in your home region is a regional outage waiting to be a company outage.
- **Hot / warm / cold tiering.** S3 Standard → Standard-IA → Glacier Instant → Glacier Deep Archive (and Azure/GCS equivalents) cut storage cost 5–20× — but each tier has retrieval latency, minimum-storage duration, and per-request retrieval fees that can dwarf the savings if you tier hot data. Model retrieval costs against access patterns before setting lifecycle rules (AWS S3 docs, "Storage Classes"; GCS "Storage classes").
- **Vacuum, autovacuum, and bloat (Postgres).** MVCC means every UPDATE/DELETE leaves dead tuples; autovacuum reclaims them and updates the visibility map and statistics. Misconfigured autovacuum (default thresholds on large tables, long-running transactions blocking cleanup, `xid` wraparound risk) is the single most common Postgres incident (PostgreSQL docs, "Routine Vacuuming"). Monitor `n_dead_tup`, `pg_stat_progress_vacuum`, age of the oldest transaction, and tune per-table autovacuum on hot tables.
- **Index bloat and rebuild.** Indexes also bloat under churn; rebuild online with `REINDEX CONCURRENTLY` (PG ≥12) or `pg_repack`. Track index size vs table size as an SLI on tier-0 tables.
- **Connection pooling.** Postgres' per-connection memory cost makes raw app→DB connections an outage vector; put **PgBouncer** (transaction pooling for most workloads) or RDS Proxy / Cloud SQL Auth Proxy in front. Pool saturation, not the database, is the most common "database is down" page (PgBouncer docs, "Pooling modes" — note transaction pooling breaks session-level features like `SET`, prepared statements pre-PG14, and advisory locks).
- **Stale reads from replicas.** Async read replicas always lag. Read-after-write from a replica returns yesterday's data. Either route reads-after-writes to the primary, use synchronous replicas for the affected paths, or expose the lag to the application so it can wait or fall back. Aurora replicas, Cloud SQL read replicas, and Postgres streaming replicas all document this; the application has to *use* the documentation.

## 18. Stateful workloads on Kubernetes

**Default: don't.** For 90% of teams in 2026, running Postgres / MySQL / Kafka / Redis / Elasticsearch on Kubernetes is operational masochism that buys you nothing your cloud provider's managed service doesn't already provide better and cheaper after headcount.

StatefulSets, CSI drivers, volume snapshot classes, storage-aware schedulers, and the operator ecosystem (CloudNativePG, Strimzi, Zalando Postgres Operator) have all matured — but the *failure modes* (PV stuck detaching, CSI controller wedged, operator reconciler bug eats your cluster) are exotic in a way managed services don't expose you to.

**Self-host on K8s only when** you're already running a multi-team K8s platform with a dedicated storage SRE function; *and* you have a hard requirement (air-gap, on-prem, regulated cloud-region unavailability) that rules out managed; *and* the workload is itself K8s-native and well-supported by a mature operator (CloudNativePG for Postgres, Strimzi for Kafka are the strongest examples), and you've read the Jepsen reports for the engine.

If you do: dedicated node pools for stateful workloads with taints and PDBs; local NVMe via CSI for performance-sensitive engines, replicate at the *application* layer not the storage layer; backups go to **object storage outside the cluster**, written by an identity the cluster cannot revoke; restore drills (§3) include "the entire K8s cluster is gone" as a scenario.

The honest version: most teams that ran Postgres on K8s in 2020–2023 migrated back to managed by 2025. Learn from their gas bills.

## 19. Observability of the data tier

Detail belongs in the observability chapter. The data-tier minimum: **replication lag** as an SLI with an SLO (for async standbys this is your real RPO); **backup age & last-successful-restore age** as SLIs with pages; **latest restorable timestamp** for PITR engines (§2); **query-level metrics** (`pg_stat_statements`, Performance Schema, slow logs shipped) — not just host CPU/IO; **connection pool saturation** (the most common "database is down" outage is actually pool exhaustion in the app tier); **autovacuum progress and dead-tuple counts** (Postgres); **storage growth rate** with capacity projection (Sridharan, *Distributed Systems Observability*, O'Reilly 2018).

## 20. The short list

If you read nothing else: (1) use the managed service, justify self-hosting in writing; (2) PITR on, ≥14 days, cross-account off-site copy, KMS key in a separate trust domain — and know your engine's PITR mechanics; (3) quarterly restore drills, timed, into a fresh instance *and target region*, with smoke test, dashboarded; (4) RPO/RTO derived from impact + budget + recoverability *together*, tested by game-day not by assertion; (5) one DB per service, no shared schemas across services with different release cadences; (6) multi-AZ + tested restore is the default; multi-region active/active is an exception with a written justification; (7) shard only after measuring a wall, and gate Vitess / Citus with workload tests; (8) events get a schema registry and expand/contract rollouts, just like DDL; (9) CDC + outbox instead of dual writes, idempotent consumers everywhere; (10) expand/contract migrations, reviewed by a DBA-grade reviewer, with an automated tool (Atlas / Flyway / Liquibase / Sqitch); (11) object storage with versioning, lifecycle, and provider-appropriate immutability — know the difference between S3 Object Lock, GCS Bucket Lock, and Azure immutable storage; (12) GDPR Art. 17 against backups means beyond-use + retention rolloff, not surgical deletion; crypto-shredding only with isolated subject keys and no plaintext replicas; (13) day-2 basics — vacuum, PgBouncer, key escrow, stale-read awareness — are not optional; (14) don't run stateful workloads on K8s unless you have a written reason and a storage SRE. Everything else is detail.
