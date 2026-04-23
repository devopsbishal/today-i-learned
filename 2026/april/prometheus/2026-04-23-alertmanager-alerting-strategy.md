# Alertmanager & Alerting Strategy -- Triage, Routing, and the Discipline of Not Waking People Up

> Yesterday we learned to read the ledger: PromQL turns raw scrape samples into vectors, rates, quantiles, and SLO math. Today we answer the question that matters at 3 AM -- **what happens after a query returns `true`?** A PromQL expression by itself is a silent verdict. It becomes a pager buzz, a Slack message, a ticket, or nothing at all only because a second machine -- Alertmanager -- is sitting downstream of Prometheus deciding who to tell, when, how often, and when to shut up.
>
> The core analogy for the day: **Alertmanager is the triage desk at a hospital emergency room.** Prometheus is the patient-vitals monitor wheeled up beside each bed -- blood pressure, heart rate, oxygen sat, scraped every 15 seconds. Alerting rules are the "if vitals cross this line, raise the flag" conditions the monitor is checking. But the monitor does not know which doctor is on shift tonight, whether the patient in bed 7 is already being worked on for cardiac arrest (don't also page the dermatologist), whether Wing B is closed for cleaning (don't page anyone in there), or whether the ten beds in the same bay all threw the same alarm because the HVAC failed (one page, not ten). The **triage nurse** -- Alertmanager -- sits between the monitor and the on-call roster and does exactly that work: she groups cases so the attending doctor gets one coherent briefing instead of ten fragmented alarms, she silences rooms under maintenance, she inhibits redundant pages when a bigger alarm already covers the smaller ones, and she routes each case to the right specialist based on severity and symptoms. A hospital without a triage nurse is a hospital where every alarm pages every doctor, and by the end of the first shift nobody answers their pager anymore. That is exactly what an un-tuned Alertmanager does to your on-call rotation.
>
> The triage nurse carries a routing cheat-sheet that works like an old telephone switchboard -- labels on the alert determine which "extension" (Slack, PagerDuty, email) the case gets patched through to, and some calls get patched to multiple extensions at once. That cheat-sheet is what Alertmanager calls the **routing tree**. Keep the nurse as the main character; the switchboard is just the tool she's holding.

---

**Date**: 2026-04-23
**Topic Area**: prometheus
**Difficulty**: Intermediate / Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Prometheus alerting rule | A vital-signs monitor's "if heart rate > 180 for 5 min, raise flag" threshold | PromQL `expr` + `for:` duration + labels + annotations; evaluated on schedule |
| `for:` duration | The nurse waits a few minutes to confirm the vital isn't a blip before paging | Time the condition must hold before state transitions from `pending` -> `firing` |
| `keep_firing_for` | The nurse keeps the flag up a minute longer after vitals normalize, in case they dip again | Prometheus v2.42+: delays the `firing` -> `resolved` transition to suppress flap storms |
| Alertmanager | The hospital triage nurse | Separate binary that takes firing alerts and decides who to tell, how, when |
| Alertmanager pipeline | Triage desk checklist: check inhibit list, check silenced rooms, route to specialist, bundle, call | 5 stages: inhibit -> silence -> route -> group/dedupe -> notify |
| Route | Switchboard operator's decision flowchart | Tree of label matchers that picks the receiver for each alert |
| `matchers:` (modern) | The operator's decision criteria on the sheet | v0.22+ syntax: `[severity="critical", team="platform"]`; replaces deprecated `match`/`match_re` |
| `group_by` | Bundling the whole bay's alarms into one call to the attending | Labels whose distinct combinations define one notification bundle |
| `group_wait` | "Give me 30 seconds in case siblings trickle in before I call" | Delay before first notification for a new group |
| `group_interval` | "After the initial call, wait 5 min before adding new siblings in a follow-up" | Minimum time between updates to the same group when NEW alerts join |
| `repeat_interval` | "If the alarm is still going, re-page every 4 hours so nobody forgets" | How often to re-notify for alerts that are still firing |
| `continue: true` | "After telling this extension, keep trying the rest of the list too" | Traversal does NOT stop at first match; enables multi-receiver fan-out |
| Inhibition rule | Cardiac arrest alarm silences the dermatology alarm for the same patient | Declarative, config-driven: one alert's presence automatically suppresses others |
| Silence | "Wing B is closed for cleaning tonight, ignore alarms from there until 8 AM" | Imperative, time-bound, human-created suppression via UI/API/`amtool` |
| Receiver | The extension the call is finally patched to | Slack / PagerDuty / OpsGenie / email / webhook endpoint config |
| Severity label | The tag pinned on the chart: red / yellow / green | Convention: `critical` (page) / `warning` (ticket or Slack) / `info` (log only) |
| `runbook_url` annotation | The laminated procedure card taped to the bed rail | Link in the alert's annotation pointing to the "what do I do about this" doc |
| Error budget | Monthly allowance of downtime you can spend before it's a contract violation | `1 - SLO` fraction of requests you're allowed to fail |
| Burn rate | How fast you're eating the monthly error budget | Current error rate / acceptable error rate; `>14.4` = burning a month's budget in 2 days |
| Multi-window multi-burn-rate | Two nurses independently confirming a pattern before pulling the fire alarm | Short + long windows AND-ed together; reduces false positives from transient spikes |
| nflog (notification log) | Triage desk's shared paper log of "who we already called" | Gossip-replicated record of sent notifications; what makes HA Alertmanager dedupe work |
| HA Alertmanager | Three triage nurses at the desk who gossip with each other | 3-replica cluster on `:9094` gossip port; all Prometheus replicas fan out to all AMs |
| `PrometheusRule` CRD | Shipping the vital-signs-monitor's thresholds as declarative Kubernetes YAML | Namespace-scoped rule object, selected by the `Prometheus` CRD's `ruleSelector` |
| `AlertmanagerConfig` CRD | Each hospital ward owns its own triage sub-flowchart | Prometheus Operator v0.43+: namespace-scoped routing config for multi-tenant clusters |
| Alert fatigue | The boy who cried wolf -- but on-call | When pages lose meaning because too many fire, the actionable ones get ignored |

---

## The Pipeline in One Picture

Before we zoom in, look at the whole flow once. Every alert that Prometheus fires walks this five-station assembly line inside Alertmanager:

```
                                 ALERTMANAGER PIPELINE
   +---------------------------------------------------------------------+
   |                                                                     |
   |  Prometheus                                                         |
   |  (rule engine evaluates                                             |
   |   PromQL every 30s)                                                 |
   |       |                                                             |
   |       | HTTP POST /api/v2/alerts                                    |
   |       | (active alert set, pushed continuously while firing)        |
   |       v                                                             |
   |   +-------+      +--------+      +-------+      +---------+      +----+
   |   |INHIBIT|  ->  |SILENCE |  ->  | ROUTE |  ->  | GROUP + |  ->  |NOT |
   |   |       |      |        |      |       |      | DEDUPE  |      |IFY |
   |   +-------+      +--------+      +-------+      +---------+      +----+
   |      |               |               |               |              |
   |   Drop if a       Drop if a      Walk the        Bundle          Send to
   |   "cover"         matching       routing        siblings          Slack /
   |   alert is        silence is      tree; pick    sharing            PD /
   |   firing          active         receiver      group_by           webhook
   |                                                  labels;          (with
   |                                                 wait for          retry)
   |                                                  group_wait;
   |                                                dedupe via
   |                                                 nflog
   +---------------------------------------------------------------------+
```

**Five things to remember about this pipeline:**

1. **It runs per-alert, continuously.** Prometheus re-POSTs the full set of firing alerts every evaluation interval. Alertmanager is not a queue you drop things into once; it is a rolling set of "these are currently firing, please do the right thing."
2. **The stages are ordered.** Inhibit runs before silence, silence runs before routing. A silenced alert still gets checked against inhibitors first -- doesn't usually matter, but worth knowing.
3. **Grouping is the stage most people get wrong.** Bad `group_by` is the #1 cause of either (a) storms of separate notifications that should have been one, or (b) one giant notification that hides which service is actually broken.
4. **Notification is the only stage that leaves Alertmanager.** Everything else is internal bookkeeping. When a notification is sent, it's logged in the **nflog**, which HA replicas gossip among themselves to avoid duplicate sends.
5. **Resolution is its own mini-pipeline.** When Prometheus stops sending an alert (the PromQL expression no longer matches), Alertmanager waits `resolve_timeout` (default 5 min from the last seen evaluation) then fires a "resolved" notification through the same receiver. The routing/grouping/inhibition all apply again.

---

## Part 1: Alerting Rules -- Where the Conversation Starts

Alertmanager only ever sees alerts that Prometheus has already decided to fire. Those alerts are defined as **alerting rules** -- YAML that lives next to the Prometheus server (or in a `PrometheusRule` CRD on Kubernetes).

### Full Anatomy of a Rule

