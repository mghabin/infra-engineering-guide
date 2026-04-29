# Chapter 05 — Observability

Opinionated, cloud-agnostic defaults for instrumenting and operating
production systems. Deliberately heretical against the "three pillars,
buy-a-vendor, slap-on-dashboards" school: the goal is to **ask new
questions of a running system without shipping new code**.

> **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only
> with measurement or a documented constraint.

---

## 1. The mental model: events, not pillars

- **Observability ≠ monitoring.** Monitoring answers questions you
  already knew to ask; observability lets you debug *novel* failure
  modes by slicing arbitrary high-cardinality dimensions without a
  redeploy (Majors, Fong-Jones & Miranda, *Observability Engineering*,
  O'Reilly 2022, ch. 1).
- **Reject "the three pillars" as a design framework.** Logs / metrics /
  traces is a *vendor billing taxonomy*. The unit of observability is a
  **wide structured event** ("wide event" / "structured event" are
  short forms) with as many high-cardinality dimensions as you can
  afford (user_id, tenant_id, build_sha, shard, k8s_pod,
  feature_flag_state). Metrics and traces are projections of that
  stream; logs are degenerate events with bad schemas. Distinct from
  ch08's *domain events* — see glossary.
- **Cardinality is an asset *in the right signal*.** A wide event with
  per-request user/tenant/build fields answers "why is p99 bad for
  tenant X on canary Y in eu-west-1?" without code changes; the same
  fields on a Prometheus metric are a billing incident — see §6.

> **Smell test.** If your debugging loop is "open dashboard → eyeball graph → SSH → grep logs → guess", you have monitoring, not observability.

---

## 2. Instrumentation: OpenTelemetry as the preferred default

- **Use OpenTelemetry (OTel) for all new instrumentation.** Merged
  OpenTracing/OpenCensus, CNCF; SDKs, OTLP, and the Collector are
  accepted by every serious backend
  ([spec](https://opentelemetry.io/docs/specs/otel/);
  [CNCF](https://www.cncf.io/projects/opentelemetry/)). Don't instrument
  against vendor SDKs (Datadog tracer, NewRelic agent, AppInsights) for
  new code; vendor agents may *receive* OTLP, your code emits OTel.
- **OTel reduces instrumentation lock-in, not backend lock-in.**
  Switching emitters is cheap; switching backends still costs query
  languages (PromQL/LogQL/SPL/DD-query), alerts, dashboards, retention,
  operator muscle memory — Datadog and Honeycomb publicly agree
  ([DD](https://www.datadoghq.com/blog/opentelemetry-instrumentation/);
  [Honeycomb](https://www.honeycomb.io/blog)).
  Plan migrations on the *backend* axis separately.
- **Propagate W3C Trace Context (`traceparent` / `tracestate`) on every
  hop** ([Trace Context](https://www.w3.org/TR/trace-context/);
  [Baggage](https://www.w3.org/TR/baggage/)). When migrating from a
  legacy propagator (B3, Jaeger, X-Ray), use OTel's *composite*
  propagator until the legacy producer retires.
- **Baggage is for small, non-sensitive correlation keys** (tenant tier,
  feature flag bucket, request class). No PII, auth tokens, or large
  payloads — baggage propagates downstream and crosses trust boundaries.
- **Auto-instrument first, hand-instrument the domain.** Auto for
  HTTP/gRPC/SQL/queue clients; manual spans only for *business-meaningful*
  operations (`checkout.authorize`, `pricing.recompute`).

---

## 3. Collector deployment: pick by use case, not ranking

Run an OpenTelemetry Collector between apps and backends so batching,
retries, redaction, sampling, enrichment, and backend swapping live
outside application code
([Collector](https://opentelemetry.io/docs/collector/);
[deployment patterns](https://opentelemetry.io/docs/collector/deployment/)).

| Pattern | Use when | Avoid when |
|---|---|---|
| **Agent / DaemonSet (per node)** | Default for K8s node-local collection: host metrics, kubelet/cAdvisor scrape, log tail from `/var/log/pods`, OTLP on `localhost`. | You need cross-node aggregation (tail sampling, dedup) — DaemonSets only see one node. |
| **Gateway / Deployment (central)** | Fleet-wide processors needing a *full view*: tail-sampling, span metrics, cardinality enforcement, fan-out, egress auth, cost routing. Scaled horizontally behind an LB. | Tiny estates where one agent tier is enough; latency-sensitive paths. |
| **Sidecar (per pod)** | Strict per-workload isolation (multi-tenant, regulated workloads needing per-team redaction), or apps that can only emit to `localhost`. | Default case — sidecars multiply CPU/memory/config per replica. |

- **Typical shape:** apps → DaemonSet agent → Gateway (tail sampling,
  routing, redaction) → backends. Sidecars only where justified. Tail
  sampling needs a gateway tier with trace-affinity LB (§5a).
- **Collector v1 is per-module, not a single GA.** Core API/config
  reached 1.0 incrementally in 2024–25; the *contrib* distro
  (`tailsamplingprocessor`, `loadbalancingexporter`, `filelogreceiver`,
  most vendor exporters) is still pre-1.0 — pin versions and read
  CHANGELOG on every upgrade
  ([release docs](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/release.md)).

---

## 4. Structured logs: cheap events, not prose

- **Logs are JSON, wired through the OTel Logs Bridge.** Plain-text
  logs are unparseable at scale. Prefer the framework-native logger
  plus a first-party OTel bridge (Go `log/slog` + `otelslog`; .NET
  `Microsoft.Extensions.Logging` + OTel logging provider; Python
  `logging`/`structlog`; Node `pino`). `zap`/`zerolog`/`Serilog` remain
  acceptable on existing services. One JSON object per event
  ([12-Factor](https://12factor.net/logs);
  [Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)).
  Logs Bridge API is Stable; Logs SDK is Stable in Java/.NET/C++/PHP
  and Beta-or-earlier in Go/Python/JS — which is why the bridge pattern
  is doing real work ([status](https://opentelemetry.io/status/)).
- **Adopt a single field schema.** **OTel Logs** (preferred for new
  systems — same attribute model as traces) or **Elastic Common Schema
  (ECS)** (legacy ELK). Mixing schemas across services makes
  cross-service queries impossible.
- **Standard fields on every event** (per OTel semconv): `timestamp`
  (RFC 3339 UTC), `severity_text`+`severity_number`, `service.name`,
  `service.version`, `deployment.environment`, `trace_id`, `span_id`,
  `event.name`, `code.namespace`/`code.function`. Every log inside a
  traced operation carries `trace_id`/`span_id`
  ([Bridge API](https://opentelemetry.io/docs/specs/otel/logs/bridge-api/)).
- **Don't use logs as metrics** (orders of magnitude more expensive
  than a counter) and **don't log at INFO in the hot path**. Reserve
  INFO for state transitions, WARN for degraded but serving, ERROR for
  "a human should look".

### 4a. Log shippers and ingestion

- **Use a dedicated log shipper, not the application.** App writes JSON
  to stdout; a node-local shipper handles buffering, batching, retry,
  fan-out, back-pressure. Defaults: **Vector** (Rust, expressive VRL);
  **Fluent Bit** (C, tiny, de-facto K8s node agent); **OTel Collector
  with `filelog` receiver** when you want one binary for all signals.
  Don't ship logs from app code over the network — lose events on
  restart, leak credentials ([vector](https://vector.dev/docs/);
  [fluent-bit](https://docs.fluentbit.io/)).

### 4b. PII, redaction, and audit logs

- **Redact at the trust boundary you control** — in the shipper or
  Collector (Vector VRL, OTel `attributes`/`transform`) *before* egress.
  Backend-side redaction means raw PII is on disk you don't own.
  Maintain a **field allowlist** (public / internal / PII / secret);
  anything else is dropped or hashed. Never log secrets, full auth
  headers, raw card numbers, or full request bodies; reference by
  opaque IDs that resolve in a separate, access-controlled store.
- **Audit logs are not observability logs.** SOC 2 / HIPAA / PCI / GDPR
  events need a **separate pipeline** (append-only, tamper-evident,
  multi-year retention), different access controls, and a different
  schema (mandatory actor, action, resource, outcome, source IP,
  request ID). Canonical definition is owned by ch06 §15 — this section
  covers only *transport*.

---

## 5. Distributed tracing and sampling

- **Trace every external request and every async boundary.** A trace
  whose spans stop at your service edge is theatre — value is in the
  *causal chain* across services, queues, DBs. Instrument inbound
  HTTP/gRPC, outbound clients, queue publishers/consumers, DB drivers
  ([trace semconv](https://opentelemetry.io/docs/specs/semconv/general/trace/)).
- **Sample at trace granularity, never at span granularity.** A trace
  with half its spans dropped is worse than no trace — keep the whole
  tree or none of it
  ([sampling](https://opentelemetry.io/docs/concepts/sampling/)). Use
  W3C `traceparent` (§2).

### 5a. Tail-based sampling: rules and constraints

**Tail-based sampling** ("tail sampling" short form) decides after the
trace completes, in a Collector gateway tier
([tailsamplingprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)).
Policies are **fixed**: `status_code`, `latency` (fixed-ms), attribute,
`probabilistic`, `rate_limiting`, composites — *not* dynamic "current
p99". Derive thresholds from your SLO.

- **Policy template** (compose with `and`/`composite`): `status_code =
  ERROR` → keep; `latency > <SLO ms>` → keep; high-value tenant → keep;
  `probabilistic` baseline (1–10%); `rate_limiting` cap on spans/sec.
- **Trace affinity is required.** All spans must reach the *same*
  collector instance, or the sampler decides on a partial trace. Run a
  single gateway replica (small scale) or a `loadbalancing` exporter
  keyed on `trace_id` in front of the tail-sampling tier
  ([loadbalancingexporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter)).
- **Memory cost is real.** Buffers all spans for `decision_wait` (10–30s);
  budget `peak spans/sec × decision_wait × avg span size` plus headroom.
  By scale: < ~100 traces/sec keep 100%. Mid/high — keep all errors and
  SLO-violators with a probabilistic baseline and `rate_limiting` cap;
  route overflow to cheap storage or drop with a counter so the rate
  stays observable.
- **Alarm on processor health, not just OOM** —
  `otelcol_processor_tail_sampling_*` queue depth and dropped-trace
  counters. At very high volume, a purpose-built sampler (Honeycomb
  Refinery) or head + reservoir sampling (X-Ray / Datadog APM) is a
  legitimate alternative — the goal is policy-driven retention, not
  the specific processor
  ([Refinery](https://docs.honeycomb.io/manage-data-volume/refinery/)).

### 5b. Linking metrics to traces with exemplars

Exemplars attach a sampled `trace_id` to a metric so a latency spike
pivots to an example trace
([OTel](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars);
[Prom](https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage)).
Every link must support them: client records, wire carries (OpenMetrics
or OTLP — legacy Prometheus text format does **not**), storage persists
(Prom exemplar storage feature-flagged in-memory; **RW 2.0** preserves
by spec but Mimir/Thanos versions vary), UI surfaces. Most reliable on
the **Grafana LGTM stack** with Prometheus 3.0+. Vendor APM (Datadog,
NewRelic, Dynatrace) pivot via their own correlation, *not* the
exemplar wire format. Don't promise "click latency to trace" until
demonstrated on the *current* backend.

### 5c. Browser and mobile RUM

RUM measures user-perceived latency a server-side trace cannot. OTel
JS-browser/Mobile SDKs are pre-Stable; production RUM is dominated by
vendor SDKs (Datadog RUM, NewRelic Browser, Sentry, Honeycomb web)
shipping session replay, error grouping, Core Web Vitals. Pragmatic
pattern: vendor RUM correlates to server-side OTel traces by
`session_id`, *not* by a unified W3C `traceparent` from browser to
backend (`Server-Timing` exists but is rarely wired). Migrate when OTel
client SDKs reach Stable.

---

## 6. Metrics

- **Services use RED, resources use USE, edge uses Four Golden Signals.**
  **RED** (Rate/Errors/Duration) per service endpoint;
  **USE** (Utilization/Saturation/Errors) per resource;
  **Four Golden Signals** at mesh/gateway
  ([Wilkie](https://thenewstack.io/monitoring-microservices-red-method/);
  [Gregg](https://www.brendangregg.com/usemethod.html);
  [SRE Book ch. 6](https://sre.google/sre-book/monitoring-distributed-systems/)).
- **Default to Prometheus exposition + OTLP** — pull-based scrape for
  long-lived in-cluster workloads; push (OTLP/HTTP) for serverless and
  short-lived jobs ([naming](https://prometheus.io/docs/practices/naming/)).
  Base SI units, `_total` for counters, `_seconds`, `_bytes`.
- **Prometheus 3.0+ is itself an OTLP receiver**
  (`/api/v1/otlp/v1/metrics`), with **Remote Write 2.0** (exemplars,
  metadata, native histograms) and UTF-8 metric/label names so OTel
  dot-notation no longer needs underscore mangling
  ([Prom 3.0](https://prometheus.io/blog/2024/11/14/prometheus-3-0/);
  [RW 2.0](https://prometheus.io/docs/specs/remote_write_spec_2_0/)).
  Prefer Collector → OTLP-write into Prometheus over text-format scrape
  for new pipelines. **Native histograms remain experimental** in
  Prometheus 3.x (`--enable-feature=native-histograms`); opt-in only.

### 6a. Signal vs cardinality

| Signal | Cardinality budget | Retention | Put here |
|---|---|---|---|
| **Metrics** (Prometheus / OTLP) | **Bounded** — series count must be a small multiple of label-domain product; no unbounded labels. | Long (months–years), aggregate. | Rates, percentiles, resource utilisation, SLI numerators/denominators. |
| **Traces** | High per trace; total bounded by sampling. | Days–weeks. | Per-request causal chains, slow/error exemplars, cross-service timing. |
| **Wide events / structured logs** | Highest; one row per request is fine. | Short by default, longer for audit. | Per-request debug context, business events, anything with `user_id` / `request_id` / `build_sha`. |
| **Profiles** | Aggregated by symbol; bounded. | Short rolling window. | CPU / memory / lock attribution to source lines. |

- Hard rule: **never** put `user_id`, `request_id`, `trace_id`, full
  URL paths, or raw error messages on a Prometheus metric label
  ([labels](https://prometheus.io/docs/practices/naming/#labels)); put
  them on the wide event or span attribute. Normalise route templates
  (`/users/{id}`), never raw paths.
- **In OTel, the closest thing to a wide event is a richly-attributed
  span.** Default to over-attributing spans (with bounded keys and §4b
  PII rules). A separate event stream is justified only for operations
  with no natural span (batch finalisation, business outcome) or when
  committed to a wide-event backend (Honeycomb, ClickHouse-based).

### 6b. Enforcing cardinality in production

`promtool check metrics` validates exposition *format and naming* but
cannot enforce cardinality. Do this at runtime:

- **Relabel/drop at scrape time** (`metric_relabel_configs` with
  `action: drop`/`labeldrop`/`replace`) to neutralise labels before the
  TSDB; **allowlist labels per metric** in the Collector
  (`transform`/`attributes`) so unknown labels are stripped at the
  gateway, not the application
  ([relabel config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#metric_relabel_configs)).
- **Top-N series dashboards** (`topk(20, count by (__name__)({...}))`)
  reviewed weekly; **alert on `prometheus_tsdb_head_series`** at 80%
  capacity; recording rules track series count per metric so cost
  regressions show up as a graph, not a bill. CI is best-effort:
  `promtool check rules`, lint label names against an allowlist, reject
  obvious offenders (`user_id`, `request_id`, `email`) by regex — CI
  cannot catch unbounded values.
- **For multi-tenant TSDBs (Mimir/Cortex/Thanos), enforce backend-side
  per-tenant limits** — `max_global_series_per_user`,
  `max_global_series_per_metric`, ingester/receive limits — the actual
  write-path enforcement; scrape-time relabel does not protect a
  shared remote-write target
  ([Mimir](https://grafana.com/docs/mimir/latest/configure/configure-limits/)).

---

## 7. Long-term storage and federation

- **Single-binary Prometheus is fine until ~1M active series or local
  retention (default 15d).** Past that, push remote-write to a scale-out
  backend ([remote-write](https://prometheus.io/docs/practices/remote_write/)).
  OSS scale-out: **Mimir**
  ([arch](https://grafana.com/docs/mimir/latest/references/architecture/));
  **Thanos** ([components](https://thanos.io/tip/components/));
  **VictoriaMetrics** ([docs](https://docs.victoriametrics.com/)). Don't
  run Cortex new — Mimir is the maintained fork.
- **Storage backend defaults:**

| Signal | OSS default | Scale-out OSS | Managed/SaaS |
|---|---|---|---|
| Metrics | Prometheus (≤15d local) | Mimir / Thanos / VictoriaMetrics | Grafana Cloud, AMP, Datadog |
| Logs | Loki | Loki + S3 | Grafana Cloud, Datadog, Splunk |
| Traces | Tempo / Jaeger | Tempo + S3 | Honeycomb, Datadog |
| Wide events | — | ClickHouse (SigNoz, Highlight) | Honeycomb |
| Profiles | Pyroscope / Parca | Pyroscope + S3 | Polar Signals, Grafana Cloud |

- **Backend choice — pick the default for your shape, don't shop the matrix every quarter.** Instrumentation stays OTel either way, so a backend swap is mechanical rewiring of exporters, not a re-instrumentation project ([OpenTelemetry — vendor-neutral, portable telemetry](https://opentelemetry.io/docs/what-is-opentelemetry/)).
  - **< 50 engineers, no platform team, fast-moving:** standardize on a single managed SaaS — **Datadog**, **Honeycomb**, **Grafana Cloud**, or **New Relic** — and accept ~2 years of lock-in. Time-to-insight dominates unit cost at this size; running Mimir/Loki/Tempo yourself is a second product you can't afford. ([Datadog pricing](https://www.datadoghq.com/pricing/); [Honeycomb on the real cost of observability](https://www.honeycomb.io/blog/cost-complexity-observability-2-0); [Grafana Cloud pricing](https://grafana.com/pricing/)).
  - **50–500 engineers, platform team exists:** managed SaaS for **traces + RUM** (where vendor query UX and session replay still beat OSS), OSS **Grafana stack (Mimir / Loki / Tempo)** for **metrics + logs** *if* the platform team has the on-call capacity to run it; otherwise stay all-SaaS. The split is justified only when the OSS unit-cost win clears the platform-team payroll ([Mimir sizing & operations](https://grafana.com/docs/mimir/latest/manage/run-production-environment/planning-capacity/); [Loki sizing](https://grafana.com/docs/loki/latest/setup/size/)).
  - **500+ engineers, regulated or cost-pressured:** OSS **Grafana stack (Mimir / Loki / Tempo)** or **Elastic / OpenSearch** in-house, with FinOps governance on telemetry spend (ch10). Accept the platform-team cost; at this scale SaaS list-price scales worse than headcount ([Honeycomb — observability TCO](https://www.honeycomb.io/blog/cost-complexity-observability-2-0); [Grafana Mimir architecture](https://grafana.com/docs/mimir/latest/references/architecture/)).
  - **In all three shapes:** keep instrumentation on OTel SDKs/Collector, keep dashboards/alerts in code (Grafana JSON / Jsonnet / Terraform), prefer query languages with multiple implementations (PromQL, LogQL). Then a backend swap costs weeks, not quarters.

---

## 8. Telemetry cost model

Telemetry is a workload with a budget. Allocation, showback/chargeback,
anomaly detection are owned by ch10 §11–12.

| Signal | Dominant cost driver | Notes |
|---|---|---|
| Metrics | active series × retention | Cheap per series, brutal when cardinality leaks. |
| Traces | spans/sec × retention × sampling | Cheap with tail sampling; ruinous at 100% head. |
| Logs | bytes × indexed fields × retention | Most expensive in practice; index discipline > volume. |
| Wide events | events/sec × attributes × retention | Cheaper than indexed logs in columnar stores. |
| Profiles | hosts × profile rate × retention | Cheap with sub-1% sampling. |

- **Set per-team/per-service budgets** in units the backend bills in
  (series, GB, span count). Surface burn-down alongside SLOs.
- **Drop, don't store-and-archive, what you don't query.** Cold logs
  nobody opens are a compliance question, not an observability one.
- **Vendor switching cost is dominated by non-OTel pieces:** dashboards,
  alert rules, query muscle memory, runbooks. Keep dashboards/alerts in
  code (Grafana JSON / Jsonnet / Terraform); prefer query languages
  with multiple implementations (PromQL, LogQL).

---

## 9. Alerting: SLO-based for user-facing services, plus carve-outs

- **For user-facing request/response services, alert on user-visible
  symptoms, not causes.** "CPU 90%" is not a page; "p99 latency > 300ms
  for 5m on the checkout endpoint" is. Page on **SLO burn**, not raw
  infra metrics (Hidalgo, *Implementing Service Level Objectives*,
  O'Reilly 2020, ch. 7–8). Use **multi-window, multi-burn-rate** alerts:
  fast (5m + 1h, burn ≥ 14.4) plus slow (30m + 6h, burn ≥ 6)
  ([SRE Workbook ch. 5 §6, Table 5-3](https://sre.google/workbook/alerting-on-slos/#6-multiwindow-multi-burn-rate-alerts)).
- **Codify SLOs — see ch09 §2.2 (canonical owner).** The spec must
  generate the §2.1 alert rules, not be paraphrased into them. ch09 owns
  tool selection (Sloth, Pyrra, OpenSLO, slo-generator, managed
  equivalents) and the spec-as-code workflow.

### 9a. Legitimate non-symptom pages (carve-outs)

Symptom-first ≠ "no cause-based alerts." Page on these without an SLO
violation, provided each is actionable with a runbook:

- **Capacity/saturation with lead time** — disk > 85% and growing,
  queue depth growing for N min, connection pool > 90%, cert/credential
  expiring < 14 days, cloud quota > 80%.
- **Hard dependency down or security event** — upstream API 100%
  errors, primary DB failover, broker partition unavailable, IDS/EDR
  alert, anomalous auth, audit pipeline gap, key rotation overdue.
- **Batch, pipelines, internal control planes** without continuous
  request traffic — job failure, SLA miss, dead-letter growth, backup
  restore-test failure, CI/CD or IdP outage. (IdP outage is the canonical break-glass scenario — the bypass-federation identity that lets you sign in when the IdP is the incident is owned by [ch12 §8 Break-glass](./12-identity.md#8-break-glass).)

### 9b. Synthetic monitoring (zero-traffic gap)

SLO burn alerts don't fire when the denominator collapses to zero — a
regional outage, off-hours B2B traffic, or broken ingress can leave a
dead service "technically meeting SLO." Run synthetic probes where
plausible: **Blackbox Exporter** (HTTP/TLS/cert/TCP/ICMP), **Grafana
k6** (scripted journeys), or vendor synthetics — a synthetic failure
pages even when SLO burn is nominal
([blackbox](https://github.com/prometheus/blackbox_exporter);
[k6](https://k6.io/docs/)).

---

## 10. Profiling and eBPF (maturity-gated)

Continuous profiling and eBPF earn their place *after* metrics, logs,
and traces are working.

### 10a. Continuous profiling

- **Adopt when** unexplained CPU/memory cost regressions, latency
  outliers traces don't explain, or non-trivial cloud bill from
  compute. Tooling: Pyroscope, Parca, Polar Signals, Datadog Continuous
  Profiler, Grafana Cloud Profiles. Profile format: `pprof`. **OTel
  Profiling signal** data model merged 2024 but no SDK is Stable yet —
  use vendor/OSS profilers
  ([Pyroscope](https://grafana.com/oss/pyroscope/);
  [Parca](https://www.parca.dev/);
  [OTel Profiling](https://opentelemetry.io/blog/2024/profiling/)).
  Verify overhead in your environment; sub-1% is realistic for sampled
  CPU profiling, heap/lock can cost more.

### 10b. eBPF

- **Adopt when** you need language-agnostic kernel-truth signals — L4/L7
  network flows (Cilium Hubble), whole-system CPU profiling (Parca /
  Polar Signals), runtime security (Falco, Tetragon), or zero-code
  RED-metric/HTTP-span coverage on uninstrumented services (**Grafana
  Beyla**) ([Hubble](https://docs.cilium.io/en/stable/observability/hubble/);
  [Beyla](https://grafana.com/docs/beyla/latest/);
  Brendan Gregg, *BPF Performance Tools*, 2019). Beyla is a defensible
  Tier-1 stop-gap before OTel SDK instrumentation lands. eBPF sees
  syscalls and packets, not business semantics — not a general
  replacement for app-level instrumentation.
- **Platform caveats:** recent kernel (Linux ≥ 4.18, ≥ 5.x for many
  modern probes); elevated capabilities (`CAP_BPF`/`CAP_SYS_ADMIN`)
  blocked by hardened/FedRAMP/STIG images; not available in
  serverless/managed runtimes (Lambda, Cloud Run, Fargate without
  privileged mode, App Service); Windows eBPF lags Linux.

---

## 11. Anti-patterns (reject in review)

- **Dashboard-driven debugging** — pre-built dashboards encode the last
  incident's questions; build them as a *result* of investigation.
- **Buying a vendor before having an instrumentation strategy.**
- **Logs-as-metrics** (counting log lines for rates) or **metrics-as-logs**
  (identifiers as labels — cardinality death).
- **Custom trace propagation headers** — use W3C Trace Context.
- **Per-service log schemas** — pick OTel logs or ECS, enforce in pipeline.
- **Paging on causes for user-facing services without an SLO link** —
  use the §9a carve-outs deliberately.
- **Head-only sampling at low rates** — throws away the traces you want.
- **Instrumenting only with a vendor SDK** for new code.
- **PII in logs, baggage, or metric labels**, or audit logs sharing the
  observability pipeline.
- **App code emitting syslog (RFC 3164/5424) directly in cloud
  workloads** — write JSON to stdout, let the node agent route; syslog
  is acceptable only as transport from network appliances or legacy hosts.

---

## 12. PR / repo checklists by maturity tier

Apply the tier matching the service's stage; promote upward as scale grows.

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

- [ ] **Tail sampling** in a Collector gateway with trace-affinity LB
      (§5a); buffering memory budgeted and alarmed.
- [ ] **Exemplars** end-to-end, verified on the live backend.
- [ ] **Long-term metrics storage** (Mimir / Thanos / VictoriaMetrics)
      with retention tiers and recording rules.
- [ ] **Continuous profiling** (§10a, overhead measured) and eBPF where
      platform supports (§10b); wide-event store on highest-traffic
      services.
- [ ] Game-day or chaos exercise diagnosed using *only* observability
      tooling — no SSH, no grep.
