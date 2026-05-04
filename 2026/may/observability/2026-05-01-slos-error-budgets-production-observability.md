# SLOs, Error Budgets & Production Observability -- The Hospital ER Triage Contract

> The last nine days built the entire observability stack. Apr 20-23 stood up the metrics pillar (Prometheus TSDB, PromQL, exemplars, Alertmanager). Apr 24 hung the dashboards on the wall (Grafana). Apr 26 wired the K8s metrics stack with kube-prometheus-stack and PrometheusRule. Apr 27-28 brought up traces (OTel SDK, Collector pipelines, W3C Trace Context, Tempo, tail sampling). Apr 29 added the third pillar (Loki, structured metadata, LogQL). Apr 30 stitched all three together with exemplars + tracesToLogsV2 + derivedFields + the LogQL ruler. **Today caps the arc** -- and the question shifts from "what signals do I emit" to "what reliability promises do I make, and how do I run an organization around them." Every rate(), every histogram bucket, every burn-rate alert from the prior nine docs is *fuel*. SLOs are the *engine* that turns that fuel into organizational decisions.
>
> **The core analogy for the day: an SLO is the triage commitment of a hospital ER, and the error budget is the operating-room margin you can spend on training, equipment failures, and bad days.** A real ER does not promise "we will see every patient instantly forever." It triages: "We commit that 99% of category-3 (urgent but stable) patients will be seen by an attending within 30 minutes -- measured over a rolling 30-day window." That single sentence has all four parts of an SLO: a **target population** (category-3 patients, not coughs and not gunshot wounds), an **SLI** (time from arrival to attending greeting), a **threshold** (30 min), and a **target** (99%). The 1% who are not seen in 30 minutes is the **error budget** -- it is *deliberately allocated* and exists to cover staff training, equipment failures, mass-casualty surges, and the sheer reality that ERs are not magic. **The budget is not "how many failures the hospital will tolerate before it fires people." It is "how many failures the hospital can afford before it must change behavior."** When the budget is healthy, the ER is allowed to run resident-supervised training, swap out fluoroscopy machines, and roll out new triage software. When the budget is half-spent, training pauses, change windows tighten, and the chief of medicine starts attending shift huddles. When the budget is exhausted, the ER goes to **diversion status** -- non-critical patients are routed to other hospitals while the team recovers, conducts blameless postmortems, and rebuilds the budget. **This is the entire SLO program in one paragraph.** Everything below is mechanism: how to pick the SLI (the right thing to measure), how to set the target (the right number), how to compute the budget (the right math), how to alert on it (the right speed), and how to wire the consequence ladder so that "budget exhausted" actually causes diversion in your engineering org instead of just a sad gauge on a dashboard.
>
> Three corollaries the analogy carries that show up everywhere below. **(1) The promise must match what users actually feel.** A hospital that measures "average door-to-doctor time" hides the patient who waited four hours behind the patient seen in two minutes; users feel the four-hour patient. This is why SLIs are *percentile-against-threshold* (e.g., "99% of requests faster than 250ms"), not averages. The Apr 21 PromQL doc taught you `histogram_quantile(0.99, ...)`; today's doc tells you *which* threshold to put in front of it and *why*. **(2) Each "nine" is an order of magnitude more expensive than the one before it.** Going from 99% to 99.9% means cutting the budget from 7.2 hours/month to 43.2 minutes/month, which means single-AZ deployments are no longer acceptable, deploy-and-pray is no longer acceptable, and your test pyramid has to actually exist. Going to 99.99% means active-active multi-region, deploy automation that catches regressions in seconds, and a runbook for every alert. The cost curve is exponential, and the "Just Barely Good Enough" principle from the SRE Workbook says: **never set an SLO higher than what users can actually distinguish from perfection.** **(3) The budget creates a self-correcting loop.** Spend it fast (bad release, infra incident) and the org reflexively slows down -- feature freeze, more review, on-call escalation -- until the budget rebuilds. Spend it slow (no incidents) and the org reflexively speeds up -- ship faster, take more risk, lower review bar -- because reliability is *too high* relative to the SLO. The budget is the **organizational thermostat** for the velocity-vs-reliability tradeoff. SLOs without a written policy are a thermostat that is not wired to the furnace.

---

**Date**: 2026-05-01
**Topic Area**: observability
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| SLI (Service Level Indicator) | The metric the ER charts on the wall | The actual measurement -- e.g., `good_events / valid_events` over time |
| SLO (Service Level Objective) | The triage commitment ("99% in 30 min") | A target value of an SLI over a rolling window; the contract |
| SLA (Service Level Agreement) | The hospital's contract with the city | The legally enforceable, customer-facing version with money/credits attached; usually looser than the SLO |
| Error budget | The 1% margin you can spend on training and bad days | `1 - SLO`; the engineering currency for change |
| Error budget burn | Spending the margin faster than expected | The instantaneous rate at which budget is consumed; the alert-worthy signal |
| Burn rate | "How many days until the ER hits diversion if this continues?" | Multiple of normal consumption; 14.4x = 36h to exhaust 100% budget |
| Multi-window multi-burn-rate (MWMBR) | Different alarms for different urgencies | 4-tier alert hierarchy: page-on-fast-burn, ticket-on-slow-burn |
| Error budget policy | The hospital bylaws that say what triggers diversion | Written, signed artifact mapping budget thresholds to organizational consequences |
| Request-driven SLI | "% of patients seen within 30 min" | Latency/availability of synchronous request-response services -- the RED method (Apr 21) |
| Pipeline SLI | "% of lab results delivered within 1h of being drawn" | Data-freshness or completeness for batch/stream pipelines |
| Storage SLI | "Will this medical record still be readable in 7 years?" | Durability, availability, and correctness of data at rest |
| User-journey composite SLI | "% of patients who completed admit-triage-treatment-discharge in 4h" | Multi-step composite; component SLOs multiply, not add |
| Good events / Valid events | Patients seen in time / patients who showed up | The canonical SLI formula; "valid" excludes synthetic and not-yet-eligible traffic |
| 14.4x / 6x / 3x / 1x burn | Four flavors of "we're spending too fast" | The canonical SRE Workbook table; each row is a (burn_rate, window, severity) tuple |
| Sloth | A YAML-to-PrometheusRule generator | CLI tool: input one SLO spec, output ~10 rules (recording + multi-burn-rate alerts) |
| Pyrra | A Kubernetes-native SLO operator with a UI | CRD `ServiceLevelObjective`, controller reconciles to PrometheusRules + serves a burndown UI |
| OpenSLO | The vendor-neutral SLO YAML schema | Like OpenTelemetry but for SLOs; `Service`/`SLI`/`SLO`/`AlertPolicy` kinds |
| Grafana SLO plugin | Grafana's first-party SLO management | Enterprise/Cloud-only; manages SLOs from the Grafana UI with auto-generated rules |
| Nobl9 | The commercial SaaS SLO platform | Multi-data-source, supports OpenSLO; for orgs that don't want to self-host |
| Burndown chart | The fuel gauge on the dashboard | Time-series of remaining budget over the SLO window; the canonical SLO viz |
| Runbook-as-code | The triage flowchart printed and laminated | Markdown docs in Git, linked from `runbook_url` annotation on every alert |
| Observability maturity | "Does the ER actually know when it's failing?" | Five-capability framework (Charity Majors / Liz Fong-Jones); SLOs are a level-3+ practice |

---

## The Whole Picture in One Diagram

Before diving in, here is the entire SLO program on one page. Every section below ties to a node in this graph.

```
THE SLO PROGRAM, END TO END
================================================================================

  +---------------------------+
  |  USER JOURNEY MAP         |     "What do users actually feel?"
  | - critical paths          |
  | - acceptable thresholds   |
  +-------------+-------------+
                |
                v
  +---------------------------+
  |  SLI SELECTION            |     The hardest step.
  | - request-driven (RED)    |     Output: a PromQL ratio
  | - pipeline / freshness    |     good_events / valid_events
  | - storage / durability    |
  | - composite (AND/OR)      |
  +-------------+-------------+
                |
                v
  +---------------------------+
  |  SLO TARGET               |     "Just barely good enough."
  | - cost curve              |     Output: a number like 99.9%
  | - dependency math         |     and a window like 30d rolling.
  | - user JND threshold      |
  +-------------+-------------+
                |
                v
  +---------------------------+
  |  ERROR BUDGET (math)      |     1 - SLO, materialised as
  | - 30d / 90d windows       |     remaining minutes.
  | - rolling vs calendar     |
  +-------------+-------------+
                |
        +-------+--------+
        |                |
        v                v
+---------------+  +-----------------------+
| BURN-RATE     |  | ERROR BUDGET POLICY   |   The org artifact.
| ALERTING      |  | - signed by Eng+Prod  |   Without this, SLOs
| (Apr 23 base, |  | - escalation triggers |   are a vanity dashboard.
| deeper here)  |  | - feature freeze      |
| 14.4/6/3/1    |  | - postmortem feedback |
+-------+-------+  +-----------+-----------+
        |                      |
        v                      v
+--------------------------------------------+
|        TOOLING (one of these)              |
| Sloth (CLI)  Pyrra (CRD+UI)  Grafana SLO   |
| OpenSLO (spec)  Nobl9 (SaaS)               |
| -> generates PrometheusRule(s)             |
+--------------------------+------------------+
                           |
                           v
+--------------------------------------------+
|  PROMETHEUS / MIMIR (Apr 20, Apr 26)        |
|  - recording rules: SLI ratios per window   |
|  - alert rules: burn-rate AND-of-2-windows  |
|  - feeds Alertmanager (Apr 23)              |
+--------------------------+------------------+
                           |
                           v
+--------------------------------------------+
|   ALERTMANAGER -> ON-CALL                   |
|  - severity matrix:                         |
|    fast-burn 14.4x -> PAGE                  |
|    medium-burn 6x  -> PAGE                  |
|    slow-burn 3x    -> TICKET                |
|    crawl-burn 1x   -> DIGEST                |
|  - runbook_url -> Markdown in Git           |
+--------------------------+------------------+
                           |
                           v
+--------------------------------------------+
|   GRAFANA: SLO BURNDOWN DASHBOARDS          |
|  - remaining budget gauge                   |
|  - 30d burndown line                        |
|  - per-feature breakdown                    |
+--------------------------+------------------+
                           |
                           v
+--------------------------------------------+
|   POSTMORTEM -> POLICY -> SLO REVISION      |
|  - blameless retro                          |
|  - did the alert fire fast enough?          |
|  - did the runbook work?                    |
|  - is the SLO still right?                  |
+--------------------------------------------+

   THE FEEDBACK LOOP (the part most teams miss)
   --------------------------------------------
   Postmortem findings feed back into:
   - SLI definition (was the wrong thing measured?)
   - SLO target (too tight? too loose?)
   - Burn-rate thresholds (alert fired too late?)
   - Runbook (the flowchart at 3 AM)
   - Policy (did the consequences trigger correctly?)
```