```yaml
groups:
- name: api.rules
  interval: 30s                 # how often this group is evaluated (defaults to global interval)
  rules:
  - alert: HighAPIErrorRate     # the alert NAME -- becomes label `alertname`; MUST be unique per group
    expr: |                     # the PromQL EXPRESSION -- when it returns a non-empty vector, rule fires
      (
        sum by (service) (rate(http_requests_total{status=~"5..",job="api"}[5m]))
        /
        sum by (service) (rate(http_requests_total{job="api"}[5m]))
      ) > 0.05
    for: 10m                    # condition must hold continuously for 10m before transitioning to firing
    keep_firing_for: 5m         # keep firing 5m AFTER expression stops matching (Prometheus v2.42+)
    labels:                     # labels attached to the outgoing alert (merged on top of expression labels)
      severity: critical
      team: payments
      tier: user-facing
    annotations:                # free-form metadata for humans; NOT used for routing
      summary: "{{ $labels.service }} is returning >5% 5xx errors"
      description: |
        Service {{ $labels.service }} has an error rate of {{ $value | humanizePercentage }}
        for the last 10 minutes. This likely impacts all /api/* traffic.
      runbook_url: https://runbooks.example.com/payments/high-error-rate
      dashboard_url: https://grafana.example.com/d/abc123/payments?var-service={{ $labels.service }}
```

Every field has a specific job:

| Field | Role | Analogy |
|-------|------|---------|
| `alert` | Unique name within the group | The name of the flag on the vital-signs monitor |
| `expr` | The boolean condition (PromQL) | The threshold check on the monitor |
| `for` | Stability delay before firing | "Wait 10 min to confirm before pressing the alarm" |
| `keep_firing_for` | Hysteresis after expr goes false | "Keep the flag up 5 min longer to make sure it stays down" |
| `labels` | Structured routing metadata | The color-coded tags on the chart that triage uses |
| `annotations` | Human-readable context | The handwritten notes on the chart |

### The State Machine -- inactive / pending / firing / resolved

Every alert instance (identified by its label set) walks a four-state machine:

```
                               expr starts
                               matching for
                                this series
                                     |
                                     v
      +-----------+   expr     +-----------+   expr held      +-----------+
      |  INACTIVE | ---------> |  PENDING  | ---------------> |  FIRING   |
      | (no alert)|  matches   | (counting |  for full `for`  | (pushed to |
      |           |            |  down `for`) duration         | Alertmgr) |
      +-----------+            +-----------+                  +-----------+
           ^                         |                              |
           |                         | expr stops                   | expr stops
           |                         | matching                     | matching
           |                         v                              v
           |                    INACTIVE                +--------------------------+
           |                                            | (optional) `keep_firing_ |
           |                                            | for` delay before        |
           |                                            | transitioning            |
           |                                            +-----------+--------------+
           |                                                        |
           |                                                        v
           +--------------------------------------------------------+
                                 (series becomes INACTIVE; Alertmanager eventually
                                  emits `resolved` notification after resolve_timeout)
```

**The most important row in that diagram**: `pending` does NOT page. Nothing leaves Prometheus during `pending`. The `for:` duration is your anti-flap insurance. Set it to zero (`for: 0s` or omit it entirely) and every transient spike becomes a page.

### Templating: `$value` and `$labels`

Inside `annotations` (and `labels`), you can interpolate **Go templates**:

- `{{ $value }}` -- the numeric result of the expression for this series
- `{{ $labels.service }}` -- a label from the series
- `{{ $value | humanize }}` -- format large numbers (`1.2M`, `34.5k`)
- `{{ $value | humanizePercentage }}` -- `0.0523` becomes `5.23%`
- `{{ $value | humanizeDuration }}` -- seconds become `1h 23m`

These run in **Prometheus's template engine** (not Alertmanager's). Alertmanager has its own, separate template engine for receiver payloads that uses `.CommonLabels`, `.Alerts`, etc. Mixing them up is a classic first-week mistake -- see the gotchas section.

### Production Rule Conventions

Every production-grade rule you write should have:

1. **A `severity` label** -- always one of `critical`, `warning`, `info`. This is what the routing tree pivots on.
2. **A `team` (or `owner`) label** -- which team owns this alert. Makes multi-team routing sane.
3. **A `runbook_url` annotation** -- a link to a real doc explaining what the alert means and what to do. If you can't write a runbook, you don't need the alert.
4. **A `summary` annotation** -- one-line description with enough context to be useful in a push notification.
5. **A `description` annotation** -- long form, templated with `$value` and key labels.
6. **A `for:`** -- never zero. Minimum 2-5 minutes for most things, 10-15 minutes for anything subject to normal deploy noise.

---

## Part 2: The Routing Tree -- The Hardest Part, the Part That Matters Most

The routing tree is Alertmanager's core primitive. Everything else (grouping, continuation, inhibition interaction) is defined in terms of the tree. Spending an extra hour on this section now saves a dozen "why did this page the wrong team" incidents later.

### The Tree Has Exactly One Root

```yaml
route:                                 # THE root -- mandatory, unique
  receiver: default-null               # fallback receiver if nothing nested matches
  group_by: [alertname, cluster]       # default grouping for every child unless overridden
  group_wait: 30s                      # 30s default wait for siblings
  group_interval: 5m
  repeat_interval: 4h
  routes:                              # children -- tried in order
    - matchers: [...]
      receiver: ...
```

The root has a receiver that acts as a catch-all. If no nested route matches, the alert lands here. **Always** set the root receiver to something non-null and at least loud enough that you'll notice an alert falling through routing misconfiguration -- a "devops-catchall" Slack channel is the minimum.

### Matchers: the Modern Syntax

Alertmanager v0.22+ introduced a cleaner matcher syntax that replaces the legacy `match` (exact) and `match_re` (regex) fields:

```yaml
# MODERN (v0.22+) -- use this
matchers:
  - severity="critical"
  - team="platform"
  - alertname=~"High.*ErrorRate"       # regex with =~
  - environment!~"staging|dev"         # negation with !~

# LEGACY -- still works, but don't write new configs this way
match:
  severity: critical
  team: platform
match_re:
  alertname: High.*ErrorRate
```

The modern form supports `=`, `!=`, `=~`, `!~` in one unified list -- same operators as PromQL label matchers. All matchers in the list are **AND-ed**. There is no OR in a single route; use separate routes or regex alternation.

### `group_by` -- Deciding What's One Notification

`group_by` controls how many **notifications** a burst of related alerts produces. The rule is: alerts with the same values for all the labels in `group_by` become one bundled notification.

Think of `group_by` as **GROUP BY in SQL**, applied to the incoming stream of firing alerts:

```
Incoming alerts (10 firing):                   group_by: [alertname, cluster]

  PodCrashLoop {cluster=prod, pod=api-1}       PodCrashLoop / prod         <- 5 pods bundled
  PodCrashLoop {cluster=prod, pod=api-2}                                       into 1 notification
  PodCrashLoop {cluster=prod, pod=api-3}                                       listing all 5
  PodCrashLoop {cluster=prod, pod=api-4}
  PodCrashLoop {cluster=prod, pod=api-5}
  PodCrashLoop {cluster=staging, pod=web-1}    PodCrashLoop / staging      <- 1 alert
  NodeDown     {cluster=prod, node=n-7}        NodeDown / prod             <- 1 alert
  NodeDown     {cluster=prod, node=n-8}                                       (2 nodes bundled)
  HighLatency  {cluster=prod, service=api}     HighLatency / prod          <- 1 alert
  HighLatency  {cluster=eu, service=api}       HighLatency / eu            <- 1 alert

   10 firing alerts -> 5 distinct group keys -> 5 notifications
```

| `group_by` choice | Effect | When to use |
|--------------|--------|-------------|
| `[alertname]` | One notification per alert *type*, regardless of cluster/service/pod | Rare; loses cluster context |
| `[alertname, cluster, service]` | **Sensible default**: one notification per (alert, cluster, service) combo | Most use cases |
| `[alertname, cluster, namespace]` | Per-namespace scoping | Multi-tenant clusters |
| `[...]` (literal ellipsis) | One notification per unique full label set -- basically no grouping | Almost never -- causes storms |
| `[]` (empty) | One giant notification for EVERYTHING currently firing | Never |

**The rule of thumb**: include enough labels that each group has a clear "owner" reading the page, but not so many that you split alerts that are obviously the same incident.

### `group_wait`, `group_interval`, `repeat_interval` -- the Three Timers

These three timers confuse everyone on first read. Get them straight once and you'll use them for life:

| Timer | Default | Controls | Analogy |
|-------|---------|----------|---------|
| `group_wait` | 30s | Time to wait **before the first notification** for a new group, hoping siblings arrive | "Hold on a second, see if more alarms from the same bay ring in" |
| `group_interval` | 5m | Minimum time between notifications for the same group **when new alerts join** | "After the first briefing, wait 5 min before I'll add newcomers into an update" |
| `repeat_interval` | 4h | How often to **re-send** the same group if nothing has changed | "If the alarm is still going, re-remind the doctor every 4 hours" |

Timeline for a single group:

