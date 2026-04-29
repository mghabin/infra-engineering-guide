# Chapter 09 — Reliability (SRE)

Opinionated, cloud-agnostic operational discipline. The Google SRE Book and SRE Workbook are
the canon; this chapter distills them into defaults you can land in a PR. Reliability is a
product feature with a budget — not a vibe, not a status page.

> Conventions: **Do** = required default; **Don't** = reject in review unless a written
> exception exists; **Prefer** = strong default, deviate only with measurement.

---

## 1. SLI, SLO, and the error budget

**100% is the wrong target.** Past a certain point, each extra nine costs more than the user
perceives, and chasing it freezes the product. Pick a target that matches user expectations
and the dependency chain beneath you (Beyer et al., *SRE Book* ch.4 "Service Level
Objectives"; Hidalgo, *Implementing Service Level Objectives* ch.3).

- **SLI** (indicator): a *ratio of good events to valid events*, measured at the point
  closest to the user (e.g. load balancer, CDN edge, mobile RUM). Prefer ratios over
  averages — averages hide the bad tail (SRE Workbook ch.2). SLIs are
  aggregations *of* the wide structured events defined in ch05 §1; carrying
  the high-cardinality dimensions on the events themselves does not violate
  the metrics-cardinality rule in ch05 §6a — only Prometheus-class label
  series have a cardinality budget.
- **SLO** (objective): a target on that ratio over a *rolling window*. Pick one default
  window per org and stick to it; **30 days** matches the SRE Workbook worked examples
  (ch.5, Table 5-3) and is the recommended default. Quarterly is fine for batch.
- **Error budget**: `1 − SLO` expressed in the same units as the SLI.

### 1.1 Event-based vs time-based budgets

Pick the one that matches how the SLI is counted (SRE Workbook ch.2; Hidalgo ch.4):

- **Event-based.** SLI is `good_events / valid_events`; budget is *bad events allowed*.
  99.9% over 30d on 1B requests → 1,000,000 bad requests. Use for request-driven services
  where traffic is the natural denominator.
- **Time-based.** SLI is `good_time / valid_time` (e.g. probe minutes); budget is *minutes
  of badness*. 99.9% over 30d ≈ 43m 12s; over 28d ≈ 40m 19s. Use for low-traffic services
  and synthetic probing where event counts are too sparse to drive decisions.
- **Window consistency.** Whatever window you pick (28d for sprint alignment, 30d as
  default), the SLI math, alert table, and policy doc must all reference the *same* window.

### 1.2 Picking SLIs

**Choose SLIs from user journeys, not from CPU graphs.** For each critical journey ("user
logs in", "checkout completes", "search returns results"), pick at most 2–3 SLIs from this
menu (SRE Workbook ch.2; Hidalgo ch.4):

| Journey shape          | SLI category   | Example                                        |
| ---------------------- | -------------- | ---------------------------------------------- |
| Synchronous request    | Availability   | `2xx+3xx / valid_requests`                     |
| Synchronous request    | Latency        | `p99 < 400ms` measured as `fast / valid`       |
| Async pipeline         | Freshness      | `messages processed within 5min / total`       |
| Storage / read         | Correctness    | `bytes_returned == bytes_expected`             |
| Stream / batch         | Throughput     | `events_per_sec ≥ floor`                       |

- **User-journey SLOs are the primary contract** with product and customers; they drive
  error-budget policy and paging.
- **Component SLOs** (a microservice, a queue, a cache) are *diagnostic and supporting* —
  they help locate burn and let platform teams negotiate internal targets, but should not
  page on their own unless they are the journey (SRE Workbook ch.2 §"SLOs for services with
  dependencies").
- **Don't** define a *user-facing* SLO on a metric the user cannot feel (CPU%, queue depth,
  GC pauses) — those are *causes*; user SLOs measure *symptoms* (SRE Book ch.6).

### 1.3 Error budget policy and enforcement

The policy is a written contract signed by product + eng + SRE *before* you ever exceed it
(SRE Workbook ch.4). Minimum clauses:

1. **Healthy budget** → ship faster, run gamedays, raise risk tolerance.
2. **Exhausted budget** → service enters *protected mode* until the rolling window recovers.
3. **Exception process** with a named approver and duration cap.

**Enforcement must be mechanical**, not aspirational:

- **Trigger.** On-call SRE (or service owner) opens a tracked "protected mode" issue when
  the budget-exhaustion alert fires; product lead is auto-notified.
- **What it blocks.** CI/CD gate rejects non-reliability changes (label `reliability` or
  `rollback` required); merge rules require a second reviewer from the on-call rotation; new
  feature flags are frozen.
- **Exceptions.** Named approver (e.g. director-on-call) can grant a time-boxed exception
  (≤72h) recorded in the issue. Exceptions consume future budget — track them.
- **Recovery.** Protected mode lifts automatically when the rolling SLI is back above SLO
  for a configured cool-down (e.g. 24h) *and* the burn-event postmortem is published.

---

## 2. Alerting: multi-window, multi-burn-rate

Page humans on **symptoms** (the SLO is burning), not on causes (a disk is 80% full). For
mature, high-traffic, user-facing services, the multi-window/multi-burn-rate scheme from SRE
Workbook ch.5 is the default — it balances detection time and false-positive rate better
than fixed thresholds. It is **not** universal:

- For **low-traffic services** the short window is statistically noisy; prefer longer
  windows, synthetic probing, or time-based SLIs.
- For **batch / async pipelines**, alert on freshness or backlog SLIs with windows tuned to
  the batch period.
- For **internal/diagnostic SLOs**, a ticket-only policy is often correct.

### 2.1 Default thresholds (30-day window)

**Burn rate** = how fast the budget is being consumed relative to the SLO window. A burn
rate of 1 exhausts the budget exactly at the window boundary; a burn rate of 14.4 over 1
hour burns 2% of a 30-day budget (SRE Workbook ch.5, Table 5-3).

| Severity | Long window | Short window | Burn rate | Budget burned |
| -------- | ----------- | ------------ | --------- | ------------- |
| Page     | 1h          | 5m           | 14.4      | 2%            |
| Page     | 6h          | 30m          | 6         | 5%            |
| Ticket   | 3d          | 6h           | 1         | 10%           |

For a 28-day window, the *budget burned* column is identical (it's a fraction of the budget)
but absolute event/minute counts differ; recompute from `1 − SLO` over your chosen window.
Generate the table from config; do not hand-maintain.

The **short window** must also be firing — that's what kills the false-positive rate from a
long-window-only alert. Implement in Prometheus with two `for:` expressions ANDed; in cloud-
native monitors, use composite conditions.

### 2.2 Alert-fatigue controls

- **Deduplication and grouping** by service + SLO so one burn doesn't fan out across N
  causal alerts; **inhibition** rules so a SEV1 page suppresses dependent component pages.
- **Maintenance suppression** with explicit owner and end time; no open-ended silences.
- **Ownership review** quarterly: every alert has a service owner; orphan alerts are
  deleted, not adopted. Alerts with zero true-positive fires in 90d are proposed for
  deletion in a tracked review.

**Don't** alert on raw thresholds for user-visible services ("error_rate > 1%"). They have
no relationship to the contract and page forever during a slow brownout. Causal alerts
(disk, memory) belong as *tickets* unless they're on the path of a runbook.

---

## 3. Incident response

Adopt an explicit Incident Command System for **major incidents** (SEV1/SEV2), FEMA ICS-100
lineage adapted by Google (SRE Book ch.14 "Managing Incidents").

### 3.1 Roles, scaled to incident size

- **Small / early** (one responder, no confirmed customer impact): one engineer holds all
  roles; declare severity within 10 min.
- **Promoted to major** (customer impact confirmed, duration > 30 min, or multi-team): split
  into the three roles below. Promotion is itself a tracked event.
- **Incident Commander.** Owns the incident, drives callouts, declares severity, decides on
  rollback. In a major incident, IC does not make changes — separation reduces tunnel vision
  (SRE Book ch.14).
- **Communications Lead.** External + stakeholder updates on a fixed cadence (every 30 min
  for SEV1/2). Posts to a status channel separate from the working channel.
- **Operations / SME(s).** Investigates and changes things. Destructive actions (`kubectl
  delete`, schema migration, traffic shift) in a major incident are confirmed by IC before
  execution.

### 3.2 Mechanics and severity

- **Channel separation.** One *working* channel (`#inc-1234-payments`), one *broadcast*
  channel (`#incidents`). Mixing them collapses signal.
- **Preserve the timeline.** A bot or scribe captures actions, commands, graph links, and
  decisions with UTC timestamps. Slack history is a log; the timeline is curated afterward.

Define SEV1–SEV4 from **customer/business impact** primarily, with these inputs: scope (%
users, segments, geographies); nature of impact (data loss, security/privacy exposure,
regulatory breach, financial loss, full outage vs degraded); duration/trajectory; and
reliability signal (SLO burn rate, number of SLOs burning). Document one row per severity
with concrete examples, not just thresholds. Burn rate is a *trigger* for declaring; impact
decides the *level*.

---

## 4. Blameless postmortems

"Blameless" does **not** mean "no accountability". The document and the meeting do not
assign individual blame; people remain accountable for follow-up actions and for telling the
truth (Allspaw, *Etsy Debriefing Facilitation Guide*; Dekker, *Field Guide to Understanding
Human Error*, ch.1). Mandatory for every SEV1/SEV2 and any near-miss involving customer
data, auth, or budget exhaustion. Published within 5 business days.

**Template** (SRE Book ch.15 "Postmortem Culture", with Allspaw / Dekker "new-view"
framing):

1. **Summary** (≤3 sentences a VP can read).
2. **Impact.** Users affected, $ if known, SLO budget burned, regulatory exposure.
3. **Timeline** (UTC): first detection, first page, IC assigned, key decisions, mitigation,
   resolution. Reconstruct from the audit-log pipeline defined in ch06 §15
   (separate from observability logs) plus the standard observability
   stream — the audit log is the authoritative record for who-did-what-when.
4. **Trigger.** The proximate event that started the incident (a deploy, a config change, a
   traffic spike).
5. **Contributing factors.** *Plural.* Conditions that allowed the trigger to become an
   incident (missing guardrail, stale runbook, unowned alert). Avoid single-cause framing;
   complex systems fail for multiple reasons (Dekker, *Field Guide* ch.4).
6. **Detection.** How and when we noticed; gap between start and detection.
7. **Mitigation and recovery.** What stopped customer impact, what restored normal
   operation, and what helped (lucky breaks count and must be named).
8. **Systemic gaps.** Patterns this incident exposes across services or teams.
9. **Action items** with owner, ticket, due date. Tag each as **prevent** (stop recurrence)
   vs **mitigate** (reduce next blast radius). Mitigation work counts.
10. **Glossary / supporting links.**

**Reject in review:**

- "Operator error" or "human error" as a contributing factor — that's the *start* of the
  investigation, not the end (Dekker, *Field Guide* ch.3).
- Action items with no owner or no date.
- Any use of "root cause" singular that closes off further inquiry.

---

## 5. Runbooks (as code)

A runbook exists for **every paging alert**. No runbook → the alert is a ticket, not a page
(SRE Book ch.11; Workbook ch.8). A good runbook is a **decision tree, not a narrative**, and
is treated as code:

```
ALERT: checkout-latency-burn-fast
1. Verify: open dashboard <link>. Is p99 > 400ms for /checkout?
   - No  → ack as flap, file ticket to tune alert.
   - Yes → continue.
2. Check upstream: payments-gw error rate <link>.
   - >1% → page payments on-call, set IC, comms cadence 30m.
   - <1% → continue.
3. Check recent deploys (<link to deploy log>).
   - Deploy in last 60m → roll back via `make rollback SERVICE=checkout`. STOP.
   - Else → continue to capacity check below.
```

Rules:

- **Linked from the alert payload.** If the on-call has to grep a wiki at 3am, the runbook
  does not exist.
- **Executable snippets** (`make`, `kubectl`, CLI invocations) instead of prose. Each
  command has an **owner team** recorded next to it so escalation is unambiguous.
- **Automated preflight checks** wrap destructive commands: confirm cluster context, confirm
  change window, confirm operator has the required role.
- **CI validation.** A pipeline test asserts that every alert payload's runbook URL resolves
  (HTTP 200) and that referenced commands exist (`make -n`, `helm template`, etc.).
- **Tested.** Runbook steps are exercised in gameday at least quarterly. An untested runbook
  is a hypothesis.
- **Version-controlled** in the same repo as the service. Docs that drift from code are
  worse than no docs.

---

## 6. On-call hygiene

Treat the numbers below as *sustainability thresholds* — the line at which to investigate,
not laws of nature (SRE Workbook ch.11; Fong-Jones, "Sustainable On-Call", 2019).

- **Page load**: aim for ≤2 actionable pages per shift on average; sustained breach is a
  tuning, staffing, or architecture signal.
- **Out-of-hours pages** for non-tier-1 services should trend to zero; if a service pages at
  night, it should have a tier-1 SLO and a paying customer journey.
- **Follow-up SLA.** Every page generates a ticket; tickets >30 days without action escalate
  to the service owner's manager.
- **Handoff** at the start of every shift: open incidents, suppressed alerts, in-flight
  changes, planned maintenance. 15 min, written.

### 6.1 Rotation models

Pick the *minimum viable* model and grow into the ideal:

| Model                        | Headcount | Coverage          | Notes                                 |
| ---------------------------- | --------- | ----------------- | ------------------------------------- |
| Primary + manager backstop   | 2–3       | Business hours    | Acceptable for early-stage / low SEV  |
| Primary + secondary          | 4–5       | Extended hours    | Secondary covers escalation + relief  |
| Single-region 24x7           | 6+        | 24x7 in one TZ    | Watch for night-shift attrition       |
| Follow-the-sun               | 6+ across ≥2 TZs | True 24x7  | Ideal for global tier-1 services      |

Anything thinner than "primary + backstop" is a single point of failure, not a rotation.

### 6.2 Compensation and page-budget review

On-call is paid (cash, time off, or both). Volunteer on-call is unpaid labor; document the
comp model alongside the rotation. Monthly, count pages per service and per shift; any
service exceeding its page budget loses the right to deploy until alerts are tuned — the
on-call equivalent of an error-budget freeze.

---

## 7. Toil

**Toil** = manual, repetitive, automatable, tactical, devoid of enduring value, and scaling
linearly with growth (SRE Book ch.5). Restarting a pod by hand is toil; writing a controller
that restarts it is engineering.

- **Cap toil at 50%** of any SRE / platform engineer's time, measured. Sustained breach is a
  staffing or roadmap signal — escalate, don't grind.
- **Track it.** Quarterly survey or time-tracked tags on tickets; publish the chart, which
  is the argument for headcount.
- **Top-3 toil items per quarter** become explicit roadmap projects with the same rigor as
  feature work (design doc, owner, exit criteria). Intentional manual gates (compliance
  approvals) are not toil — label `governance` and exclude.

---

## 8. Chaos engineering (maturity-gated)

Chaos engineering injects failure to surface latent risk *before* it surfaces you (Rosenthal
& Jones, *Chaos Engineering*, O'Reilly, ch.2). It is high-leverage *only* once the
prerequisites below are in place; otherwise it manufactures incidents.

### 8.1 Maturity ladder — walk in order, do not skip

1. **Tabletop exercises.** Whiteboard a failure; talk through detection, comms, mitigation.
   Cheap, finds runbook gaps.
2. **Blast-radius drills in pre-prod.** Inject faults (kill a pod, drop a network link, slow
   a dependency) in a staging cell. Validates tooling and on-call muscle memory.
3. **Dependency-failure rehearsals.** Stub or fault-inject specific downstreams; verify
   timeouts, retries, and graceful degradation trigger as designed.
4. **Production chaos in a canary cell** (1% of traffic), with documented hypothesis and
   automated abort.
5. **Production chaos at broader scope**, only after months of clean canary runs and
   postmortem-driven improvements.

### 8.2 Principles and gates

- **Hypothesis first.** "We believe checkout tolerates loss of one of three payments
  replicas with no SLO impact." No hypothesis → it's an outage.
- **Blast radius minimized initially**, then expanded. Every experiment has a documented
  kill switch and an owner watching live SLOs.
- **Gameday discipline.** Quarterly tabletop + live exercise per tier-1 service; output is
  updated runbooks and filed bugs.
- **Don't run chaos when** the service has no SLO, the on-call is over page budget, or
  recovery depends on a single SME and is not automated.

---

## 9. Resilience patterns

Defaults for any inter-service call. The order below is the order signals flow on a request
path; each layer assumes the ones above it (Brooker, "Timeouts, Retries, and Backoff with
Jitter" and "Avoiding Overload", AWS Builders' Library; Nygard, *Release It!* 2e ch.5).

1. **Deadline propagation.** Carry a single end-to-end deadline (gRPC deadline, HTTP
   `X-Request-Deadline`, context cancellation) so downstream work stops as soon as the
   caller has given up. Per-hop timeouts without propagation produce wasted work.
2. **Per-hop timeouts**, strictly shorter than the inherited deadline. The default in most
   client libraries is "infinity"; that is a bug.
3. **Bounded concurrency / backpressure.** Cap in-flight requests per client and per
   dependency (semaphores, bounded queues). Bounded queues surface overload as latency *and*
   shed load deterministically; unbounded queues hide it until OOM.
4. **Safe retries with idempotency.** Retry only transient, safe failures (timeouts, 5xx
   without side effects, explicit retryable errors). Use exponential backoff with full
   jitter (`sleep = random(0, min(cap, base * 2^attempt))`) and a **retry budget** capping
   retries to ~10% of base traffic per client (Brooker, AWS Builders' Library). Mutations
   require **idempotency keys** end-to-end; without them, retries are duplicate-charges-as-
   a-service.
5. **Circuit breakers / admission control.** Trip on sustained downstream failure or queue
   saturation; half-open probes recover. Apply to dependencies with known failure modes, not
   every call (it's state to maintain). Admission control on the server side rejects new
   work when the in-flight pool is saturated.
6. **Graceful degradation / brownout.** Identify per-feature degradation modes (return
   cached data, hide non-critical UI, skip personalization). Trigger from SLO burn or
   admission-control pressure, not from manual toggles only.
7. **Load shedding** at the edge before queues build. Reject early with `503 Retry-After`;
   prioritize by request class so paying / critical journeys win.
8. **Bulkheads.** Isolate resources (thread pools, connection pools, queues) per dependency
   so one slow downstream cannot consume all of them.

### 9.1 Automated canary analysis

Every deploy that changes user-facing behavior runs through automated canary analysis: a
small % of traffic to the new version, with a controller comparing SLI metrics (error rate,
latency percentiles, saturation) against the baseline and aborting on regression. Manual
canary "look at the graph" is not canary analysis (SRE Workbook ch.16 "Canarying Releases").

---

## 10. Capacity planning

Capacity planning is *queueing theory plus headroom*, not a spreadsheet of last quarter's
CPU (SRE Book ch.18; Workbook ch.11 "Managing Load").

- **Leading indicators**: traffic forecast, product roadmap, known seasonality. Drive
  provisioning.
- **Lagging indicators**: utilization, queue depth, p99 latency creep. Drive "you were
  wrong" alarms.
- **Headroom and the latency knee.** Latency under load grows non-linearly as utilization
  approaches saturation, but the *location* of the knee depends on service-time variability,
  concurrency model (single-server M/M/1 is rarely a fit for real services), queue design,
  and request mix (Gunther, *Guerrilla Capacity Planning*; Brooker, "Avoiding Overload").
  Anchor headroom targets in **measured load tests**, not a folklore "70%" number.
- **Tiered headroom defaults** (justify deviations in the design doc):

  | Tier             | Failure-domain headroom         | Steady-state utilization budget | Why                                |
  | ---------------- | ------------------------------- | ------------------------------- | ---------------------------------- |
  | Tier-1 (revenue / safety-critical) | N+2 zones/cells | Measured knee, typically ≤60%  | Survive simultaneous failure + spike |
  | Tier-2 (important, recoverable)    | N+1             | Measured knee, typically ≤75%  | Survive single-domain loss          |
  | Tier-3 (best-effort, internal)     | N (autoscale)   | ≤85% with autoscale headroom   | Cost over resilience                |

- **Forecast horizon ≥ lead time** for the slowest resource (often GPUs, DB shards, network
  capacity, IP space). If procurement is 12 weeks, forecast 16.
- **Load test to the cliff** quarterly. The point is not to confirm your model; it's to find
  the cliff before customers do.

---

## 11. Org model — a maturity ladder

Reliability org structure should evolve with coordination cost, not be imposed up front (SRE
Book ch.1, ch.32; Skelton & Pais, *Team Topologies* ch.3–4).

1. **Service ownership + on-call accountability.** Every service has a named owning team;
   sufficient for small orgs, no SRE function needed.
2. **Reliability guild / virtual team.** Engineers from product teams share SLO templates,
   runbook patterns, postmortem reviews. Cheap coordination layer.
3. **Enabling team** (temporary). Reliability specialists embed for a quarter to bootstrap
   SLOs, alerting, gamedays, then leave.
4. **Platform team** owns shared infra (CI/CD, observability, deploy, incident tooling) as a
   product with internal SLOs, once shared-infra load justifies dedicated headcount.
5. **Embedded SRE / dedicated SRE team** for tier-1 services. Highest cost and leverage;
   introduce only when error-budget and toil data demonstrate the need.

**Mikey Dickerson's reliability hierarchy** (SRE Book ch.1) sets the order of *investment*:
monitoring → incident response → postmortems → testing/release → capacity → development →
product. Skipping levels — e.g. chaos before postmortems — wastes the work above. **Safety
culture is built, not declared** (Dekker, *The Safety Anarchist*; *Field Guide* ch.6): it
works when people report near-misses unprompted, postmortems name systems, and on-call
complaints get roadmap slots.

---

## 12. Disaster recovery

DR is a *practiced* capability; an untested DR plan is a fiction (SRE Book ch.33; AWS
Builders' Library, "Static Stability Using Availability Zones").

- **RTO / RPO per service**, set by the business. Tier-1 starting default: RTO ≤ 1h, RPO
  ≤ 5m. Document the *actual achievable* values; if they don't match, escalate.
- **DR drill cadence**: full region-loss drill ≥ annually for tier-1; partial (single AZ,
  single dependency) quarterly. Drill failures are SEV3 postmortems. The
  timed end-to-end DR drill itself is owned by ch08 §3 (quarterly floor,
  monthly for tier-0); this section owns the *automated restore
  verification* loop below.
- **Restore, don't just back up.** A backup never restored is Schrödinger's backup;
  automated restore test weekly (scripted PITR into a throwaway instance with
  checksum compare), alert on age of last successful restore. This weekly
  loop is distinct from the quarterly DR drill in ch08 §3 — keep both.
- **Runbooks + access in a separate trust domain.** A DR runbook only in the wiki of the
  cloud you just lost does not exist — mirror to a second provider or printed binder.
- **Write down what you will *not* recover** (analytics replicas, dev envs, derived caches);
  scope discipline shortens RTO more than any tool.
- **Multi-region: decide topology here, not in passing.** The decision to go
  active/active vs active/passive vs single-region-with-cross-region-restore
  is owned by this section. Synthesise the inputs from the other chapters:
  data replication, RPO/RTO, and tier classification from ch08 §4–§5;
  network primitives (anycast, hub-and-spoke, PrivateLink) from ch07; HA vs
  DR distinction from §1 of this chapter. Default for tier-1: multi-AZ HA
  plus a tested cross-region restore (per ch08 §3 DR drill). Active/active is
  an exception with a written justification (regulator, business loss per
  minute, measured exposure to provider regional incidents) — the cost in
  write latency, conflict handling, and split-brain bug classes is real.

---

## 13. Metrics that don't matter

- **Status-page uptime %** — self-reported and disconnected from any SLO.
- **MTBF** — useful for hardware, meaningless for software that changes daily (Allspaw,
  *Adaptive Capacity Labs*; Dekker, *Field Guide* ch.5).
- **Number of incidents** — encourages hiding. Track *minutes of budget burned* and
  *postmortem action-item completion rate* instead.
- **Per-engineer ticket counts** — Goodhart's Law at speed.
- **Any reliability metric not tied to a written SLO.** If no one signed up to the target,
  the number is decoration.

DORA's delivery keys (lead time, deploy frequency, change-failure rate,
failed-deployment recovery time; Forsgren et al., *Accelerate* ch.2) are the
*delivery* counterparts and correlate with reliability outcomes — measure them
too, but they are not a substitute for SLOs. Canonical metric set and count
live in ch11 §14.

---

## Checklist — what reviewers should reject

- New user-facing service ships without ≥1 user-journey SLO and an error-budget policy with
  mechanical enforcement.
- Paging alert without a linked, CI-validated runbook.
- Runbook that is a wall of prose with no decision tree or executable commands.
- Postmortem with "operator error" / single root cause, or no owner on action items.
- Retry logic without jitter, without a budget, or on a non-idempotent call without an
  idempotency key.
- Network call with no timeout or no propagated deadline.
- DR plan that has never been executed end-to-end.
- On-call rotation without a documented compensation and escalation model.
- Production chaos experiment without a documented hypothesis, abort, and prior pre-prod
  blast-radius drill.
