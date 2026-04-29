# 08 — Data & State

Opinionated, cloud-agnostic defaults for running stateful infrastructure in 2026. This chapter is about **operating** databases, caches, brokers, and object stores — not about schema design, ORMs, or query tuning. Application-side data modelling belongs elsewhere; here we treat data as a *workload* with an SLO, a blast radius, and a recovery clock.

> Conventions: **Do** = required default. **Don't** = reject in review unless a written exception exists. **Prefer** = strong default; deviate only with measurement or a documented constraint.

The single most expensive lesson in this domain: **the only durability metric that matters is restore time on real data.** Everything else — replication lag, backup success rate, RPO targets in a slide deck — is a leading indicator at best and a comforting lie at worst (Google SRE Book, ch. 26, "Data Integrity: What You Read Is What You Wrote").

## 1. Managed vs self-hosted

**Default to managed.** In 2026, for Postgres, MySQL, Redis/Valkey, Kafka, object storage, and search, the managed offering from your primary cloud (or a reputable specialist: Aiven, Confluent Cloud, MongoDB Atlas, Neon, PlanetScale, Crunchy Bridge, Timescale Cloud) is the right starting point **unless one of the disqualifying conditions below applies**. The operational surface — patching, minor-version upgrades, backup orchestration, failover, TLS rotation, OS-level CVEs — is a *full-time team* you do not have (AWS RDS *Shared responsibility model*, https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.SecuringRDSResources.html ; Google Cloud SQL *Overview of high availability*, https://cloud.google.com/sql/docs/postgres/high-availability ; Campbell & Majors, *Database Reliability Engineering*, ch. 1).

**Self-host (or move to a non-hyperscaler vendor) only when one of these documented exceptions applies** — phrase the ADR as a checkbox against this list, not "we want control":

- **Regulatory / sovereignty** — data cannot leave a tenancy or region your provider doesn't serve (EU sovereign cloud, air-gapped, government, certain health/finance regimes; see §14).
- **Cost at scale** — past ~$50k/month managed spend on one engine, with a TCO model that includes 2+ SREs and the egress bill. Discord's ScyllaDB migration off managed Cassandra ("How Discord Stores Trillions of Messages", Discord Engineering, Mar 2023) is the canonical published example; analytics-tier ClickHouse / DuckDB / Iceberg-on-S3 vs a managed warehouse is another.
- **Blocked extension or version** — RDS Postgres still does not support TimescaleDB or Citus (AWS RDS Postgres "Supported extensions"); Cloud SQL trails Postgres minor versions; if your workload genuinely needs the missing piece and the gap is documented, self-host.
- **License / redistribution rights** — if you ship the engine inside a customer-deployed product, the SSPL/RSALv2 (Redis), ELv2 (Elastic), BSL (CockroachDB <24), and Confluent Community License all matter. Forks (Valkey, OpenSearch, Redpanda) exist precisely to cover this case.
- **Latency floor** — co-tenant placement (same rack / same NUMA node) the provider won't expose, validated with measurement.

"We're special" and "we want control" are not on the list. For most teams below ~$500k/yr in DB spend, the ops-FTE cost of self-hosting still dwarfs the cloud premium.

**Don't operate two of the same engine.** One Postgres fleet, one Kafka, one cache flavor. Multi-engine sprawl is how on-call dies.

## 2. Backups: the 3-2-1 rule, restated

The 3-2-1 rule (US-CERT TA14-353A; Krogh, *The DAM Book*) is the floor: **3** copies, **2** media, **1** off-site. In cloud terms: (1) the live DB, (2) automated managed snapshots in the same account, (3) **cross-account, cross-region** export pushed by a *different* identity than the one that can write production — object-locked / WORM where the engine supports it (S3 Object Lock; Azure Blob immutable storage; GCS Bucket Lock — see §13 for per-provider differences).

**Replicas are not backups.** A read replica or multi-AZ standby replays every `DELETE` and `DROP TABLE` your primary commits. They protect against *hardware* and *AZ* failure, not against operator error, ransomware, application bugs, or `DELETE FROM users WHERE id = $1` with a NULL bind. Treat replicas as an **availability** mechanism, backups as a **durability** mechanism (Google SRE ch. 26 "Defense in depth"; AWS Well-Architected REL09).

**Encrypt backups with a key in a different trust domain** than the source DB. If the same KMS key is used and it's revoked or destroyed, both the DB and the "backup" become bricks (NIST SP 800-57 Pt. 1 Rev. 5, §6 "Key management phases"; AWS KMS "Key policies" — separate `kms:ScheduleKeyDeletion` from workload identity).

### 2.1 PITR is engine-specific — read the fine print

Snapshots alone give RPO = snapshot interval. PITR layers log/WAL replay on top, but the mechanics, RPO floor, restore target, and consistency guarantees differ sharply by engine. The single bullet "PITR on, ≥14 days" is not enough; this is the table to keep on the wall:

| Engine                          | RPO floor          | Restore target            | Cross-DB consistency           | Engine gotcha                                                                 |
|---------------------------------|--------------------|---------------------------|--------------------------------|-------------------------------------------------------------------------------|
| RDS (non-Aurora) Postgres/MySQL | ~5 min             | New instance              | N/A (one DB per instance)      | Tx logs uploaded to S3 every ~5 min; `LatestRestorableTime` lags by that much |
| Aurora (classic)                | ~1 s               | New cluster               | Yes (one cluster)              | Continuous archive to distributed storage; backtrack is in-place but limited  |
| Aurora DSQL                     | ~1 s               | New cluster (PITR preview)| Cluster-wide                   | Different model — see §5; check current preview limits                        |
| Cloud SQL Postgres/MySQL        | seconds            | New (cloned) instance     | N/A                            | RFC3339 timestamp; logical-replication slots **not** preserved across restore |
| AlloyDB                         | seconds            | New cluster               | Cluster-wide                   | Continuous backup; restore is always to a new cluster                         |
| Azure SQL / Flexible Server     | seconds            | New server                | Per-DB                         | Differential snapshot + log; long-term retention separate                     |
| RDS for SQL Server              | ~1 s               | New instance              | **Per-DB ±1 s, not atomic**    | Cross-DB transactions can be inconsistent; `BULK_LOGGED` silently breaks PITR |
| DynamoDB                        | ~1 s               | New table                 | Per-table only                 | Cross-table referential restores are an app reconciliation problem            |
| Cosmos DB (continuous backup)   | ~100 s (async)     | **New account**           | Per-account                    | Cannot restore in place; restore-account RBAC is its own onboarding           |
| MongoDB Atlas                   | ~1 s (oplog)       | New cluster               | Per-cluster                    | Sharded clusters need `pointInTime` queries; cross-shard atomicity caveats    |
| Self-hosted PG (`pgBackRest`)   | depends on archive | Anywhere you choose       | Yes                            | You own archiver, retention, integrity checks, and the on-call for them       |

Set retention to ≥14 days, **alert on `LatestRestorableTime` (or engine equivalent) drifting beyond its expected lag** — not just "backup job succeeded" — and rehearse the restore (§3). A backup that can only be restored in your home region or to a specific account is a regional outage waiting to be a company outage.

## 3. Restore drills: the only metric that matters

> "An untested backup is not a backup. It's a hope."

There are **two distinct activities** the industry calls "restore drills", and they belong to different chapters:

- **Automated restore verification** (weekly, scripted, checksum compare) — owned by ch09 §12. Catches silent corruption and broken backup pipelines on a short loop.
- **Timed end-to-end DR drill** (this section) — **quarterly is the floor, monthly for tier-0**, with restore time recorded against RTO and a smoke test pointed at the restored DB. Catches the things only a real game-day finds: runbook gaps, IAM holes, cross-account/region permissions, staffing.

**Schedule both as recurring infrastructure work**, not as a DR exercise once a year.

A timed end-to-end DR drill is not complete until: (1) a full PITR restore to a *new* instance from the off-site copy succeeded; (2) schema and row-count checksums match a known-good fixture; (3) the restore *time* was recorded and is within RTO; (4) an application instance was pointed at the restored DB and a smoke test passed; (5) the **target region was different from the source** at least once per year. Anything less is a "the snapshot exists" check.

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

## 5. HA topologies and the managed-DB picker

Three reference models:

- **Primary + sync/async standby** (Postgres, MySQL, SQL Server). Failover seconds-to-minutes; data loss bounded by replication lag for async. This is "Multi-AZ" on RDS, Azure Flexible Server, Cloud SQL HA, AlloyDB — and is the **correct default for the vast majority of workloads**.
- **Distributed SQL with consensus** (Spanner, CockroachDB, YugabyteDB, **Aurora DSQL**, TiDB). Quorum writes (Paxos for Spanner, Raft per range/tablet for the others), no failover ceremony, paid for in write latency. Aurora classic is a hybrid: shared distributed storage, single writer (Verbitski et al., "Amazon Aurora", SIGMOD 2017).
- **Globally-distributed multi-master with tunable consistency** (Cosmos DB, DynamoDB Global Tables, Cassandra, Scylla). Always-writable; conflict resolution is *your* problem (LWW, CRDTs, app-level merge). Read Vogels, "Eventually Consistent" (ACM Queue 2008) before adopting one.

### 5.1 Distributed SQL — what you actually pay

Distributed-SQL engines are not free HA. Quantify before adopting:

- **Write latency floor = quorum RTT.** Intra-region 2–5 ms, multi-region 50–200 ms. Single-row OLTP that took 1 ms on a vertically-scaled Postgres primary will take 5–10× that under a Raft quorum. Load-test p99 writes against an inter-zone quorum *before* committing.
- **Clock-skew sensitivity.** Spanner relies on TrueTime (GPS + atomic clocks). CockroachDB uses HLC + a configured `--max-offset`; exceeding it can stall or, historically, produce anomalies. TiDB uses the PD timestamp oracle (a logically singleton service per cluster) for global ordering.
- **Range/tablet splits and rebalances are operational events.** P99 latency spikes during splits are normal; alert and capacity-plan for them.
- **Read the Jepsen reports for your version**, not "Jepsen says it's fine": see jepsen.io/analyses for CockroachDB (2017, 2020, 23.1.6), YugabyteDB (1.3.1, 2019), TiDB (3.0.0, 2019). Anomalies were real and most have been fixed; the discipline is to know which version closed which bug.

### 5.2 Aurora DSQL — new shape, narrow fit (preview/early-GA)

Amazon Aurora DSQL was announced at re:Invent (Dec 2024) and reached GA in May 2025. It is a serverless, Postgres-wire-compatible distributed SQL engine with **active-active multi-region writes**, strong snapshot isolation, optimistic concurrency, and a published **99.999% multi-region / 99.99% single-region** availability target. Architecture is disaggregated transaction-log + storage quorum (similar in spirit to Spanner / Cockroach), with a third "witness" region required for multi-region clusters.

- **Use it when** you have a *measured* requirement for active-active multi-region writes on a Postgres-shaped workload — global uniqueness, write-anywhere SaaS, regulator-mandated regional independence with RPO=0.
- **Do not use it as a drop-in for Aurora Postgres.** At preview / early-GA it has materially fewer Postgres features (foreign keys, several extensions, certain DDL patterns); the optimistic-concurrency model surfaces serialization errors the app must retry. Re-validate the gap list at the time you adopt — it is moving.
- **For everything else, Aurora Postgres or Cloud SQL / AlloyDB / Azure Flexible Server with Multi-AZ + an async cross-region replica is cheaper, more compatible, and easier to operate.**

### 5.3 Postgres-on-cloud picker

Most teams will pick one of these. The differences that actually matter on day 2:

| Offering                       | Sweet spot                                    | Watch out for                                                              |
|--------------------------------|-----------------------------------------------|----------------------------------------------------------------------------|
| RDS Postgres                   | Boring, predictable, broad ext list           | No TimescaleDB; minor-version cadence trails community by months           |
| Aurora Postgres                | High-throughput OLTP, fast read replicas      | Cost; storage-layer behaviour differs from vanilla PG (vacuum, replication)|
| Aurora DSQL                    | Active-active multi-region writes (§5.2)      | Feature gaps vs Aurora PG; OCC retries in app                              |
| Cloud SQL Postgres             | Default GCP managed PG                        | Trails community minors; HA failover slower than Aurora                    |
| AlloyDB                        | OLTP+light-analytics on GCP, columnar engine  | Newer, smaller ext set; pricing model                                      |
| Azure DB for PostgreSQL Flex   | Default Azure managed PG                      | Burstable SKUs hide CPU credits; HA zone-redundant only in some regions    |
| Neon                           | Serverless, branchable, scale-to-zero         | Cold-start latency; smaller blast radius per project                       |
| Crunchy Bridge / Postgres      | Vanilla PG, fast adoption of community minors | Smaller vendor; you trade hyperscaler integration for PG fidelity          |
| Supabase                       | App-tier bundle (auth, edge fns) on PG        | Coupling to the bundle; bring-your-own ops at scale                        |
| Timescale Cloud                | TimescaleDB / time-series                     | Vendor-specific extensions; locks you into the engine                      |

**Multi-region active/active is an exception, not a default.** Adopt it only with a quantified justification: a documented business loss per minute of regional outage, a regulator that mandates it, or measured exposure to a provider's regional incidents. The cost is real — write latency, conflict handling, double the compliance surface, and split-brain bug classes that don't exist in single-writer designs. **Multi-AZ HA + a tested cross-region restore is the right answer for the overwhelming majority of tier-1 systems.**

**Multi-primary RDBMS is hard.** Galera, MySQL Group Replication, and BDR all work in demos and have specific, documented failure modes under partition and failover. See Kyle Kingsbury's Jepsen analyses of *MySQL 8.0.34* (https://jepsen.io/analyses/mysql-8.0.34, 2023 — lost-update under `READ COMMITTED`/`REPEATABLE READ`) and the broader Jepsen index for MariaDB Galera write-skew and stale-read findings under partitions (https://jepsen.io/analyses); also Kleppmann, *Designing Data-Intensive Applications*, ch. 5 (https://dataintensive.net/). If you think you need multi-primary write-anywhere SQL, you almost certainly want a distributed-SQL engine, or a single-writer design with regional read replicas.

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

## 8. Caches — and the Redis → Valkey reset

**Patterns:** *cache-aside* (default — read miss fills, writes invalidate; thundering-herd failure mode, fix with single-flight); *write-through* (strong freshness, doubles write latency); *write-behind* (fastest, most dangerous; data loss on cache failure unless the cache is durable); *refresh-ahead* for predictable hot keys. Pick cache-aside unless you've measured a reason to do otherwise.

**"There are only two hard things in CS: cache invalidation and naming things."** — Phil Karlton. Invalidation is fundamentally distributed coordination; the cheapest correct answer is a short TTL plus event-driven busts on writes. Avoid relying on TTL alone for correctness; avoid relying on busts alone for availability.

### 8.1 The Redis license change and Valkey

On **20 Mar 2024** Redis Inc. relicensed Redis from BSD-3 to dual **SSPLv1 / RSALv2**; on **28 Mar 2024** the Linux Foundation announced **Valkey**, a BSD-3 fork of Redis 7.2.4, backed by AWS, Google Cloud, Oracle, Ericsson, and Snap. AWS ElastiCache and MemoryDB now ship Valkey as a first-class engine (often at lower per-node price than Redis OSS); Google Memorystore added Valkey support in 2024; Azure Cache for Redis remains on the commercial product.

- **Default for new workloads: Valkey-compatible managed offerings** (ElastiCache for Valkey, MemoryDB for Valkey, Memorystore for Valkey). Wire-protocol-compatible with existing Redis clients.
- **Existing Redis OSS deployments are not on fire** — if you only consume managed Redis as a backing service over the wire and never redistribute the binary, the SSPL change has limited practical effect. Migrate at next major upgrade, not as an emergency.
- **If you redistribute** (you ship Redis inside a customer-deployable product), SSPL/RSALv2 is incompatible with most commercial usage; Valkey is the answer.
- **DragonflyDB and KeyDB** are performance-focused alternatives, but coverage of the full Redis modules ecosystem (Search, JSON, TimeSeries, Bloom) is partial — verify your module use-cases before adopting.
- **Redis 7.4+ is a commercial product.** Treat it as such in licensing review, including any "free tier" or "developer edition" terms.

### 8.2 Engine choice (general)

**Valkey / Redis** for almost everything (data structures, pub/sub, streams, Lua atomicity, persistence) — managed; cluster mode for horizontal scale. **Memcached** when you need *only* a memory-bounded LRU and zero features. **CDN/HTTP caches** (CloudFront, Front Door, Cloud CDN, Fastly) for cacheable HTTP — almost always cheaper and faster than an origin-side cache.

**Caches are not databases.** Anything you cannot regenerate from the system of record does not belong in a cache. Period.

## 9. Message brokers

**Kafka is overkill for most teams.** It's right when you need durable, replayable, partitioned event logs at high throughput with multiple independent consumer groups — log aggregation, CDC fan-out, analytics pipelines, event sourcing at scale. It's wrong when you need a work queue with retries and a DLQ.

| Need                                            | Pick                                                              |
|-------------------------------------------------|-------------------------------------------------------------------|
| Work queue, per-message ack, DLQs               | SQS, Service Bus Queues, Cloud Tasks, RabbitMQ                    |
| Pub/sub fan-out, low ops                        | SNS+SQS, Service Bus Topics, Pub/Sub                              |
| Durable replayable log, low latency             | Kafka (MSK / Confluent / Aiven), Redpanda, Pulsar, Kinesis        |
| Durable replayable log, cost-optimized          | WarpStream, AutoMQ, Confluent Freight (S3-backed; see §9.2)       |
| In-cluster, lightweight, request/reply          | NATS, NATS JetStream                                              |
| Database CDC                                    | Debezium → Kafka, or DMS / Datastream                             |

### 9.1 Kafka 4.0: KRaft only, ZooKeeper gone

**Apache Kafka 4.0 (released 18 Mar 2025) is the first major release operating entirely without ZooKeeper.** KRaft is the only supported control plane. Operational consequences:

- **Migration path is two hops.** Any pre-3.x ZK-based cluster must upgrade to 3.x KRaft first (`kafka-metadata-quorum.sh` migration), then to 4.0. Skipping is not supported.
- **Java 17 required for brokers / Connect / Tools** (clients still work on Java 11). Bake it into base images.
- **KIP-848 next-generation consumer rebalance protocol is GA** — eliminates stop-the-world rebalances that used to spike p99 across the consumer group on every membership change. Worth the consumer-library bump.
- **KIP-932 "Queues for Kafka" is in early access** — Kafka-as-work-queue with per-message ack semantics is finally on the roadmap; do not bet production on it yet.
- **MSK has supported KRaft since 2023.** In any picker, mark ZooKeeper-backed managed brokers as legacy / EOL.
- **KRaft is not free of ops.** You still need 3–5 dedicated controller nodes; the line "no separate ensemble" is technically true but doesn't mean lower hardware count for small clusters.

### 9.2 S3-backed brokers — the cost reset

**Object-storage-backed Kafka-compatible brokers** — WarpStream, AutoMQ, Confluent Freight — write directly to S3 / GCS / Blob and eliminate inter-AZ replication traffic plus broker disks. Confluent **acquired WarpStream in Sep 2024** and folded the BYOC architecture into Freight clusters. The trade is real and quantifiable:

- **Cost:** ~80–90% lower TCO at high throughput (the inter-AZ replication bill is often the largest single line on a self-hosted Kafka).
- **Latency:** produce p99 is **single-digit seconds, often ~400 ms–1 s**, vs tens of milliseconds for disk-replicated Kafka. Hard ceiling for request/response or any path with a user waiting.
- **Failure-mode concentration:** an S3 (or regional object-store) event becomes a broker outage. The blast radius shifts from "broker fleet" to "object store".

Decision rule: **S3-backed for analytics / log / CDC / fan-out where ≥500 ms is fine and cost dominates; classic disk-replicated Kafka (or Redpanda) for low-latency request/response and exactly-once transactional pipelines.** Add a "broker storage model" column to your internal picker — it is now a first-class axis, not an implementation detail.

### 9.3 Kafka vs Pulsar vs Redpanda — short note

Kafka couples broker compute and storage on the same node (per-partition log files, plus tiered storage via KIP-405). Pulsar separates compute (brokers) from storage (BookKeeper bookies), making per-tenant elasticity and tiered storage cleaner but adding a second distributed system to operate (Apache Pulsar docs, "Architecture overview"). Pulsar's geo-replication is built into the broker (per-topic via `pulsar-admin namespaces set-clusters`); Kafka's cross-cluster replication is external — **MirrorMaker 2** or **Confluent Replicator**. **Redpanda** is a single-binary C++ Kafka-protocol-compatible broker (no JVM, no ZK ever) — strong for low-latency / on-prem / edge.

**Kafka usually wins on ecosystem maturity** (Connect, Streams, ksqlDB, Schema Registry, the connector catalogue). Pick Pulsar when multi-tenancy isolation, geo-replication, or storage/compute separation is a *measured* requirement; pick Redpanda when you want the wire protocol without the JVM; otherwise the marginal complexity isn't paid back.

**Prefer cloud-native managed brokers.** Self-hosting Kafka well takes a dedicated team — KRaft upgrades, partition rebalancing, broker tuning — and is the most common "managed would have been cheaper" story.

**Exactly-once is a marketing term.** What you get is *effectively-once* through idempotent consumers + dedupe keys, or transactional outbox patterns (§11). Design every consumer to be idempotent (Helland, "Life Beyond Distributed Transactions", CIDR 2007).

## 10. Schema evolution & compatibility

Schema evolution applies to **events on the wire** as much as to database DDL (§15). The failure mode is the same: a producer ships a change, a consumer crashes in production, and the rollback is a release window. The discipline is *contracts, registered and enforced*.

- **Use a schema registry** for any non-trivial event bus — Confluent Schema Registry, AWS Glue Schema Registry, Azure Schema Registry, Apicurio. Pick Avro, Protobuf, or JSON Schema; the registry enforces compatibility on publish (Confluent docs, "Schema Evolution and Compatibility").
- **Default to backward-compatible changes:** add optional fields with defaults, never remove or rename a field, never change a type, never change the meaning of an enum value. Field numbers/positions are permanent (Google Protocol Buffers docs, "Updating A Message Type").
- **Consumer-first vs producer-first rollouts.** For a *backward-compatible* change (new readers can read old data), roll **consumers first**: deploy the new consumer, verify it still handles old events, then ship the producer. For a *forward-compatible* change (old readers can read new data — rarer), roll **producers first**. **Breaking changes** (incompatible in either direction) require a parallel topic / new subject + dual-publish + drain old consumers + retire — the event-bus analogue of expand/contract (§15).
- **Version events explicitly** in the topic or schema name (`orders.v2`) when a breaking change is unavoidable. Never silently re-purpose a topic.
- **Compatibility mode is a registry setting, not a convention.** Configure `BACKWARD` (or `BACKWARD_TRANSITIVE` if you replay history) and let CI fail builds that violate it.

## 11. Change data capture & the outbox

CDC is how you escape **dual writes** — the anti-pattern where application code writes to a database *and* publishes an event, and the second write fails. There is no `try/catch` that fixes this; the database's write log must *be* the source of truth.

- **Log-based CDC.** Read the engine's replication log directly: **Debezium** for Postgres logical decoding (`pgoutput`), MySQL binlog (`ROW` format), MongoDB oplog, SQL Server CDC tables (Debezium docs, "Connector for PostgreSQL"; AWS DMS / GCP Datastream are managed equivalents). Provides ordered, at-least-once delivery of every committed change with no application code changes. Watch for: **replication-slot bloat on Postgres** if a consumer falls behind (the WAL is retained until the slot advances — alert on `pg_replication_slots.confirmed_flush_lsn` lag), schema-change handling, and initial-snapshot load on large tables.
- **Transactional outbox via Debezium Outbox Event Router.** When you need to publish an event *as part of* a business transaction, write it to an `outbox` table in the same DB transaction, and have Debezium tail that table with the **Outbox Event Router SMT** (shipped in Debezium 2.x). The standard column shape is `id, aggregatetype, aggregateid, type, payload[, tracingspancontext]`; the SMT routes each row to a topic named by `aggregatetype`, with `aggregateid` as the Kafka message key — preserving per-aggregate ordering and giving idempotency keys for free (Debezium docs, "Outbox event router"; Morling, "Reliable Microservices Data Exchange With the Outbox Pattern", Debezium Blog 2019; Richardson, *Microservices Patterns*, ch. 3).
- **Avoid dual writes.** If you find yourself writing `db.save(x); broker.publish(x)` in app code, you have a bug waiting for the next network blip. Replace with outbox + CDC.
- **Idempotency keys on every consumer.** Every event must carry a stable ID (often source PK + change LSN, or the outbox `id`), and consumers must dedupe on it. CDC is at-least-once; replays happen on every restart, failover, or slot reset.
- **Tombstones for deletes.** Compacted Kafka topics use a `null` value as a tombstone to signal deletion downstream (Apache Kafka docs, "Log Compaction"). Consumers must handle them; sinks (search indexes, caches, materialized views) must propagate the delete or you've built a GDPR Art. 17 problem (§14).

## 12. Event sourcing & CQRS — operationally

Event sourcing solves real problems (auditability, temporal queries, replayable read models) and creates real ones: schema evolution of events forever (§10), snapshotting, projection rebuild times, GDPR Art. 17 right-to-erasure against an immutable log (§14). CQRS adds a second store you must monitor and reconcile.

**Adopt only when** audit/regulatory requirements demand the full event history, *or* multiple read models materially diverge from the write model, *or* you have an operational appetite for two stores and projection-lag SLOs. For most CRUD systems, a normal database with an outbox table and CDC into Kafka (§11) gives you 80% of the benefits at 20% of the cost. (Helland, "Immutability Changes Everything", CACM 2016.)

## 13. Object storage and immutable backup vaults

Object storage is the closest thing infrastructure has to a universal primitive — default for blobs, backups, build artifacts, logs, analytics input/output, ML datasets, anything large and cold-ish. Two distinctions matter and most chapters blur them:

- **S3-compatible API ≠ feature parity.** R2, MinIO, Wasabi, B2, Ceph RGW implement most of the S3 *wire protocol* and core verbs, with varying coverage of newer features (multipart edge cases, conditional writes, SSE-KMS, request-payer). GCS and Azure Blob expose **S3-compatible endpoints** for interop (GCS "XML API"; Azure Blob "S3 API support"), but their *native* APIs and feature models are distinct. Test against the specific provider, not "S3".

### 13.1 Immutability is not one feature — it's a per-cloud table

"We use object lock" tells a compliance reviewer nothing. The mode, scope, and bypass semantics differ at every cloud and even between two AWS services:

| Control                       | Scope                  | Modes / states                      | Bypass during retention                                    | Notes                                                                              |
|-------------------------------|------------------------|-------------------------------------|------------------------------------------------------------|------------------------------------------------------------------------------------|
| **AWS S3 Object Lock**        | Per-object-version     | Governance / Compliance + Legal Hold| Governance: `s3:BypassGovernanceRetention`. Compliance: **none**, including root | Requires versioning; new versions and delete markers can still be added on top      |
| **AWS Backup Vault Lock**     | Per backup vault       | Governance / Compliance             | Governance: IAM-bypassable. Compliance: **none, including AWS**, after grace     | Cohasset-assessed for SEC 17a-4 / FINRA / CFTC; "retention=Always" footgun is permanent |
| **Azure Blob immutable**      | Container or version   | Time-based / Legal Hold; locked vs unlocked | Unlocked policies: mutable. Locked policies: **WORM, irreversible**       | "Locked" is the only true WORM state; container-level vs version-level differ      |
| **Azure Backup Immutable Vault** | Backup vault        | Disabled / Enabled / **Enabled+Locked** | Only Enabled+Locked is irreversible; plain Enabled can be turned off          | Backup-vault WORM still preview in several regions as of late 2024 — check matrix  |
| **GCS Bucket Lock**           | Bucket-wide retention  | Retention policy + (locked) flag    | Locked policy: cannot be reduced or removed                                      | Applies bucket-wide to every object until aged past the policy                     |

**Practical rules:**

- **For the off-site copy in 3-2-1 (§2): Compliance-mode WORM** (S3 Object Lock Compliance / AWS Backup Vault Lock Compliance / Azure Immutable Vault Enabled+Locked / GCS Bucket Lock locked) — **scoped to tier-0 or regulated data only.**
- **Set a grace period (≥3 days) before promoting any vault to Compliance/Locked**, and run a CI test that creates → restores → expires a recovery point inside the grace window. Compliance mode is genuinely irreversible; "retention=Always" applied by mistake is permanent unbounded spend that *AWS itself cannot delete*.
- **Restore drills (§3) must include a negative test:** attempt to delete an in-retention object using the root / owner identity, expect failure, log the failure. A locked vault that was never tested for being actually locked is folklore.
- **Don't apply Compliance mode to dev / staging vaults.** Use Governance mode or shorter retention there.

### 13.2 Object-storage defaults

- **Versioning on** for any bucket holding state you'd cry over. Pair with a lifecycle policy that expires noncurrent versions after N days. S3 Object Lock requires versioning; deleting a versioned object creates a delete marker, not a true deletion.
- **Lifecycle policies** to tier objects to IA / Cold / Archive by age. Most buckets pay 5–10× more than necessary because nobody set this. Model retrieval cost (and minimum-storage-duration fees) against access patterns *before* setting the rule.
- **Cross-region replication ≠ versioning.** CRR replicates deletes (unless excluded); versioning protects against deletes. You usually want both, and the replica bucket in a separate account with its own Object Lock.
- **Block public access at the account level**, not per-bucket. Every major data-leak postmortem of the last decade includes "the bucket was public."
- **Server-side encryption with a CMK** for anything sensitive. Default encryption is table stakes; key custody is the control.

## 14. Data sovereignty & residency

GDPR, Schrems II, the EU AI Act, India's DPDP Act, China's PIPL, and sectoral regimes (HIPAA, PCI-DSS, FedRAMP, FINMA) make region a *first-class* deployment property, not an afterthought.

### 14.1 The Schrems II → DPF arc, and why sovereign cloud exists

- **Schrems II (CJEU C-311/18, 16 Jul 2020)** invalidated Privacy Shield and tightened the bar for EU→non-EU personal-data transfers: Standard Contractual Clauses + a transfer impact assessment + supplementary measures (typically encryption with EU-held keys).
- **EU–US Data Privacy Framework (Commission Implementing Decision (EU) 2023/1795, 10 Jul 2023)** restored adequacy for self-certified US organisations under the DPF Principles. **It does not eliminate SCCs for non-DPF-certified sub-processors,** and a Schrems III challenge is pending — treat the framework as *politically fragile* in long-horizon designs.
- **Sovereign-cloud offerings** (AWS European Sovereign Cloud — announced 2023, GA target 2025; Microsoft Cloud for Sovereignty; Google Sovereign Controls partnerships with T-Systems "S3NS" and similar) exist for the residual cases: regulated sectors that demand EU-only operations and personnel, or the strictest reading of the EU AI Act for high-risk systems. **Adopt as exception-only, with explicit regulatory citation** — they cost more and have smaller service catalogues. Do not default to sovereign cloud for ordinary B2B SaaS; DPF + SCCs + EU-resident KMS is sufficient.

### 14.2 Operating defaults

- **Region pinning by tenant.** Each tenant has a home region recorded at provisioning. Primary stores, backups, and *logs* live in or replicate only to permitted regions.
- **Logs and traces are personal data too.** A request log with an IP, user ID, or header is in scope for GDPR. Pipe regional traffic to regional log stores.
- **Encryption keys in-region**, with a key per residency zone. CMKs in EU KMS for EU data; same for other regimes. Key material does not cross borders.
- **EU AI Act overlay.** For systems classified as high-risk or providing general-purpose AI, training-data residency, model-output logging, and provider transparency obligations land on the data tier. Inventory which datasets and projections feed model training and apply the same residency rules.

### 14.3 GDPR Art. 17 against backups — beyond use, not surgical delete

GDPR Article 17 (right to erasure) does **not** require immediate, surgical deletion from immutable backups. The recognised lawful approach (UK ICO "Right to erasure and backups"; EDPB Guidelines 5/2019; consistent CNIL and Datatilsynet positions) is:

1. **Delete from production** within the legal deadline.
2. **Put backup copies "beyond use"** — taken offline, access-restricted, and **excluded from any restore that would reintroduce the subject's data without re-deleting it**. This means runbook gates: any PITR or DR restore that crosses an erasure event must replay the deletion as part of the restore.
3. **Let backup copies age out** under the documented retention schedule.

**Crypto-shredding is a complement, not a substitute.** It only achieves erasure when (a) each subject's PII is encrypted with an isolated, per-subject (or per-tenant) key; (b) the key lives in a KMS where revocation is auditable and irreversible; and (c) **no plaintext replicas** of the PII exist downstream — no warehouse copy, no search index, no cache, no analytics extract, no broker topic, no log line. Shredding the key only renders ciphertext useless, not plaintext elsewhere (NIST SP 800-88 Rev. 1 "Cryptographic Erase"; ENISA, "Pseudonymisation Techniques and Best Practices", 2019). Inventory every plaintext sink before claiming crypto-shredding as your erasure mechanism, and document the gap to your DPO if you can't satisfy all three.

**Sectoral overrides.** Art. 17(3) carves out legal-obligation retention; health, finance, and tax regimes routinely impose minimum retention that overrides erasure requests. The DPO, not the engineer, decides which one wins.

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

**In transit:** TLS 1.2 minimum, 1.3 preferred, on every hop including intra-VPC database connections. "It's a private network" is not a defence; the cloud substrate is shared, and lateral-movement attacks routinely cross VPC boundaries. (Service-to-service mTLS is owned by ch07.)

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
- **Replication-slot bloat (Postgres).** Logical replication slots (used by Debezium and many CDC tools) retain WAL until the consumer advances `confirmed_flush_lsn`. A stuck consumer fills the WAL volume and takes the primary down. Alert on slot lag with the same severity as disk-full.

## 18. Stateful workloads on Kubernetes

**Default: don't.** For 90% of teams in 2026, running Postgres / MySQL / Kafka / Redis / Valkey / Elasticsearch on Kubernetes is operational masochism that buys you nothing your cloud provider's managed service doesn't already provide better and cheaper after headcount.

StatefulSets, CSI drivers, volume snapshot classes, storage-aware schedulers, and the operator ecosystem (CloudNativePG, Strimzi, Zalando Postgres Operator) have all matured — but the *failure modes* (PV stuck detaching, CSI controller wedged, operator reconciler bug eats your cluster) are exotic in a way managed services don't expose you to.

**Self-host on K8s only when** you're already running a multi-team K8s platform with a dedicated storage SRE function; *and* you have a hard requirement (air-gap, on-prem, regulated cloud-region unavailability) that rules out managed; *and* the workload is itself K8s-native and well-supported by a mature operator (CloudNativePG for Postgres, Strimzi for Kafka are the strongest examples), and you've read the Jepsen reports for the engine.

If you do: dedicated node pools for stateful workloads with taints and PDBs; local NVMe via CSI for performance-sensitive engines, replicate at the *application* layer not the storage layer; backups go to **object storage outside the cluster**, written by an identity the cluster cannot revoke; restore drills (§3) include "the entire K8s cluster is gone" as a scenario.

The honest version: most teams that ran Postgres on K8s in 2020–2023 migrated back to managed by 2025. Learn from their gas bills.

## 19. Observability of the data tier

Detail belongs in the observability chapter. The data-tier minimum: **replication lag** as an SLI with an SLO (for async standbys this is your real RPO); **backup age & last-successful-restore age** as SLIs with pages; **`LatestRestorableTime` (or engine equivalent)** for PITR engines (§2.1); **query-level metrics** (`pg_stat_statements`, Performance Schema, slow logs shipped) — not just host CPU/IO; **connection pool saturation** (the most common "database is down" outage is actually pool exhaustion in the app tier); **autovacuum progress and dead-tuple counts** (Postgres); **logical replication slot lag** for any CDC pipeline; **storage growth rate** with capacity projection (Sridharan, *Distributed Systems Observability*, O'Reilly 2018).

## 20. The short list

If you read nothing else:

1. Use the managed service; justify self-hosting in writing against the §1 exception list.
2. PITR on, ≥14 days, cross-account off-site copy, KMS key in a separate trust domain — and **know your engine's PITR mechanics from §2.1** (RDS ~5 min, Aurora ~1 s, Cosmos ~100 s to a *new account*, SQL Server per-DB-not-cross-DB, BULK_LOGGED breaks PITR).
3. Quarterly restore drills (monthly for tier-0), timed, into a fresh instance *and target region*, with smoke test, dashboarded. Automated weekly verification belongs to ch09.
4. RPO/RTO derived from impact + budget + recoverability *together*, tested by game-day not by assertion.
5. One DB per service, no shared schemas across services with different release cadences.
6. Multi-AZ + tested restore is the default; multi-region active/active is an exception with a written justification. Aurora DSQL / Spanner / Cockroach / Yugabyte / TiDB earn their complexity only above a measurable cross-region write requirement.
7. Shard only after measuring a wall, and gate Vitess / Citus with workload tests.
8. Default new caches to **Valkey** (managed); treat Redis 7.4+ as a commercial product.
9. **Kafka 4.0 = KRaft only**; ZooKeeper-backed brokers are legacy. For cost-dominated, latency-tolerant pipelines, S3-backed brokers (WarpStream / AutoMQ / Freight) are now a first-class choice.
10. Events get a schema registry and expand/contract rollouts, just like DDL.
11. CDC + Debezium **Outbox Event Router** instead of dual writes; idempotent consumers everywhere; alert on replication-slot lag.
12. Expand/contract migrations, reviewed by a DBA-grade reviewer, with an automated tool (Atlas / Flyway / Liquibase / Sqitch).
13. Object storage with versioning, lifecycle, and **cloud-specific** immutability — S3 Object Lock vs AWS Backup Vault Lock vs Azure Immutable Vault vs GCS Bucket Lock are *not* interchangeable; Compliance/Locked is irreversible and tier-0-only; always set a grace period.
14. GDPR Art. 17 against backups means **beyond-use + retention rolloff + restore-time re-deletion**, not surgical purge; crypto-shredding only with isolated subject keys and zero plaintext replicas. Schrems II → DPF restored adequacy in 2023 but is politically fragile; sovereign cloud is exception-only.
15. Day-2 basics — vacuum, PgBouncer, key escrow, stale-read awareness, replication-slot lag — are not optional.
16. Don't run stateful workloads on K8s unless you have a written reason and a storage SRE.

Everything else is detail.