```
t=0         first alert in new group arrives
            |
            +-- group_wait (30s) --+
                                   |
t=0:30                  FIRST NOTIFICATION (includes all siblings that arrived in that 30s)
            |
            |
            +-- group_interval (5m) --+
                                      |
t=5:30      if more new alerts joined during that 5m, NOTIFICATION UPDATE sent
            (if nothing new joined, nothing is sent here)
            |
            |
            +-- repeat_interval (4h) --+
                                       |
t=4h        even if nothing changed, same group is RE-SENT (anti-forget reminder)
```

**Key subtlety**: `repeat_interval` is a floor, not a ceiling. Its minimum allowed value is `group_interval` -- if you configure `repeat_interval: 1m` and `group_interval: 5m`, Alertmanager rounds up to 5m silently. Don't try to outsmart this.

**Another subtlety worth internalizing**: `group_interval` is NOT a fixed scheduled "tick" -- it's a **floor on the time between notifications for a group**. Alertmanager evaluates "can I send an update?" whenever the alert set changes. If a change happens 10s after the last notification, it waits until `group_interval` has elapsed before sending. If a change happens 6m after the last notification (and `group_interval` is 5m), the update fires immediately because the floor is already cleared. The mechanism is "wait until `group_interval` has elapsed since last notification, then send any pending changes," not a scheduled timer that ticks every 5 minutes.

### `continue: true` -- Fan-out to Multiple Receivers

By default, Alertmanager **stops at the first matching route**. That is often what you want ("critical alerts go to PagerDuty, stop there"). But sometimes you want an alert to hit two places at once -- e.g., critical goes to PagerDuty AND to a shared `#incidents` Slack channel. That's what `continue: true` is for.

```yaml
route:
  receiver: default-slack
  routes:
    - matchers: [severity="critical"]
      receiver: pagerduty
      continue: true              # after PagerDuty, keep walking the tree
    - matchers: [severity="critical"]
      receiver: incidents-slack   # gets reached because of continue above
    - matchers: [severity="warning"]
      receiver: warnings-slack
```

Without `continue: true`, the first `critical` match would stop the traversal and the Slack channel would never hear about it. Think of `continue: true` as the switchboard operator saying "after connecting the PagerDuty call, also patch it through to the Slack line."

### Nested Routes Inherit from Parents

Everything a parent route declares (`group_by`, `group_wait`, `group_interval`, `repeat_interval`) is inherited by child routes unless explicitly overridden. This keeps configs DRY:

```yaml
route:
  group_by: [alertname, cluster]       # inherited everywhere below
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: default
  routes:
    - matchers: [severity="warning"]
      receiver: team-slack
      group_wait: 2m                  # override ONLY for warnings -- less urgent
      repeat_interval: 12h            # override: re-nag every 12h, not 4h
      # group_by and group_interval still inherited
```

**Common inheritance mistake**: a nested route that specifies `group_by` **replaces** (not merges with) the parent's `group_by`. If you want to add a label to the parent's group_by for this subtree, you must rewrite the full list.

### A Full, Realistic Multi-Team Routing Tree

```yaml
route:
  receiver: default-null-catchall          # fallback -- never lose an alert
  group_by: [alertname, cluster, service]  # default bundling inherited by all children
  group_wait: 30s                          # first notification delay
  group_interval: 5m                       # updates when new siblings join
  repeat_interval: 4h                      # re-page reminder for still-firing

  routes:
  # ---- Platform team (infra-level criticals) ----
  - matchers:
      - severity="critical"
      - team="platform"
    receiver: platform-pagerduty
    continue: true                         # also send to the shared incident channel
  - matchers:
      - severity="critical"
      - team="platform"
    receiver: incidents-slack

  # ---- Other critical alerts (catch-all page) ----
  - matchers: [severity="critical"]
    receiver: oncall-pagerduty             # whoever's primary rotation for the cluster

  # ---- Per-team warnings ----
  - matchers: [severity="warning", team="payments"]
    receiver: payments-slack
    group_wait: 2m                         # warnings less urgent, longer bundling window
    repeat_interval: 12h                   # don't re-nag every 4h for non-pages
  - matchers: [severity="warning", team="platform"]
    receiver: platform-slack
    group_wait: 2m
    repeat_interval: 12h

  # ---- Info alerts go to a digest channel (not individual pings) ----
  - matchers: [severity="info"]
    receiver: slack-info-digest
    group_wait: 5m                         # batch aggressively
    group_interval: 1h                     # drip-feed updates
    repeat_interval: 24h                   # once a day is plenty
```

Walk through it: the **root** defines sensible defaults. **Platform criticals** are a special case -- they page AND hit the incident channel (the `continue: true` trick). **All other criticals** fall through to a generic on-call rotation. **Warnings** go per-team to Slack with longer timers. **Infos** get aggressively batched into a single digest channel because the goal is signal, not paging.

### Routing Tree Visual Editor Workflow

The official tool at `prometheus.io/webtools/alerting/routing-tree-editor/` renders any routing config as a tree diagram. Paste your config, then feed it test alerts by providing label sets -- it shows which receiver each would hit and the effective inherited timers at each node.

**Your development loop should be**: write config locally -> `amtool check-config alertmanager.yml` (syntax) -> paste into routing-tree editor (semantics) -> test with `amtool alert add ...` (end-to-end) -> ship.

---

## Part 3: Inhibition Rules -- Automatic Suppression

Inhibition is the **cardiac-arrest-silences-the-dermatologist** principle, written as config. When a bigger alarm is firing, the smaller alarms it implies should not also page.

### The Three Fields

```yaml
inhibit_rules:
- source_matchers:                # alerts that TRIGGER inhibition (the cardiac arrest)
    - alertname="NodeDown"
  target_matchers:                # alerts that GET SUPPRESSED (the pod-level alerts)
    - alertname=~"KubePod.*|KubeletDown"
  equal: [cluster, instance]      # ONLY suppress target alerts with matching cluster AND instance
```

**All three fields matter**:

- `source_matchers`: labels identifying the "covering" alerts. If ANY alert matching all these matchers is firing, inhibition is active.
- `target_matchers`: labels identifying what to suppress. Alerts matching all these matchers are dropped for the duration of the source being firing.
- `equal`: safety guard -- a source alert only suppresses a target alert when all labels in this list have the same values in both. If you omit `equal`, a single `NodeDown` on any node would suppress every `KubePodCrashLooping` cluster-wide.

### The Canonical Examples

**Node down suppresses all pod alerts on that node**:
```yaml
- source_matchers: [alertname="KubeNodeNotReady"]
  target_matchers: [alertname=~"KubePodCrashLooping|KubePodNotReady|KubeletDown"]
  equal: [cluster, instance]      # same cluster AND same node
```

**Cluster-wide outage suppresses per-service alerts**:
```yaml
- source_matchers: [alertname="ClusterAPIUnreachable"]
  target_matchers: [alertname=~"KubeDeploymentReplicasMismatch|KubeStatefulSetReplicasMismatch"]
  equal: [cluster]                # same cluster (any service in it)
```

**Critical silences its warning sibling**:
```yaml
- source_matchers: [severity="critical"]
  target_matchers: [severity="warning"]
  equal: [alertname, cluster, service]   # same alert type on same service
```
The last one is particularly useful when you have paired rules (`HighErrorRateCritical` at 5% and `HighErrorRateWarning` at 2%) -- when the critical fires, the warning is obviously also firing; suppressing it prevents duplicate pages.

### Inhibition vs Silence -- Don't Conflate Them

| Dimension | Inhibition | Silence |
|-----------|------------|---------|
| Defined by | YAML config (code-reviewed, versioned) | UI / API / `amtool` (imperative, logged) |
| Lifetime | Permanent, condition-driven | Time-bound (has an explicit expiry) |
| Trigger | Another alert being firing | Human decision |
| Use case | "X implies Y is also bad, don't duplicate" | "I'm deploying / doing maintenance, ignore noise" |
| Scope | Label-matched (automatic) | Label-matched (manual choice) |
| Audit trail | Git history of the config | Alertmanager silence log |

In one sentence: **inhibition is declarative (code); silence is imperative (ops)**. Never mix them up in a design review.

---

## Part 4: Silences -- Temporary Shut-Up Orders

A silence is a human saying "I know about this, stop telling me for the next N hours." Unlike inhibitions, silences have an explicit expiry.

### Creating Silences

**Via CLI (preferred -- leaves an audit trail)**:
```bash
amtool silence add \
  alertname="HighErrorRate" \
  service="checkout" \
  --duration=4h \
  --comment="known issue, fix shipping 18:00 UTC, owner: bishal" \
  --author="bishal@example.com"
```

**Via UI**: `/#/silences/new` in the Alertmanager web UI. Comment is mandatory in most production configs.

**Via API** (for automation in deploy pipelines -- e.g., auto-silence during canary rollout):
```bash
curl -X POST http://alertmanager:9093/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [
      {"name": "service", "value": "checkout", "isRegex": false}
    ],
    "startsAt": "2026-04-23T18:00:00Z",
    "endsAt":   "2026-04-23T20:00:00Z",
    "createdBy": "deploy-bot",
    "comment":   "canary silence for release v2.45.0"
  }'
```

