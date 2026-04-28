# Chapter 09 — Reliability (SRE)

Opinionated, cloud-agnostic operational discipline. The Google SRE Book and SRE
Workbook are the canon; this chapter distills them into defaults you can land in
a PR. Reliability is a product feature with a budget — not a vibe, not a status
page.

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint.

---

## 1. SLI, SLO, and the error budget

**100% is the wrong target.** Past a certain point, each extra nine costs more
than the user perceives, and chasing it freezes the product. Pick a target that
matches user expectations and the dependency chain beneath you (Beyer et al.,
*SRE Book* ch.4 "Service Level Objectives"; Hidalgo, *Implementing Service Level
Objectives* ch.3).

- **SLI** (indicator): a *ratio of good events to valid events*, measured at the
  point closest to the user (e.g. load balancer, CDN edge, mobile RUM). Prefer
  ratios over averages — averages hide the bad tail (SRE Workbook ch.2).
- **SLO** (objective): a target on that ratio over a *rolling window* (28 days
  is the default; quarterly is fine for batch). The SLO is the contract.
- **Error budget**: `1 − SLO`. If the SLO is 99.9% over 28d, the budget is
  ~40m 19s of "bad" events. Spend it on releases, experiments, and chaos —
  not heroics.

**Choose SLIs from user journeys, not from CPU graphs.** For each critical
journey ("user logs in", "checkout completes", "search returns results"), pick
at most 2–3 SLIs from this menu (SRE Workbook ch.2; Hidalgo ch.4):

| Journey shape          | SLI category   | Example                                        |
| ---------------------- | -------------- | ---------------------------------------------- |
| Synchronous request    | Availability   | `2xx+3xx / valid_requests`                     |
| Synchronous request    | Latency        | `p99 < 400ms` measured as `fast / valid`       |
| Async pipeline         | Freshness      | `messages processed within 5min / total`       |
| Storage / read         | Correctness    | `bytes_returned == bytes_expected`             |
| Stream / batch         | Throughput     | `events_per_sec ≥ floor`                       |

**Don't** define an SLO on a metric the user cannot feel (CPU%, queue depth, GC
pauses). Those are *causes*; SLOs measure *symptoms* (Beyer et al., SRE Book
ch.6 "Monitoring Distributed Systems").

**Error budget policy is a written contract** signed by product + eng + SRE
before you ever exceed it. Minimum clauses (SRE Workbook ch.4):

1. What happens when budget is **healthy**: ship faster, run gamedays, raise
   risk tolerance.
2. What happens when budget is **exhausted**: feature freeze, only reliability
   work and rollbacks merge, until the rolling window recovers.
3. Who can grant an exception, and the duration cap.

---

## 2. Alerting: multi-window, multi-burn-rate

Page humans on **symptoms** (the SLO is burning), not on causes (a disk is
80% full). The only alerting policy that does not destroy on-call is the
multi-window, multi-burn-rate scheme from SRE Workbook ch.5.

**Burn rate** = how fast the budget is being consumed relative to the SLO
window. A burn rate of 1 exhausts the budget exactly at the window boundary; a
burn rate of 14.4 over 1 hour burns 2% of a 30-day budget.

Default policy (SRE Workbook ch.5, Table 5-3) for a 30-day window:

| Severity | Long window | Short window | Burn rate | Budget burned |
| -------- | ----------- | ------------ | --------- | ------------- |
| Page     | 1h          | 5m           | 14.4      | 2%            |
| Page     | 6h          | 30m          | 6         | 5%            |
| Ticket   | 3d          | 6h           | 1         | 10%           |

The **short window** must also be firing — that's what kills the false-positive
rate from a long-window-only alert. Implement in Prometheus with two `for:`
expressions ANDed; in cloud-native monitors, use composite conditions.

**Don't** alert on raw thresholds for user-visible services
("error_rate > 1%"). They have no relationship to the contract and they page
forever during a slow brownout. Causal alerts (disk, memory) belong as
*tickets* unless they're on the path of a runbook.

---

## 3. Incident response

Adopt an explicit Incident Command System (PagerDuty, *Incident Response
Documentation*, response.pagerduty.com; FEMA ICS-100 lineage). Three roles,
even for a one-person incident:

- **Incident Commander (IC).** Owns the incident. Does not type at a keyboard.
  Drives structured callouts, declares severity, decides on rollback.