Six things to remember as we go:

1. **SLI selection is the hardest step.** Once the SLI is right, everything else is mechanical. Most failed SLO programs failed at SLI selection, not at math.
2. **The error budget is denominated in minutes, not percentages.** "We have 18 minutes left this month" is a sentence engineers can act on. "We are at 99.62%" is a sentence that gets ignored.
3. **The policy is the lever.** SLOs without consequences are vanity dashboards. The policy is the contract between engineering and product about what happens when the budget is spent.
4. **Burn-rate alerts are about *speed of consumption*, not *current value*.** Apr 23 derived this from the SRE Workbook; today's doc goes deeper into *why* 14.4 specifically and how to differentiate alert tiers.
5. **SLO tooling is mostly cosmetic.** Sloth/Pyrra/Grafana SLO/Nobl9 all generate the same underlying PrometheusRules. Pick on operability, not features.
6. **Maturity matters more than tooling.** A team with a clear SLO and a 1-page runbook beats a team with Nobl9 and no policy every time.

---

## Part 1: SLI -- What to Measure

The single hardest decision in an SLO program is **what to measure**. Not "Prometheus or Datadog?" Not "99.9 or 99.95?" The hardest call is: of the thousand metrics your service emits, which one or two represent *what users actually feel?* Get this wrong and every other piece of the program -- the target, the budget, the alerts, the policy -- is built on sand.

The SRE Workbook frames it as a two-step process: **specification** (what do users care about, in plain English?) followed by **implementation** (how do I express that as a query my monitoring system can run?). Most teams skip step 1 and go straight to "rate of HTTP 5xx," which is *an* SLI but is rarely *the* SLI users actually feel.

### The four canonical SLI categories

The Workbook (Chapter 2) recognizes four kinds of services, each with its own SLI shape:

| Service type | What users feel | Canonical SLI shape | Example |
|--------------|------------------|----------------------|---------|
| **Request-driven** | "Did my request succeed, and was it fast enough?" | `successful_requests / valid_requests` and `requests_under_threshold / valid_requests` | Web API, GraphQL endpoint, RPC service |
| **Pipeline / data processing** | "Is my data fresh and complete?" | `events_processed_within_freshness_threshold / events_in` and `events_processed_correctly / events_in` | Kafka -> Flink job, ETL, log ingestion, ML feature pipeline |
| **Storage** | "Will my data be there, and will it be correct?" | `successful_reads / read_attempts` (availability), `bytes_intact / bytes_stored` (durability), p99 read latency | S3, RDS, DynamoDB-backed services, blob stores |
| **Composite / user journey** | "Did the entire flow work?" | Product of upstream SLIs, OR a journey-instrumented end-to-end probe | Login (auth + session + profile), checkout (cart + payment + inventory + email) |

The Apr 21 PromQL doc covered the request-driven case exhaustively (RED method: Rate, Errors, Duration). Today's doc is about the three categories that are usually skipped, and about composing them into journey-level SLOs.

### Pipeline / data-freshness SLIs (the most-skipped category)

Half of every modern stack is asynchronous: Kafka topics, Kinesis streams, Airflow DAGs, Spark jobs, Flink stream processors, dbt models, ML feature pipelines. **None of these have HTTP requests, so RED-method SLIs do not apply.** Yet they are exactly the components that page on-call when "the dashboard says 0" or "yesterday's report is wrong."

The right SLI for pipelines is almost always **freshness** plus **correctness**:

- **Freshness:** what fraction of events are processed within an acceptable lag of being produced?
- **Correctness:** what fraction of events are processed without producing a downstream error or schema violation?

Concrete example for a Kafka -> Flink -> S3 event pipeline:

```promql
# Freshness SLI: 99% of events processed within 5 minutes of producer timestamp.
# Requires the consumer to emit a histogram of (processing_time - event_time).

sum(rate(pipeline_event_lag_seconds_bucket{le="300", job="flink-events"}[5m]))
/
sum(rate(pipeline_event_lag_seconds_count{job="flink-events"}[5m]))
```

The denominator is "events that arrived in the last 5 minutes." The numerator is "events whose `processed_time - event_time` was under 300s." The ratio is the freshness SLI; an SLO of 99.5% means "99.5% of events processed end-to-end within 5 minutes of being produced, measured over the SLO window."

For pipelines that emit *batches* rather than per-event metrics (e.g., daily dbt runs), the SLI shifts to "did the run complete within the SLA window?" -- typically a recording rule on `kube_job_status_completion_time - kube_job_status_start_time` against an SLA threshold.

The trap that pipeline SLIs uncover: many teams *think* they have a request-driven service when they actually have a pipeline. A "real-time" recommendation API that fetches features from a feature store updated by a streaming job has a *user-facing* request-driven SLI (latency/availability of the API) **and** an *internal* pipeline SLI (feature freshness). If you only set the request SLI, you will get paged because users complain that recommendations are stale -- the API was 100% available but the data behind it was 6 hours old.

### Storage SLIs -- availability, durability, correctness are different

Storage services have *three* SLIs that are often conflated:

1. **Availability:** can the storage service answer my read/write right now? `successful_ops / total_ops`.
2. **Durability:** if I wrote data 5 years ago, is it still there? Measured against expected loss probability over time -- e.g., S3's 99.999999999% (eleven nines) is a durability claim, not an availability one.
3. **Correctness:** the bytes I read back are the bytes I wrote (no silent corruption). Often instrumented as periodic checksum verification.

Most teams set an availability SLO and call it done. But for any storage system holding regulated data (PII, PHI, financial records), **the durability SLO is the one that matters in court**. Implementation is harder because durability is a probability over time, not a ratio over events. Typical implementations use:

- **Background scrubbers** that read every object on a schedule (S3 erasure-coded background scrubbing, Cassandra repair, ZFS scrub) and emit `objects_scrubbed_with_checksum_match / objects_scrubbed`.
- **Replication lag** to a separate region/account as a freshness floor on durability.
- **Restore tests** -- regularly restoring from backup and emitting `restore_successful / restore_attempted`.

### User-journey composite SLIs

The most user-honest SLI is the multi-step journey: **"% of users who completed the entire login flow in under 3 seconds without error."** This is what users actually feel. They do not care that your auth service is 99.95% if your session service is 99.5% and 0.5% of logins fail at the seam.

Composing journey SLOs is **multiplicative, not additive**. If a flow has three serial steps each at 99.9%, the journey availability is `0.999^3 = 0.997`, or 99.7%. The Workbook is explicit on this: **the SLO of a composite service can never exceed the product of its dependencies' SLOs.** This is the "dependency math" pillar covered below in target setting.

Two implementation patterns:

1. **Composite from per-step SLIs (cheap, approximate).** Express each step as its own ratio, multiply, alert on the product. Works when steps are independent. Fails when the same request triggers multiple steps that fail correlated.
2. **End-to-end synthetic journey (expensive, accurate).** A dedicated synthetic job (Grafana k6, AWS Synthetics, Datadog Synthetics) executes the full flow on a schedule from N regions and emits `journey_succeeded / journey_attempted`. This is the *correct* answer for tier-0 user journeys and the *wrong* answer for tier-3 internal services.

### Traffic-weighted vs population-weighted SLIs -- coverage as a first-class signal

Standard request-based SLIs are **traffic-weighted**: the denominator is total events, so a chatty 10% of users dominate the ratio while a quiet 90% are statistically invisible. This works for symmetric failure modes (a deploy that breaks everything for everyone) but **systematically hides skewed failures**: a stuck Kafka partition that affects 5% of users but they happen to be the low-traffic tail; a per-tenant cache that's wedged for one customer; a consistent-hashing pod that's serving stale data only to users whose user-id hashes to it.

The fix is a **coverage SLI** that measures the *population*, not the *traffic*:

```
good_events  = unique active users served correctly in the window
valid_events = total unique active users in the window
```

This pairs with a request-rate SLI to give you both signals: **freshness/correctness asks "is what we're serving right?"; coverage asks "is everyone getting it?"** A user-facing service that fronts asynchronous data (recommendation feature stores, search indexes, notification timelines) almost always needs both, because the failure mode where "10% of users see hours-old data while 90% are fine" is invisible to traffic-weighted SLIs but is the failure that wakes Customer Success up at 9 AM Monday. The implementation cost is real -- coverage requires a unique-users counter (HyperLogLog in Redis, or a recording rule with `count(count by (user_id)(...))`) -- but it pays for itself the first time a partition gets stuck.

### The "good events / valid events" formula -- and why "valid" matters

The canonical SLI formula is:

```
SLI = good_events / valid_events
```

`good_events` is intuitive -- the requests that succeeded, the events that were fresh, the bytes that were intact. **The subtle word is `valid_events` -- not `total_events`.** "Valid" excludes:

- **Synthetic / health-check traffic.** Including health checks in the denominator inflates SLIs to ~100% during real incidents (because health-checks keep succeeding while users fail), masking the outage.
- **Requests we intentionally rejected.** A 401 from a missing API key is *correct* behavior, not a service failure. It belongs out of both numerator and denominator.
- **Requests blocked by WAF / bot mitigation.** Dropping malicious traffic is success, not failure.
- **Requests during planned maintenance windows.** If your error budget policy explicitly allows them, exclude. Otherwise include.

The most expensive trap: **dividing by zero during outages.** If your service goes completely silent (no traffic at all), `valid_events = 0`, your SLI ratio is undefined, and a naive Prometheus expression returns `NaN`. The wrong attempts you will see in the wild:

```promql
# Naive: NaN during full outage
sum(rate(http_requests_total{code!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Looks right but is WORSE: silently reports 0% SLI during full outage,
# because numerator=0 / denominator=1 (forced via vector(1)) = 0. This
# floods burn-rate alerts during a real outage with the wrong signal.
(sum(rate(http_requests_total{code!~"5.."}[5m])) or vector(0))
/
(sum(rate(http_requests_total[5m])) > 0 or vector(1))
```

The correct pattern is to track the **error ratio** (not success ratio) and clamp the denominator so zero traffic yields a 0 error ratio (not NaN, not a misleading "100% errors"). Then use `absent_over_time` separately to alert that the service is gone:

```promql
# Right: error ratio with clamped denominator. Returns 0 during zero
# traffic, which correctly says "no errors observed." Burn-rate alerts
# fire only when errors are actually accumulating against the budget.
sum(rate(http_requests_total{code=~"5.."}[5m]))
/
clamp_min(sum(rate(http_requests_total[5m])), 1)

# Plus a separate "service is silent" alert -- distinct signal:
absent_over_time(http_requests_total{job="checkout-api"}[10m])
```

The first query gets the SLI math right; the second handles "is the service even running?" without conflating it into the SLI denominator. **Sloth and Pyrra both follow this two-query pattern** -- if you hand-roll PrometheusRules, replicate it. The `or vector(0/1)` idiom is a common foot-gun that produces a syntactically valid but semantically wrong SLI during the exact moments you most need it to be right.

### Anti-pattern: averaging latency as an SLI

The single most common SLI mistake: using `avg(rate(http_request_duration_sum[5m]) / rate(http_request_duration_count[5m]))` as a latency SLI. **This is broken in three independent ways:**

1. **Averages hide tail latency.** A service serving 99% of requests in 50ms and 1% in 30 seconds has an average of 350ms -- looks fine. The 1% who waited 30s feel terrible.
2. **Averages cannot be SLO'd against a meaningful threshold.** "Is 350ms good?" requires comparing to the user JND (just-noticeable difference); 99th percentile vs 250ms is unambiguous.
3. **Averages can't be combined across services.** Avg of avgs is not avg of total -- it's mathematically wrong (Apr 21 covered this for `histogram_quantile`).

The correct pattern: **bucket-against-threshold**. Pick a latency threshold users feel (e.g., 250ms), and the SLI is "fraction of requests under that threshold":

```promql
sum(rate(http_request_duration_seconds_bucket{le="0.25"}[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))
```

This is **not** the same as `histogram_quantile(0.99, ...) < 250ms`. The bucket-against-threshold form is a ratio (an SLI); the quantile form is a scalar (a value). You SLO the ratio. The Apr 21 doc covered the quantile form for dashboards and ad-hoc analysis; for SLOs, always use the ratio form.

---

## Part 2: SLO Target -- Picking the Number

Once you know what to measure, the next question is: how good is good enough? **99% or 99.9% or 99.99%?** This decision is *political* as much as technical, because each "nine" added is roughly an order of magnitude more expensive in engineering effort.

### The cost curve

Empirically:

| SLO | Budget / month | Implications |
|-----|-----------------|--------------|
| 99% | 7.2 hours / month | Single-AZ is fine, deploy-and-pray is fine, manual remediation is acceptable. Budget covers a typical "afternoon outage." |
| 99.5% | 3.6 hours / month | Need decent monitoring. Manual deploys with a runbook. Budget covers a full incident response. |
| 99.9% | 43.2 minutes / month | Multi-AZ. Automated rollback. Health checks that work. Budget = one bad deploy. |
| 99.95% | 21.6 minutes / month | Active health probes. Canary deploys. Tested DR runbooks. Budget = one alert-triage cycle. |
| 99.99% | 4.32 minutes / month | Active-active multi-region. Automated remediation. SLI computed from real user telemetry. Budget = one auto-rollback. |
| 99.999% | 26 seconds / month | The realm of telcos and core financial systems. Most companies should NOT promise this. Budget = a single TCP timeout. |

Each row is approximately 10x the engineering effort of the row above it, because:

- Detection time has to drop by 10x (you can't burn 10% of a 26-second budget on a 60-second alert delay).
- The blast radius of any single failure has to drop by 10x.
- The rate of human error has to drop by 10x -- which means manual operations stop scaling and you need automation.

### Dependency math (the constraint nobody enforces)

**Your SLO can never exceed the product of your dependencies' SLOs.** If your API calls a database with a 99.9% SLO and a cache with a 99.95% SLO, the absolute ceiling on your API's user-perceived availability is `0.999 * 0.9995 = 0.99850`, or about 99.85%.

In practice it's worse, because:

- **You also depend on the network** (typically 99.99-99.999% SLA from cloud providers).
- **You depend on AWS region availability** (publicly, EC2's SLA is 99.99%, but multi-AZ designs hit higher).
- **You depend on your own deploy pipeline** (every deploy is a controlled outage; the budget should account for it).

So a service that *claims* a 99.95% SLO while sitting on a 99.9% RDS dependency is making a promise it provably cannot keep. The SRE Workbook calls this **the SLO target ceiling**, and the rule is: **set your SLO at most equal to the lowest SLO of any *critical* dependency, and ideally one nine looser.**

If you cannot meet your target because of dependencies, you have three options:

1. **Lower your SLO** (the honest answer).
2. **Add redundancy** -- multi-region, multi-AZ, multi-cloud -- to make the effective dependency SLO higher.
3. **Renegotiate with the dependency owner** -- "we need RDS to be 99.95% for our 99.9% to be defensible."

> **The independence assumption is also wrong.** The product-of-SLOs formula assumes failures are *independent* events. In real cloud architectures they aren't: shared AWS control planes, IAM service availability, regional metadata services, and DNS resolution mean that a single underlying event takes down multiple "independent" dependencies at once. An IAM regional disruption simultaneously degrades RDS, S3, Lambda, and DynamoDB control-plane operations -- the math says `0.9995 * 0.999 * 0.9999...` but the *real* failures are correlated and the realistic ceiling is **0.5-1× nines lower than the math predicts**. Build the dependency-math number as the *optimistic* ceiling, then derate it for correlated-failure risk before promising anything externally. This is also why "we run on AWS" doesn't get you 99.99% just because each AWS service publishes 99.99% -- shared-fate components mean the effective compound SLA is lower than the multiplicative product.

### User JND -- "just barely good enough"

The Workbook's principle: **"set the SLO target as the minimum value users won't notice."** Users *cannot tell* the difference between 99.99% and 99.999% of a web app. They *can* tell the difference between 99% and 99.9% (the former feels flaky). The SLO should sit just above the threshold of user-perceptibility, not at theoretical maximum.

Practical guidance:

- **Web app, request latency:** 99-99.9% under 250-500ms is the perceptual band. Anything tighter wastes engineering for invisible gains.
- **Background job freshness:** 99% within `1.5 * promised_window`. Users notice "stale dashboard" but tolerate 2-minute delays.
- **Storage durability:** match the regulatory floor (typically 11 nines) -- this one is *not* about user perception, it's about audit.

### The vanity SLO anti-pattern

If your service has been running at 99.97% for the last 90 days, **do not set an SLO at 99.97%.** The SLO must leave headroom for change -- that's literally what the budget is for. Setting an SLO at observed performance produces:

- A budget of 0 (because you've consumed all of it just maintaining the historical average).
- Constant burn-rate alerts (because normal variation crosses the threshold).
- An organizational reflex to *avoid changes* rather than spend the budget on improvements.

The rule: **SLO < observed performance, with the gap = one full incident's worth of budget.** If your service runs at 99.97% and a typical incident burns 30 minutes of budget, set the SLO at 99.9% (43.2 min/month), giving you ~13 minutes of headroom for routine variation plus one full incident before alerts fire.

---

## Part 3: Error Budget Math (The Numbers)

The error budget is `1 - SLO`, expressed as time. Memorize these numbers; they come up in every SLO conversation:

| SLO | % budget | Min/30d | Min/90d | Min/year |
|-----|----------|---------|---------|----------|
| 99.0% | 1.0% | 432 (7.2h) | 1296 (21.6h) | 5256 (87.6h) |
| 99.5% | 0.5% | 216 (3.6h) | 648 (10.8h) | 2628 (43.8h) |
| 99.9% | 0.1% | **43.2** | 129.6 | 525.6 (8.76h) |
| 99.95% | 0.05% | **21.6** | 64.8 | 262.8 (4.38h) |
| 99.99% | 0.01% | **4.32** | 12.96 | 52.56 (52.6m) |
| 99.999% | 0.001% | 0.43 (26s) | 1.30 | 5.26 |

**Rule of thumb:** 30 days has 43,200 minutes. Multiply by `(1 - SLO)` to get the budget in minutes.

### Calculating remaining budget in PromQL

The canonical "remaining budget" query over a rolling 30-day window:

```promql
# Convention: slo:sli_error:ratio_rateXX is the *error* ratio (5xx / total)
# evaluated over the window XX, defined as a recording rule.

# Error budget remaining = 1 - (observed_error_ratio / allowed_error_ratio)
1 - (
  slo:sli_error:ratio_rate30d{slo_id="checkout-api-availability"}  # observed error ratio
  /
  (1 - 0.999)                                                       # allowed error ratio (99.9% SLO)
)
```

When this expression is `1.0`, the budget is full. When it is `0.0`, the budget is exhausted. When it is *negative*, the SLO has been violated and the team is in budget-exhausted territory.

The 30d window is computed via **recording rules** -- never query 30d windows live. A 30d range query over a histogram that's scraped every 30 seconds is effectively a Prometheus DOS (millions of samples per query). The standard pattern: a stack of recording rules at 5m, 1h, 6h, 1d, 3d, 30d, each computed at progressively longer evaluation intervals to keep the query cost bounded.

Sloth and Pyrra both auto-generate this stack. If you hand-roll it (covered in Part 7), the rule taxonomy looks like:

```
slo:sli_error:ratio_rate5m         <- raw SLI, evaluated every 30s
slo:sli_error:ratio_rate30m        <- aggregated, evaluated every 5m
slo:sli_error:ratio_rate1h         <- aggregated, evaluated every 5m
slo:sli_error:ratio_rate6h         <- aggregated, evaluated every 30m
slo:sli_error:ratio_rate1d         <- aggregated, evaluated every 1h
slo:sli_error:ratio_rate3d         <- aggregated, evaluated every 1h
slo:sli_error:ratio_rate30d        <- aggregated, evaluated every 1h
```

The longer-window rules sum the shorter-window rules; the aggregation cost is paid at evaluation time, not query time. This is the single most important performance pattern in SLO tooling.

### Rolling window vs calendar window

**Always use a rolling window for the SLO, never a calendar-month boundary.** Calendar months reset on the 1st, which means:

- An incident on the 31st burns 100% of the budget for one day.
- The same incident on the 1st starts the next month with 100% budget restored.
- The team learns nothing about whether the SLO is realistic over time; they see a sawtooth.

Rolling 30-day windows give a continuous, smoothly-decaying view of reliability that reflects actual risk to users. Customer-facing SLAs may use calendar billing windows for legal reasons -- that's fine, but the *engineering* SLO is rolling.

---

## Part 4: The Error Budget Policy -- The Org Artifact

This is the part most engineers skip. Without a written, signed policy, SLOs are decoration. The policy is the contract between **engineering** (who own velocity and reliability) and **product / business** (who own feature priorities and customer commitments) about *what happens when the budget is spent*.

### What an error budget policy contains

The Google SRE Workbook (Chapter 4) provides the canonical template. The minimum-viable policy has six sections:

1. **Goals and non-goals.** Why this SLO exists, what reliability tier it represents, what it does *not* cover.
2. **The SLOs and their targets.** Explicit target, window, SLI definition. (This is the "what" -- the technical part.)
3. **The escalation ladder.** What happens at each budget threshold.
4. **The release policy.** What kinds of releases are allowed at each budget level.
5. **The dispute resolution process.** What happens when engineering and product disagree on whether to invoke a consequence.
6. **The signature block.** Who signed off (engineering lead, product lead, SRE/platform lead minimum).

### The escalation ladder (the heart of the policy)

A typical escalation ladder maps budget thresholds to organizational consequences:

| Budget remaining | State | Consequences |
|-----------------|-------|--------------|
| > 75% | Healthy | Normal velocity. Risky deploys allowed. Experimental features OK. |
| 50-75% | Cautious | Monitor more closely. Defer non-critical risky changes. Increased eng review on deploys. |
| 25-50% | Yellow | Feature freeze on non-customer-facing work. Mandatory pre-deploy review. SRE attends standups. |
| 10-25% | Red | Hard feature freeze. All deploys require sign-off. Daily incident review. Reliability work prioritized. |
| < 10% or exhausted | Code Yellow (org-level) | Customer-facing communication. War-room status. Reliability becomes top priority for entire team. Possible "diversion" -- routing critical traffic only. |
| Sustained breach | Policy review | After 2-3 sustained breaches, the policy itself is up for review -- is the SLO too aggressive? Is the dependency graph wrong? |

The point of the ladder is that **consequences must be automatic and unambiguous.** "We'll have a meeting" is not a consequence. "All non-bug-fix PRs are blocked from merging until the budget recovers above 25%" is a consequence -- it's enforceable in CI.

### The blameless-postmortem feedback loop

The policy must wire postmortems back into SLO governance:

1. **Every burn that consumes >10% of budget** triggers a postmortem.
2. **Every postmortem** asks: was the SLO appropriate? Did the alerts fire fast enough? Was the runbook useful?
3. **Postmortem findings feed back into SLI/SLO/alert/runbook revision.** If the SLO is consistently over- or under-spent, it gets adjusted at the next review cycle.
4. **The policy itself is revisited quarterly** -- not because policies should be unstable, but because services evolve and the policy needs to evolve with them.

### Dispute resolution

The hardest section of the policy: what happens when a feature freeze is triggered but Product believes the freeze is wrong (e.g., a budget breach was caused by a third-party outage outside engineering's control)?

The Workbook's recommendation: **predefined exceptions are allowed but must be recorded.** Common exceptions:

- **Force majeure / upstream provider outages.** AWS US-East-1 going down is generally considered an external event; budget consumption may be deducted with explicit documentation.
- **Planned maintenance windows** explicitly excluded *in advance*.
- **Bug-fix releases** that demonstrably *reduce* error rate may be exempted from freeze.

What should *not* be allowed: "well, this is an important feature so we're going to ignore the freeze just this once." The policy collapses the moment exceptions become discretionary. **The escape hatch is the formal exception process; the freeze itself is not negotiable.**

### Anti-pattern: SLOs without consequences

The most common failure mode: a team sets up beautiful Grafana dashboards showing burndown charts, brags about the SLO program in all-hands, and then *nothing changes when the budget is breached*. This is worse than not having SLOs at all, because:

- It signals to the org that reliability commitments are theater.
- It teaches engineers that SLO breaches are normal.
- It demoralizes SRE / platform teams whose job is to defend the SLO.

The single most important question for an SLO program: **"What stops happening when the budget is spent?"** If the answer is "nothing," you have a vanity dashboard, not an SLO program.

### The success metric: coverage vs enforcement

When platform teams report on SLO programs to leadership, the natural metric is *coverage*: "we have SLOs on 12 services this quarter, 20 next quarter, 35 by year-end." This is the wrong metric. **The right success metric is enforcement: how many services have a signed error budget policy and have *acted on it* -- frozen launches, declared code-yellow, paused feature work because the math said so.**

The failure mode the wrong metric hides: a team can spin up SLOs on 30 services with beautiful Sloth-generated rules and Grafana burndowns, blow through three budgets in a quarter, ship 47 features anyway, and report "30 services covered" as success. Coverage went up. Reliability culture went nowhere. Adding 8 more services to that program is just the same problem at scale -- 8 more dashboards no one acts on.

The interview-grade reframe when leadership asks to expand the program: **"Before we add 8 more services, let me show you Q3 -- 3 of our existing 12 breached budget, 47 features shipped, 0 retros mentioned the breach. We don't have an SLO program; we have SLO infrastructure. The next 8 services without a signed policy is the same theater 8 more times. Let me pilot a real policy on one team this quarter and use that as the template."** Coverage is a leading indicator that's easy to game. Enforcement is the lagging indicator that actually predicts reliability outcomes.

This is the difference between **SLO theater** (instrumented, dashboarded, never acted on) and **SLO practice** (signed policy, three-way handshake, demonstrated freezes). Promote the metric you want to see; if you report coverage, you'll get coverage.

---

## Part 5: Multi-Window Multi-Burn-Rate Alerting -- Going Deeper Than Apr 23

The Apr 23 Alertmanager doc covered the basics: the 14.4/6/3/1 burn rate table, fast-burn vs slow-burn PAGE rules, AND-ed short+long windows, and the burn-rate-vs-threshold decision framework. **Today's deeper view: why those specific numbers, what happens at different SLO tiers, and how to wire severity routing.**

### Why 14.4 specifically -- the math

The SRE Workbook walks through six iterations of alerting design (target-error-rate, increased-window, alert-duration, burn-rate, multi-burn-rate, multi-window-multi-burn-rate). Each iteration fixes a flaw in the previous one. The Workbook's canonical table (Ch.5, Table 5-6) has **three** rows:

| Burn rate | Long window | Short window | Budget consumed | Time to exhaustion at this rate | Severity |
|-----------|-------------|--------------|-----------------|----------------------------------|----------|
| 14.4x | 1h | 5m | 2% in 1h | ~50h to exhaust full budget | PAGE (fast burn) |
| 6x | 6h | 30m | 5% in 6h | ~5d to exhaust full budget | PAGE (medium burn) |
| 1x | 3d | 6h | 10% in 3d | ~30d (= window) to exhaust | TICKET (crawl burn) |

In production it is common to add a **fourth tier** between the Workbook's medium and crawl rows -- 3x / 1d long / 2h short for a "slow burn" ticket -- popularized by [Grafana Labs' MWMBR blog](https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/) and the SoundCloud engineering post. The 4-row schema below is this common extension, **not** the original Workbook table:

| Burn rate | Long window | Short window | Budget consumed | Severity |
|-----------|-------------|--------------|-----------------|----------|
| 14.4x | 1h | 5m | 2% in 1h | PAGE (fast burn) |
| 6x | 6h | 30m | 5% in 6h | PAGE (medium burn) |
| 3x | 1d | 2h | 10% in 1d | TICKET (slow burn) -- *Grafana/SoundCloud extension, not in Workbook* |
| 1x | 3d | 6h | 10% in 3d | TICKET (crawl burn) |

The math behind 14.4: if your SLO window is 30 days = 720 hours, and the budget allows 0.1% errors over that window (for a 99.9% SLO), then a sustained burn rate of 14.4× means **2% of the entire monthly budget consumed every hour** -- equivalently, the full budget exhausted in `720h / 14.4 = 50h` if the burn does not stop. The "consume 2% per hour" framing is the cleaner one: it makes the alert threshold legible in budget terms, not multiplier terms. The 1h-long-window plus 5m-short-window combination then gives:

- **Long window (1h):** detects sustained burn -- prevents alerting on a single bad scrape.
- **Short window (5m):** confirms the burn is *currently* happening -- prevents alerting on a 1-hour-old issue that's already resolved.

The AND of both means: the burn must be sustained over an hour AND still happening in the last 5 minutes. **Detection latency** for a fast burn is the time for the 5-minute short window to qualify (i.e., ~5 minutes from the start of a sustained burn) plus any `for:` clause you add. Workbook examples deliberately omit `for:` on burn-rate alerts -- the AND-of-windows construct already serves the purpose of `for:`. Adding a 2-minute `for:` on top of a 1h+5m AND construct delays paging to ~7 minutes (see gotcha #9). Apr 23 derived this; the deeper insight today is that **the same 14.4/6/3/1 schema does not work uniformly across SLO tiers.**

### Burn rates differ by SLO tier

A 99.99% SLO has a budget of 4.32 minutes/month. A 14.4x burn rate threshold means "a rate that would exhaust the budget in 50 hours." For a 99.99% SLO, **a 14.4x burn for 5 minutes consumes 17% of the entire monthly budget.** That's not a "fast burn alert" -- that's "the budget is gone."

The SRE Workbook acknowledges this implicitly. In practice, tighter SLOs need:

- **Lower burn rate thresholds** (e.g., 7.2x and 3x for 99.99%) -- because even small bursts matter.
- **Shorter detection windows** (e.g., 30m and 2m, not 1h and 5m) -- because you can't afford to miss the first 5 minutes.
- **More aggressive routing** -- the "crawl burn" alert for a 99.99% service is what would be the "slow burn" page for a 99.9% service.

A practical schema for differentiated tiers:

| SLO tier | Fast burn | Medium burn | Slow burn | Crawl burn |
|----------|-----------|-------------|-----------|------------|
| 99% | 14.4x / 1h+5m | 6x / 6h+30m | 3x / 1d+2h | 1x / 3d+6h |
| 99.9% | 14.4x / 1h+5m | 6x / 6h+30m | 3x / 1d+2h | 1x / 3d+6h |
| 99.95% | 12x / 30m+2m | 5x / 4h+15m | 2.5x / 18h+1h | 1x / 3d+6h |
| 99.99% | 7.2x / 30m+2m | 3x / 2h+10m | 1.5x / 12h+30m | 1x / 3d+6h |

These are starting points; tune on real production data over a quarter. The principle: **detection time should fit inside the budget**. A 99.99% SLO's 4.32-minute budget means alerts must fire and humans must respond in well under 4 minutes of cumulative burn time -- which often means tighter integration with auto-remediation than with on-call paging.

### "You cannot alert your way to four-9s"

The deeper structural point is that at 99.99% and above, the human response chain is fundamentally too slow to be the primary mitigation. Concretely: a 14.4x fast-burn alert fires when 2% of monthly budget has been consumed in 1 hour. For a 99.9% SLO that's ~52 seconds of cumulative downtime equivalent -- the on-call has ~42 minutes of budget left when they engage. For a 99.99% SLO, 2% of 4.32 minutes is **~5 seconds**. The realistic floor for human response (PagerDuty delivery + acknowledgment + opening laptop + engaging + diagnosis) is **3-10 minutes minimum**. That is *longer than the entire monthly budget*.

The implication is qualitative, not quantitative: **at 99.99%+ the alerting model has to invert -- automated mitigation runs first, alerts confirm or escalate it.** Specifically:

- **Canary deploys with automatic rollback** on SLO breach -- most outages are deploy-induced, so this is the highest-leverage lever.
- **Circuit breakers** on dependencies so degradation contains itself instead of cascading.
- **Regional failover with health-checked traffic shifting** -- the system mitigates without paging a human at all.
- **Load shedding / graceful degradation** that triggers on error-rate thresholds in seconds, not minutes.
- **Pages fire on automation failure** ("auto-rollback didn't help") or sustained burn after mitigation -- not as the front line.

This is also why the cost curve for nines is exponential: each nine forces a *qualitative* change in incident response architecture, not just tighter alert thresholds. A 99.95% SLO can be defended by a strong on-call rotation; a 99.99% SLO requires automation that runs without humans in the loop. Setting 99.99% without that infrastructure is expensive vanity -- the budget is gone before anyone reads the page.

### Severity routing matrix -- the signal-to-noise win

The deeper payoff of MWMBR over threshold alerting is **a 10x improvement in signal-to-noise.** Threshold alerts (e.g., "page when error rate > 1%") flap on every brief spike. Burn-rate alerts only fire when burn is sustained AND current.

The routing matrix combines burn rate with budget remaining:

| Burn rate | Budget > 50% | Budget 25-50% | Budget < 25% |
|-----------|--------------|----------------|--------------|
| Fast (14.4x) | PAGE on-call | PAGE on-call + escalate to manager | PAGE on-call + war room + customer comm |
| Medium (6x) | PAGE on-call | PAGE on-call | PAGE on-call + escalate |
| Slow (3x) | TICKET | PAGE on-call | PAGE on-call |
| Crawl (1x) | DIGEST | TICKET | PAGE on-call |

Implementation in Alertmanager (extending the Apr 23 routing):

```yaml
route:
  receiver: 'default'
  group_by: [alertname, slo_id]
  routes:
    # Critical: fast burn ALWAYS pages
    - matchers: [severity="critical", burn_rate="fast"]
      receiver: 'pagerduty-critical'
      continue: true

    # Slow burn with low budget escalates
    - matchers: [severity="warning", burn_rate="slow", budget_remaining="<25"]
      receiver: 'pagerduty-warning'
      continue: true

    # Slow burn with healthy budget is a ticket
    - matchers: [severity="warning", burn_rate="slow"]
      receiver: 'jira-tickets'
```

The `budget_remaining` label is set by a recording rule that buckets the budget value (`<25`, `25-50`, `>50`) -- this avoids the cardinality explosion of putting a numeric budget value as a label.

### PrometheusRule with full burn-rate stack

The full PrometheusRule for an availability SLO at 99.9% with all four burn rates:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: checkout-api-availability-slo
  namespace: monitoring
  labels:
    release: prometheus  # match kube-prometheus-stack ruleSelector (Apr 26)
spec:
  groups:
    - name: slo-checkout-api-availability-recording
      interval: 30s
      rules:
        # Raw SLI: error ratio, evaluated every 30s
        - record: slo:sli_error:ratio_rate5m
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[5m]))
          labels:
            slo_id: checkout-api-availability
            slo_target: "0.999"

        - record: slo:sli_error:ratio_rate30m
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[30m]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[30m]))
          labels:
            slo_id: checkout-api-availability

        - record: slo:sli_error:ratio_rate1h
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[1h]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[1h]))
          labels:
            slo_id: checkout-api-availability

        - record: slo:sli_error:ratio_rate2h
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[2h]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[2h]))
          labels:
            slo_id: checkout-api-availability

        - record: slo:sli_error:ratio_rate6h
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[6h]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[6h]))
          labels:
            slo_id: checkout-api-availability

        - record: slo:sli_error:ratio_rate1d
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[1d]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[1d]))
          labels:
            slo_id: checkout-api-availability

        - record: slo:sli_error:ratio_rate3d
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[3d]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[3d]))
          labels:
            slo_id: checkout-api-availability

        # The full SLO window error ratio -- this is the denominator for
        # budget-remaining. 30d range queries are expensive; that's exactly
        # why we recording-rule it once instead of running it from alerts.
        - record: slo:sli_error:ratio_rate30d
          expr: |
            sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[30d]))
            /
            sum(rate(http_requests_total{job="checkout-api"}[30d]))
          labels:
            slo_id: checkout-api-availability

        # Budget remaining over the SLO window.
        # Formula: 1 - (observed_error_ratio / allowed_error_ratio)
        #   - allowed_error_ratio = 1 - SLO_target = 1 - 0.999 = 0.001
        #   - observed_error_ratio = ratio_rate30d (computed above)
        # Yields 1.0 (= 100%) when no errors, 0 when budget fully spent,
        # negative when overspent.
        - record: slo:budget_remaining:ratio30d
          expr: |
            1 - (
              slo:sli_error:ratio_rate30d{slo_id="checkout-api-availability"}
              /
              (1 - 0.999)
            )

    - name: slo-checkout-api-availability-alerts
      rules:
        # Fast burn (14.4x) - 2% budget consumed in 1h
        # NOTE: no `for:` clause -- the AND-of-windows already serves the
        # debounce purpose. Adding `for: 2m` would delay paging to ~7 min.
        - alert: ErrorBudgetBurnFast
          expr: |
            (
              slo:sli_error:ratio_rate1h{slo_id="checkout-api-availability"} > (14.4 * 0.001)
              and
              slo:sli_error:ratio_rate5m{slo_id="checkout-api-availability"} > (14.4 * 0.001)
            )
          labels:
            severity: critical
            slo_id: checkout-api-availability
            burn_rate: fast
            team: payments
          annotations:
            summary: "Checkout API burning error budget at 14.4x rate"
            description: "Current 1h error rate {{ $value | humanizePercentage }} exceeds 14.4x normal. Budget exhaustion in ~50h if sustained."
            runbook_url: "https://runbooks.example.com/checkout-api/error-budget-burn-fast"

        # Medium burn (6x) - 5% budget consumed in 6h
        - alert: ErrorBudgetBurnMedium
          expr: |
            (
              slo:sli_error:ratio_rate6h{slo_id="checkout-api-availability"} > (6 * 0.001)
              and
              slo:sli_error:ratio_rate30m{slo_id="checkout-api-availability"} > (6 * 0.001)
            )
          for: 15m
          labels:
            severity: critical
            slo_id: checkout-api-availability
            burn_rate: medium
            team: payments
          annotations:
            summary: "Checkout API burning error budget at 6x rate"
            runbook_url: "https://runbooks.example.com/checkout-api/error-budget-burn-medium"

        # Slow burn (3x) - 10% budget consumed in 1d
        - alert: ErrorBudgetBurnSlow
          expr: |
            (
              slo:sli_error:ratio_rate1d{slo_id="checkout-api-availability"} > (3 * 0.001)
              and
              slo:sli_error:ratio_rate2h{slo_id="checkout-api-availability"} > (3 * 0.001)
            )
          for: 1h
          labels:
            severity: warning
            slo_id: checkout-api-availability
            burn_rate: slow
            team: payments
          annotations:
            summary: "Checkout API slowly burning error budget"
            runbook_url: "https://runbooks.example.com/checkout-api/error-budget-burn-slow"

        # Crawl burn (1x) - sustained baseline burn
        - alert: ErrorBudgetBurnCrawl
          expr: |
            (
              slo:sli_error:ratio_rate3d{slo_id="checkout-api-availability"} > 0.001
              and
              slo:sli_error:ratio_rate6h{slo_id="checkout-api-availability"} > 0.001
            )
          for: 6h
          labels:
            severity: info
            slo_id: checkout-api-availability
            burn_rate: crawl
            team: payments
          annotations:
            summary: "Checkout API steadily consuming budget at SLO baseline"
            runbook_url: "https://runbooks.example.com/checkout-api/error-budget-burn-crawl"