### Listing and Expiring Silences

```bash
amtool silence query                          # list all active silences
amtool silence query service="checkout"      # filter by matcher
amtool silence expire <SILENCE_ID>            # cancel early
```

### The "Silence That Outlived the Incident" Anti-Pattern

The most common silence mistake is creating a silence with a long duration, fixing the issue, and **forgetting to expire it**. Real alerts for the same problem come in later and get silently dropped. Countermeasures:

- **Default durations short.** Culture: silences are max 4h unless explicitly justified. If you need longer, you need a ticket.
- **Mandatory `--comment`.** Force the creator to write WHY. "temp" as a comment is cause for a code review conversation.
- **Weekly silence review.** Cron a job that lists silences older than 24h into a Slack channel. Stale silences stand out.
- **Narrow matchers.** A silence with just `alertname="HighErrorRate"` silences the alert for every service, every cluster. Always add scoping labels.

---

## Part 5: Receivers -- Where Notifications Actually Go

A **receiver** is a named block that configures how to send notifications. Receivers are referenced by name from routes.

### Slack

```yaml
receivers:
- name: payments-slack
  slack_configs:
  - api_url: https://hooks.slack.com/services/TXXXX/BXXXX/YYYYYYY
    channel: '#payments-alerts'
    send_resolved: true                    # also send "resolved" notifications
    title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}'
    text: |
      {{ range .Alerts }}
      *Service:* {{ .Labels.service }}
      *Severity:* {{ .Labels.severity }}
      *Summary:* {{ .Annotations.summary }}
      *Description:* {{ .Annotations.description }}
      *Runbook:* {{ .Annotations.runbook_url }}
      {{ end }}
    color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

Key points:
- `api_url` is the webhook; treat it as a secret.
- `send_resolved: true` is almost always what you want -- tells you when the alert stops.
- Templates use Alertmanager's template engine (`.Status`, `.Alerts`, `.CommonLabels`, `.GroupLabels`), NOT the Prometheus rule engine's `$labels`/`$value`.
- Modern Slack supports **Block Kit** via `slack_configs.blocks`, which renders cleaner than the legacy `attachments`. Worth migrating.

### PagerDuty

```yaml
receivers:
- name: oncall-pagerduty
  pagerduty_configs:
  - routing_key: 'R01XXXXXXXXXXXXXX'     # Events API v2 integration key
    severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
    class: '{{ .CommonLabels.alertname }}'
    component: '{{ .CommonLabels.service }}'
    group: '{{ .CommonLabels.cluster }}'
    description: '{{ .CommonAnnotations.summary }}'
    details:
      runbook: '{{ .CommonAnnotations.runbook_url }}'
      dashboard: '{{ .CommonAnnotations.dashboard_url }}'
```

**PagerDuty gotcha -- `dedup_key`**: by default, Alertmanager uses a hash of the group labels as the dedup_key. If your grouping is too granular, PagerDuty creates a new incident per distinct group key, meaning one outage creates twenty PagerDuty incidents. Make sure your `group_by` rolls alerts up to a sensible dedup granularity (usually per-service or per-cluster, not per-pod).

### OpsGenie

```yaml
- name: oncall-opsgenie
  opsgenie_configs:
  - api_key: 'xxxxxxxx'
    priority: '{{ if eq .CommonLabels.severity "critical" }}P1{{ else }}P3{{ end }}'
    tags: '{{ .CommonLabels.team }},{{ .CommonLabels.cluster }}'
```

### Generic Webhook (Jira, Teams, custom)

```yaml
- name: jira-webhook
  webhook_configs:
  - url: https://internal.example.com/alertmanager-jira-bridge
    send_resolved: true
    http_config:
      bearer_token_file: /etc/alertmanager/secrets/jira-token
    max_alerts: 50        # limit payload size
```

Alertmanager POSTs a JSON body matching the [webhook API schema](https://prometheus.io/docs/alerting/latest/configuration/#webhook_config). The receiver responds with a 2xx for success; 5xx triggers retry with exponential backoff. 4xx is treated as a permanent failure (no retry).

### Email

```yaml
- name: ops-email
  email_configs:
  - to: oncall@example.com
    from: alertmanager@example.com
    smarthost: smtp.sendgrid.net:587
    auth_username: apikey
    auth_password_file: /etc/alertmanager/secrets/sendgrid-key
    require_tls: true
```

Common gotchas: SMTP TLS negotiation (`require_tls` vs implicit TLS on port 465 vs STARTTLS on 587), and `from:` addresses that the SMTP provider doesn't authorize.

### Severity-to-Receiver Matrix (Decision Framework)

| Severity | Primary | Secondary | Tertiary | Rationale |
|----------|---------|-----------|----------|-----------|
| `critical` | PagerDuty (on-call) | Slack `#incidents` | -- | Must wake someone; visibility in shared channel |
| `warning` | Slack (team channel) | Jira ticket (via webhook, optional) | -- | Actionable but not urgent; don't page |
| `info` | Slack `#alerts-digest` (batched) | -- | -- | FYI only; hides in dedicated low-noise channel |

Never page on `warning`. If a warning is actually worth waking someone for, re-label it `critical`.

---

## Part 6: SLO-Based Burn-Rate Alerting -- The Senior-Engineer Section

Back to the triage nurse for a moment: a thermometer reading `39.5 C` is not the same story as a patient's temperature rising `0.5 C per hour`. The first is a snapshot; the second is a **rate of change** that tells you whether you've got minutes or hours to intervene. Threshold alerts are snapshots. Burn-rate alerts measure the rate at which a service is using up its allowed downtime, and give you the "minutes vs hours" signal the nurse actually needs.

Threshold alerts (`errors per second > 500`) are the first thing you learn and the thing you should outgrow fastest. They're broken for two reasons:

1. **They don't scale with traffic.** 500 errors/sec is trivial at peak load and catastrophic at 3 AM. The alert fires either way.
2. **They're not tied to user impact.** Paging on "errors > X" tells you a symptom; paging on "we're eating our error budget too fast" tells you **whether users are getting hurt right now, proportionally to how much they notice**.

The fix is **burn-rate alerting based on SLOs**, from the Google SRE Workbook Chapter 5.

### The Math in One Table

An SLO like **99.9% of requests succeed over 30 days** implies:

- **Error budget** = `1 - 0.999` = `0.001` = `0.1%` of all requests are allowed to fail
- In 30 days = 43.2 minutes of full-outage equivalent
- **Burn rate** = current error rate / acceptable error rate. A burn rate of 1 means you'll use exactly 100% of your budget in 30 days. A burn rate of 14.4 means you'll use 100% of it in... `30 days / 14.4 = ~50 hours = ~2 days`.

The canonical table (SRE Workbook, Ch. 5, Table 5-8) has **three rows** -- two pages and a ticket. Each row pairs a **long window** (for the primary burn confirmation) with a **short window** (for "is the burn still happening *right now*?"):

| Burn rate | Long window | Short window | % budget consumed | Alert severity |
|-----------|-------------|--------------|-------------------|----------------|
| 14.4 | 1 hour | 5 minutes | 2% | Page (fast burn) |
| 6 | 6 hours | 30 minutes | 5% | Page (slow burn) |
| 1 | 3 days | 6 hours | 10% | Ticket |

Translation: if you're burning at 14.4x the acceptable rate sustained over an hour (AND still burning in the last 5 minutes), you've eaten 2% of your month's budget in one hour. At that rate the month's budget is gone in ~2 days. Page now. The slower 6x/6h confirms a sustained but not catastrophic burn -- still a page, but you've got hours, not minutes. The 1x/3d burn is the "nobody's dying today but this will hurt by month-end" ticket.

### Why Multi-Window Multi-Burn-Rate

A single-window burn-rate alert (say, 1h @ 14.4) has a problem: a 5-minute spike can trip the 1h average and fire a false positive. The fix is to **require two windows to both show high burn simultaneously**:

```promql
(
  burn_rate_1h > 14.4
  and
  burn_rate_5m > 14.4
)
```

The short window (5m) confirms the burn is *right now*, not historical. The long window (1h) confirms it's sustained. AND-ed together, transient spikes don't fire the alert.

### Concrete PromQL: Burn Rate

```promql
# Error ratio over 5m
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)
```

For an SLO of 99.9% (acceptable error ratio = 0.001), the burn rate is just `error_ratio / 0.001`. But rather than hardcoding `0.001`, encode it as a **recording rule**:

```yaml
groups:
- name: slo.recording
  interval: 30s
  rules:
  - record: service:slo_target
    expr: 0.999
    labels: {service: api}
  - record: service:error_ratio_5m
    expr: |
      sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
      /
      sum by (service) (rate(http_requests_total[5m]))
  - record: service:error_ratio_30m
    expr: |
      sum by (service) (rate(http_requests_total{status=~"5.."}[30m]))
      /
      sum by (service) (rate(http_requests_total[30m]))
  - record: service:error_ratio_1h
    expr: |
      sum by (service) (rate(http_requests_total{status=~"5.."}[1h]))
      /
      sum by (service) (rate(http_requests_total[1h]))
  - record: service:error_ratio_6h
    expr: |
      sum by (service) (rate(http_requests_total{status=~"5.."}[6h]))
      /
      sum by (service) (rate(http_requests_total[6h]))
  # Traffic-volume gate: used in alerts to suppress false positives on low-volume endpoints
  - record: service:request_rate_5m
    expr: sum by (service) (rate(http_requests_total[5m]))
```