- **Communications Lead.** Owns external + stakeholder updates on a fixed
  cadence (every 30 min for SEV1/2). Posts to a status channel, never the same
  channel as the working channel.
- **Operations / Subject Matter Expert(s).** Investigates and changes things.
  Only ops-lead can `kubectl delete` etc.

**Channel separation.** One *working* channel (`#inc-1234-payments`) for the
responders, one *broadcast* channel (`#incidents`) for everyone else. Mixing
them collapses signal and turns the incident into a stand-up
(charity.wtf, "On-Call Shouldn't Suck"; PagerDuty IR docs §"Roles").

**Preserve the timeline.** A bot or a dedicated scribe captures every
significant action, command, graph link, and decision with timestamps in UTC.
Slack history is *not* a timeline — it's a log; the timeline is curated
afterwards. Without this, the postmortem is fiction.

**Severity ladder.** Define SEV1–SEV4 with concrete entry criteria tied to
SLOs ("burn rate > 14.4 for >5m on a tier-1 SLO" = SEV2). Don't let severity
be vibes.

---

## 4. Blameless postmortems

"Blameless" does **not** mean "no accountability". It means the document and
the meeting do not assign individual blame; people remain accountable for
follow-up actions and for telling the truth. The goal is to maximize learning,
which only happens if responders can speak freely (Allspaw, *Etsy Debriefing
Facilitation Guide*; Dekker, *Field Guide to Understanding Human Error*, ch.1).

**Mandatory for every SEV1/SEV2** and any near-miss involving customer data,
auth, or the budget being exhausted. Published within 5 business days.

**Template (SRE Book ch.15 "Postmortem Culture", with Allspaw additions):**

1. **Summary** (≤3 sentences a VP can read).
2. **Impact** (users affected, $ if known, SLO budget burned).
3. **Timeline** (UTC, every state change; first detection, first page, IC
   assigned, mitigation, resolution).
4. **Root causes** — *plural*. Single-cause postmortems are almost always
   wrong (Allspaw, "Three Mile Island and the SCAM principle"; Dekker, *The
   Field Guide* ch.4 — the "new view" of human error).
5. **What went well / what went poorly / where we got lucky.** The "lucky"
   row is the one that finds latent risk.
6. **Action items** with owner, ticket, and due date. Tag each as **prevent**
   (stop recurrence) vs **mitigate** (reduce next blast radius). You will
   never prevent everything; mitigation work counts.
7. **Glossary / supporting links.**

**Two failure modes to reject in review:**

- "Operator error" as a root cause. That's the *start* of the investigation,
  not the end (Dekker, *Field Guide* ch.3).
- Action items with no owner or no date. They are decoration.

---

## 5. Runbooks

A runbook exists for **every paging alert**. No runbook → the alert should be
a ticket, not a page (SRE Book ch.11; Workbook ch.8).

A good runbook is a **decision tree, not a narrative**:

```
ALERT: checkout-latency-burn-fast
1. Verify: open dashboard <link>. Is p99 > 400ms for /checkout?
   - No  → ack as flap, file ticket to tune alert.
   - Yes → continue.
2. Check upstream: payments-gw error rate <link>.
   - >1% → page payments on-call, set IC, comms cadence 30m.
   - <1% → continue.
3. Check recent deploys (<link to deploy log>).
   - Deploy in last 60m → roll back via <runbook-link>. STOP.
   - Else → continue to capacity check below.
```

Rules:

- **Linked from the alert payload.** If the on-call has to grep a wiki at
  3am, the runbook does not exist.
- **Tested.** Runbook steps are exercised in gameday at least quarterly. An
  untested runbook is a hypothesis.
- **Version-controlled** in the same repo as the service. Docs that drift
  from code are worse than no docs.

---

## 6. On-call hygiene

On-call sustainability is a hard SLO of its own. Targets (Fong-Jones,
"Sustainable On-Call"; charity.wtf, "On-Call Shouldn't Suck"):

- **≤ 2 actionable pages per on-call shift, on average.** More than that and
  the rotation is broken — fix the alerts, not the humans.
- **0 pages outside business hours** for non-tier-1 services. If a service
  pages at night, it must have a tier-1 SLO and a paying customer journey.
- **Rotation size ≥ 6 people** for 24x7; otherwise adopt follow-the-sun
  across timezones. Anything smaller is a burnout pipeline.