```

This single PrometheusRule represents one SLO. A real platform has dozens, generated by Sloth or Pyrra rather than hand-written.

---

## Part 6: SLO Tooling Landscape

Five real options, each with a different operational profile.

### Sloth -- the YAML-to-rules CLI

**Sloth** is a CLI that takes a YAML SLO spec and generates a complete PrometheusRule. The tool is stateless; it does not run in your cluster. You feed it specs in CI and apply the generated rules via your usual GitOps pipeline.

A complete Sloth spec for an availability + latency SLO:

```yaml
# checkout-api-slos.yaml
version: "prometheus/v1"
service: "checkout-api"
labels:
  team: "payments"
  tier: "1"
slos:
  - name: "availability"
    objective: 99.9
    description: "99.9% of HTTP requests succeed (non-5xx) over 30 days"
    sli:
      events:
        error_query: |
          sum(rate(http_requests_total{job="checkout-api",code=~"5.."}[{{.window}}]))
        total_query: |
          sum(rate(http_requests_total{job="checkout-api"}[{{.window}}]))
    alerting:
      name: CheckoutAPIAvailability
      labels:
        category: "availability"
      annotations:
        runbook_url: "https://runbooks.example.com/checkout-api/availability"
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning

  - name: "latency-p99-250ms"
    objective: 99.0
    description: "99% of requests complete in under 250ms over 30 days"
    sli:
      events:
        error_query: |
          (
            sum(rate(http_request_duration_seconds_count{job="checkout-api"}[{{.window}}]))
            -
            sum(rate(http_request_duration_seconds_bucket{job="checkout-api",le="0.25"}[{{.window}}]))
          )
        total_query: |
          sum(rate(http_request_duration_seconds_count{job="checkout-api"}[{{.window}}]))
    alerting:
      name: CheckoutAPILatency
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning
```

Generation:

```bash
sloth generate -i checkout-api-slos.yaml -o checkout-api-slos-rules.yaml
kubectl apply -f checkout-api-slos-rules.yaml
```

The generated rules cover all six time windows, the burn-rate alert hierarchy, the metadata recording rules, and a catalog rule for inventory queries -- about 25 rules per SLO. The Sloth `{{.window}}` template variable is what lets a single SLI declaration expand into the full window stack.

**Strengths:** simple, stateless, GitOps-native, no runtime dependency. Wide adoption.
**Weaknesses:** no UI -- you build burndown dashboards yourself. No Kubernetes CRD -- the source of truth is a YAML file, not a cluster object.

### Pyrra -- the Kubernetes-native operator with UI

**Pyrra** is a Kubernetes operator + UI. You install Pyrra into your cluster; it watches `ServiceLevelObjective` CRD instances and reconciles them into PrometheusRules. The UI shows a live burndown for every SLO.

```yaml
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: checkout-api-availability
  namespace: payments
  labels:
    pyrra.dev/team: payments