All four burn-rate windows plus a traffic-rate recording rule (needed for the low-traffic-denominator gate -- covered in the gotchas section). The pattern generalizes: for every burn-rate alert you want to write, you need a recording rule at both its short-window and long-window durations.

Then the alerts:

```yaml
- name: slo.alerts
  rules:
  # Fast burn: 14.4x for 2% budget in 1h -- PAGE
  - alert: SLOFastBurnRate
    expr: |
      (
        service:error_ratio_5m / on(service) (1 - service:slo_target) > 14.4
        and
        service:error_ratio_1h / on(service) (1 - service:slo_target) > 14.4
      )
    for: 2m
    labels:
      severity: critical
      slo_type: fast_burn
    annotations:
      summary: "{{ $labels.service }} burning SLO budget at 14.4x"
      description: |
        Service {{ $labels.service }} has burned ~2% of its monthly error budget
        in the last hour (5m and 1h both confirming). At this rate, the full
        month's budget exhausts in ~2 days.
      runbook_url: https://runbooks.example.com/slo-fast-burn

  # Slow burn: 6x for 5% budget in 6h -- PAGE (but less urgent)
  - alert: SLOSlowBurnRate
    expr: |
      (
        service:error_ratio_30m / on(service) (1 - service:slo_target) > 6
        and
        service:error_ratio_6h  / on(service) (1 - service:slo_target) > 6
      )
    for: 15m
    labels:
      severity: critical
      slo_type: slow_burn
    annotations:
      summary: "{{ $labels.service }} sustained SLO burn at 6x for 6h"
      description: |
        Service {{ $labels.service }} has burned ~5% of its monthly error budget
        in the last 6 hours (30m and 6h both confirming). Sustained but not
        catastrophic -- fix within hours, not minutes.
      runbook_url: https://runbooks.example.com/slo-slow-burn
```

### The Low-Traffic Denominator Trap (the #1 burn-rate gotcha)

Burn-rate math is a **ratio**, and ratios on small denominators are unstable. If your endpoint gets 10 requests/minute and one of them errors, the error ratio is 10% -- which at a 99.9% SLO is a burn rate of `0.10 / 0.001 = 100`. The 14.4 threshold trips, the 5m short window agrees (still high), and you page on-call for what was effectively one bad request.

Translate it to the triage nurse: she's measuring "percentage of patients with abnormal vitals in the last hour." When the ER is slow and only one patient has walked in, a single abnormal reading becomes "100% of patients are dying." She'd be a fool to call the code team for that.

The fix is a **traffic-volume gate**: require the denominator to exceed a floor before the alert is allowed to fire. The cleanest form uses an `and` with the recording rule you saw above:

```promql
(
  service:error_ratio_5m / on(service) (1 - service:slo_target) > 14.4
  and
  service:error_ratio_1h / on(service) (1 - service:slo_target) > 14.4
  and
  service:request_rate_5m > 0.5    # at least 30 req/min sustained over 5m
)
```

The magic number (`0.5 req/s` here) depends on your service. The rigorous way to pick it:

- Burn-rate threshold as an error ratio: `burn_rate × (1 - SLO)`. For 14.4x at 99.9% = `14.4 × 0.001 = 1.44%`.
- A single error should NOT be able to clear that ratio on its own: `1/N ≤ 0.0144` → `N ≥ ~70`.
- So the 5m-window denominator floor should be at least 70 requests. At 5m = 300s, that's `70 / 300 ≈ 0.23 req/s`. Round up to `0.5 req/s` (= 150 req / 5m) for a bit of safety margin.

Derive this number from first principles every time you set an SLO on a new service -- don't copy-paste `0.5` and hope. Services with 99.99% SLOs have a `(1 - SLO)` of 0.0001, so the math changes significantly: `1/N ≤ 14.4 × 0.0001 = 0.00144` → `N ≥ 695`. A 99.99% SLO service needs ~10x the traffic floor of a 99.9% service.

**Alternative gating strategy -- absolute error count**: gate on the numerator rather than the denominator, e.g. `and increase(errors_total[5m]) >= 3`. Says "I only care about burn rate if at least 3 errors happened in the last 5 minutes, regardless of traffic." Simpler to reason about, and often a better fit for services where you genuinely don't care about *ratio* below a small absolute number of failures.

**Alternative**: gate on absolute error count rather than rate, e.g., `sum(increase(errors_total[5m])) > 5` AND-ed with the burn rate. Either works; pick one and apply consistently.

Every production burn-rate alert you write needs this gate. It is the single most common omission that turns a good SLO alert into a pager-generator nobody trusts.

### Why Burn-Rate Beats Threshold

| Dimension | Threshold alert | Burn-rate alert |
|-----------|-----------------|-----------------|
| Scales with traffic | No | Yes (ratio, not absolute) |
| Tied to user impact | Indirectly | Directly (error budget = user pain) |
| False positive rate | High (transient spikes) | Low (multi-window AND) |
| Severity tiers | Ad-hoc | Natural (fast/slow burn) |
| Explains itself on-call | "errors > 500, now what?" | "budget at X%, slow enough to wait / fast enough to page" |

If you take one habit away from this doc, let it be: **write your important alerts as burn-rate alerts on documented SLOs, and let thresholds die off naturally**.

### Decision Framework: Threshold vs Burn Rate

| Situation | Use threshold | Use burn rate |
|-----------|---------------|---------------|
| Hard-limit resource (disk full, memory OOM) | YES | No -- it's not about user ratio |
| Binary health (service down, pod crash looping) | YES | No -- the "error rate" is 100% |
| Request error rate / latency of a user-facing API | No | **YES** |
| Background-job queue depth | YES (absolute numbers) | No |
| Cert expiring in N days | YES | No |
| Anything you SLO-ed | No | **YES** |

The rule of thumb: **if it has an SLO, alert on the SLO; if it's a hard limit or a binary state, threshold is fine.**

---

## Part 7: Kubernetes Integration -- PrometheusRule and AlertmanagerConfig CRDs

The Prometheus Operator turns everything above into Kubernetes objects. Instead of mounting files into Prometheus pods, you create CRDs and the operator handles the reload.

### PrometheusRule CRD

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payments-alerts
  namespace: payments               # namespace-scoped
  labels:
    release: kube-prometheus-stack  # must match the Prometheus CRD's ruleSelector
    team: payments
spec:
  groups:
  - name: payments.rules
    interval: 30s
    rules:
    - alert: PaymentsHighErrorRate
      expr: |
        sum by (service) (rate(http_requests_total{status=~"5..",namespace="payments"}[5m]))
        / sum by (service) (rate(http_requests_total{namespace="payments"}[5m]))
        > 0.02
      for: 10m
      labels:
        severity: critical
        team: payments
      annotations:
        summary: "{{ $labels.service }} error rate > 2%"
        runbook_url: https://runbooks.example.com/payments-errors
```

The `Prometheus` CRD's `ruleSelector` decides which `PrometheusRule` objects get loaded:

```yaml
kind: Prometheus
spec:
  ruleSelector:                     # pick up any PrometheusRule with label release=kube-prometheus-stack
    matchLabels:
      release: kube-prometheus-stack
  ruleNamespaceSelector: {}         # ...from any namespace
```

**Subtle but important**: `ruleSelector: {}` (empty object) matches ALL `PrometheusRule` objects; `ruleSelector:` with null matches NOTHING. Same for `ruleNamespaceSelector`. Getting this wrong is a surprisingly common "my rules aren't loading" root cause.

### AlertmanagerConfig CRD (Prometheus Operator v0.43+)

This is the multi-tenant superpower. Before `AlertmanagerConfig`, every team wanting a route or receiver had to PR into a central `alertmanager.yml`. After it, each team owns their routing config in their own namespace. The CRD itself landed in **v0.43.0** (Oct 2020); inhibit rules in `AlertmanagerConfig` came in **v0.52.0** (Nov 2021); and `alertmanagerConfigMatcherStrategy` (see below) was added in **v0.55.0** (Mar 2022).

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: payments-routing
  namespace: payments
  labels:
    alertmanagerConfig: payments
spec:
  route:
    receiver: payments-slack
    groupBy: [alertname, cluster, service]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    matchers:
      - name: team
        value: payments
        matchType: "="
    routes:
      - receiver: payments-pagerduty
        matchers:
          - name: severity
            value: critical
            matchType: "="
  receivers:
    - name: payments-slack
      slackConfigs:
        - apiURL:
            name: payments-slack-webhook    # k8s Secret name
            key: url
          channel: '#payments-alerts'
          sendResolved: true
    - name: payments-pagerduty
      pagerdutyConfigs:
        - routingKey:
            name: payments-pagerduty-key
            key: key
```