- **Compensation.** On-call is paid (cash, time off, or both). Volunteer
  on-call is exploitation dressed up as culture.
- **Handoff** at the start of every shift: open incidents, suppressed alerts,
  in-flight changes, planned maintenance. 15 min, written.
- **Follow-up SLA.** Every page generates a ticket; tickets older than 30
  days without action are escalated to the service owner's manager.

**Page budget review** monthly: count pages per service, per shift; any
service exceeding the budget loses the right to deploy until alerts are tuned.
This is the on-call equivalent of an error-budget freeze.

---

## 7. Toil

**Toil** = manual, repetitive, automatable, tactical, devoid of enduring
value, and scaling linearly with service growth (Beyer et al., *SRE Book*
ch.5 "Eliminating Toil"). Restarting a pod by hand is toil. Writing a
controller that restarts it is engineering.

- **Cap toil at 50%** of any SRE / platform engineer's time, measured.
  Sustained breach is a staffing or roadmap signal — escalate, don't grind.
- **Track it.** Quarterly survey or time-tracked tags on tickets. Categories:
  on-call, manual ops, meetings, eng. Publish the chart; the chart is the
  argument for headcount.
- **Top-3 toil items per quarter** become explicit roadmap projects with the
  same rigor as feature work (design doc, owner, exit criteria).

Toil that is *intentional* (rare manual gate, compliance approval) is not
toil — label it `governance` and exclude.

---

## 8. Chaos engineering