spec:
  target: "99.9"
  window: 30d
  description: "99.9% of HTTP requests succeed over 30 days"
  indicator:
    ratio:
      errors:
        metric: http_requests_total{job="checkout-api",code=~"5.."}
      total:
        metric: http_requests_total{job="checkout-api"}
  alerting:
    name: CheckoutAPIAvailability
    labels:
      team: payments
```

Pyrra reconciles this into PrometheusRules in the same namespace. The UI (default `:9099`) shows current burn rate, remaining budget, time-to-exhaustion projection, and historical SLI -- all without you wiring up a Grafana dashboard.

**Strengths:** Kubernetes-native, live UI, supports both `ratio` and `latency` SLIs natively, ConfigMap-mode fallback for clusters without Prometheus Operator.
**Weaknesses:** runtime dependency (the operator must be running for new SLOs to be reconciled); the UI is read-only -- editing SLOs still goes through Git/`kubectl`.

### Grafana SLO -- first-party, Cloud/Enterprise only

The **Grafana SLO** plugin manages SLOs from the Grafana UI itself. You define the SLO in a wizard; Grafana auto-generates the recording and alert rules in your Prometheus / Mimir backend, plus the burndown dashboard.

**Strengths:** zero additional infrastructure if you already have Grafana Cloud or Enterprise. Native integration with Grafana alerting (Apr 24's split). UI-first workflow.
**Weaknesses:** **OSS Grafana cannot manage SLOs** -- this feature is paid-tier only. Vendor lock-in: SLOs are stored in Grafana's database, not exposed as PrometheusRule files. Migration to/from this format is non-trivial.

### OpenSLO -- the vendor-neutral spec

**OpenSLO** is not a tool; it's a YAML schema, like OpenTelemetry Protocol. The spec defines four kinds: `Service`, `SLI`, `SLO`, `AlertPolicy`. Multiple tools (Nobl9, Sloth's `--openslo` mode, others) accept OpenSLO as input.

Example OpenSLO doc:

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: checkout-api-availability
  displayName: Checkout API Availability
spec:
  service: checkout-api
  description: 99.9% of requests succeed over 30d
  indicator:
    metadata:
      name: error-ratio
    spec:
      ratioMetric:
        counter: true
        good:
          metricSource:
            type: prometheus
            spec:
              query: sum(rate(http_requests_total{job="checkout-api",code!~"5.."}[1m]))
        total:
          metricSource:
            type: prometheus
            spec:
              query: sum(rate(http_requests_total{job="checkout-api"}[1m]))
  timeWindow:
    - duration: 30d
      isRolling: true
  budgetingMethod: Occurrences
  objectives:
    - displayName: Availability
      target: 0.999
```