### alertmanagerConfigMatcherStrategy -- the Multi-Tenant Safety Net

The operator auto-adds a `namespace` matcher to every route in an `AlertmanagerConfig`. So the payments team's routes only match alerts with `namespace="payments"`. This prevents one team from accidentally routing (or stealing) alerts from another team. The **field** `alertmanagerConfigMatcherStrategy` was introduced in **operator v0.55.0** specifically to let you *opt out* of this injection (set `type: None`) -- the injection itself was the pre-existing behavior and remains the default.

**The silent failure mode worth knowing cold**: the injected matcher is **invisible in the YAML the team authored**. If the payments team's `AlertmanagerConfig` lives in the `payments` namespace but their service runs in a shared `api` namespace (a `PrometheusRule` in `api` carries `namespace="api"` on its alerts), the operator-compiled route effectively becomes `namespace="payments" AND team="payments"`, which the alert fails because `namespace="api"`. The whole route is skipped, the alert falls through to the global catchall, and the team debugs the matchers they *wrote* rather than the matchers they *got*. Options when this bites you: (a) disable `OnNamespace` cluster-wide (`type: None`) and rely on teams writing scoped matchers with discipline + platform-reviewed CI lint, (b) move the `PrometheusRule` into the team's namespace, (c) co-locate the `AlertmanagerConfig` with the rule. The first trades safety for flexibility; the second/third trade flexibility for safety. Pick based on whether your platform team can review every team's matchers or not. A common pragmatic compromise: keep `OnNamespace` on, enforce via CI that every team's root route includes explicit `team=` and `service=` matchers, and set the global catchall receiver to ping the platform channel so nothing is silently dropped.

```yaml
kind: Alertmanager
spec:
  alertmanagerConfigSelector:
    matchLabels:
      alertmanagerConfig: payments
  alertmanagerConfigNamespaceSelector: {}
  alertmanagerConfigMatcherStrategy:
    type: OnNamespace     # DEFAULT -- injects namespace matcher automatically
    # Other option: None -- disables the injection (rarely needed; explicitly lets one team route across namespaces)
```

### Built-in Rules via kube-prometheus-stack

The `kube-prometheus-stack` chart ships ~100 default rules (`KubePodCrashLooping`, `KubeDeploymentReplicasMismatch`, `KubeletDown`, `NodeFilesystemSpaceFillingUp`, etc.). You customize them via Helm values:

```yaml
# values.yaml
defaultRules:
  create: true
  disabled:
    - KubeMemoryOvercommit       # too noisy on our clusters
    - KubeCPUOvercommit

# Layer custom rules on top without forking the chart
additionalPrometheusRulesMap:
  custom-slos:
    groups:
      - name: slo.alerts
        rules:
          - alert: SLOFastBurnRate
            expr: ...
            for: 2m
            labels:
              severity: critical
```

### Decision Framework: PrometheusRule vs AlertmanagerConfig

| Concern | PrometheusRule | AlertmanagerConfig |
|---------|----------------|---------------------|
| What it defines | Alert conditions (PromQL) | Where alerts GO (routes + receivers) |
| Scope | Which metrics, which threshold | Which team's Slack, which PD integration |
| Owned by | Whoever writes the query (often the service team) | Team owning the receiver (often also the service team, sometimes platform) |
| Cluster-admin concern | Only the `ruleSelector` on `Prometheus` CRD | The `alertmanagerConfigSelector` and matcher strategy on `Alertmanager` CRD |
| Multi-tenant safety | Labels-based convention | Namespace matcher auto-injection |

In a healthy multi-tenant cluster: **platform owns the `Alertmanager` and `Prometheus` CRDs; every team owns their `PrometheusRule` and `AlertmanagerConfig` in their namespace.**

---

## Part 8: High-Availability Alertmanager

Picture the triage desk again, but now with **three nurses** sharing it during a shift. Each nurse hears every monitor beep independently (the Prometheus replicas fan out to all three). Between them they share one paper log -- "who we already called, which doctor we already paged" -- and they glance at it before picking up the phone so they don't all call the same specialist for the same patient. That shared paper log is the **nflog**, gossip-replicated across the replicas. If one nurse steps out for a coffee, the other two keep the desk running; if network gossip breaks and they can't see each other's notes, each will notify independently and you get triple-paged.

One Alertmanager is a single point of failure. Two is a gossip-redundant pair. Three is paranoid. Pick based on your paranoia budget.

### The HA Topology

```
             +---------------------------------------------------+
             |                                                   |
   +---------+---------+   +---------+---------+   +-------------+-----+
   | Prometheus Rep A  |   | Prometheus Rep B  |   | Prometheus Rep C  |
   | (identical rules, |   | (identical rules, |   | (identical rules, |
   |  evaluates        |   |  evaluates        |   |  evaluates        |
   |  independently)   |   |  independently)   |   |  independently)   |
   +-------+-----------+   +-----------+-------+   +-------+-----------+
           |                           |                   |
           |    fan-out to ALL AMs:    |                   |
           +-------------+-------------+-------+-----------+
                         |             |       |
                         v             v       v
                  +------+-----+  +----+-------+  +-----+-----+
                  | AM replica |  | AM replica |  | AM replica|
                  |     1      |<-|     2      |<-|     3     |
                  +------+-----+  +------+-----+  +-----+-----+
                         ^               ^              ^
                         |   gossip (:9094 tcp+udp)     |
                         +-------- nflog replication ---+
                         +-------- silence replication -+
                         +-------- alert dedup ---------+
                                         |
                                         v
                                  +-------------+
                                  | PagerDuty / |
                                  | Slack / etc.|
                                  +-------------+
                                  (one notification
                                   per alert, even
                                   though all AMs
                                   see it)
```