Chaos engineering is the disciplined practice of injecting failure to surface
latent risk *before* it surfaces you (Rosenthal & Jones, *Chaos Engineering*,
O'Reilly, ch.2 "Principles of Chaos"; Netflix Chaos Monkey lineage).

**Principles to enforce in review:**

- **Hypothesis first.** "We believe the checkout path tolerates the loss of
  one of three payment-service replicas with no SLO impact." No hypothesis
  → it's an outage, not an experiment.
- **Blast radius minimized initially**, then expanded. Start in a canary
  cell or 1% of traffic.
- **Run in production.** Staging chaos is theatre — the dependencies, the
  load, and the configs are different (Rosenthal, *Chaos Engineering* ch.3;
  Bryant Butow, "Chaos Engineering at Dropbox / Gremlin").
- **Abort button.** Every experiment has a documented kill switch and an
  owner watching.
- **Gameday discipline.** Quarterly tabletop + live exercise per tier-1
  service. The output is updated runbooks and filed bugs, not vibes.

**When NOT to do chaos:**

- The service has no SLO. You don't know what "broken" means.
- The on-call team is already over page budget. You're adding to a fire.
- Recovery is not automated and depends on a single SME. Fix that first.

ThoughtWorks Tech Radar has held chaos engineering in **Adopt** since 2020
for orgs with mature observability; for orgs without, it stays in *Assess*.

---

## 9. Resilience patterns

Defaults for any inter-service call (Brooker, "Timeouts, Retries, and
Backoff with Jitter", AWS Builders' Library; Nygard, *Release It!* 2e):

- **Timeouts on every network call.** No exceptions. The default in most
  client libraries is "infinity"; that is a bug. Set timeouts shorter than
  the caller's timeout (timeout budget shrinks down the stack).
- **Retries with exponential backoff + full jitter.** Synchronized retries
  ("retry storm") are the second outage. `sleep = random(0, min(cap, base *
  2^attempt))` (Brooker, AWS Builders' Library).
- **Retry budget.** Cap retries to ~10% of base traffic per client. Above
  that, fail fast. Unbounded retries turn a brownout into an outage.
- **Circuit breakers** on calls to dependencies with known failure modes;
  half-open probes to recover. Don't put a breaker on every call — it's
  state to maintain.
- **Bulkheads.** Isolate resources (thread pools, connection pools, queues)
  per dependency so one slow downstream cannot consume all of them.
- **Idempotency keys** on any retried mutation. Without them, retries are
  duplicate-charges-as-a-service.
- **Load shedding** at the edge before queues build. Reject early with 503
  - Retry-After; do not let work pile up unbounded.

---

## 10. Capacity planning

Capacity planning is *queueing theory plus headroom*, not a spreadsheet of
last quarter's CPU (SRE Book ch.18 "Software Engineering in SRE"; Workbook
ch.11).

- **Leading indicators**: traffic forecast, product roadmap (new feature
  launches), known seasonality. Drives provisioning.
- **Lagging indicators**: utilization, queue depth, p99 latency creep.
  Drives "you were wrong" alarms.
- **Headroom targets**: a service should sustain N+2 (lose two zones / cells
  / replicas) at peak with utilization ≤ 60%. Above ~70%, latency goes
  non-linear (Little's Law / M/M/1).
- **Forecast horizon ≥ lead time** to provision the slowest resource (often
  GPUs, DB shards, network capacity, IP space). If procurement is 12 weeks,
  forecast 16.
- **Load test to the cliff** quarterly. The point is not to confirm your
  model; the point is to find the cliff before customers do.

---

## 11. Org model

Two patterns work; everything else collapses (SRE Book ch.1, ch.32; Forsgren
et al., *Accelerate* ch.3 "Measuring and Changing Culture"):

| Model                      | Fits                                  | Watch out for                       |
| -------------------------- | ------------------------------------- | ----------------------------------- |
| **You build it, you run it** ("dual-pizza", AWS) | Small/medium orgs; product-aligned teams | On-call burnout if no platform team |
| **Embedded SRE / platform team**                 | Large orgs with shared infra         | Becoming an ops dumping ground       |

**Mikey Dickerson's reliability hierarchy** (Google site-reliability pyramid)
sets the order of investment: monitoring → incident response → postmortems →
testing/release → capacity → development → product. Skipping levels — e.g.
investing in chaos before postmortems — wastes the work above.

**Safety culture is built, not declared** (Dekker, *The Safety Anarchist*;
*Field Guide* ch.6). Indicators that it's working: people report near-misses
unprompted; postmortems name systems, not people; on-call complaints get
roadmap slots. Indicators it isn't: "human error" appears as a root cause;
the same alert pages weekly; new hires shadow on-call without ever being
primary because "the system is too complex".

---

## 12. Disaster recovery

DR is a *practiced* capability. An untested DR plan is a fiction (SRE Book
ch.33 "Lessons Learned"; AWS Builders' Library, "Static Stability").

- **RTO / RPO per service**, set by the business, not by infra. Tier-1: RTO
  ≤ 1h, RPO ≤ 5m. Document the actual achievable values; if they don't
  match, escalate.
- **DR drill cadence**: full region-loss drill ≥ annually for tier-1; partial
  (single AZ, single dependency) quarterly. Drill failures are SEV3
  postmortems, not "we'll fix it next time".
- **Restore, don't just back up.** A backup that has never been restored is
  Schrödinger's backup. Automated restore test weekly; alert on age of last
  successful restore.
- **Runbooks + access in a separate trust domain.** If your DR runbook lives
  only in the wiki of the cloud you just lost, it does not exist. Mirror to
  a second provider or printed binder for the critical 10 pages.
- **Write down what you will *not* recover** — analytics replicas, dev
  envs, derived caches. Scope discipline shortens RTO more than any tool.

---

## 13. Metrics that don't matter

- **Status-page uptime %.** Self-reported, coarse-grained, and almost
  always disconnected from any SLO. Investors look at it; on-calls don't.
- **MTBF (Mean Time Between Failures).** Useful for hardware in steady
  state; meaningless for software systems that change daily (Allspaw,
  *Adaptive Capacity Labs*; Dekker, *Field Guide* ch.5).
- **Number of incidents.** Encourages hiding incidents. Track *minutes of
  budget burned* and *postmortem action-item completion rate* instead.
- **Per-engineer ticket counts.** Goodhart's Law at speed.
- **Any reliability metric not tied to a written SLO.** If no one signed up
  to the target, the number is decoration.

DORA's four keys (lead time, deploy frequency, change-failure rate, MTTR;
Forsgren et al., *Accelerate* ch.2) are the *delivery* counterparts and
correlate with reliability outcomes — measure them too, but they are not a
substitute for SLOs.

---

## Checklist — what reviewers should reject

- New user-facing service ships without ≥1 SLO and an error-budget policy.
- Paging alert without a linked runbook.
- Runbook that is a wall of prose with no decision tree.
- Postmortem with "operator error" as the root cause, or no owner on action
  items.
- Retry logic without jitter, without a budget, or on a non-idempotent call
  without an idempotency key.
- Network call with no timeout.
- DR plan that has never been executed end-to-end.
- On-call rotation with < 6 people and no follow-the-sun.
- Chaos experiment in production without a documented hypothesis and abort.