**Strengths:** vendor-neutral; portable across tools. Good answer to "what if I move from Sloth to Nobl9?"
**Weaknesses:** still maturing. Tool support is uneven (some kinds aren't fully implemented in some tools).

### Nobl9 -- the commercial SaaS

**Nobl9** is a hosted SLO platform. You point it at your Prometheus / Datadog / CloudWatch / etc., declare SLOs in their UI or via OpenSLO, and they handle storage, alerting, dashboards, and reports. **Strengths:** zero ops burden; multi-data-source support out of the box. **Weaknesses:** SaaS cost (typically $5-15 per SLO per month at scale); data leaves your infrastructure (compliance considerations); vendor risk.

### Decision matrix

| Need | Pick |
|------|------|
| GitOps shop, Prometheus-only, want full control | **Sloth** |
| Kubernetes-native, want a built-in UI, multi-tenant | **Pyrra** |
| Already on Grafana Cloud / Enterprise, UI-first workflow | **Grafana SLO** |
| Multi-vendor (Datadog + Prometheus + CloudWatch), want SaaS | **Nobl9** |
| Vendor-neutral standard for portability | **OpenSLO** spec, with Sloth or Nobl9 as the engine |
| Hand-rolled, want to learn the mechanics deeply | Write the PrometheusRule yourself (Part 5 above) -- valid for ~5-10 SLOs, painful past that |

In practice, most organizations end up with **Sloth or Pyrra** for OSS Prometheus, **Grafana SLO** for Grafana Cloud, or **Nobl9** for multi-vendor environments. The choice is about operational fit, not features -- all five generate substantively similar PrometheusRules (or proprietary equivalents).

---

## Part 7: Runbook Patterns

The `runbook_url` annotation on every alert from Part 5 is only useful if the runbook on the other end actually helps the on-call. **A runbook that says "investigate the issue" is worse than no runbook**, because it teaches on-call to ignore the link.

### What makes a runbook useful at 3 AM

The 3 AM test: an on-call engineer who has never seen this service before should be able to follow the runbook and either resolve the incident or escalate confidently within 15 minutes. Concretely, the runbook must contain:

1. **Severity context.** "What does this alert mean? Is the customer impacted?"
2. **Quick triage.** A 5-step diagnostic flow with copy-paste commands. Not "check the database" -- `kubectl logs -n payments deploy/checkout-api --tail=200 | grep ERROR`.
3. **Common root causes** ranked by historical frequency, each with confirmation steps.
4. **Mitigation actions** -- not fixes, but ways to stop the bleeding (rollback, traffic shift, scale up).
5. **Escalation path** with names, Slack channels, and pager schedules.
6. **"Did the alert do its job?"** -- a section the on-call fills in after, feeding back into postmortem.

### Runbook-as-code

Runbooks live in Git, in a dedicated repo or a `/runbooks` folder of the service repo. The `runbook_url` annotation points to the rendered Markdown URL. Common patterns:

- **One Markdown file per alert name.** `runbooks/CheckoutAPIAvailability.md` -- the URL is deterministic from the alert name.
- **Jinja-templated runbooks** generated from common templates -- "all availability burn alerts" share a base template, individual services add specifics.
- **CI checks** that verify every alert in PrometheusRules has a `runbook_url` annotation pointing to an existing file.

### Anti-patterns

- **"See the dashboard"** -- the runbook *is* what the dashboard does not tell you (judgment, decision logic, escalation).
- **Untested runbooks.** Runbooks must be exercised in chaos tests / DR drills. An untested runbook is worse than no runbook.
- **Stale runbooks.** Every postmortem updates the runbook for the alert that fired. This is non-negotiable.

---

## Part 8: Observability Maturity Model

The Charity Majors / Liz Fong-Jones framing (from *Observability Engineering*, Chapter 22) gives a way to assess organizational maturity along five capability dimensions. SLOs sit near the top of this hierarchy -- they are a level-3+ practice.

### The five capabilities

1. **Response to system failure.** How quickly does the org detect, diagnose, and recover from production incidents? SLOs and burn-rate alerts are the artifact here.
2. **Delivering high-quality code.** Does observability let engineers verify their changes work as intended in production? Pre-deploy traces, exemplar-driven debugging, canary metrics.
3. **Managing complexity and technical debt.** Can the org locate the source of slowness or flakiness in a complex distributed system? Tracing + service maps.
4. **Predictable release cadence.** Does the org ship at a steady, sustainable rate? SLOs gate this -- you cannot ship faster than your error budget allows.
5. **User behavior understanding.** Do you know what users actually do? Composite SLOs and user-journey instrumentation are the bridge between user analytics and reliability.

### Where SLOs fit

SLOs are an **integrative** practice -- they require *all three pillars* of observability (metrics for the SLI, logs for the diagnostic context, traces for the deep-dive when an alert fires) to be in place and *correlated* (Apr 30's three-witnesses framing). An organization that has metrics but no traces will set SLOs but be unable to diagnose breaches; an org with traces but no SLOs will know what's slow but not whether it matters to users.

The maturity ladder, paraphrased:

- **Level 1 (Reactive).** Logs only. Alerting on log pattern matches. No SLOs.
- **Level 2 (Monitoring).** Metrics + Prometheus. Threshold alerts. RED/USE dashboards. No SLOs (or SLOs without policy).
- **Level 3 (Observability).** Three pillars + correlation. SLOs with burn-rate alerts. Written policy. Postmortems feed SLO revision.
- **Level 4 (Engineering culture).** SLOs drive product priorities. Engineers write SLIs as part of feature design. Error budgets directly gate release decisions.
- **Level 5 (Organizational).** Reliability is a first-class product attribute. Customer SLAs derive from internal SLOs. Engineering and product agree the SLO is the contract.

Most orgs sit at Level 2-3. The leap to Level 3+ is **not** about better tools -- it's about wiring the SLO program into actual decision-making.

---

## Part 9: Production Examples -- Runnable Code

### Error budget policy template

```markdown
# Error Budget Policy: Checkout API

**Effective:** 2026-05-01
**Review cycle:** Quarterly
**Owners:**
- Engineering Lead: Alice (signed)
- Product Lead: Bob (signed)
- SRE / Platform Lead: Carol (signed)

## 1. Goals and Non-Goals

### Goals
- Provide a reliable checkout experience for customers.
- Balance velocity (feature delivery) with stability.
- Make reliability decisions data-driven, not opinion-driven.

### Non-Goals
- This policy does NOT cover: third-party dependencies (Stripe, FedEx APIs)
  beyond their published SLAs; planned-maintenance windows announced 24h in
  advance; force majeure events affecting AWS us-east-1.

## 2. SLOs

### Availability
- **Target:** 99.9% over rolling 30 days.
- **SLI:** ratio of non-5xx HTTP responses to total responses, excluding
  health-check traffic and authenticated bot traffic.
- **Window:** 30-day rolling.
- **Budget:** 43.2 minutes / 30 days.

### Latency
- **Target:** 99.0% of requests under 250ms over rolling 30 days.
- **SLI:** ratio of requests under 250ms to total requests.
- **Window:** 30-day rolling.
- **Budget:** 1% of requests / 30 days.

## 3. Escalation Ladder

| Budget remaining | State | Consequences |
|---|---|---|
| > 75% | Healthy | Normal velocity. |
| 50-75% | Watchful | Defer high-risk experiments. |
| 25-50% | Yellow | Feature freeze on non-customer-facing work. Pre-deploy review mandatory. |
| 10-25% | Red | Hard feature freeze. Daily incident review with SRE. |
| < 10% | Code Yellow | Customer comm, war room, reliability work top priority. |

## 4. Release Policy

| Release type | Healthy | Watchful | Yellow | Red |
|---|---|---|---|---|
| Bug fixes | Allowed | Allowed | Allowed | Allowed |
| Feature releases | Allowed | Allowed | Blocked* | Blocked |
| Infrastructure changes | Allowed | Pre-review | Blocked* | Blocked |
| Experimental features | Allowed | Blocked | Blocked | Blocked |

*Blocked = requires written exception from Eng Lead + Product Lead.

## 5. Dispute Resolution

If product believes a freeze is incorrectly triggered:
1. File an exception request with documented evidence.
2. Eng Lead + Product Lead sign off jointly within 24h.
3. All exceptions are logged and reviewed at the quarterly policy review.

## 6. Postmortem Feedback

Every burn that consumes >10% of budget triggers:
1. Blameless postmortem within 5 business days.
2. SLO appropriateness review: was the target right? Were the alerts fast enough?
3. Runbook update.
4. Findings reviewed at next quarterly policy review.
```

### Runbook template

```markdown
# Runbook: CheckoutAPIAvailability — Fast Burn Alert

**Severity:** Critical (PAGE)
**Service:** Checkout API
**Owner:** Payments team (#payments-on-call)

## What this alert means

The checkout API is burning its error budget at 14.4x the normal rate.
At this rate, the entire monthly error budget will be exhausted in ~36 hours.
**This usually correlates with customer impact** -- checkout failures cost
revenue directly.

## Quick triage (5 minutes)

1. Check the dashboard:
   `https://grafana.example.com/d/checkout-api/checkout-api-overview`
2. Identify the error class:
   ```
   sum by (code) (rate(http_requests_total{job="checkout-api",code=~"5.."}[5m]))
   ```
3. Check recent deploys:
   ```
   kubectl rollout history -n payments deploy/checkout-api
   ```
4. Check downstream dependencies (RDS, Stripe, inventory-svc):
   - Stripe status: https://status.stripe.com
   - RDS CloudWatch: [link]
   - inventory-svc dashboard: [link]
5. Pull recent error logs:
   ```
   kubectl logs -n payments deploy/checkout-api --tail=200 | grep -i error
   ```

## Common root causes (ranked by historical frequency)

### #1 -- Recent deploy regression (40% of cases)
- **Confirm:** check `kubectl rollout history`. Is the latest revision <30 min old?
- **Mitigate:** `kubectl rollout undo -n payments deploy/checkout-api`
- **Verify:** error rate drops to baseline within 2 minutes.

### #2 -- Stripe API degradation (25% of cases)
- **Confirm:** Stripe status page; logs show `stripe.api.error` patterns.
- **Mitigate:** enable degraded-mode flag (queue payments for retry):
  ```
  kubectl set env -n payments deploy/checkout-api STRIPE_DEGRADED_MODE=true
  ```
- **Verify:** errors drop; queued payments visible in DLQ dashboard.

### #3 -- RDS connection exhaustion (20% of cases)
- **Confirm:** RDS DatabaseConnections metric near max; logs show connection-pool errors.
- **Mitigate:** scale checkout-api down by 30% to reduce connection demand:
  ```
  kubectl scale -n payments deploy/checkout-api --replicas=$(($(kubectl get deploy -n payments checkout-api -o jsonpath='{.spec.replicas}') * 7 / 10))
  ```
- **Verify:** RDS connection count drops; error rate recovers.

### #4 -- Other (15% of cases)
Escalate to senior on-call (#payments-escalate). Document new pattern in this
runbook after the incident.

## Escalation

- After 15 minutes without progress: page Senior on-call (#payments-escalate)
- After 30 minutes: invoke Code Yellow (notify #incident-commander)
- Customer comms: trigger templated status page update (link to template)

## Did the alert do its job?

After the incident, fill in:
- **Detection time:** [minutes between burn start and alert fire]
- **Diagnosis time:** [minutes between alert fire and root cause identified]
- **Did this runbook help?** [yes / no -- if no, update before next handoff]
```

### Grafana burndown dashboard JSON snippet

The canonical SLO burndown panel is a time-series of remaining budget with a 0% horizontal threshold:

```json
{
  "type": "timeseries",
  "title": "Error Budget Remaining (30d rolling)",
  "targets": [
    {
      "expr": "slo:budget_remaining:ratio30d{slo_id=\"checkout-api-availability\"} * 100",
      "legendFormat": "Budget %"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "min": -50,
      "max": 100,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"color": "red", "value": null},
          {"color": "orange", "value": 10},
          {"color": "yellow", "value": 25},
          {"color": "green", "value": 50}
        ]
      },
      "custom": {
        "lineWidth": 2,
        "fillOpacity": 10
      }
    }
  }
}
```

Pair this with a **stat panel showing remaining minutes** (more concrete than %) and a **time-to-exhaustion projection** based on current burn rate.

---

## Part 10: Production Gotchas

### 1. Calendar-month vs rolling-30-day window
**Gotcha:** Using calendar-month windows means the budget resets on the 1st. An incident on the 31st burns 100% of the budget for one day; the same incident on the 1st starts the next month with full budget.
**Why this matters:** the team gets a misleading "we're fine" signal at month-start. Reliability work feels less urgent because the budget always recovers.
**Fix:** rolling 30-day window in the SLO. Calendar windows are fine for legal SLAs (which are billing-period-based) but never for the engineering SLO.

### 2. Burn-rate alerts without recording rules = Prometheus DOS
**Gotcha:** Alerting `rate(...[1h]) > 14.4 * 0.001` directly evaluates a 1h range every alert evaluation interval. Multiply by dozens of SLOs and Prometheus queries thrash.
**Why this matters:** burn-rate alerting is supposed to *improve* reliability, not break it.
**Fix:** always layer recording rules. Apr 21 covered this; the SLO use case is its single biggest application.

### 3. Health-check traffic in the SLO denominator
**Gotcha:** if your `valid_events` includes Kubernetes liveness/readiness probes (which always succeed), your SLI is artificially inflated to ~100% during real outages, because the probe traffic dilutes user error rate.
**Why this matters:** the SLO becomes a vanity number that doesn't reflect user experience.
**Fix:** filter health checks out of both numerator AND denominator. Common patterns: exclude by user-agent (`kube-probe`), by path (`/healthz`), or by header (`x-internal-probe`).

### 4. Composite SLOs -- multiplying availabilities, not adding
**Gotcha:** "Login flow has 3 steps each at 99.9%" is a 99.7% journey SLO, not a 99.9% one.
**Why this matters:** teams set journey SLOs at 99.9% and constantly miss; the math says they cannot win.
**Fix:** explicitly compute `Π(component_SLOs)` as the journey ceiling. Set the journey SLO at most equal to the product, ideally one nine looser to leave headroom.

### 5. Aurora Multi-AZ failover -- counts toward budget?
**Gotcha:** Aurora failovers take 30-60 seconds. During that window, writes fail. Does this consume the database SLO budget?
**Why this matters:** Aurora's published SLA is 99.99% across the cluster -- which already accounts for failover time. But your *application's* SLO will see those 30-60 seconds as errors. If the app SLO is 99.95% (~21 min/month), one Aurora failover consumes 5% of the monthly budget.
**Fix:** decide explicitly in the policy. Either (a) include failover time as legitimate budget burn (forces the app to handle failover gracefully), or (b) define a documented exclusion window when Aurora notifies of planned failover. Most teams choose (a) -- it's a forcing function for retry logic and connection-pool resilience.

### 6. Holiday / incident exclusion windows -- do NOT silently exclude
**Gotcha:** "We don't count Thanksgiving Day" or "we exclude that one really bad incident" -- silent exclusions destroy trust in the SLO program.
**Why this matters:** if exclusions are discretionary, the SLO becomes whatever number Engineering wants this month.
**Fix:** if exclusions are necessary, declare them *in advance* in the policy and *record* them on the burndown chart with annotations. Never quietly subtract.

### 7. SLOs in dev / staging are noise
**Gotcha:** Cookie-cutter SLO setup creates SLOs in every environment. Dev and staging SLOs constantly fire because there are no users -- the statistical sample is too small.
**Why this matters:** alert fatigue. On-call gets paged for staging "outages" that don't matter.
**Fix:** SLOs in production only. Dev/staging gets ordinary RED dashboards (Apr 21) but no SLOs and no burn-rate alerts.

### 8. "Per-customer" SLOs explode cardinality
**Gotcha:** "Let's set an SLO per customer so we know which ones are unhappy." With 10K customers, this creates 10K time series per recording rule, times 6 windows, times N SLOs -- quickly millions of series.
**Why this matters:** Prometheus OOM. The burn-rate recording rules become unsustainable.
**Fix:** SLOs aggregate across customers. For per-customer visibility, use sampled traces or a separate analytics pipeline (e.g., dump per-customer success rate to ClickHouse, query there). Not every metric belongs in Prometheus.

### 9. Alertmanager `for:` clause vs burn-rate window math interaction
**Gotcha:** burn-rate alerts already have a long window (e.g., 1h). Adding `for: 5m` on top means the alert must be true for the entire 5m AFTER the 1h window already qualifies. This pushes detection time from 2 minutes to 7 minutes.
**Why this matters:** every minute of detection time is budget burned.
**Fix:** for fast-burn alerts, `for: 0` or `for: 1m` is sufficient (the long-window AND-short-window construct already filters noise). The Apr 23 doc covered Alertmanager's `for:` semantics; the SLO use case requires extra care.

### 10. Long windows (30d) make alerts slow to fire
**Gotcha:** a naive "alert when 30d SLI > error_budget" expression can take days to fire because the 30d window slowly accumulates the breach.
**Why this matters:** by the time it fires, the budget is irrecoverable.
**Fix:** never alert on the 30d SLI directly -- that's a *reporting* metric. Alert on burn rates over short windows (1h/5m, 6h/30m). The 30d SLI is for dashboards and policy decisions.

### 11. Rate-of-change alerts vs burn-rate alerts -- different signals
**Gotcha:** `delta(error_rate[1h]) > 0.005` (rate of change) detects regressions; `error_rate > 14.4 * 0.001` (burn rate) detects sustained burn. They're different signals and one cannot replace the other.
**Why this matters:** teams confuse "the error rate doubled" with "we're burning budget" -- doubling from 0.001% to 0.002% is a regression but not a budget burn.
**Fix:** keep both alert classes. Burn-rate for SLO consequences; rate-of-change for deploy regression detection.

### 12. Recording rule cardinality bombs in burn rate calculations
**Gotcha:** A burn-rate recording rule with `by (instance, pod)` instead of `by (service)` creates one time series per pod per window per SLO -- with pod churn, hundreds of thousands of series.
**Why this matters:** the recording rules that were supposed to *save* Prometheus query cost end up *exploding* its memory.
**Fix:** aggregate aggressively in recording rules. SLO recording rules should typically be `by (slo_id)` or `by (service, slo_id)` -- not by pod, not by instance.

### 13. Sloth/Pyrra rule-name collisions in shared Prometheus
**Gotcha:** Both Sloth and Pyrra default to recording rule names like `slo:sli_error:ratio_rate5m`. If you run both tools (during migration) or use the same Prometheus across teams, rules collide.
**Why this matters:** silently, the second-evaluated rule overwrites the first. Burndown queries return wrong values.
**Fix:** prefix recording rule names by team/service: `payments:slo:checkout_availability:ratio_rate5m`. Both Sloth and Pyrra support custom prefixes.

### 14. Dashboarding the budget -- burndown vs gauge vs stat
**Gotcha:** A radial gauge showing "67% budget remaining" looks fine but hides the trend. The team learns nothing about *whether the budget is getting worse*.
**Why this matters:** the burndown chart's *slope* is the actionable signal -- "we're spending budget twice as fast as last week."
**Fix:** primary viz is a **time-series of remaining budget** with thresholds at 50/25/10%. Stat panel is supplementary. Avoid radial gauges -- they're decorative, not informative.

### 15. SLI denominator clamping during outages
**Gotcha:** during a complete service outage, traffic drops to zero, `valid_events = 0`, `SLI = 0/0 = NaN`. Some recording rules then propagate `NaN` indefinitely until traffic resumes. The `or vector(0)` / `> 0 or vector(1)` idiom looks like a fix but is *worse*: filling numerator with 0 and denominator with 1 yields `1 - (0/1) = 1.0`, i.e., 100% error rate during silence. For a 99.9% SLO that's a `1.0 / 0.001 = 1000x` burn rate -- every threshold (fast, medium, slow, crawl) shatters simultaneously, paging on-call at 4 AM during a quiet weekend with the most expensive false-positive page possible.
**Why this matters:** burndown charts either go blank or show wildly wrong values during exactly the moments you most need them. Worse, the bug is *masked* by looking defensive: `or vector(...)` reads like "we handled the edge case" so reviewers nod through it without simulating the math.
**Fix:** the cleanest pattern is to drop the defensive fillers entirely and let the plain ratio return empty during silence (empty vector × any threshold = empty, so burn-rate alerts naturally don't evaluate), then add a *separate* alert for the silence case: `absent(rate(...)) or sum(rate(...)) == 0` with a `for: 10m` window and its own runbook. Alternative: track the **error ratio** with `clamp_min(denominator, 1)` -- error ratio stays 0 during silence, also correct. Either way, **the SLI measures quality of served traffic; a separate alert measures presence of traffic** -- conflating them produces misleading pages and hides genuinely different failure modes (broken vs silent have different runbooks). Sloth/Pyrra both follow this two-query pattern; hand-rolled rules often don't.
**PR review checklist for any SLI query:** force reviewers to walk through the four corner cases explicitly -- *what does this evaluate to during {zero traffic, all-failure, all-success, scrape failure}?* -- before stamping +1. SLI queries are operational code; "looks defensive" is not a substitute for simulating the math.

### 16. The "recovered budget" illusion
**Gotcha:** rolling 30-day windows mean a bad week 28 days ago will *fall off* the window today, and the budget will appear to recover even though nothing improved.
**Why this matters:** teams celebrate a recovering budget as evidence of a fix -- when it's just the calendar moving.
**Fix:** annotate the burndown chart with deploy markers (Apr 24 covered this with Grafana annotations). Look at the *slope*, not the level. A budget that drifts up because old incidents fall off is not the same as a budget that's improving.

---

## Part 11: Decision Frameworks

### 1. When to set an SLO at all

Not every service deserves an SLO. Setting one creates engineering overhead -- recording rules, alerts, runbooks, policy. Use an SLO when:

| Use SLO | Skip SLO |
|---------|----------|
| Customer-facing service | Internal tool used only by engineers |
| Has measurable user impact | Background experimentation / research |
| Has stable, well-understood traffic | New service still iterating on shape |
| Multiple teams depend on it | Single team owns end-to-end |
| Has change pressure from product | Stable, deployed-and-forgotten |

For services that don't need SLOs, RED dashboards (Apr 21) and threshold alerts are sufficient.

### 2. 99.9% vs 99.95% vs 99.99% by service tier

| Tier | SLO | When |
|------|-----|------|
| Tier 0 | 99.99% | Authentication, billing, anything where downtime = revenue loss within minutes |
| Tier 1 | 99.95% | Core product features (checkout, search, primary user-facing journey) |
| Tier 2 | 99.9% | Important but not blocking (notifications, recommendations, account settings) |
| Tier 3 | 99% | Internal tooling, async batch jobs where 1h delay is acceptable |
| Tier 4 | No SLO | Experimental, internal-only, fully behind feature flag |

### 3. Sloth vs Pyrra vs Grafana SLO vs Nobl9

| Dimension | Sloth | Pyrra | Grafana SLO | Nobl9 |
|-----------|-------|-------|-------------|-------|
| Architecture | CLI generator | K8s Operator | Grafana feature | SaaS |
| Source of truth | YAML in Git | CRD in cluster | Grafana DB | SaaS DB |
| UI | None (build your own) | Built-in | Built-in | Built-in |
| GitOps native | Yes | Yes (CRD) | No (UI-driven) | Partial |
| Multi-data-source | Prometheus only | Prometheus only | Multi-source | Multi-source |
| Cost | Free | Free | Cloud / Enterprise | Per-SLO/month |
| Ops burden | Lowest | Medium | Lowest | Zero |

### 4. Burn-rate alert vs threshold alert vs both

| Use burn-rate | Use threshold | Use both |
|---------------|---------------|----------|
| SLO-driven services | Hard limits (RAM 95%, disk 90%) | Tier-1 services where you want both early-warning and SLO breach signal |
| Slow-degradation detection | Resource exhaustion | |
| Multi-team consequences | Component-specific | |

### 5. In-house SLO platform vs SaaS

| Build (Sloth/Pyrra) | Buy (Nobl9 / Grafana Cloud) |
|---------------------|------------------------------|
| Have a platform team | No platform team |
| Prometheus already self-hosted | Multi-vendor data sources |
| Strong GitOps discipline | UI-first culture |
| <100 SLOs | 100+ SLOs across orgs |
| Compliance prevents data leaving cluster | Compliance allows SaaS |

---

## Part 12: 10 Commandments of SLOs

1. **Thou shalt set an SLO based on what users feel, not what your service emits.** The SLI starts with the user journey, not with the metric you happen to have.
2. **Thou shalt express SLIs as `good_events / valid_events`, never as averages.** Averages hide tail behavior; ratios make the SLO calculable.
3. **Thou shalt set the SLO target as the minimum users won't notice, no tighter.** Each "nine" is 10x the engineering cost; spend that cost only if users will see the difference.
4. **Thou shalt write the error budget policy and have it signed.** SLOs without policy are vanity dashboards.
5. **Thou shalt alert on burn rate, not on SLI threshold.** Burn-rate alerting catches sustained degradation while ignoring transient spikes; threshold alerting flaps.
6. **Thou shalt back every alert with a runbook that passes the 3 AM test.** "Investigate the issue" is not a runbook.
7. **Thou shalt use rolling windows for engineering SLOs, never calendar windows.** Calendar windows produce sawtooth incentives.
8. **Thou shalt aggregate SLO recording rules by service, never by pod.** Pod-level aggregation explodes cardinality and breaks the rules' purpose.
9. **Thou shalt feed postmortems back into SLO revision.** SLOs that never change are SLOs that nobody trusts.
10. **Thou shalt remember that SLOs are an organizational artifact, not a technical one.** The work is 20% PromQL, 80% getting Engineering and Product to agree on what reliable means -- and what happens when it isn't.

---

## Further Reading

- **Google SRE Workbook**, Chapters 2 (Implementing SLOs), 4 (Error Budget Policy), 5 (Alerting on SLOs) -- https://sre.google/workbook/
- **Alex Hidalgo, *Implementing Service Level Objectives*** (O'Reilly, 2020) -- the deep dive into SLI shapes for batch, streaming, ML, and platform-as-a-product workloads.
- **Charity Majors, Liz Fong-Jones, George Miranda, *Observability Engineering*** (O'Reilly, 2022) -- Chapter 22 (Maturity Model) is the org-level framing.
- **Sloth documentation** -- https://sloth.dev/
- **Pyrra GitHub + live demo** -- https://github.com/pyrra-dev/pyrra and https://demo.pyrra.dev/
- **OpenSLO specification** -- https://openslo.com/
- **Grafana multi-window multi-burn-rate blog** -- https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/
- **SoundCloud Engineering: Alerting on SLOs Like Pros** -- https://developers.soundcloud.com/blog/alerting-on-slos/
- Repo cross-references: `2026-04-21-promql-deep-dive.md` (rate, histogram, recording rules), `2026-04-23-alertmanager-alerting-strategy.md` (burn-rate basics, routing tree, HA), `2026-04-26-kubernetes-metrics-stack.md` (PrometheusRule CRD, kube-prometheus-stack), `2026-04-30-unified-observability-correlation.md` (the three-witnesses framing that SLO breaches trigger).