**The dance**:
1. Every Prometheus replica independently evaluates rules (they may fire the same alert a few seconds apart due to clock drift -- that's fine).
2. Every Prometheus replica POSTs firing alerts to **every** Alertmanager replica (fan-out).
3. Each Alertmanager replica independently walks the pipeline.
4. Before notifying, each AM checks the **nflog** (notification log, gossip-replicated) to see if another replica already sent this notification.
5. If not, one of them wins the race, sends the notification, and records it in the nflog; the others see it and skip.

### Gossip Configuration

```yaml
# alertmanager args
--cluster.listen-address=0.0.0.0:9094
--cluster.peer=alertmanager-1.alertmanager-headless.monitoring.svc:9094
--cluster.peer=alertmanager-2.alertmanager-headless.monitoring.svc:9094
--cluster.peer=alertmanager-3.alertmanager-headless.monitoring.svc:9094
```

On port 9094 (TCP + UDP). Headless Services for pod-to-pod resolution. kube-prometheus-stack handles all of this automatically -- you just set `alertmanager.alertmanagerSpec.replicas: 3`.

### Why Fan-Out, Not Load Balancing

You might think "put all AMs behind a LoadBalancer and point Prometheus there." **Don't**. The gossip protocol replicates **notification state** (the nflog -- "have we already paged for this group key?") and **silences**, but **not the alerts themselves**. That means the cluster's HA guarantee assumes every AM receives every alert. Put an LB in front and suddenly each Prometheus push goes to exactly one backend. The redundancy collapses to "whichever AM the LB happened to pick."

The worst failure mode here is silent and insidious: **TCP-healthy-but-notification-pipeline-stuck**. The LB's L4 health check keeps an AM in rotation because it answers TCP probes. But that AM's Slack webhook is timing out, or it's hung on config reload, or its receiver's DNS lookup is stalling. The LB keeps routing alerts to it; the other AMs never see those alerts; no page goes out. There's no error, no alarm, nothing in the dashboard -- just "we never got woken up for that outage." With broadcast-to-all, AM2 or AM3 would independently notify and you survive it. Even an L7 health check can't save you because receivers are per-route: a pod might have broken Slack but healthy PagerDuty, and no generic health probe can test every receiver without spamming them.

The correct model is **Prometheus sends to ALL AMs, always**; the AMs gossip among themselves to dedupe via nflog. The LB belongs in front of the **UI** (so humans can reach the web console via a stable URL), never in front of the notification intake. Anti-pattern: single-AM-behind-LB is the classic SPOF you thought you fixed.

### Why 2 Replicas Is Usually Enough

Gossip tolerates N-1 failures in an N-member cluster for most purposes. Two replicas tolerate one failure (the survivor continues to work). Three replicas tolerate two failures but add gossip chatter. **For most production uses, 2 is the sweet spot**; 3 is fine if you want to survive a rolling restart during an active incident.

### HA Split-Brain

If the three AMs can't gossip (e.g., network partition within the cluster), each becomes its own island and notifies independently. Result: triple-paging. This is why `cluster.peer` arguments must be stable DNS names, not ephemeral pod IPs, and why the `cluster.reconnect-timeout` default (6h) is so generous -- better slightly stale gossip than split-brain.

---

## Part 9: Alert Fatigue -- the Anti-Patterns That Kill On-Call

The hospital analogy is not a metaphor here -- it's a literal precedent. **"Alarm fatigue"** is a documented, peer-reviewed phenomenon in hospital ICUs: when monitors beep constantly and most beeps are non-actionable, staff start ignoring them, including the real ones, and patients die. The Joint Commission (the body that accredits US hospitals) has flagged it as a "sentinel event." On-call engineering has the exact same failure mode: once every page becomes noise, the real outage gets acknowledged-without-reading at 3 AM and the incident lasts an hour longer than it had to.

More alerts = less attention paid = real alerts missed. This is the on-call equivalent of the boy who cried wolf, but with medical precedent for how fast the cognitive collapse happens. The following are the high-mortality anti-patterns:

### 1. Alerting on Causes, Not Symptoms

**Wrong**: `cpu_usage > 80%` on a node. Cause-level, not symptom-level -- high CPU is only bad if it hurts users. If p99 latency is fine, nobody cares.

**Right**: `p99_request_latency > SLO_threshold` on the user-facing API. That's a symptom your users feel. High CPU becomes a diagnostic tool in the runbook, not a page.

### 2. Missing `for:` -- Flapping Alerts

```yaml
- alert: HighCPU
  expr: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) > 0.8
  # NO `for:` ---- fires on every 15s scrape where it's true, resolves, fires, resolves...
```

Without `for:`, a noisy metric becomes a flood of firing/resolved pairs. Every alert needs at least `for: 2m`; most should be `for: 5m` or longer.

### 3. No Severity Hierarchy

If every alert is `severity: critical`, nothing is critical. If every alert pages, on-call develops "alert blindness" within a week and starts acknowledging without reading. The three-tier (critical / warning / info) hierarchy is the minimum discipline.

### 4. Wrong `group_by`

- Too broad (`[alertname]`): all clusters, all services, all namespaces bundled -- you lose the ability to tell who's affected.
- Too narrow (`[alertname, pod]`): one page per crashing pod; an outage produces forty pages. Death.

The sweet spot is usually `[alertname, cluster, service]` or `[alertname, cluster, namespace]`. Change only when you have a specific reason.

### 5. Paging on Warnings

`severity: warning` should never reach PagerDuty. Warnings are for "it's Tuesday, look at this when you have a minute." Paging on warnings is the fastest route to on-call burnout.

### 6. Too Many Alerts Per Service

A healthy service has **3-5 paging alerts** (SLO fast burn, SLO slow burn, pod not running, maybe latency spike, maybe dependency down). A service with 30 paging alerts is a service with 30 different ways to wake someone up for no reason.

### 7. No Runbook = No Alert

Culture rule: **if you can't write a runbook for the alert, you don't understand it well enough to alert on it.** The runbook URL in the annotation is not optional; it's the thing the on-call engineer reads at 3 AM.

### The On-Call Diary Discipline

Every page gets tagged by the responder the next day: `actionable` (did something), `false positive` (shouldn't have fired), `noise` (real, but nobody could do anything). Review weekly. Noise and false-positive alerts get deleted or tightened. After two months of this discipline, most teams halve their alert volume without reducing coverage.

---

## Part 10: Testing and Tooling

### promtool test rules

Test alerting rules with synthetic time series:

```yaml
# tests.yml
rule_files:
  - rules.yml

tests:
  - interval: 30s
    input_series:
      - series: 'http_requests_total{status="500",service="api"}'
        values: '10+1x30'         # 10, 11, 12, ... (30 samples, increment 1 each scrape)
      - series: 'http_requests_total{status="200",service="api"}'
        values: '1000+10x30'      # background traffic
    alert_rule_test:
      - eval_time: 15m
        alertname: HighAPIErrorRate
        exp_alerts:
          - exp_labels:
              severity: critical
              service: api
            exp_annotations:
              summary: "api is returning >5% 5xx errors"
```

Run: `promtool test rules tests.yml`

### amtool

Alertmanager CLI:

```bash
# Validate config syntax (run in CI!)
amtool check-config alertmanager.yml

# List currently firing alerts
amtool --alertmanager.url=http://alertmanager:9093 alert

# Inject a test alert (useful for testing routing)
amtool alert add alertname=TestCritical severity=critical team=platform \
  --start="2026-04-23T12:00:00Z" \
  --end="2026-04-23T13:00:00Z"

# Silences
amtool silence add alertname=Foo --duration=2h --comment="testing"
amtool silence query
amtool silence expire <ID>
```

### Routing Tree Editor

`prometheus.io/webtools/alerting/routing-tree-editor/` -- paste your config, feed test alerts as label sets, see the decision tree and which receivers get hit. Essential for `continue: true` and inheritance debugging.

---

## Part 11: Production Gotchas (Top 12)

1. **`for:` + `group_wait` compound.** Total time from expression first matching to first notification = `for` + `group_wait`. If `for: 5m` and `group_wait: 30s`, your first page arrives 5:30 after the incident starts. People forget this and set `for: 10m` thinking it's fast; it's really 10:30.

2. **`repeat_interval` has a floor of `group_interval`.** Setting `repeat_interval: 1m` while `group_interval: 5m` silently rounds up to 5m. No warning, just silent behavior.

3. **External labels are mandatory for multi-cluster.** Set `external_labels: {cluster: prod-us-east-1, region: us-east-1}` on every Prometheus. These get attached to every outgoing alert; without them, alerts from different clusters with the same service name look identical to Alertmanager and deduplicate across clusters -- you lose prod alerts because staging is also firing.

4. **A silence with only `alertname` is a grenade.** `amtool silence add alertname=HighErrorRate` silences that alert across every service, every cluster, every namespace. Always add scope: `alertname=HighErrorRate service=checkout`.

5. **PagerDuty `dedup_key` is derived from group labels -- changing `group_by` retroactively breaks incident continuity.** Alertmanager computes the PagerDuty `dedup_key` from the group-key hash, which is a function of the labels listed in `group_by`. Two consequences: (a) if your `group_by` is too granular, you get N PagerDuty incidents per outage instead of 1. (b) **far worse, and the one that surprises people**: if you edit `group_by` while an alert is actively firing, the dedup_key for the same logical alert changes on config reload. PagerDuty sees the *old* incident still open and a *new* incident with a different dedup_key and creates a duplicate. On-call now has two incidents for one problem, and the "resolve" signal from Alertmanager closes only the new one. Fix: treat `group_by` changes as "incident-affecting" -- schedule them for a quiet window, or at least page yourself before rolling out. Test in PagerDuty's non-prod environment before rolling out routing changes.

6. **HA split-brain triples pages.** If three AMs can't gossip (network partition, DNS flake), each notifies independently. Use stable DNS names for `--cluster.peer`, monitor gossip health (`alertmanager_cluster_members`).

7. **Rolling deploys flap alerts with `for: 0s` -- the deploy-flap trap.** Walk through the failure: you write an alert like

    ```yaml
    - alert: DeploymentNotAvailable
      expr: kube_deployment_status_replicas_available == 0
      # NO `for:` -- or `for: 0s`
      labels: {severity: critical}
    ```

    Now rollout drops replicas to zero for ~20 seconds mid-deploy (kubelet draining old pod, new pod not yet Ready). Alert fires. PagerDuty pages. On-call wakes up, fumbles for laptop. By the time they're logged in, the deployment completed, the alert resolved itself, and they're looking at "everything is fine" with no idea what happened. Next deploy, same cycle. Three repeats and they mute the alert.

    The fix is `for: 5m` (or longer than your longest deploy window). Some docs recommend `max_over_time(up[5m]) == 0` as a "smarter" equivalent -- in practice it behaves almost identically to `for: 5m` (both require the condition to hold for the full window), so prefer the simpler `for:` form unless you have a specific reason. The subtler fix for this *particular* metric: alert on `kube_deployment_status_replicas_unavailable > 0` with `for: 15m` -- that catches actually-broken deploys but tolerates the natural rollout transient.

8. **`resolve_timeout` default is 5 minutes.** Alertmanager considers an alert resolved if Prometheus hasn't sent it for 5 min. On a laggy rule-evaluation path or a flaky network, you can have stuck "firing" alerts because Prometheus delivery is slower than resolve_timeout would imply. Don't reduce this below 5m unless you understand the consequence.

9. **Template engine confusion: `$value` vs `.CommonLabels`.** In Prometheus rule annotations, use `$value`, `$labels.service`. In Alertmanager receiver templates, use `.CommonLabels.service`, `.Alerts`, `.Status`. They're different template engines -- `$` is Prometheus, `.` is Alertmanager. Mixing them produces `<no value>` in your Slack messages.

10. **`ruleSelector: {}` vs no ruleSelector.** On the Prometheus CRD, `ruleSelector: {}` (empty object) selects ALL rules cluster-wide; omitting the field entirely (null) selects NOTHING. Same for `ruleNamespaceSelector`. The first time a team complains "my PrometheusRule isn't firing," check this.

11. **AlertmanagerConfig doesn't support inhibition in older operator versions.** Pre-v0.52 of the operator, `AlertmanagerConfig` CRD had no way to define `inhibit_rules` -- you had to put them in the global config. Check your operator version; upgrade if you need per-team inhibition.

12. **Slack webhook rate limits.** Slack enforces per-webhook rate limits (~1 req/sec). A large alert storm can start dropping messages. If you have >100 alerts/hour, use a dedicated webhook per high-volume receiver, or switch to the Slack app with a bot token (higher limits).

---

## Part 12: Production Helm Values + CRDs

### kube-prometheus-stack `values.yaml`

```yaml
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: prod-us-east-1
      region: us-east-1
    ruleSelector:
      matchLabels:
        release: kube-prometheus-stack    # platform-managed rules
    ruleNamespaceSelector: {}             # from any namespace
    retention: 15d
    retentionSize: 90GB

alertmanager:
  alertmanagerSpec:
    replicas: 3                            # HA
    retention: 120h                        # 5 days of alert history
    externalUrl: https://alertmanager.example.com
    alertmanagerConfigSelector:
      matchLabels:
        alertmanagerConfig: team-managed
    alertmanagerConfigMatcherStrategy:
      type: OnNamespace                    # auto-inject namespace matcher (multi-tenant safety)
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi

  # Global (platform-managed) config -- team routing comes from AlertmanagerConfig CRDs
  config:
    global:
      resolve_timeout: 5m
      slack_api_url_file: /etc/alertmanager/secrets/slack-webhook/url
    route:
      receiver: default-null
      group_by: [alertname, cluster]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
    receivers:
      - name: default-null
    inhibit_rules:
      # Node down suppresses per-pod alerts on that node.
      # IMPORTANT: kube-state-metrics emits `node=<name>` for node-level alerts, while
      # kubelet self-scrape emits `instance=<host:port>`. Pick the label that ALL the
      # source AND target alerts actually carry, or the inhibitor silently never matches.
      # Here we use `instance` because the canonical kube-prometheus-stack rules
      # (`KubeNodeNotReady`, `KubePodCrashLooping`, `KubeletDown`) all agree on `instance`.
      - source_matchers: [alertname="KubeNodeNotReady"]
        target_matchers: [alertname=~"KubePodCrashLooping|KubePodNotReady|KubeletDown"]
        equal: [cluster, instance]
      # Critical suppresses warning of same alert on same service
      - source_matchers: [severity="critical"]
        target_matchers: [severity="warning"]
        equal: [alertname, cluster, service]

defaultRules:
  create: true
  disabled:
    - KubeMemoryOvercommit                 # too noisy on our cluster
    - KubeCPUOvercommit
```

### Platform Team's AlertmanagerConfig

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: platform-routing
  namespace: monitoring
  labels:
    alertmanagerConfig: team-managed
spec:
  route:
    receiver: platform-slack
    groupBy: [alertname, cluster, service]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    matchers:
      - name: team
        value: platform
        matchType: "="
    routes:
      - receiver: platform-pagerduty
        matchers: [{name: severity, value: critical, matchType: "="}]
        continue: true
      - receiver: platform-slack-incidents
        matchers: [{name: severity, value: critical, matchType: "="}]
  receivers:
    - name: platform-slack
      slackConfigs:
        - apiURL:
            name: platform-slack-webhook
            key: url
          channel: '#platform-alerts'
          sendResolved: true
    - name: platform-slack-incidents
      slackConfigs:
        - apiURL:
            name: platform-slack-webhook
            key: url
          channel: '#incidents'
          sendResolved: true
    - name: platform-pagerduty
      pagerdutyConfigs:
        - routingKey:
            name: platform-pagerduty
            key: routingKey
```

### PrometheusRule with Burn-Rate Alert + Backing Recording Rule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-slo
  namespace: api
  labels:
    release: kube-prometheus-stack
    team: api
spec:
  groups:
  - name: api.slo.recording
    interval: 30s
    rules:
    - record: api:http_error_ratio_5m
      expr: |
        sum by (service) (rate(http_requests_total{status=~"5..",namespace="api"}[5m]))
        / sum by (service) (rate(http_requests_total{namespace="api"}[5m]))
    - record: api:http_error_ratio_1h
      expr: |
        sum by (service) (rate(http_requests_total{status=~"5..",namespace="api"}[1h]))
        / sum by (service) (rate(http_requests_total{namespace="api"}[1h]))

  - name: api.slo.alerts
    rules:
    - alert: APISLOFastBurnRate
      expr: |
        (
          api:http_error_ratio_5m / 0.001 > 14.4
          and
          api:http_error_ratio_1h / 0.001 > 14.4
        )
      for: 2m
      labels:
        severity: critical
        team: api
        slo_type: fast_burn
      annotations:
        summary: "{{ $labels.service }} burning SLO budget at >14.4x"
        description: |
          Service {{ $labels.service }} error ratio = {{ $value | humanizePercentage }};
          budget exhaust ETA ~2 days. Investigate immediately.
        runbook_url: https://runbooks.example.com/api-fast-burn
        dashboard_url: https://grafana.example.com/d/slo/api?var-service={{ $labels.service }}
```

---

## Part 13: The 10 Commandments of Alerting

Mirroring yesterday's PromQL commandments, here are the ten rules to carve into your on-call brain:

1. **Thou shalt alert on symptoms, not causes.** High CPU is a clue, not a page. User-visible error rate is a page. Latency p99 over SLO is a page.

2. **Thou shalt set a `for:` on every alert.** Never zero. Minimum 2 minutes. 5-10 minutes for anything subject to deploy noise.

3. **Thou shalt tier severities correctly.** `critical` pages. `warning` tickets or Slacks. `info` logs. Confusing these tiers is the fastest path to on-call burnout.

4. **Thou shalt write a runbook for every paging alert.** No runbook, no alert. If you can't explain what to do about it, you don't understand it well enough to wake someone up for it.

5. **Thou shalt group thy alerts by `[alertname, cluster, service]`.** Start here. Deviate only with a specific reason you can defend in a design review.

6. **Thou shalt use burn-rate alerts for SLOs, not thresholds.** Multi-window, multi-burn-rate from the SRE Workbook. Threshold alerts are allowed for hard limits (disk, memory OOM) and binary states.

7. **Thou shalt inhibit, not silence, when one alert implies another.** Silences are for humans doing maintenance. Inhibitions are for "if X is firing, Y is obviously also firing -- don't say it twice."

8. **Thou shalt run Alertmanager in HA with at least 2 replicas, behind fan-out, never behind a load balancer.** Prometheus sends to every AM; gossip dedupes.

9. **Thou shalt test every non-trivial rule with `promtool test rules`.** Synthetic time series, asserted firing behavior, checked in CI.

10. **Thou shalt review thy alerts weekly.** Every page gets tagged (actionable / false positive / noise). Noise gets deleted. False positives get tightened. The on-call diary is how alerting hygiene compounds.

---

## Key Takeaways

- **Alertmanager is the triage nurse.** Prometheus decides *what* fires; Alertmanager decides *who hears about it, how, and how often*. The two binaries have complementary, non-overlapping jobs.

- **The pipeline is: inhibit -> silence -> route -> group -> notify.** Internalize those five stages and every Alertmanager behavior makes sense.

- **Routing is a tree, not a flat list.** One root, nested children, inheritance of timers, `continue: true` for fan-out. Use the visual editor until you can read a tree at a glance.

- **`group_by` controls storm vs signal.** Default `[alertname, cluster, service]` is the right starting point. Tune only with evidence from real incidents.

- **Burn-rate alerts beat threshold alerts for anything you SLO-ed.** Multi-window, multi-burn-rate. `14.4` / `6` are the numbers to remember.

- **Inhibition is declarative; silences are imperative.** Inhibition goes in code review; silences go in an operational audit log with a comment field.

- **HA Alertmanager requires fan-out, not load balancing.** Every Prometheus sends to every AM; gossip on `:9094` dedupes.

- **Multi-tenancy is what AlertmanagerConfig + `alertmanagerConfigMatcherStrategy: OnNamespace` solves.** Each team owns their routes in their namespace without stepping on other teams.

- **Alert fatigue kills on-call.** Tier severity correctly, runbook every page, review weekly. The best alerting improvement is usually deleting alerts, not adding them.

---

## Further Reading

- Prometheus Alerting Rules: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- Alertmanager Configuration: https://prometheus.io/docs/alerting/latest/configuration/
- Routing Tree Visual Editor: https://prometheus.io/webtools/alerting/routing-tree-editor/
- Google SRE Workbook Ch. 5 (Alerting on SLOs): https://sre.google/workbook/alerting-on-slos/
- Grafana burn-rate blog: https://grafana.com/blog/2022/08/18/how-to-configure-slo-burn-rate-alerts-in-prometheus-and-grafana-cloud/
- Prometheus Operator Alerting: https://prometheus-operator.dev/docs/developer/alerting/
- Repo: `2026/april/prometheus/2026-04-20-prometheus-fundamentals.md` (Prometheus scrape model, rules, HA basics)
- Repo: `2026/april/prometheus/2026-04-21-promql-deep-dive.md` (PromQL that powers `expr` and burn-rate math)
