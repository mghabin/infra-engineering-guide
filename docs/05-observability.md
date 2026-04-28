# Chapter 05 — Observability

Opinionated, cloud-agnostic defaults for instrumenting and operating production
systems. The framing is deliberately heretical against the "three pillars,
buy-a-vendor, slap-on-some-dashboards" school: the goal of observability is to
let you **ask new questions of a running system without shipping new code**, and
most of the industry is still doing monitoring in an OTel t-shirt.

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint.

---

## 1. The mental model: events, not pillars

**Observability ≠ monitoring.** Monitoring answers questions you already knew
to ask (CPU > 90, error rate > 1%). Observability is the property of a system
that lets you debug *novel* failure modes — the unknown-unknowns — by slicing
arbitrary high-cardinality dimensions of its real behaviour, without a redeploy
(Majors, Fong-Jones & Miranda, *Observability Engineering*, O'Reilly 2022, ch. 1;
Sridharan, *Distributed Systems Observability*, O'Reilly 2018, ch. 4;
[Charity Majors, "Observability — A 3-Year Retrospective"](https://thenewstack.io/observability-a-3-year-retrospective/)).

**Reject "the three pillars" as a design framework.** Logs / metrics / traces
is a *vendor billing taxonomy*, not a model of your system. The unit of
observability is a **wide, structured event** with as many high-cardinality
dimensions attached as you can afford (user_id, tenant_id, build_sha,
shard, k8s_pod, feature_flag_state, …). Metrics and traces are projections of
that event stream; logs are degenerate events with bad schemas
(Sridharan, "Monitoring and Observability", 2017,
<https://copyconstruct.medium.com/monitoring-and-observability-8417d1952e1c>;
Majors, "Metrics: not the observability droids you're looking for",
<https://charity.wtf/2018/02/19/metrics-not-the-observability-droids-youre-looking-for/>).

**Cardinality is the asset, not the cost.** A metric `http_requests_total{code,
route}` with 50 routes and 5 codes is 250 series — useless for debugging *which
customer* is hitting *which build* on *which pod*. A wide event with the same
fields at request granularity is what lets you answer "why is p99 latency bad
for tenant X on canary Y in eu-west-1?" without code changes
(*Observability Engineering*, ch. 2 & 7; Majors,
<https://charity.wtf/2020/06/08/are-we-there-yet-the-thing-no-one-tells-you-about-observability/>).

> **Smell test.** If your debugging loop is "open dashboard → eyeball graph →
> SSH → grep logs → guess", you have monitoring, not observability.

---

## 2. Instrumentation: OpenTelemetry, full stop

**Use OpenTelemetry (OTel) for *all* new instrumentation.** OTel is the merged
OpenTracing/OpenCensus standard, a CNCF graduated/incubating project, and the
unambiguous winner of the instrumentation wars; the SDKs, the wire protocol
(OTLP), and the Collector are vendor-neutral and supported by every serious
backend (Honeycomb, Datadog, New Relic, Grafana, Splunk, Dynatrace, AWS, Azure,
GCP) ([OpenTelemetry spec](https://opentelemetry.io/docs/specs/otel/);
[CNCF — OpenTelemetry](https://www.cncf.io/projects/opentelemetry/);
[ThoughtWorks Tech Radar — OpenTelemetry: Adopt](https://www.thoughtworks.com/radar/tools/opentelemetry)).

**Don't** instrument directly against a vendor SDK (Datadog tracer, NewRelic
agent, AppInsights SDK) in 2026. You are deliberately re-creating the lock-in
that OTel was designed to remove. Vendor agents may *receive* OTLP, but your
code emits OTel.

**Run an OpenTelemetry Collector** as a sidecar (or per-node DaemonSet, or
deployment, in that order of preference) between your apps and your backend.
The Collector is where you do batching, retries, redaction, tail sampling,
attribute enrichment, and *backend swapping* — not in the application
([OTel Collector docs](https://opentelemetry.io/docs/collector/);
[OTel Collector deployment patterns](https://opentelemetry.io/docs/collector/deployment/)).

**Propagate W3C Trace Context (`traceparent` / `tracestate`) on every hop.**
This is the W3C Recommendation for distributed trace propagation and the OTel
default; deviating breaks every cross-vendor trace
([W3C Trace Context](https://www.w3.org/TR/trace-context/);
[W3C Baggage](https://www.w3.org/TR/baggage/)).

**Auto-instrument first, hand-instrument the domain.** Use language
auto-instrumentation for HTTP/gRPC/SQL/queue clients to get free spans; add
manual spans only for *business-meaningful* operations (`checkout.authorize`,
`pricing.recompute`) with rich attributes. Instrumenting every function is
noise — you want spans where decisions happen.

---

## 3. Structured logs: cheap events, not prose

**Logs are JSON. Always.** Plain-text logs are unparseable at scale and
hostile to correlation. Use a structured logger (`zap`/`zerolog` in Go,
`Serilog` in .NET, `structlog` in Python, `pino` in Node) emitting one JSON
object per event ([12-Factor — Logs](https://12factor.net/logs);
[OTel Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/);
Sridharan, *Distributed Systems Observability*, ch. 3).

**Adopt a single field schema.** Pick one and stick to it: **OpenTelemetry
Logs** (preferred for new systems — it's the same attribute model as traces)
or **Elastic Common Schema (ECS)** (good in legacy ELK shops). Mixing schemas
across services makes cross-service queries impossible
([OTel Logs Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/general/logs/);
[Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/index.html)).

**Every log line carries `trace_id` and `span_id`.** This is the cheapest
upgrade in observability and is automatic if your logger is wired to the OTel
context. Without it you cannot pivot from a slow trace to its logs in one
click ([OTel Logs Bridge API](https://opentelemetry.io/docs/specs/otel/logs/bridge-api/)).

**Don't use logs as metrics.** Counting log lines to derive a rate is the
classic "log-as-metric" anti-pattern: it is 100–1000× more expensive than a
counter, and unindexed substring matches are the #1 driver of $7-figure
SIEM bills (Sridharan, ch. 3; Majors,
<https://charity.wtf/2019/02/05/logs-vs-structured-events/>).

**Don't log at INFO in the hot path.** A request-rate log line at INFO is a
metric; demote it. Reserve INFO for state transitions, WARN for degraded but
serving, ERROR for "a human should look".

---

## 4. Distributed tracing

**Trace every external request and every async boundary.** A trace whose
spans stop at your service edge is theatre — the value of tracing is in the
*causal chain* across services, queues, and DBs. Instrument inbound HTTP/gRPC,
outbound clients, queue publishers and consumers, and DB drivers
(*Observability Engineering*, ch. 6;
[OTel semconv: trace](https://opentelemetry.io/docs/specs/semconv/general/trace/)).

**Sample tail-based, not head-based, for production.** Head sampling
(decide-at-root) keeps random 1% — and throws away the 1% of slow/failed
requests you actually need. Tail sampling (decide-after-trace-completes,
typically in the OTel Collector or a vendor) lets you keep 100% of errors,
100% of slow traces, and a small probabilistic sample of normal traffic
([OTel sampling](https://opentelemetry.io/docs/concepts/sampling/);
[OTel Collector tail-sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor);
*Observability Engineering*, ch. 17).

**Link metrics to traces with exemplars.** Prometheus and OTLP both support
exemplars — a `trace_id` attached to a histogram bucket — so a spike in p99
latency is one click away from an example trace that caused it. This kills
the "I see the spike but can't explain it" loop
([Prometheus exemplars](https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage);
[OTel exemplars spec](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars);
[Grafana — Exemplars](https://grafana.com/docs/grafana/latest/fundamentals/exemplars/)).

**Don't roll your own propagation.** No custom `X-My-Trace-Id` headers. Use
the W3C `traceparent` and let OTel handle it.

---

## 5. Metrics

**Services use RED, resources use USE.**

- **RED** (Rate / Errors / Duration) per service endpoint — Tom Wilkie's
  refactor of the four golden signals for request-driven services
  ([Tom Wilkie — RED method](https://thenewstack.io/monitoring-microservices-red-method/);
  [Grafana — RED method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)).
- **USE** (Utilization / Saturation / Errors) per resource (CPU, disk, NIC,
  pool) — Brendan Gregg's method for finding bottlenecks in resources
  ([Gregg — USE method](https://www.brendangregg.com/usemethod.html)).
- **Four Golden Signals** (latency, traffic, errors, saturation) at the
  service mesh / gateway layer — Google SRE Book
  ([Beyer et al., *SRE Book* ch. 6](https://sre.google/sre-book/monitoring-distributed-systems/)).

These are *complementary*, not competing. RED tells you "is the service
healthy from the user's point of view?"; USE tells you "is the box about to
fall over?"; golden signals are the executive summary.

**Default to Prometheus exposition + OTLP.** Pull-based scrape is the
operational sweet spot for in-cluster workloads; push (OTLP/HTTP) is correct
for serverless and short-lived jobs. Both are first-class citizens in OTel
([Prometheus best practices](https://prometheus.io/docs/practices/);
[Prometheus instrumentation](https://prometheus.io/docs/practices/instrumentation/);
[Prometheus naming](https://prometheus.io/docs/practices/naming/)).

**Cardinality discipline.** A label with unbounded values (user_id, request_id,
URL with path params, full error string) on a Prometheus metric will cost you
more than the rest of your observability stack combined
([Prometheus — labels best practices](https://prometheus.io/docs/practices/naming/#labels);
[Grafana Labs — cardinality](https://grafana.com/blog/2022/10/20/how-to-manage-high-cardinality-metrics-in-a-prometheus-environment/)).
Hard rules:

- No user identifiers, request IDs, or trace IDs on a Prom metric. Put them
  on the wide event / span instead.
- Normalise route templates (`/users/{id}`), never raw paths.
- Cap label cardinality per metric (~10⁴ series); review in CI via
  `promtool check metrics` and per-metric series count alerts.

**Don't** invent your own units. Follow Prometheus naming: base SI units,
`_total` suffix for monotonic counters, `_seconds` for durations,
`_bytes` for sizes.

---

## 6. Sampling

**Errors and slow requests: keep 100%.** Probabilistic sampling has no
business throwing away the data you actually need to debug. Configure the
Collector's tail-sampling processor with rules: `status==ERROR → keep`,
`duration > p99 → keep`, `tenant in {…allowlist…} → keep`, else 1–10%
probabilistic ([OTel tail-sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor);
*Observability Engineering*, ch. 17).

**Sample at trace granularity, never at span granularity.** A trace with
half its spans dropped is worse than no trace; the propagation contract is
"keep the whole tree or none of it."

**Logs: dynamic sampling on the high-volume class only.** If you're paying
for every DEBUG line at 50k RPS you have a budget problem; sample by event
class with a key (e.g. `route` + `status`) and emit a count so the rate is
preserved (Honeycomb's `dynsampler-go` is the reference implementation:
<https://github.com/honeycombio/dynsampler-go>).

---

## 7. Storage backends

**Defaults that survive contact with reality:**

| Signal | OSS default | Scale-out OSS | Managed/SaaS |
|---|---|---|---|
| Metrics | Prometheus (≤15d local) | Mimir / Thanos / VictoriaMetrics | Grafana Cloud, AMP, Datadog |
| Logs | Loki | Loki + S3 | Grafana Cloud, Datadog, Splunk |
| Traces | Tempo / Jaeger | Tempo + S3 | Honeycomb, Lightstep, Datadog |
| Wide events | — | ClickHouse (SigNoz, Highlight) | Honeycomb |
| Profiles | Pyroscope / Parca | Pyroscope + S3 | Polar Signals, Grafana Cloud |

**Single-binary Prometheus is fine until ~1M active series or 15 days of
retention.** Past that, push remote-write into Mimir, Thanos, or
VictoriaMetrics for horizontal scale and object-store backed long-term
retention. Don't run a 12-node Cortex cluster for 200k series
([Grafana Mimir](https://grafana.com/oss/mimir/);
[Thanos](https://thanos.io/);
[VictoriaMetrics](https://docs.victoriametrics.com/);
[Grafana Labs — LGTM stack](https://grafana.com/blog/2022/10/10/the-lgtm-stack-grafana-labs-approach-to-open-source-observability/)).

**Vendor vs OSS.** Honest tradeoff:

- **OSS LGTM (Loki/Grafana/Tempo/Mimir + Pyroscope)** — total control, lower
  unit cost at scale, but you operate it. Reasonable from ~50 engineers and
  a dedicated platform team.
- **Honeycomb** — best-in-class wide-event / high-cardinality querying;
  pick it if your debugging style is "ask a new question every time."
- **Datadog / New Relic / Dynatrace** — turnkey, expensive, easy to outgrow
  budget; right when time-to-first-signal beats unit cost.
- **ClickHouse-based stacks (SigNoz, Highlight, Uptrace)** — the rising
  middle ground: OTLP-native, single store for logs/metrics/traces, much
  cheaper than ELK at log scale ([SigNoz architecture](https://signoz.io/docs/architecture/);
  [ClickHouse for observability](https://clickhouse.com/use-cases/observability)).

**Don't** buy "monitoring as a service" before you have an instrumentation
strategy. The fastest way to a $500k/yr observability bill is to ship a
vendor agent into prod and let it index everything by default.

---

## 8. Alerting: SLO-based, multi-burn-rate

**Alert on user-visible symptoms, not causes.** "CPU 90%" is not a page; "p99
latency > 300ms for 5m on the checkout endpoint" might be. The Google SRE
canon is unambiguous: page on **service level objective (SLO) burn**, not on
infrastructure metrics ([SRE Book ch. 6](https://sre.google/sre-book/monitoring-distributed-systems/);
Beyer et al., *SRE Workbook* ch. 5,
<https://sre.google/workbook/alerting-on-slos/>; Hidalgo,
*Implementing Service Level Objectives*, O'Reilly 2020, ch. 7–8).

**Use the multi-window, multi-burn-rate pattern.** A single threshold either
pages too late (slow burn ignored) or too often (transient spike). The SRE
workbook's canonical pattern combines a *fast* window (e.g. 5m + 1h, burn
rate ≥ 14.4) with a *slow* window (e.g. 30m + 6h, burn rate ≥ 6) so you page
fast on real outages and slow on real budget loss
([SRE Workbook ch. 5 — Multi-window, multi-burn-rate](https://sre.google/workbook/alerting-on-slos/#6-multiwindow-multi-burn-rate-alerts);
[Liz Fong-Jones — SLO alerting](https://www.honeycomb.io/blog/alerting-on-slos);
[Honeycomb — SLO alerting math](https://www.honeycomb.io/blog/the-math-behind-slo-burn-alerts)).

**Every page must be actionable, novel, and about real user pain.** If the
on-call's first action is "check if it self-resolves," delete it. The SRE
workbook's "alerting on SLOs" chapter quantifies this: ticket-noise alerts
correlate with on-call attrition, not reliability
([SRE Workbook ch. 5](https://sre.google/workbook/alerting-on-slos/);
*SRE Book* ch. 6, "Setting Reasonable Expectations for Monitoring").

**Codify SLOs.** Use an SLO-as-code tool (`sloth`, `pyrra`, OpenSLO,
`slo-generator`) so SLO docs and alert rules cannot drift
([OpenSLO](https://openslo.com/); [sloth](https://sloth.dev/);
[pyrra](https://github.com/pyrra-dev/pyrra)).

---

## 9. Profiling: the fourth signal

**Run continuous profiling in production.** Sub-1% overhead profiling
(Pyroscope, Parca, Polar Signals, Datadog Continuous Profiler, Grafana Cloud
Profiles) closes the last gap: traces show *which* request is slow, profiles
show *which line of code* burned the CPU during it. Treat it as a peer of
metrics, not a special-occasion tool
([Grafana Pyroscope](https://grafana.com/oss/pyroscope/);
[Parca](https://www.parca.dev/);
[Polar Signals — continuous profiling](https://www.polarsignals.com/blog/posts/2022/04/26/start-continuous-profiling/);
[ThoughtWorks Tech Radar — Continuous profiling: Trial](https://www.thoughtworks.com/radar/techniques/observability-for-ci-cd-pipelines)).

**Default to pprof + eBPF where the language allows.** `pprof` is the
de-facto profile format (Go, Java, Python, Ruby, Node via wrappers). For
languages without good runtime profilers, use eBPF-based whole-system
profilers (Parca Agent, Polar Signals Agent) — no app changes, kernel-level
sampling ([pprof format](https://github.com/google/pprof/blob/main/doc/README.md);
[Parca Agent](https://www.parca.dev/docs/parca-agent);
Brendan Gregg, [*Systems Performance, 2nd ed.*](https://www.brendangregg.com/systems-performance-2nd-edition-book.html), ch. 6;
Gregg, [Flame Graphs](https://www.brendangregg.com/flamegraphs.html)).

---

## 10. eBPF: kernel-level observability for free

**Use eBPF tools for cheap, language-agnostic, kernel-truth signals.** eBPF
lets you observe syscalls, network flows, and process events without
instrumenting the application — useful for legacy services, polyglot
estates, and security/network telemetry
([Pixie](https://docs.px.dev/about-pixie/what-is-pixie/);
[Cilium Hubble](https://docs.cilium.io/en/stable/observability/hubble/);
[Parca Agent](https://www.parca.dev/docs/parca-agent);
Brendan Gregg, [*BPF Performance Tools*](https://www.brendangregg.com/bpf-performance-tools-book.html), 2019;
[ThoughtWorks Tech Radar — eBPF: Adopt](https://www.thoughtworks.com/radar/platforms/ebpf);
[Frederic Branczyk — eBPF for observability](https://www.polarsignals.com/blog/posts/2022/12/01/ebpf-and-continuous-profiling)).

**Realistic uses in 2026:**

- L4/L7 network observability and policy enforcement (Cilium + Hubble).
- Continuous CPU profiling without language agents (Parca Agent, Polar
  Signals Agent).
- Auto-instrumented golden signals from kernel sockets (Pixie, Beyla,
  Coroot).
- Runtime security (Falco, Tetragon).

**Don't** treat eBPF as a replacement for app-level instrumentation. It
sees syscalls and packets, not your business semantics. It's a complement.

---

## 11. Anti-patterns (reject in review)

- **Dashboard-driven debugging.** Pre-built dashboards encode last incident's
  questions; they cannot answer the next one. Build dashboards as a *result*
  of an investigation in a query tool, not as the investigation itself
  (*Observability Engineering*, ch. 1 & 11; Majors, "Observability — A 3-Year
  Retrospective").
- **Buying a monitoring vendor before having an instrumentation strategy.**
  You will pay 3–10× and lock yourself into their schema.
- **Logs-as-metrics** (counting log lines for rates, parsing them in alert
  rules). Use a counter.
- **Metrics-as-logs** (encoding identifiers as labels). Cardinality death.
- **Custom trace propagation headers.** Use W3C Trace Context.
- **Per-service log schemas.** Pick OTel logs or ECS, enforce in CI.
- **Alerting on causes (CPU, memory, queue depth) instead of symptoms.**
  Pages the on-call for things users do not feel.
- **Head-only sampling at low rates.** Throws away exactly the traces you
  want.
- **Instrumenting only with a vendor SDK.** Re-creates the lock-in OTel
  exists to remove.

---

## 12. PR / repo checklist

- [ ] All new services emit OTel traces, metrics, and logs via OTLP to a
      Collector — no direct vendor SDKs.
- [ ] W3C `traceparent` propagation verified end-to-end in an integration test.
- [ ] Every log line carries `trace_id` / `span_id` (OTel logs bridge enabled).
- [ ] Logs are JSON, validated against OTel-logs or ECS schema in CI.
- [ ] No high-cardinality labels on Prometheus metrics (`promtool` check +
      per-metric series budget alert).
- [ ] RED metrics for every HTTP/gRPC endpoint; USE for every owned resource.
- [ ] Tail sampling configured in the Collector with `error → keep`,
      `slow → keep`, plus probabilistic baseline.
- [ ] Exemplars enabled on histogram metrics; one-click jump from latency
      panel to trace works.
- [ ] At least one SLO defined per user-facing service, codified
      (`sloth`/`pyrra`/OpenSLO), with multi-window multi-burn-rate alerts.
- [ ] No alert pages on raw infra metrics without a documented SLO link.
- [ ] Continuous profiling (Pyroscope/Parca) enabled in prod, <1% overhead
      verified.
- [ ] Runbook for every paging alert; alert fires include trace exemplars
      and dashboard deep-links.
