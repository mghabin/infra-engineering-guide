# Chapter 05 — Observability

Opinionated, cloud-agnostic defaults for instrumenting and operating production
systems. The framing is deliberately heretical against the "three pillars,
buy-a-vendor, slap-on-some-dashboards" school: the goal of observability is to
let you **ask new questions of a running system without shipping new code**, and
most of the industry is still doing monitoring in an OTel t-shirt.

> Conventions: **Do** = required default. **Don't** = reject in review unless
> a written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint.

---

## 1. The mental model: events, not pillars

- **Observability ≠ monitoring.** Monitoring answers questions you already
  knew to ask. Observability lets you debug *novel* failure modes by
  slicing arbitrary high-cardinality dimensions of real behaviour, without
  a redeploy (Majors, Fong-Jones & Miranda, *Observability Engineering*,
  O'Reilly 2022, ch. 1; Sridharan, *Distributed Systems Observability*,
  O'Reilly 2018, ch. 4).
- **Reject "the three pillars" as a design framework.** Logs / metrics /
  traces is a *vendor billing taxonomy*. The unit of observability is a
  **wide structured event** (canonical term in this guide; "wide event"
  and "structured event" are short forms used interchangeably below) with
  as many high-cardinality dimensions as
  you can afford (user_id, tenant_id, build_sha, shard, k8s_pod,
  feature_flag_state). Metrics and traces are projections of that event
  stream; logs are degenerate events with bad schemas. Distinct from
  ch08's *domain events* (event sourcing) — see the glossary entry
  "Wide Event".
- **Cardinality is an asset *in the right signal*.** A wide event with
  per-request user/tenant/build fields lets you answer "why is p99 bad
  for tenant X on canary Y in eu-west-1?" without code changes. The same
  fields on a Prometheus metric are a billing incident — see §6.

> **Smell test.** If your debugging loop is "open dashboard → eyeball graph → SSH → grep logs → guess", you have monitoring, not observability.

---

## 2. Instrumentation: OpenTelemetry as the preferred default

- **Use OpenTelemetry (OTel) for all new instrumentation.** OTel is the
  merged OpenTracing/OpenCensus standard and a CNCF project; SDKs, the
  wire protocol (OTLP), and the Collector are vendor-neutral and accepted
  by every serious backend
  ([OpenTelemetry specification](https://opentelemetry.io/docs/specs/otel/);
  [CNCF — OpenTelemetry](https://www.cncf.io/projects/opentelemetry/)).
  Don't instrument directly against a vendor SDK (Datadog tracer,
  NewRelic agent, AppInsights SDK) for new code; vendor agents may
  *receive* OTLP, but your code emits OTel.
- **OTel reduces instrumentation lock-in, not backend lock-in.** Switching
  emitters is cheap; switching backends still costs you query languages
  (PromQL/LogQL/SPL/DD-query), alert syntax, dashboards, retention
  schemas, and operator muscle memory. Plan migrations on the *backend*
  axis separately from the *instrumentation* axis.
- **Propagate W3C Trace Context (`traceparent` / `tracestate`) on every
  hop** — the W3C Recommendation and the OTel default
  ([W3C Trace Context](https://www.w3.org/TR/trace-context/);
  [W3C Baggage](https://www.w3.org/TR/baggage/)). During migration from a
  legacy propagator (B3, Jaeger, X-Ray), configure OTel's *composite*
  propagator to inject and extract both formats until the legacy producer
  is retired
  ([OTel propagators](https://opentelemetry.io/docs/specs/otel/context/api-propagators/)).
- **Baggage is for small, non-sensitive correlation keys** (tenant tier,
  feature flag bucket, request class). Don't put PII, auth tokens, or
  large payloads in baggage — it propagates to every downstream and
  often crosses trust boundaries ([W3C Baggage §3](https://www.w3.org/TR/baggage/)).
- **Auto-instrument first, hand-instrument the domain.** Use language
  auto-instrumentation for HTTP/gRPC/SQL/queue clients; add manual spans
  only for *business-meaningful* operations (`checkout.authorize`,
  `pricing.recompute`) with rich attributes.

---

## 3. Collector deployment: pick by use case, not ranking

Run an OpenTelemetry Collector between apps and backends so batching,
retries, redaction, sampling, attribute enrichment, and backend swapping
live outside application code
([OTel Collector](https://opentelemetry.io/docs/collector/);
[deployment patterns](https://opentelemetry.io/docs/collector/deployment/)).
There is no single "best" topology — the patterns compose:

| Pattern | Use when | Avoid when |
|---|---|---|
| **Agent / DaemonSet (per node)** | Default for Kubernetes node-local collection: host metrics, kubelet/cAdvisor scrape, log tail from `/var/log/pods`, OTLP receiver on `localhost`. Predictable per-node footprint. | You need cross-node aggregation (tail sampling, dedup) — DaemonSets only see one node's traffic. |
| **Gateway / Deployment (central)** | Fleet-wide processors that need a *full view*: tail-sampling, span metrics, cardinality enforcement, multi-backend fan-out, egress auth, cost-control routing. Scaled horizontally behind a load balancer. | Tiny estates where a single agent tier is enough; latency-sensitive paths where an extra hop matters. |
| **Sidecar (per pod)** | Strict per-workload isolation (multi-tenant clusters, regulated workloads needing per-team redaction), or apps that can only emit to `localhost`. | Default case — sidecars multiply CPU, memory, and config surface per replica and complicate upgrades. |

- **Typical shape:** apps → DaemonSet agent (collection + light processing)
  → Gateway deployment (tail sampling, routing, redaction) → backends.
  Sidecars only where justified in writing. For tail sampling specifically,
  a gateway tier with trace-affinity load balancing is required (§5a).

---

## 4. Structured logs: cheap events, not prose

- **Logs are JSON. Always.** Plain-text logs are unparseable at scale.
  Use a structured logger (`zap`/`zerolog` in Go, `Serilog` in .NET,
  `structlog` in Python, `pino` in Node) emitting one JSON object per
  event ([12-Factor — Logs](https://12factor.net/logs);
  [OTel Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)).
- **Adopt a single field schema.** Pick **OpenTelemetry Logs** (preferred
  for new systems — same attribute model as traces) or **Elastic Common
  Schema (ECS)** (legacy ELK). Mixing schemas across services makes
  cross-service queries impossible
  ([OTel logs semconv](https://opentelemetry.io/docs/specs/semconv/general/logs/);
  [ECS](https://www.elastic.co/guide/en/ecs/current/index.html)).
- **Standard fields on every event** (per OTel semantic conventions):
  `timestamp` (RFC 3339 UTC), `severity_text` + `severity_number`,
  `service.name`, `service.version`, `deployment.environment`, `trace_id`,
  `span_id`, `event.name`, `code.namespace` / `code.function`. **Every
  log line carries `trace_id` and `span_id`** when emitted inside a traced
  operation — automatic if the logger is wired to the OTel context
  ([OTel Logs Bridge API](https://opentelemetry.io/docs/specs/otel/logs/bridge-api/)).
- **Don't use logs as metrics.** Counting log lines for a rate is the
  classic anti-pattern: orders of magnitude more expensive than a counter,
  and unindexed substring matches drive most multi-million log bills.
- **Don't log at INFO in the hot path.** A request-rate INFO line is a
  metric in disguise; demote it. Reserve INFO for state transitions, WARN
  for degraded but serving, ERROR for "a human should look".

### 4a. Log shippers and ingestion

- **Use a dedicated log shipper, not the application, to ship logs.** App
  writes JSON to stdout; a node-local shipper handles buffering, batching,
  retry, multi-backend fan-out, and back-pressure. Default choices:
  **Vector** ([vector.dev](https://vector.dev/docs/)) — Rust, expressive
  VRL transforms, multi-sink; **Fluent Bit**
  ([docs.fluentbit.io](https://docs.fluentbit.io/)) — C, very small
  footprint, the de-facto Kubernetes node agent; **OTel Collector with
  `filelog` receiver** when you want one binary for logs/metrics/traces.
- **Don't** ship logs directly from app code over the network — you lose
  events on restart, leak credentials into every service, and couple
  deploys to backend availability.

### 4b. PII, redaction, and audit logs

- **Redact at the trust boundary you control** — in the shipper or
  Collector (Vector VRL, OTel `attributes`/`transform` processors) *before*
  egress. Backend-side redaction means raw PII is already on disk
  somewhere you don't own. Maintain an explicit **allowlist** of fields
  that may contain user data (public / internal / PII / secret); anything
  else is dropped or hashed. Never put secrets, full auth headers, raw
  card numbers, or full request bodies in logs; reference them by opaque
  IDs that resolve in a separate, access-controlled store.
- **Audit logs are not observability logs.** Audit events (SOC 2 / HIPAA /
  PCI / GDPR) need a **separate pipeline** with append-only, tamper-
  evident storage and multi-year retention; **different access controls**
  (compliance/security/legal, often break-glass); and a **different
  schema** with mandatory actor, action, resource, outcome, source IP,
  request ID. Mixing them with observability logs means one stream's
  retention silently breaks the other's compliance posture. The canonical
  definition of an audit log (storage, access, retention) is owned by
  ch06 §15 — this section only covers the *transport* separation.

---

## 5. Distributed tracing and sampling

- **Trace every external request and every async boundary.** A trace whose
  spans stop at your service edge is theatre — the value is in the *causal
  chain* across services, queues, and DBs. Instrument inbound HTTP/gRPC,
  outbound clients, queue publishers and consumers, and DB drivers
  ([OTel trace semconv](https://opentelemetry.io/docs/specs/semconv/general/trace/)).
- **Sample at trace granularity, never at span granularity.** A trace with
  half its spans dropped is worse than no trace; the contract is "keep the
  whole tree or none of it" ([OTel sampling](https://opentelemetry.io/docs/concepts/sampling/)).
  Don't roll your own propagation — use W3C `traceparent` (§2).

### 5a. Tail-based sampling: rules and constraints

**Tail-based sampling** (OTel canonical wording — used as the canonical
form across this guide; "tail sampling" appears as a short form below) is
decide-after-trace-completes, in a Collector gateway tier
([tailsamplingprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)).
It supports **fixed policies** — `status_code`, `latency` (fixed
millisecond threshold), `string_attribute`, `numeric_attribute`,
`probabilistic`, `rate_limiting`, and composites — *not* dynamic "current
p99" predicates. Derive thresholds from your SLO and review on the same
cadence.

- **Policy template** (compose with `and` / `composite`): `status_code =
  ERROR` → keep; `latency > <SLO-derived ms>` → keep; high-value tenant
  attribute → keep; `probabilistic` baseline (1–10%) for the rest;
  `rate_limiting` cap on total spans/sec to bound cost during incidents.
- **Trace affinity is required.** All spans for a trace must reach the
  *same* collector instance, or the sampler decides on a partial trace
  and you get orphan spans. Either run a single gateway replica (small
  scale) or put a `loadbalancing` exporter keyed on `trace_id` in front of
  the tail-sampling tier
  ([loadbalancing exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter)).
- **Memory cost is real.** The processor buffers all spans for
  `decision_wait` (commonly 10–30s); budget memory as `peak spans/sec ×
  decision_wait × avg span size` plus headroom, and alarm on processor
  queue depth.
- **By scale:** low volume (< ~100 traces/sec) — no sampling, keep 100%.
  Mid/high volume — keep all errors and SLO-violating traces with a
  probabilistic baseline and `rate_limiting` cap; during cascading
  failures the cap will sample errors too, so route overflow to cheaper
  storage or drop with a counter so the rate stays observable.

### 5b. Linking metrics to traces with exemplars

Exemplars attach a sampled `trace_id` to a metric observation so a latency
spike can pivot to an example trace
([OTel exemplars](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars);
[Prometheus exemplars](https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage)).
Useful *only when the whole path supports them*: client library records
exemplars (recent Prometheus client libs; OTel SDK with exemplar
reservoirs); scrape format carries them (OpenMetrics or OTLP — the legacy
Prometheus text format does not); storage path persists them (local
Prometheus exemplar storage is in-memory and feature-flagged; remote-write
preserves them only with compatible protocol versions, and many long-term
stores or older Mimir/Thanos/Cortex versions drop or short-retain — verify
against release notes); UI surfaces them (Grafana panels with "Exemplars"
enabled, or vendor-native rendering). Don't advertise "click latency to
trace" in runbooks until you've demonstrated the round-trip end-to-end on
the *current* backend.

---

## 6. Metrics

- **Services use RED, resources use USE, edge uses Four Golden Signals.**
  **RED** (Rate / Errors / Duration) per service endpoint
  ([Wilkie](https://thenewstack.io/monitoring-microservices-red-method/));
  **USE** (Utilization / Saturation / Errors) per resource
  ([Gregg](https://www.brendangregg.com/usemethod.html)); **Four Golden
  Signals** (latency, traffic, errors, saturation) at mesh / gateway
  ([SRE Book ch. 6](https://sre.google/sre-book/monitoring-distributed-systems/)).
- **Default to Prometheus exposition + OTLP** — pull-based scrape for
  in-cluster long-lived workloads; push (OTLP/HTTP) for serverless and
  short-lived jobs ([Prometheus instrumentation](https://prometheus.io/docs/practices/instrumentation/);
  [naming](https://prometheus.io/docs/practices/naming/)). Don't invent
  units — base SI, `_total` for monotonic counters, `_seconds` for
  durations, `_bytes` for sizes.

### 6a. Signal vs cardinality

| Signal | Cardinality budget | Retention | Put here |
|---|---|---|---|
| **Metrics** (Prometheus / OTLP) | **Bounded** — series count must be a small multiple of label-domain product; no unbounded labels. | Long (months–years), aggregate. | Rates, percentiles, resource utilisation, SLI numerators/denominators. |
| **Traces** | High per trace; total bounded by sampling. | Days–weeks. | Per-request causal chains, slow/error exemplars, cross-service timing. |
| **Wide events / structured logs** | Highest; one row per request is fine. | Short by default, longer for audit. | Per-request debug context, business events, anything with `user_id` / `request_id` / `build_sha`. |
| **Profiles** | Aggregated by symbol; bounded. | Short rolling window. | CPU / memory / lock attribution to source lines. |

- Hard rule: **never** put `user_id`, `request_id`, `trace_id`, full URL
  paths, or raw error messages on a Prometheus metric label
  ([Prometheus label best practices](https://prometheus.io/docs/practices/naming/#labels));
  put them on the wide event or span attribute instead. Normalise route
  templates on metrics (`/users/{id}`), never raw paths.

### 6b. Enforcing cardinality in production

`promtool check metrics` validates exposition *format and naming*; it does
**not** see real label values and cannot enforce cardinality budgets. Do
this at runtime:

- **Relabel / drop rules at scrape time** (`metric_relabel_configs` with
  `action: drop` / `labeldrop` / `replace`) to neutralise labels before
  they hit the TSDB
  ([Prometheus relabel config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#metric_relabel_configs));
  **allowlist labels per metric** in the Collector
  (`transform` / `attributes` processors) so unknown labels are stripped at
  the gateway, not the application.
- **Top-N series dashboards** (`topk(20, count by (__name__)({...}))`)
  reviewed weekly; **alert on `prometheus_tsdb_head_series`** total and
  per-job (e.g. page at 80% of provisioned capacity); **recording rules
  tracking series count per metric** so cost regressions show up as a
  graph, not a quarterly bill.
- **CI signal is best-effort:** `promtool check rules`, lint label names
  against an allowlist, reject obvious offenders (`user_id`, `request_id`,
  `email`) by regex. CI cannot catch unbounded values.

---

## 7. Long-term storage and federation

- **Single-binary Prometheus is fine until ~1M active series or your
  configured local retention (default 15d).** Past that, push remote-write
  to a horizontally scalable backend
  ([Prometheus storage](https://prometheus.io/docs/prometheus/latest/storage/);
  [remote-write](https://prometheus.io/docs/practices/remote_write/)).
  OSS scale-out: **Grafana Mimir** (receiver-mode, microservice, large
  multi-tenant —
  [architecture](https://grafana.com/docs/mimir/latest/references/architecture/));
  **Thanos** (sidecar uploads blocks to object storage; query federates —
  [components](https://thanos.io/tip/components/));
  **VictoriaMetrics** (single-binary or cluster, lower op surface —
  [docs](https://docs.victoriametrics.com/)). Don't run Cortex new — Mimir
  is the maintained fork.
- **Storage backend defaults:**

| Signal | OSS default | Scale-out OSS | Managed/SaaS |
|---|---|---|---|
| Metrics | Prometheus (≤15d local) | Mimir / Thanos / VictoriaMetrics | Grafana Cloud, AMP, Datadog |
| Logs | Loki | Loki + S3 | Grafana Cloud, Datadog, Splunk |
| Traces | Tempo / Jaeger | Tempo + S3 | Honeycomb, Datadog |
| Wide events | — | ClickHouse (SigNoz, Highlight) | Honeycomb |
| Profiles | Pyroscope / Parca | Pyroscope + S3 | Polar Signals, Grafana Cloud |

- **Vendor vs OSS:** **OSS LGTM** — control, lower unit cost at scale, you
  operate it; reasonable from ~50 engineers and a dedicated platform team.
  **Honeycomb** — best-in-class wide-event / high-cardinality query.
  **Datadog / New Relic / Dynatrace** — turnkey, expensive, easy to
  outgrow budget; right when time-to-first-signal beats unit cost.
  **ClickHouse-based stacks (SigNoz, Highlight, Uptrace)** — OTLP-native,
  one store for logs/metrics/traces, much cheaper than ELK at log scale.

---

## 8. Telemetry cost model

Treat telemetry as a workload with a budget. Allocation, showback /
chargeback, and anomaly detection on that budget follow the same rules as
any other cloud cost line and are owned by ch10 §11 (allocation) + ch10
§12 (anomaly detection); this section covers only the *generation-side*
cost knobs.

| Signal | Dominant cost driver | Notes |
|---|---|---|
| Metrics | active series × retention | Cheap per series, brutal when cardinality leaks. |
| Traces | spans/sec × retention × sampling | Cheap with tail sampling; ruinous at 100% head sampling. |
| Logs | bytes ingested × indexed fields × retention | Most expensive in practice; index discipline > volume. |
| Wide events | events/sec × attributes × retention | Cheaper than indexed logs in columnar stores; expensive in row-oriented backends. |
| Profiles | hosts × profile rate × retention | Cheap with sub-1% sampling. |

- **Set per-team / per-service budgets** in the units the backend bills in
  (series, GB ingested, span count). Surface burn-down alongside SLOs.
- **Drop, don't store-and-archive, what you don't query.** Cold logs nobody
  opens are a compliance question, not an observability one.
- **Vendor switching cost is dominated by non-OTel pieces:** dashboards,
  alert rules, query muscle memory, runbooks, custom integrations. Keep
  dashboards and alerts in code (Grafana JSON / Jsonnet / Terraform); prefer
  query languages with multiple implementations (PromQL, LogQL).

---

## 9. Alerting: SLO-based for user-facing services, plus carve-outs

- **For user-facing request/response services, alert on user-visible
  symptoms, not causes.** "CPU 90%" is not a page; "p99 latency > 300ms
  for 5m on the checkout endpoint" is. Page on **SLO burn**, not raw infra
  metrics ([SRE Book ch. 6](https://sre.google/sre-book/monitoring-distributed-systems/);
  [SRE Workbook ch. 5](https://sre.google/workbook/alerting-on-slos/);
  Hidalgo, *Implementing Service Level Objectives*, O'Reilly 2020,
  ch. 7–8). Use **multi-window, multi-burn-rate** alerts: fast window
  (e.g. 5m + 1h, burn ≥ 14.4) plus slow window (e.g. 30m + 6h, burn ≥ 6)
  so you page fast on real outages and slow on real budget loss
  ([SRE Workbook — multi-window multi-burn-rate](https://sre.google/workbook/alerting-on-slos/#6-multiwindow-multi-burn-rate-alerts)).
- **Codify SLOs.** Use an SLO-as-code tool (`sloth`, `pyrra`, OpenSLO,
  `slo-generator`) so SLO docs and alert rules cannot drift
  ([OpenSLO](https://openslo.com/); [sloth](https://sloth.dev/);
  [pyrra](https://github.com/pyrra-dev/pyrra)).

### 9a. Legitimate non-symptom pages (carve-outs)

Symptom-first ≠ "no cause-based alerts." Page on these without an SLO
violation, provided each is actionable with a runbook:

- **Capacity / saturation with lead time** — disk > 85% and growing, queue
  depth growing for N minutes, connection pool > 90%, certificate or
  credential expiring < 14 days, cloud quota > 80%.
- **Hard dependency down or security event** — upstream API 100% errors,
  primary DB failover, broker partition unavailable, IDS/EDR alert,
  anomalous auth, audit pipeline gap, key rotation overdue.
- **Batch, pipelines, and internal control planes** without continuous
  request traffic — job failure, SLA miss, dead-letter growth, backup
  restore-test failure, CI/CD or IdP outage.

---

## 10. Profiling and eBPF (maturity-gated)

Continuous profiling and eBPF observability are powerful but earn their
place *after* metrics, logs, and traces are working.

### 10a. Continuous profiling

- **Adopt when** you have unexplained CPU/memory cost regressions, latency
  outliers traces don't explain, or non-trivial cloud bill from compute.
  Tooling: Pyroscope, Parca, Polar Signals, Datadog Continuous Profiler,
  Grafana Cloud Profiles. Profile format: `pprof`
  ([pprof](https://github.com/google/pprof/blob/main/doc/README.md);
  [Pyroscope](https://grafana.com/oss/pyroscope/);
  [Parca](https://www.parca.dev/)).
- **Verify overhead in your environment** before declaring it "free."
  Sub-1% is realistic for sampled CPU profiling; heap/lock profiling can
  cost more depending on runtime.

### 10b. eBPF

- **Adopt when** you need language-agnostic kernel-truth signals — L4/L7
  network flows (Cilium Hubble), whole-system CPU profiling without
  language agents (Parca Agent, Polar Signals Agent), runtime security
  (Falco, Tetragon)
  ([Cilium Hubble](https://docs.cilium.io/en/stable/observability/hubble/);
  Brendan Gregg, *BPF Performance Tools*, 2019). Not a replacement for
  app-level instrumentation — it sees syscalls and packets, not business
  semantics.
- **Platform caveats:** recent kernel (broadly Linux ≥ 4.18, ≥ 5.x for
  many modern probes); elevated capabilities (`CAP_BPF` / `CAP_SYS_ADMIN`)
  blocked by some hardened or FedRAMP/STIG images; not available in
  serverless / managed runtimes (Lambda, Cloud Run, Fargate without
  privileged mode, App Service); Windows eBPF coverage lags Linux.

---

## 11. Anti-patterns (reject in review)

- **Dashboard-driven debugging** — pre-built dashboards encode last
  incident's questions; build them as a *result* of investigation in a
  query tool, not the investigation itself.
- **Buying a monitoring vendor before having an instrumentation strategy.**
- **Logs-as-metrics** (counting log lines for rates) or **metrics-as-logs**
  (encoding identifiers as labels — cardinality death).
- **Custom trace propagation headers** — use W3C Trace Context.
- **Per-service log schemas** — pick OTel logs or ECS, enforce in pipeline.
- **Paging on causes for user-facing services without an SLO link** — use
  the §9a carve-outs deliberately.
- **Head-only sampling at low rates** — throws away the traces you want.
- **Instrumenting only with a vendor SDK** for new code.
- **PII in logs, baggage, or metric labels**, or audit logs sharing the
  observability pipeline and retention.

---

## 12. PR / repo checklists by maturity tier

Apply the tier matching the service's stage. Promote upward as scale and
reliability requirements grow. Don't demand high-scale controls from a
greenfield internal tool.

### Tier 1 — Minimum viable (every service, day one)

- [ ] Structured **JSON logs** on stdout with `service.name`,
      `service.version`, `deployment.environment`, severity, timestamp.
- [ ] **RED metrics** for every external endpoint via Prometheus or OTLP.
- [ ] **W3C `traceparent`** propagated on inbound and outbound HTTP/gRPC.
- [ ] One **SLO** per user-facing service with a symptom-based alert and
      runbook link.
- [ ] No `user_id` / `request_id` / raw URL on any Prometheus label.

### Tier 2 — Growth (multi-service, on-call, paying customers)

- [ ] All telemetry through an **OTel Collector** (DaemonSet/gateway per
      §3); no app-direct vendor SDKs in new code.
- [ ] Logs carry **`trace_id` / `span_id`** via OTel logs bridge.
- [ ] **Log shipper** (Vector / Fluent Bit / Collector `filelog`) for
      buffering, redaction, routing.
- [ ] **PII allowlist** enforced; audit logs on a separate pipeline.
- [ ] **Multi-window, multi-burn-rate** SLO alerts, codified.
- [ ] Cardinality controls live: `metric_relabel_configs`, top-N
      dashboard, `prometheus_tsdb_head_series` alert.
- [ ] **Telemetry budget** per team/service, tracked against bill.

### Tier 3 — High scale (high-volume, multi-region, regulated)

- [ ] **Tail sampling** in a Collector gateway with trace-affinity load
      balancing (§5a); buffering memory budgeted and alarmed.
- [ ] **Exemplars** end-to-end, verified on the live backend.
- [ ] **Long-term metrics storage** (Mimir / Thanos / VictoriaMetrics)
      with defined retention tiers and recording rules.
- [ ] **Continuous profiling** (§10a, overhead measured) and eBPF where
      platform supports (§10b); wide-event store (Honeycomb or
      ClickHouse-based) on highest-traffic services.
- [ ] Game-day or chaos exercise diagnosed using *only* observability
      tooling — no SSH, no grep.
