# Grafana Fundamentals -- The Mission Control Room for Your Entire Observability Stack

> The past four days built the backend of observability. Apr 20 wheeled in the vital-signs monitor (Prometheus scraping metrics every 15 seconds). Apr 21 taught us to read the ledger (PromQL turning raw samples into vectors, rates, and quantiles). Apr 23 installed the triage nurse at the ER front desk (Alertmanager grouping, inhibiting, routing pages). All of that is plumbing. None of it has pixels. Nobody at your company is going to squint at Prometheus's `/graph` endpoint at 3 AM, and no executive is going to log into Alertmanager to check whether checkout is up. The part humans actually touch -- the part that turns "we have metrics" into "we run this system with confidence" -- is the one we've been putting off: **Grafana**.
>
> The core analogy for the day: **Grafana is the mission control room at Houston**. The rockets, fuel pumps, life-support sensors, guidance computers, comms radios -- those are Prometheus, Loki, Tempo, CloudWatch, PostgreSQL, Stripe's API. Each system speaks its own protocol, has its own telemetry cadence, and sits in its own black box of a data center. Mission control doesn't replace any of those systems; it is the **wall of monitors** that pulls a thread of telemetry from every one of them onto the same floor, in front of the same flight directors, in a visual grammar those directors can scan at a glance. One console shows orbital trajectory (Prometheus metrics). Another shows voice-loop transcripts (Loki logs). Another shows thrust-vector timelines (Tempo traces). Another shows weather at the launch pad (CloudWatch). Crucially, mission control is also where the **flight rules** live -- the red-line conditions that the flight director has pre-declared will scrub the mission, posted on clipboards at every station (Grafana alert rules). The dashboards are the monitors; the variables are the dials on each console that re-scope everyone's view to the same spacecraft stage; the provisioning YAML is the printed run-book that says exactly which console shows what, taped to every wall so a new flight director on console can reproduce the room from blueprints. An observability stack without Grafana is mission control with no monitors -- all the telemetry in the world, nobody looking at it.
>
> Keep mission control as the main character today. The data sources are comms feeds from different spacecraft systems; dashboards are individual consoles on the wall; panels are the specific gauges on each console; variables are the spacecraft-selector dial that re-scopes every gauge at once; transformations are the signal-conditioning boxes that clean a raw feed before it hits the gauge; Terraform is the procedure document that lets you rebuild the room from scratch.

---

**Date**: 2026-04-24
**Topic Area**: grafana
**Difficulty**: Intermediate

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Grafana | The mission control room at Houston | The visual front-end that pulls every telemetry source onto one wall of monitors |
| Data source | A comms feed from a different spacecraft subsystem | A pluggable adapter that knows how to authenticate to and query a backend (Prometheus, Loki, Tempo, CloudWatch, ...) |
| Mixed data source | Split-screen feeds from two spacecraft on one monitor | A special pseudo-source letting one panel combine queries from multiple real sources |
| Dashboard | A single console on the wall | A collection of panels, variables, rows, and time range defining one coherent view |
| Panel | One gauge on the console | The actual visualization -- time series, stat, gauge, table, heatmap, etc. |
| Row | A horizontal shelf of gauges on the console | A collapsible band of panels; can repeat per variable value |
| Variable | The "select spacecraft" dial on every console | A dashboard-level parameter interpolated into queries as `$var` / `${var}` |
| Annotation | A grease-pencil mark on the monitor at 14:03 UTC | Time-stamped event overlay on time-series panels (deploys, incidents) |
| Transformation | A signal-conditioning box between raw feed and gauge | Client-side data pipeline (rename/join/calculate/reduce) applied after the query returns |
| Library panel | A pre-approved gauge design any console can pull off the shelf | Panel definition stored once, referenced by many dashboards; edits propagate |
| Scenes runtime | The rebuilt backplane wiring behind the wall of monitors | Unified React-based scene-graph runtime (2024), replacing the legacy per-panel render loop and the last AngularJS bindings; enables dynamic dashboards |
| Dashboard schema v2 | Blueprints in CUE instead of one monolithic engineering drawing | Kubernetes-style resource schema (preview in Grafana 12, GA-default in Grafana 13 with auto-migration from v1) |
| Provisioning (file) | Printed run-book taped to the wall | YAML in `/etc/grafana/provisioning/*` defining sources/dashboards/alerts at boot |
| Terraform `grafana_*` | Manufacturing the mission control room from CAD files | Declarative IaC for folders, dashboards, data sources, rules, contact points |
| Folder + RBAC | Which flight directors can touch which console | Org -> folder -> dashboard permission hierarchy (Viewer / Editor / Admin / Team) |
| Grafana-managed alert | A flight rule stored in mission control's own binder | Rule stored in Grafana SQL DB, evaluated by Grafana, works across any data source |
| Data-source-managed alert | A flight rule stored in the spacecraft's own onboard computer | Rule stored in Prometheus/Mimir; Grafana just lists and forwards |
| Service account + token | A badge + key card for a headless console operator | Replaces deprecated API keys; the only sane CI auth path since Grafana 10 |
| `$__rate_interval` | "Pick the right shutter speed automatically" | Auto-computed interval guaranteeing `rate()` sees >=4 samples at current zoom |
| Recorded query | A materialized view of an expensive derivation, broadcast as its own telemetry channel | Grafana Cloud feature: periodically evaluates a query and writes the result back as a new time series so every console re-uses it cheaply |
| Public dashboard | The giant public countdown clock outside the building | Read-only, anonymous-visible share link with its own token |
| Git Sync (GA in G13) | Syncing the run-book to a versioned offsite Git safe | 2-way reconciliation with GitHub/GitLab/Bitbucket/Git (GitHub App auth), PR-based dashboard review |
| Dynamic dashboards (GA in G13) | Consoles that re-arrange themselves per spacecraft phase | Tabs, nested rows, conditional panels, responsive layout on the Scenes/v2 stack |
| Grafana Advisor (GA in G13) | The QA inspector walking the room checking wiring | Automated health checks on data sources, plugins, SSO, misconfig -- with AI-assisted remediation |
| Grafana Assistant (OSS in G13) | The junior flight controller you can ask "what's this metric?" | AI agent for dashboard suggestions, query help, SQL refinement; now in OSS + Enterprise, previously Cloud-only |

---

## The Room, in One Picture

Before zooming in, look at the whole observability floor once. Every piece below either hangs on the wall, lives in the backplane, or sits on a shelf of printed procedures:

```
                   MISSION CONTROL FLOOR (Grafana Server)
   +-----------------------------------------------------------------------+
   |                                                                       |
   |   +------------- WALL OF MONITORS (Dashboards) -------------+         |
   |   |                                                         |         |
   |   |  [ Console:                  [ Console:                 |         |
   |   |    k8s-overview ]              checkout-SLO ]           |         |
   |   |   +--+ +----+ +---+            +----+ +-----+           |         |
   |   |   |G | |TS  | |Tbl|            |Stat| | Heat|           |         |
   |   |   +--+ +----+ +---+            +----+ +-----+           |         |
   |   |                                                         |         |
   |   |  Variables (dashboard-wide):  $cluster  $namespace      |         |
   |   |  Time range: last 1h                                    |         |
   |   +----------+----------------------------+-----------------+         |
   |              |                            |                           |
   |              v                            v                           |
   |   +---------------------+      +---------------------+                |
   |   | DATA SOURCE PROXY   |      | ALERTING ENGINE     |                |
   |   | (Grafana backend    |      | (Grafana-managed    |                |
   |   |  adapters)          |      |  rule evaluator)    |                |
   |   +----+----+----+----+-+      +----------+----------+                |
   |        |    |    |    |                   |                           |
   |        v    v    v    v                   v                           |
   |     Prom  Loki Tempo  CW            Contact Points                    |
   |    metrics logs traces Cloud        (Slack, PagerDuty,                |
   |                                      email, webhook)                  |
   |                                                                       |
   |   +------------- BACKPLANE (Grafana SQL DB) -----------------+        |
   |   |  users | teams | folders | dashboards | alert_rules |            |
   |   |  silences | notification_policies | service_accounts |           |
   |   +---------------------------------------------------------+        |
   |                                                                       |
   |   +------------- PROVISIONING (printed run-books) -----------+        |
   |   |  /etc/grafana/provisioning/{datasources,dashboards,     |         |
   |   |                             alerting,plugins}/*.yaml    |         |
   |   |  OR: Terraform grafana_* resources via HTTP API         |         |
   |   +---------------------------------------------------------+        |
   +-----------------------------------------------------------------------+
```

Six things to remember about this picture:

1. **Grafana stores almost nothing about your metrics.** Series, samples, log lines, spans -- those live in Prometheus, Loki, Tempo, CloudWatch. Grafana owns only the *questions* (dashboards, queries, alert rules) and *identity* (users, teams, folders, permissions). This is why a Grafana backup is tiny and why losing Grafana never loses your observability data.
2. **Every panel is a query + a visualization + an optional transformation pipeline.** That triple is the unit of composition. You can change any one leg without touching the others.
3. **The data source proxy is what makes variables and RBAC work.** The browser doesn't talk directly to Prometheus; it posts to Grafana, which injects tenant headers, applies query-time filters, and forwards. This is how a single Grafana can front a multi-tenant Mimir or enforce per-team data-source access.
4. **Alerting splits along a clean axis.** Either the rule lives in the Grafana SQL DB (Grafana-managed -- can query *any* source) or it lives in the data source itself (data-source-managed -- Prometheus Mimir ruler evaluates it). Grafana's UI shows both; only one of them actually evaluates each rule.
5. **The SQL DB is mutable state, and it is the thing provisioning is trying to pin down.** Every "I edited it in UI and now I can't repro it" disaster is a fight between humans editing rows in this database and YAML/Terraform trying to overwrite those rows.
6. **Scenes + schema v2 don't change what Grafana *is*.** They change how it's rendered and serialized. The mental model above (wall of monitors, data source proxy, SQL DB, provisioning) is stable across versions.

---

## Part 1: Data Sources -- The Comms Feeds

A **data source** in Grafana is the adapter between a specific backend's protocol and Grafana's internal query interface. There are 150+ official and community data sources; you will mostly use five to ten of them. Each data source plugin contributes three things:

1. A **configuration page** (URL, auth, TLS, tenant headers).
2. A **query editor** (Prometheus gets PromQL with autocomplete; CloudWatch gets namespace/metric/dimension pickers; Loki gets LogQL; SQL sources get a SQL textarea).
3. A **backend proxy** (how Grafana's server forwards queries, often with server-side auth unavailable to the browser).

### The Five You Will Meet First

| Data source | Protocol | Query language | Authentication usually via |
|-------------|----------|----------------|----------------------------|
| Prometheus | HTTP `/api/v1/*` | PromQL | Bearer token, basic auth, sigv4 (for AMP), mTLS |
| Loki | HTTP `/loki/api/v1/*` | LogQL | Bearer token, X-Scope-OrgID for multi-tenant |
| Tempo | HTTP `/api/search`, `/api/traces/{id}` | TraceQL | Bearer token, X-Scope-OrgID |
| CloudWatch | AWS SDK | CloudWatch Metrics Insights, Logs Insights | IAM role (IRSA in EKS), access keys |
| PostgreSQL / MySQL | Native driver | SQL | Username/password, IAM DB auth for RDS |

A well-named data source matters more than people think. The name becomes the identifier that dashboard JSON and Terraform resources reference forever. `prom-prod-us-east` beats `Prometheus` -- the former survives a second Prometheus being added; the latter causes a mass dashboard break.

### Configuring a Prometheus Data Source (File Provisioning)

```yaml
# /etc/grafana/provisioning/datasources/prom.yaml
apiVersion: 1

datasources:
  - name: prom-prod-us-east
    uid: prom-prod-use1                    # STABLE UID -- dashboards reference by UID, not name
    type: prometheus
    access: proxy                          # ALWAYS proxy; 'direct' was deprecated in 8.3 and now silently behaves as proxy
    url: http://prometheus.monitoring.svc:9090
    isDefault: true
    editable: false                        # UI edits blocked; single source of truth is this file
    jsonData:
      httpMethod: POST                     # POST avoids URL-length limits on big queries
      timeInterval: "15s"                  # matches your scrape_interval -- enables $__rate_interval
      queryTimeout: "60s"
      prometheusType: Mimir                # or 'Prometheus' / 'Cortex' / 'Thanos'
      prometheusVersion: "2.9.1"
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo-prod        # click an exemplar, jump to Tempo trace
      httpHeaderName1: X-Scope-OrgID       # NON-secret header name lives in jsonData
    secureJsonData:
      httpHeaderValue1: "tenant-payments"  # SECRET header value lives in secureJsonData; pairs to Name1 by suffix

# Key lesson: `uid` is the forever handle. Name can change; uid must not.
# Header pairing: httpHeaderName<N> in jsonData (non-secret); httpHeaderValue<N> in secureJsonData (secret).
# They pair by the trailing number. Mis-indenting either side silently sends an unauthenticated request.
```

### Exemplars -- The Click-Through to Traces

That `exemplarTraceIdDestinations` line is doing a lot of work. An **exemplar** is a trace ID attached to a single observation in a Prometheus histogram bucket -- emitted by your instrumented app via the OpenMetrics protocol. When Grafana renders a heatmap or time-series of a histogram, exemplars appear as little diamonds on the panel; clicking one jumps you straight into the matching Tempo trace, scoped to the right time window. This is the **metrics -> traces** correlation pillar (see Apr 20 for how Prometheus stores them; see the upcoming OpenTelemetry doc for how apps emit them). Configure the destination once per Prometheus data source and every histogram panel acquires the click-through for free.

### Mixed Data Source -- The Split-Screen Monitor

One panel can query multiple backends if you select the magic **`-- Mixed --`** data source. The query editor then lets you pick a different real source per query:

```
Panel: "Checkout SLO vs deploys"
  Query A  (prom-prod-use1):   sum(rate(checkout_success_total[5m]))
                              / sum(rate(checkout_total[5m]))
  Query B  (loki-prod):        count_over_time({service="checkout",
                               level="error"}[5m])
  Query C  (postgres-ops):     SELECT time, env FROM deploys
                              WHERE service='checkout'
```

Transformations then line them up by time and you get one panel plotting success rate, error-log velocity, and deploy markers in the same graph. This is the single most underused feature in Grafana and one of its sharpest differentiators from single-backend UIs.

### Recorded Queries (Grafana Cloud)

A **recorded query** takes a live query -- say a horrifyingly expensive `sum by (pod) (rate(...[1h]))` -- and stores its result as a new Prometheus series every N minutes. Think of it as server-side recording rules, but authored from the Grafana UI, targeting the same writeback endpoint as your app metrics. Handy for dashboards that would otherwise melt the cluster at every open; less handy than a proper Prometheus recording rule for anything you'd put an alert on.

---

## Part 2: Dashboard Anatomy -- One Console on the Wall

A dashboard is a JSON document (schema v1) or a set of CUE-typed Kubernetes-style resources (schema v2, Grafana 12+). Either way, the mental model is the same: **time range + variables + panels in a grid**.

### The Pieces

```
Dashboard
├── uid                   # forever handle, don't change
├── title
├── tags                  # search + folder-browsing
├── time range (default)  # "now-6h to now", "now-24h to now"
├── refresh interval      # 30s, 1m, 5m, off
├── variables[]           # dashboard-wide selectors
├── annotations[]         # event overlays queried from data sources
├── links[]               # jump-to-other-dashboard buttons
└── panels[]
    ├── panel 1
    │   ├── gridPos (x,y,w,h on 24-col grid)
    │   ├── type (timeseries, stat, gauge, table, ...)
    │   ├── targets[] (queries)
    │   ├── transformations[]
    │   ├── fieldConfig (units, thresholds, decimals)
    │   └── options (viz-specific)
    ├── panel 2
    └── row panel (container for repeats)
```

The grid is 24 columns wide. `gridPos: {x: 0, y: 0, w: 12, h: 8}` is a half-width panel at the top-left. Dashboards don't have explicit pages -- they scroll, and **rows** are collapsible bands for grouping.

### Annotations -- The Grease-Pencil Marks on the Monitor

Annotations overlay vertical lines (point-in-time events) or shaded regions (duration events) on every time-series panel on the dashboard. Three flavors:

1. **Manual annotations** -- a human Ctrl-clicks a panel to mark "we changed config at 14:03." Stored in Grafana's SQL DB, tagged, searchable.
2. **Tag-based annotations** -- rendered automatically whenever a stored annotation matches a tag filter. Good for ad-hoc incident timelines written by on-call.
3. **Query-driven annotations** -- the important one -- configured at the *dashboard* level, where a Prometheus/Loki/SQL query returns rows that Grafana renders as annotation marks on every panel. This is how you get automatic **deploy markers** on every dashboard: a single Prometheus series `deploy_timestamp{service=~".+"}` or a Loki query `{stream="deploys"}` becomes vertical lines across every latency and error-rate chart, so "did this start at deploy?" answers itself in one glance. Configure once per dashboard; no panel edits needed.

The turning point where a team's dashboards go from "pretty" to "genuinely operational" is usually the day someone wires up a query-driven deploy-annotation source.

### Best-Practice Layout -- the "Z Pattern"

Grafana Labs' canonical guidance: lay out the dashboard so that a human's eye, scanning top-left to bottom-right in a Z, naturally encounters the most important signal first. In practice:

- **Top row**: stat panels answering "is the thing okay right now?" -- big numbers with thresholds (green/amber/red).
- **Second row**: RED/USE time-series (rate, errors, duration; or utilization, saturation, errors).
- **Third row**: supporting detail -- per-instance breakdowns, slow-query tables, log snippets.
- **Bottom**: infrastructure-level metrics (CPU, memory, disk) that you need when drilling down but should never crowd the top.

The anti-pattern: a wall of 40 identically sized panels ordered by when someone thought of them. No signal hierarchy, unreadable in an incident.

### Scenes Runtime and Dashboard Schema v2

Two architecture shifts matter for anyone running Grafana in 2025+:

**Scenes (Grafana 10+, default in 11+)**: the rendering library powering dashboards was rewritten from an AngularJS + React mix to a pure React "scenes" runtime. Same dashboard JSON, same panels, but: consistent performance, fewer re-render bugs, a clean API that app plugins (like Grafana Explore, k8s monitoring apps) can reuse to embed dashboard-like behavior inside their own UIs. You don't write Scenes code for a normal dashboard; you benefit from it.

**Dashboard Schema v2 (preview in Grafana 12, GA-default in Grafana 13 with auto-migration)**: the monolithic dashboard JSON is decomposed into CUE-typed, Kubernetes-style resources -- a `Dashboard` object with embedded `Panel` / `PanelConfig` / `VariableConfig` / `Layout` children. This unlocks:

- **Dynamic dashboards** (GA in Grafana 13): tabs, nested rows, conditional panels, responsive layout -- not just the 24-column grid.
- **Git Sync** (GA in Grafana 13): dashboards reconciled to/from a Git repo with 2-way sync, PR-based dashboard review, and GitHub App authentication.
- **Typed IaC**: Grafana Foundation SDK code-gens Go / Python / TypeScript types from the CUE schema.

**Migration note (2026-04 update):** in Grafana 13, opening a v1 dashboard auto-migrates it to v2 in memory; existing Terraform and API workflows keep working against v1 JSON. This removes the "one-way migration" gotcha that made v2 adoption scary in Grafana 12 -- you can take it dashboard-by-dashboard without breaking your provisioning pipeline.

---

## Part 3: Variables -- The Spacecraft-Selector Dial

Variables are dashboard-level parameters. You define them once at the top of the dashboard; every panel query can interpolate them as `$var` or `${var}`. They turn 50 per-service dashboards into 1 parameterised dashboard.

### The Seven Variable Types

| Type | What it is | Example |
|------|-----------|---------|
| **Query** | Options fetched from a data source | `label_values(up{job="api"}, instance)` -> list of API pods |
| **Custom** | Hard-coded list of values | `prod, staging, dev` |
| **Data source** | Picker over data sources of a given type | Pick which Prometheus to query |
| **Interval** | Time buckets (`1m, 5m, 1h, auto`) | For `rate(...[$interval])` |
| **Text box** | Free-form user input | Search substring for log queries |
| **Constant** | Hidden, fixed value | `$env = "prod"` embedded in dashboard |
| **Ad hoc filters** | Data-source-wide extra label matchers | Appended to every Prometheus query on the dashboard |

### Multi-Value Variables -- the Formatting Trap

When a variable is **multi-value** (user can Ctrl-click several values, or "All" is selected), `$var` interpolates differently depending on context. Pick the right format or you get broken queries:

```promql
# Variable $pod has values [api-1, api-2, api-3]

# DEFAULT interpolation -- PromQL-friendly regex alternation
up{pod=~"$pod"}        -> up{pod=~"api-1|api-2|api-3"}

# CSV form
$pod:csv               -> api-1,api-2,api-3

# Pipe form (same as default but explicit)
${pod:pipe}            -> api-1|api-2|api-3

# Raw (don't URL-encode, don't quote)
${pod:raw}             -> api-1,api-2,api-3

# SingleQuote (for SQL IN clauses)
${pod:singlequote}     -> 'api-1','api-2','api-3'
```

The default format is "pipe if the query is a regex, csv otherwise" -- it's context-aware but opaque. When something mysteriously doesn't interpolate right, switch to an explicit format like `${pod:pipe}` or `${pod:csv}` and the problem usually dissolves.

### Chained / Cascading Variables

One variable's options can depend on another's. The `$cluster` variable lists clusters; the `$namespace` variable lists namespaces **in the selected cluster** by using `$cluster` in its query:

```text
Variable: cluster
  Type: Query
  Query: label_values(up, cluster)

Variable: namespace
  Type: Query
  Query: label_values(up{cluster="$cluster"}, namespace)

Variable: pod
  Type: Query
  Query: label_values(up{cluster="$cluster",namespace="$namespace"}, pod)
```

Change `$cluster`, and `$namespace` and `$pod` automatically refresh. This is how you build one "service drill-down" dashboard that scales to hundreds of services.

### The `$__*` Built-ins

Grafana injects several built-in variables into every query. The ones you must know:

| Variable | What it is | When to use |
|----------|-----------|-------------|
| `$__interval` | Auto-computed interval = (time range) / (panel width in px) | For `sum by (...) (rate(...[$__interval]))` -- rarely the right answer |
| `$__rate_interval` | `max($__interval, 4 * scrape_interval)` | **The right one for `rate()` in time-series panels.** Guarantees >=4 samples |
| `$__range` | The full selected time range as a duration (`1h`, `6h`, `7d`) | `increase(http_requests_total[$__range])` -- total over visible window |
| `$__from` / `$__to` | Unix millis of start/end of selected range | For SQL `WHERE ts BETWEEN ...` clauses; format with `:date:iso`, `:date:seconds`, `:date:YYYY-MM-DD` |
| `$__dashboard` | UID of the current dashboard | Useful in links |
| `${__user.login}` / `${__user.email}` | Currently viewing user's identity | Audit-stamping queries, per-user filters in mixed-tenant SQL |
| `${__org.name}` / `${__org.id}` | Current org | Multi-tenant Grafana labelling |

**The `$__rate_interval` rule**: if you are running `rate()` in a Grafana panel, you almost always want `$__rate_interval`, not `$__interval`. The `timeInterval` on your Prometheus data source (set in provisioning YAML) is what makes `$__rate_interval` compute correctly -- forgetting to set that is the root cause of "my rate query is empty when I zoom out."

### Repeating Panels and Rows

A variable can be set to **repeat** a panel (or a whole row) once per value. One "CPU per pod" panel with `pod=All` becomes N actual panels, one per pod, laid out horizontally or vertically. Cheap to author, expensive at render time -- see gotchas.

---

## Part 4: Visualization Types -- Picking the Right Gauge

Grafana ships around 20 built-in visualization types. Picking the wrong one is the single biggest cause of "this dashboard is useless." Use this decision matrix:

Every row below maps to a physical gauge or display you'd find on a real mission control console -- that's the lens to keep in mind when picking:

| You want to show... | Use | Why | What this is on the console |
|---------------------|-----|-----|------------------------------|
| How a number changes over time | **Time series** | The default; dense, zoomable, multi-series | The main scrolling oscilloscope trace |
| The *current* value of one number, big and bold | **Stat** | Reads at a glance across a 20-ft room | The big 7-segment digital readout |
| Is this inside a safe range? | **Gauge** (radial) | Threshold colors, min/max visible | An old-school dial with red/amber/green bands |
| Horizontal "progress bar" per item | **Bar gauge** | Comparing many items' current values | A row of LED bargraphs, one per subsystem |
| Tabular breakdown (top N, per-service details) | **Table** | Sortable columns, per-cell coloring | The manifest clipboard on the clipboard shelf |
| Distribution over time (latency buckets) | **Heatmap** | X=time, Y=bucket, Z=count; essential for histograms | A signal-conditioning scope engineers stare at for oscillations |
| State transitions (up/down, phase changes) | **State timeline** | Step-function strip per series | The mission-phase strip chart across the bottom of the room |
| Long history of discrete state | **Status history** | Like state timeline but compact for many series | The annunciator panel of per-subsystem go/no-go lights over time |
| Map of a fleet (EC2 regions, user countries) | **Geomap** | Lat/lon or ISO country code panel | The world map with lit-up ground stations |
| Free-drawn infrastructure diagram | **Canvas** | Data-driven SVG; hot topology in a single glance | The schematic of the spacecraft with live-colored subsystems |
| Static text, runbook pointers, legend | **Text** | Markdown, not a visualization per se | The laminated procedure card taped next to the console |

**The three rules**:

1. **Never use a gauge (radial) when a stat works.** Stats are readable from across the room; gauges aren't.
2. **Never use a pie chart for anything.** Humans cannot compare angles accurately. Use a bar gauge or table.
3. **Prefer heatmaps over stacked time series for latency distributions.** A stacked bucket-count chart is unreadable; a heatmap is exactly what the eye wants.

---

## Part 5: Transformations -- The Signal-Conditioning Box

A transformation is a **client-side** step that runs after your query returns but before the visualization renders. The pipeline runs sequentially: each transform's output is the next one's input.

### The Transformations You Will Actually Use

| Transform | What it does | Classic use |
|-----------|--------------|-------------|
| **Reduce** | Collapse time series to one number per series (last, max, mean, sum) | Power a stat or bar gauge from a time-series query without a second query |
| **Organize fields** | Rename, reorder, hide columns | Make a table readable |
| **Filter fields by name** | Drop fields matching a regex | Strip internal fields before display |
| **Merge** | Combine multiple result sets into one frame by matching time | Unify queries from different sources for a single table |
| **Join by field** | SQL-style join on a key column | Combine Prometheus query + SQL query side by side |
| **Labels to fields** | Promote Prometheus labels to table columns | Turn a Prom query into a browsable table |
| **Calculate field** | Add a column computed from others | `error_rate = errors / total` at display time |
| **Group by** | Aggregate rows by a field | Sum requests per service client-side |
| **Partition by values** | Split one frame into many by field value | Feed many-per-series time-series panels from a single wide query |
| **Config from query** | Let a query drive panel config (thresholds, units, min/max) | Dynamic thresholds per service |

### Transform Pipeline Example -- Readable Table from a Noisy Query

Picture the raw feed coming off the spacecraft: packets labelled with internal telemetry IDs, full of low-amplitude noise, in the wrong field order for the operator's eye. The transformation pipeline is the signal-conditioning box in the backplane that cleans the feed before it hits the console:

```text
Query:   sum by (pod, namespace) (rate(http_requests_total[5m]))
          (raw feed from the spacecraft -- labels are internal IDs, one noisy number per series)

Transform 1: Labels to fields            -> columns: pod, namespace, Value
                                            (demultiplex the labels into operator-visible columns)
Transform 2: Organize fields             -> rename Value -> "req/s",
                                            reorder: namespace, pod, req/s
                                            (relabel the gain knob in human units, reorder for the eye)
Transform 3: Filter by values            -> keep rows where req/s > 1
                                            (notch filter -- drop below-noise-floor signals)
Transform 4: Sort by                     -> req/s desc
                                            (present strongest signals at the top of the trace)
```

The operator on console never sees the raw packet format; they see a top-requests-per-pod table with zero extra queries. Signal-conditioning in a box.

### The "Compute in Query vs Compute in Transform" Decision

You can compute the same thing in PromQL or in a transformation. The rule:

- **If the computation changes what gets transferred**, do it in the query. Aggregations (`sum by`), rate math, label reductions -- these reduce the bytes Prometheus returns.
- **If the computation is pure presentation**, do it in a transform. Renaming columns, hiding fields, joining with another source, computing a ratio for display.
- **Never move alerting-relevant logic into transforms.** Alerting rules evaluate the query, not the transformed output. Your error-rate `<` threshold must be in the PromQL, not in a calculate-field transform. The alert engine will not see the transform.

---

## Part 6: Provisioning -- The Printed Run-Book

Every Grafana you run in production must be rebuildable from a Git repo in under 15 minutes, with zero manual UI clicks. Two paths do this: **file-based provisioning** (YAML on disk) and **Terraform** (HTTP API calls). They compose; most teams use both.

### File-Based Provisioning

Grafana watches four directories under `/etc/grafana/provisioning/`:

```
/etc/grafana/provisioning/
├── datasources/         YAML defining data sources
├── dashboards/          YAML defining *providers* that point at JSON dashboard files on disk
├── alerting/            YAML defining rule groups, contact points, notification policies, mute timings
└── plugins/             YAML installing/configuring app plugins
```

A **dashboards provider** is the piece most people find confusing. The YAML file doesn't contain dashboards -- it defines a *provider* that scans a directory of dashboard JSON files:

```yaml
# /etc/grafana/provisioning/dashboards/provider.yaml
apiVersion: 1

providers:
  - name: "infra-team"
    orgId: 1
    folder: "Infrastructure"
    folderUid: "infra-folder"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: false              # critical: editing in UI shows error toast
    foldersFromFilesStructure: true    # /dashboards/k8s/* -> "k8s" folder, auto
    options:
      path: /var/lib/grafana/dashboards/infra
```

Dashboard JSON files then sit at `/var/lib/grafana/dashboards/infra/k8s-overview.json` and get loaded automatically. The `updateIntervalSeconds: 30` means pushing a new JSON to the filesystem causes Grafana to reconcile within 30 seconds.

Three flags govern the safety model:

- `allowUiUpdates: false` -- UI edits are rejected with a "this dashboard is provisioned" error. The *only* way to change the dashboard is to push new JSON.
- `disableDeletion: true` -- deleting from UI is rejected. Pair with `false` if you want UI deletes to be re-reconciled from disk.
- `editable: false` (on data sources) -- same idea, read-only in UI.

### Alerting Provisioning

```yaml
# /etc/grafana/provisioning/alerting/rules.yaml
apiVersion: 1

groups:
  - orgId: 1
    name: checkout-slo
    folder: "Alerts / Payments"
    interval: 1m
    rules:
      - uid: checkout-burn-rate-fast
        title: CheckoutFastBurn
        condition: C
        data:
          - refId: A
            datasourceUid: prom-prod-use1
            model:
              expr: |
                (1 - (
                  sum(rate(checkout_success_total[5m]))
                  /
                  sum(rate(checkout_total[5m]))
                )) / (1 - 0.999) > 14.4
              intervalMs: 60000
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              conditions:
                - evaluator:
                    params: [0]
                    type: gt
        noDataState: NoData
        execErrState: Error
        for: 2m
        annotations:
          summary: "Checkout burning error budget fast"
          runbook_url: "https://runbooks.example.com/checkout"
        labels:
          severity: critical
          team: payments

contactPoints:
  - orgId: 1
    name: pagerduty-payments
    receivers:
      - uid: pd-payments
        type: pagerduty
        settings:
          integrationKey: $PD_PAYMENTS_KEY
        secureFields:
          integrationKey: true

policies:
  - orgId: 1
    receiver: slack-default
    group_by: ["grafana_folder", "alertname"]
    routes:
      - receiver: pagerduty-payments
        matchers: ["severity = critical", "team = payments"]
        continue: false
```

Note how this reimplements, in Grafana's own schema, the exact routing-tree concepts from Apr 23's Alertmanager doc: `group_by`, `matchers`, `continue`, receivers. Grafana-managed alerting keeps the same mental model; it just stores the config in the Grafana SQL DB and evaluates it inside Grafana itself.

### The Canonical Anti-Pattern

Everyone falls into this trap exactly once. You demo a cool dashboard live in UI, click Save, ship it. Two weeks later you delete the Grafana pod, or restore from backup, and the dashboard is gone -- because it was never in the run-book. The fix: treat Grafana's UI as read-only in prod. Edits happen in Git; CI pushes to the provisioning path; Grafana reconciles. If someone insists on UI-first authoring, have them export the JSON and PR it.

---

## Part 7: Terraform -- Manufacturing the Room

File provisioning is great when Grafana runs next to the YAML on the same pod. Terraform is the better fit for multi-org, multi-stack, or Grafana Cloud setups where you want the same code path managing the Grafana config as the rest of your infra.

### The Core Resources

```hcl
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = "~> 4.0"     # major 4.x is current as of 2026; 3.x is EOL
    }
  }
}

provider "grafana" {
  url  = "https://grafana.example.com"
  auth = var.grafana_service_account_token  # NOT an API key; those are deprecated
}

# 1. Folder -- RBAC boundary
resource "grafana_folder" "payments" {
  title = "Payments"
  uid   = "payments-folder"
}

# 2. Data source -- identical to YAML above, but Terraform-managed
resource "grafana_data_source" "prom_prod" {
  name          = "prom-prod-use1"
  uid           = "prom-prod-use1"
  type          = "prometheus"
  url           = "http://prometheus.monitoring.svc:9090"
  is_default    = true
  access_mode   = "proxy"

  json_data_encoded = jsonencode({
    httpMethod         = "POST"
    timeInterval       = "15s"
    prometheusType     = "Mimir"
    prometheusVersion  = "2.9.1"
  })

  http_headers = {
    "X-Scope-OrgID" = "tenant-payments"
  }
}

# 3. Dashboard -- JSON model, usually loaded from file
resource "grafana_dashboard" "checkout" {
  folder      = grafana_folder.payments.uid
  config_json = file("${path.module}/dashboards/checkout.json")

  # overwrite existing dashboard with the same UID on apply
  overwrite = true
}

# 4. Team
resource "grafana_team" "payments" {
  name    = "Payments"
  members = ["alice@example.com", "bob@example.com"]
}

# 5. Folder permission -- team-scoped RBAC
resource "grafana_folder_permission" "payments" {
  folder_uid = grafana_folder.payments.uid

  permissions {
    team_id    = grafana_team.payments.id
    permission = "Edit"
  }
  permissions {
    role       = "Viewer"
    permission = "View"
  }
}

# 6. Alert rule group
resource "grafana_rule_group" "checkout_slo" {
  folder_uid       = grafana_folder.payments.uid
  name             = "checkout-slo"
  interval_seconds = 60

  rule {
    name      = "CheckoutFastBurn"
    condition = "C"
    for       = "2m"

    data {
      ref_id         = "A"
      datasource_uid = grafana_data_source.prom_prod.uid

      relative_time_range {
        from = 300
        to   = 0
      }

      model = jsonencode({
        expr         = "(1 - (sum(rate(checkout_success_total[5m])) / sum(rate(checkout_total[5m])))) / (1 - 0.999) > 14.4"
        intervalMs   = 60000
        maxDataPoints = 43200
      })
    }

    data {
      ref_id         = "C"
      datasource_uid = "__expr__"
      model = jsonencode({
        type = "threshold"
        conditions = [{
          evaluator = { params = [0], type = "gt" }
        }]
      })
    }

    labels = {
      severity = "critical"
      team     = "payments"
    }
    annotations = {
      summary     = "Checkout burning error budget fast"
      runbook_url = "https://runbooks.example.com/checkout"
    }
  }
}

# 7. Contact point
resource "grafana_contact_point" "pagerduty_payments" {
  name = "pagerduty-payments"
  pagerduty {
    integration_key = var.pd_payments_key
    severity        = "critical"
  }
}

# 8. Notification policy (the routing tree)
resource "grafana_notification_policy" "root" {
  group_by      = ["grafana_folder", "alertname"]
  contact_point = grafana_contact_point.slack_default.name

  policy {
    matcher {
      label = "severity"
      match = "="
      value = "critical"
    }
    matcher {
      label = "team"
      match = "="
      value = "payments"
    }
    contact_point = grafana_contact_point.pagerduty_payments.name
    continue      = false
  }
}
```

### The `jsonencode` Pattern

Notice how `json_data_encoded` and `model` take stringified JSON. This is because Terraform schemas can't represent arbitrary nested objects for every Grafana version, so the provider took the escape hatch: you hand it a JSON blob. The trick is to use `jsonencode({...})` with an HCL object rather than a literal heredoc -- you get variable interpolation and you don't hand-escape quotes.

### OSS vs Cloud Auth

- **OSS / on-prem Grafana**: create a service account, mint a token, set `auth = token`.
- **Grafana Cloud**: create a service account *in your Cloud Portal*, mint a token, and set `cloud_access_policy_token` for the Cloud-level API + `auth` for the Grafana-instance-level API. The provider has two provider configurations for this; see the docs.

**API keys are deprecated.** Grafana 10+ nags you loudly and auto-migrates existing keys to service-account tokens; Grafana 11+ hides the "create API key" button from the UI entirely; removal from the server has been telegraphed but, as of 2026, has not happened. Either way, no new code should mint an API key -- service accounts + tokens are the forward path. A service account has a role (Viewer/Editor/Admin), can be scoped to a single org, and its tokens can be rotated independently of the account.

---

## Part 8: Folders, Permissions, RBAC

The permissions model is a three-level hierarchy:

```
Organization  (tenant boundary; most self-hosted installs have just one)
  |
  +-- Folder
        |
        +-- Dashboard
        +-- Alert rule (lives in a folder too)
        +-- Library panel
```

The four built-in roles, inherited down the tree:

| Role | Can do |
|------|--------|
| **Viewer** | Read dashboards, run queries, view alerts |
| **Editor** | Viewer + create/edit dashboards and alerts in folders they have edit access to |
| **Admin** | Editor + manage folders, users, teams, data sources |
| **Grafana Admin** | Super-admin across the whole Grafana (org/server management) |

**Team** is the grouping primitive -- add users to teams, grant permissions to teams. Per-folder permissions override org-wide role for that folder.

### Grafana Enterprise / Cloud Fine-Grained RBAC

Enterprise and Cloud ship a **fine-grained RBAC** layer on top: roles like `datasources:creator`, `alert.rules:writer`, `dashboards:read` that can be stacked into custom roles. You reach for this when the four built-ins are too coarse -- for example, giving a team edit on dashboards but not on alert rules.

### The "Which folder?" Question

Every dashboard and alert rule has a folder. Folder permissions are your most-used RBAC lever. Design folders around **ownership**, not topology: `payments`, `search`, `platform` -- not `api`, `database`, `frontend`. A team should have edit access to one or two folders and view access to everything else.

---

## Part 9: The Alerting Split -- Grafana-Managed vs Data-Source-Managed

Back to mission control. Every flight has two kinds of red-line rules. Some are **burned into the spacecraft's own flight computer** -- the fuel pressure cutoff that trips whether or not comms are up with Houston. Others live in the **flight director's personal binder on the console in mission control** -- cross-system rules, like "if weather at the recovery zone is red AND the astronaut's pulse is >140, scrub." Grafana's two alerting flavors are exactly this split: data-source-managed rules live in the spacecraft (Prometheus/Mimir evaluates them locally, survives Grafana being down); Grafana-managed rules live in mission control's binder (Grafana evaluates them, can span any source including weather and voice-loop telemetry, but depend on Grafana being up).

This is the single most mis-understood thing about Grafana alerting, and it matters because Apr 23's deep dive taught you one flavor (data-source-managed, Prometheus + Alertmanager). Grafana introduces the second flavor -- the mission-control binder -- that lives next to it.

### The Two Paths Side by Side

```
                GRAFANA-MANAGED                      DATA-SOURCE-MANAGED
              (stored in Grafana SQL DB)           (stored in Prometheus / Mimir)
   +------------------------------+        +---------------------------------+
   |  Grafana                     |        |  Prometheus / Mimir ruler       |
   |                              |        |                                 |
   |  +-------- Rules ---------+  |        |  +---- PrometheusRule YAML --+  |
   |  |  PromQL / LogQL /      |  |        |  |  groups:                  |  |
   |  |  SQL / CloudWatch      |  |        |  |    - name: api            |  |
   |  |  (ANY data source)     |  |        |  |      rules:               |  |
   |  +------------------------+  |        |  |        - alert: ...       |  |
   |              |               |        |  +----------+----------------+  |
   |              v               |        |             |                   |
   |  +-------- Evaluator -----+  |        |             v                   |
   |  | Grafana alert engine   |  |        |  +-------- Evaluator -----+     |
   |  | queries data source,   |  |        |  | Prometheus / Mimir     |     |
   |  | evaluates threshold    |  |        |  | ruler evaluates        |     |
   |  +------------------------+  |        |  +------------------------+     |
   |              |               |        |             |                   |
   |              v               |        |             v                   |
   |  +-------- Routing -------+  |        |  +-------- Routing -------+     |
   |  | Grafana notification   |  |        |  | Alertmanager (external)|     |
   |  | policies + contact pts |  |        |  | routing tree           |     |
   |  | (GRAFANA'S OWN AM)     |  |        |  | (Apr 23 doc)           |     |
   |  +------------------------+  |        |  +------------------------+     |
   +------------------------------+        +---------------------------------+

              |                                           |
              v                                           v
          Slack / PagerDuty                         Slack / PagerDuty
```

### The Decision Framework

| You want... | Use |
|-------------|-----|
| Alerts on Prometheus + you already run Alertmanager | **Data-source-managed** -- don't move them |
| Alerts on Loki, CloudWatch, SQL, or mixed sources | **Grafana-managed** -- only one that can query those |
| The same alert rule evaluated by multiple Prometheus replicas for HA | **Data-source-managed** -- Prometheus ruler replicates natively |
| Tight coupling between a dashboard panel and its alert | **Grafana-managed** -- alerts can be authored from a panel |
| SLO burn-rate alerts driven by recording rules | **Data-source-managed** -- keep the rules next to the series |
| Alerts that span both Prometheus metrics AND a log count from Loki | **Grafana-managed** -- multi-datasource expression |

Many teams run **hybrid**: data-source-managed alerts in Prometheus/Mimir routed to Alertmanager for the Prometheus-native SLO burn-rate stack, plus Grafana-managed alerts for CloudWatch and Loki anomalies routed through Grafana's own notification policies. Both paths can even share contact points if you point Grafana-managed routing at an external Alertmanager (configurable per data source).

### The Alert State Machine

A Grafana-managed alert instance walks a state machine that a senior interviewer will ask you to draw. It is a superset of the Prometheus state model from Apr 23 plus two outcome-branch states for bad data:

```
                 (expr evaluates true for < `for` duration)
                 +------------------------------------------+
                 |                                          v
  Normal  ---->  Pending  ------(for elapses)---->  Alerting (aka Firing)
    ^              |                                        |
    |              +--(expr goes false before for elapses)  |
    |                                          |           |
    |                                          v           v
    +------------------------(expr goes false, resolve timer expires)---+

                   Side branches on query outcome:
                      expr returns empty  -> state derived from noDataState
                      expr errors         -> state derived from execErrState
                          values: NoData / Alerting / Normal / KeepLastState (+ Error for exec)
```

Key behaviors:

- **`for` is the anti-flap dwell time.** Same as Prometheus -- the rule must be true continuously for the full `for` duration before transitioning `Pending -> Alerting`. A single eval-true, eval-false flip resets Pending to Normal.
- **`keep_firing_for`** (Grafana 11+, mirrors the Prometheus field covered Apr 23) holds the alert in `Alerting` for an extra grace period after the expression goes false, preventing fluttery resolution notifications.
- **`missing_series_evals_to_resolve`** controls how many consecutive evaluations with a missing series count as "resolved" vs "paused." Default 2. Previously Grafana would resolve on first missing eval, causing spurious resolution on sparse data; this field is the fix.
- **NoData and Error are *separate input signals*, not separate states.** The `noDataState`/`execErrState` knobs translate the input into one of the four valid states (`NoData`, `Alerting`, `Normal`, `KeepLastState`). A rule with `noDataState: Alerting` will show as firing whenever its query returns empty -- which is exactly what you want for "prometheus is down" alerts and exactly what you don't want for "this metric is normally sparse."

### Recording Rules in Grafana-Managed Alerting

A common misconception: "Grafana-managed is alerts only; recording rules must stay in Prometheus." Not true since Grafana 10.2. You can define a **recording rule** in a Grafana rule group that runs on the same evaluator, writes its result back to any Prometheus-compatible source you configure as a write target, and is then usable by other alerts or dashboards. The Terraform resource is `grafana_rule_group` with a `rule` block where `is_paused = false` and the model has `refId` outputs routed to a recording target rather than a threshold expression. That said, for pure Prometheus-sourced SLO burn-rate stacks, keep recording rules data-source-managed (Prometheus/Mimir ruler). Reach for Grafana-managed recording rules when the source series come from Loki, CloudWatch, or mixed sources -- the same rationale as for Grafana-managed alerts.

### Why Grafana's Routing Looks Familiar

Grafana-managed notification policies reimplement the Apr 23 routing tree verbatim: `group_by`, `matchers`, `continue`, `receiver`. Same three timers (`group_wait`, `group_interval`, `repeat_interval`). Same inhibition rule concept under a different name (`mute timings` + `silences`). If you've internalized Apr 23, you've internalized 90% of Grafana alerting routing.

Reopen the mission-control frame here: a **data-source-managed** alert rule is a flight rule printed onto the onboard flight computer of a single spacecraft -- the spacecraft itself enforces it, even if mission control's link is down. A **Grafana-managed** rule is a flight rule in the *mission control director's personal binder* -- evaluated centrally, enforceable across any feed that comes in (Loki voice loops, CloudWatch weather, SQL supply-chain data), and lost when the binder is lost. Neither is strictly better; real missions carry both.

### The Multi-Source Empty-vs-Zero Trap

The most common production failure mode for cross-source Grafana-managed alerts is unique to this flavor and worth calling out explicitly. CloudWatch and Loki (and most SQL sources) return an **empty result**, not `0`, when nothing matches their query. If you write a rule like "alert when GuardDuty findings > 0 AND auth failures > 100," both arms of the rule sit empty most of the time, the Math expression has nothing to combine, and the rule parks in `NoData` -- not in `Normal`. Pager fatigue follows. Three fixes, used together:

1. **Set `noDataState` explicitly** (don't rely on the default). `Normal` means "absence of data = healthy," which is almost always what Security actually wants for "fire-when-both" detection rules.
2. **Use `Fill missing = 0` in the query options panel.** This coerces empty results into a `0` time series so Reduce/Math expressions have something to compute. CloudWatch and Loki sources support this; Prometheus generally doesn't need it because `rate()` over a non-existent series is already empty-but-typed.
3. **Route `execErrState` to a separate, lower-priority channel.** "My alert evaluator couldn't reach CloudWatch" is a real problem, but it's a *platform* problem, not a *Security* problem. Send it to the platform team's inbox, not to the on-call pager.

The rule shape for a multi-source AND-composition is canonical:

```text
Query A -> CloudWatch: GuardDuty HighSeverityFindingCount (period=5m, stat=Sum, Fill missing=0)
Query B -> Loki:       sum(count_over_time({service="auth"} |= "authentication failure" [5m]))

Expr  C -> Reduce(A, last)
Expr  D -> Reduce(B, last)
Expr  E -> $C > 0 && $D > 100     <- alert condition

for: 0s
noDataState: Normal
execErrState: Error -> routed to platform-oncall, not security-oncall
labels: { severity: critical, team: security }
```

The three wrong answers you will hear in design review and how to refute them:

- **"Write a Prometheus recording rule that combines them."** Prometheus can't see CloudWatch or Loki natively; you'd stand up `cloudwatch-exporter` + a Loki-to-Prometheus metric bridge just to replicate what Grafana-managed does for free. Recording rules aren't alerts anyway.
- **"Write two alerts and inhibit one with the other."** Inhibition *suppresses* B when A is firing -- that is the logical opposite of AND. Two alerts with no inhibition is OR. Neither shape produces "both true in the same window."
- **"Use Grafana-managed, but skip the Fill-missing step -- we'll just set `for: 5m`."** Doesn't help: `NoData` isn't `Normal`, so `for` never ticks and you're stuck in NoData state regardless of how long you wait.

---

## Part 10: Modern Features Worth Knowing

- **Public dashboards**: read-only, anonymous-viewable share URLs (with optional time-range locking and email-gated access). Great for a status-page-style board for customers; dangerous for anything internal. Analogy: the giant public countdown clock bolted to the outside wall of mission control.
- **Library panels**: a panel definition stored once and referenced by many dashboards. Edit the library panel, every dashboard that uses it updates. The "component library" of dashboarding.
- **Dynamic dashboards (GA in Grafana 13)**: tabs, nested rows, conditional panels, and a responsive layout engine built on Scenes + schema v2. Replaces the "one giant grid with 200 panels" design with tabbed drill-downs. In 13 these went from preview to default; new dashboards get the dynamic layout automatically.
- **Git Sync (GA in Grafana 13)**: 2-way dashboard sync to a Git repo with PR-based review. Works against GitHub, GitLab, Bitbucket, or any Git remote, with GitHub App authentication (no personal tokens). The eventual successor to file-based provisioning for cloud-native shops. Frame: the run-book lives in a versioned offsite safe; mission control becomes a read-only projector of what the safe contains.
- **Grafana Advisor (GA in Grafana 13)**: periodic automated health checks scanning every data source, plugin, SSO integration, and misconfiguration, with AI-assisted remediation suggestions. The QA inspector walking the mission-control floor before every shift checking wiring and labels. Run in production; this is the single lowest-effort governance win in a Grafana 13 upgrade.
- **Grafana Assistant (OSS + Enterprise in Grafana 13)**: AI agent embedded in the UI for dashboard-layout suggestions, PromQL/SQL refinement, and natural-language onboarding help. Previously Cloud-only; now available across editions. Reach for it when bootstrapping a new service's dashboard or when someone hands you a panel query that's "almost right."
- **Dashboard layout templates (Grafana 13)**: prebuilt layouts following DORA (deploy/lead/time-to-restore/change-failure), USE (utilization/saturation/errors), and RED (rate/errors/duration) methodologies. The "blank-page problem" killer -- new dashboards start with a layout that already encodes a sound signal hierarchy.
- **Recorded queries (Cloud)**: materialize an expensive query's output as a new time series, stored in Prometheus at a cheap cadence, so every dashboard opening doesn't re-derive it from raw telemetry (see Part 1).
- **Query caching (Enterprise / Cloud)**: a datasource-level TTL cache for query results, with cache keys that incorporate query text, time range, and variables. Set per data source (default 60s, up to the plan's max) with optional cache-tagging so specific panels can opt out. The simplest single lever for keeping a 500-panel status-page dashboard from melting Mimir at refresh time. Pairs naturally with recorded queries: cache the cheap stuff, record the expensive stuff.
- **Grafana API**: everything the UI does goes through the HTTP API. Service account tokens authenticate; dashboard `POST /api/dashboards/db` imports; snapshots, annotations, alerts, folders, users all have endpoints. The Terraform provider is a thin layer over this.

### What's New in Grafana 13 (April 21, 2026) -- the Interview One-Liner

Grafana 13 launched at GrafanaCON 2026 Barcelona. Four things matter for a senior-level conversation, in rough order of blast radius:

1. **Dynamic Dashboards went GA.** Tabs, nested rows, conditional panels, responsive layout -- the end of "200-panel grid" dashboards. Auto-migration from v1 JSON on first open; existing Terraform and API workflows keep working.
2. **Git Sync went GA with 2-way sync.** Authored dashboards sync to a Git repo as PRs; merges flow back to Grafana. Works with GitHub/GitLab/Bitbucket/Git, with native GitHub App auth (no personal tokens). The first time file-based provisioning has had a real successor.
3. **Grafana Advisor went GA.** Periodic health checks across data sources, plugins, SSO, and misconfig -- the single highest-ROI governance feature to enable on day one.
4. **Grafana Assistant is now in OSS + Enterprise.** Previously Cloud-only AI agent for dashboard/query help. Useful but not load-bearing; know it exists.

Plus, for bonus points: dashboard layout templates for DORA/USE/RED (kill the blank-page problem), updated Gauge panel GA, service-topology mapping (metrics pinned to a topology graph), and **Alloy's new OpenTelemetry Engine mode** -- Alloy can now be configured with standard OTel Collector YAML, making it drop-in for any OTel Collector deployment (relevant when the Apr 29 OpenTelemetry fundamentals doc lands).

What did *not* change: Alertmanager / Grafana-managed alerting split (all of Part 9 still applies), Prometheus pull-model, exemplars, the SQL DB as backplane, provisioning mechanics. The mental model from the first 11 parts of this doc is unaffected by the 12 -> 13 bump; these are additive capabilities layered on top.

### Enterprise-Only Features Worth Knowing Exist

Senior interviews occasionally ask whether you've run Enterprise or Cloud. You don't need deep experience, but you should be able to name the feature set:

- **Fine-grained RBAC**: custom roles composed from atomic permissions like `datasources:read`, `alert.rules:write`, `folders:edit`. Covered in Part 8.
- **Per-datasource permissions**: restrict a datasource to specific teams/users, not just folder-scoped. Useful when a single Grafana fronts multi-tenant data sources.
- **Reporting**: scheduled PDF/CSV renders of a dashboard, emailed to stakeholders. The "executive weekly health report" feature.
- **SAML SSO + Team Sync**: auto-map SSO group membership to Grafana teams. Users get the right folder permissions the first time they log in.
- **Usage Insights**: "which dashboards does anyone actually open?" -- a queryable audit log of dashboard views, folders opened, queries run. Used to sunset stale dashboards.
- **Enterprise plugins**: Splunk, Datadog, New Relic, Snowflake, Oracle DB datasources gated behind the Enterprise/Cloud SKU.
- **White-labelling**: custom logos, login page, color scheme. Platform-team-as-a-product signal.

---

## Part 11: Production Gotchas

Fifteen that bite real teams, in the order they usually bite you:

1. **Editing a provisioned dashboard in UI silently fails.** You click Save, see a success toast, close the tab. On next reconcile (30s later), Grafana rewrites the file-version over your changes. `allowUiUpdates: false` is loud about this; pair providers with it and teach your team "UI is read-only in prod."
2. **Data source UID vs name.** Dashboards reference data sources by **UID**, not name. Changing a data source's name is fine; changing its UID breaks every dashboard that uses it. Pick stable UIDs (`prom-prod-use1`, not a random 20-char hash) and set them explicitly in provisioning YAML and Terraform.
3. **`$__interval` instead of `$__rate_interval` on `rate()` panels.** Zoom out past a few hours, the interval exceeds `4 * scrape_interval`, and suddenly `rate()` returns empty. Fix: use `$__rate_interval` everywhere, and set `timeInterval` on the Prometheus data source to match your actual scrape interval.
4. **Dashboards exported from UI, imported via Terraform, then re-edited in UI.** The UI-saved JSON schema-drifts (Grafana adds new fields, rewrites booleans, reorders keys) against the Terraform source. Every apply shows a 300-line diff for cosmetic differences. Fix: use `lifecycle { ignore_changes = [config_json] }` pragmatically, or commit to pure file-based provisioning, or use Grafonnet/CUE to generate the JSON deterministically.
5. **Service account token scoped too broadly.** A Terraform CI SA with Grafana Admin across all orgs can nuke everything on a typo. Create SAs per-folder or per-org where possible; the fine-grained RBAC roles in Enterprise/Cloud let you issue a token that can only write to one folder.
6. **Repeated panels are expensive.** A `pod=All` variable repeating a panel across 200 pods generates 200 panel queries on every dashboard render, every refresh. Even if each is cheap, the aggregate is not. Prefer a single table panel with per-row values, or cap the repeat at 20 and offer drill-downs.
7. **Variable interpolation inside alert queries.** Dashboard variables do **not** exist when alerts evaluate (the alert engine has no panel, no dashboard, no user selecting a dropdown). If you copy a panel query like `rate(http_requests_total{service=~"$service"}[5m])` into an alert, the literal string `$service` survives into PromQL -- and the failure mode is *worse* than you think. With the regex-match operator `=~`, Prometheus parses `$service` as "end-of-string anchor (`$`) followed by the literal letters `service`." Since nothing follows end-of-string, the regex matches *zero series*, not an empty label value. Your query returns empty, the rule parks in `NoData` state, and the original alert condition is never evaluated against real data -- you get silence during real incidents and `DatasourceNoData` noise the rest of the time. Fix: hardcode values, interpolate at Terraform apply-time via `${var.service}`, or use multi-dimensional alerting with `sum by (service)` so one rule emits one alert instance per service. For multi-service rules, Terraform `for_each` is reserved for cases where the *threshold or logic itself* differs between services, not just the label value.
8. **Transformations hide alerting truth.** If you compute error rate in a `Calculate field` transform rather than in PromQL, a "create alert from this panel" flow uses the *raw query*, not the transformed output. The alert fires on something different from what the panel shows. Move the math to PromQL before alerting.
9. **Grafana-managed rules fired from a paused data source still show as firing.** `noDataState` and `execErrState` control what happens when the query returns empty or errors. The valid `noDataState` values are `NoData` (default -- fires a distinct `DatasourceNoData` alert), `Alerting` (treat missing data as firing), `Normal` (treat missing data as resolved), and `KeepLastState` (stay firing/resolved as you were; this is how you end up stuck-firing after a Prometheus outage). `execErrState` takes the same values plus `Error`. Pick deliberately: `Normal` for "lack of data means no alert," `Alerting` for "lack of data is itself a problem," and only `KeepLastState` when you know what you're asking for. **UI-label-vs-API-value gotcha:** in older Grafana (pre-10) the UI dropdown labelled the `Normal` value as "OK," and that label still shows up in third-party tutorials and community posts. In current YAML / Terraform / API, the value is `Normal`. If you see docs or answers saying "set noDataState to OK," translate to `Normal` before applying.
10. **Dashboard JSON model drift between Grafana versions.** Upgrading Grafana silently rewrites dashboards on first open to normalize them to the new schema. If you're storing the pre-upgrade JSON in Terraform, the first post-upgrade apply tries to roll them back. Plan to re-export canonical JSON after every major upgrade.
11. **Mixed data source queries fail silently for non-time-aligned results.** If query A returns every 15s and query B returns every 1m, the join-by-time transformation drops unaligned points and your overlay panel shows an incomplete picture. Three fixes, in order of preference: (1) set the panel's **Min interval** to the coarser cadence (`1m` in this example) so both sources bucket to the same step; (2) add a **Merge** transform which inner-joins on timestamp rather than field-by-field; (3) for panels that must combine Prometheus and SQL, use `$__interval_ms` in the SQL query to bucket its `GROUP BY time()` to Grafana's chosen step -- this forces both sides to the same grid.
12. **Public dashboards leak multi-tenant data.** A public dashboard bypasses user auth but **not** data-source tenant headers -- unless you've set them as default headers on the data source, in which case every public viewer sees them. Double-check any data source used by a public dashboard has no per-user context baked in.
13. **Grafana's own SQL DB is a SPOF.** Dashboards, users, alert rules, silences all live there. Losing SQLite or Postgres = losing Grafana state. Run Postgres in HA, back it up, and keep provisioning-as-code as the disaster-recovery source of truth.
14. **Alert rule UIDs collide across environments.** If dev and prod both provision an alert rule with uid `checkout-burn-rate-fast`, they're fine (separate Grafana instances). But a shared Grafana with multi-org can have two orgs' rules collide. Prefix UIDs with env/org when unsure.
15. **Time range "Browser" vs "UTC" is a debugging nightmare.** A user with browser TZ = PST looks at a 3 AM UTC incident and sees 7 PM previous-day. When correlating across tools, pin dashboards (or the user account) to UTC and document it.

---

## Part 12: Decision Frameworks

### Framework 1: Grafana-Managed vs Data-Source-Managed Alerting

```
Is the alert on a non-Prometheus source (Loki, CloudWatch, SQL, mixed)?
    YES -> Grafana-managed (only option)
    NO  -> continue

Do you already run Alertmanager + Prometheus rule files in Git?
    YES -> Data-source-managed (keep the stack uniform)
    NO  -> continue

Do you want the alert authored next to a dashboard panel?
    YES -> Grafana-managed (UI-first authoring flow)
    NO  -> Data-source-managed (IaC-first flow)
```

### Framework 2: File Provisioning vs Terraform vs UI Export

```
Is Grafana running as a Kubernetes pod next to your app?
    YES -> File provisioning via ConfigMaps (+ FluxCD/ArgoCD to sync)

Is Grafana running as SaaS (Grafana Cloud)?
    YES -> Terraform (no filesystem to mount)

Is Grafana running on a VM / shared with other orgs?
    YES -> Terraform (cleaner separation)

Is the dashboard a one-off experiment?
    YES -> UI, but *export the JSON immediately* and commit it before it's lost
```

### Framework 3: Scenes / Schema v2 Adoption

```
Are you on Grafana < 11?
    YES -> Stick with legacy; no Scenes available
    NO  -> continue

Do you want dynamic dashboards (tabs, conditional panels)?
    YES -> Schema v2, but opt-in per dashboard, starting in dev folder
    NO  -> Scenes is on by default; you're already using it transparently

Are you running the Grafana Operator on Kubernetes?
    YES -> Schema v2 + Git Sync is the path forward; evaluate it now
    NO  -> Schema v1 remains fine; watch schema v2 maturity for six months
```

### Framework 4: Transformations vs Query-Side Computation

```
Does the computation change the BYTES leaving the data source?
    YES -> Do it in the query (PromQL `sum by`, `rate`, `/`)
    NO  -> Do it in a transformation

Is the computation feeding an alert condition?
    YES -> Do it in the query, always
    NO  -> transformation is safe

Are you joining two different data sources?
    YES -> Transformation (Merge, Join by field) is the only option
```

---

## The 10 Commandments of Grafana

1. **Thou shalt not edit provisioned dashboards in UI.** The run-book is the source of truth.
2. **Thou shalt pick stable UIDs and never change them.** They are forever handles.
3. **Thou shalt use `$__rate_interval` for every `rate()` in every panel.** `$__interval` is a trap at zoom-out.
4. **Thou shalt design dashboards top-down for signal hierarchy.** Stats on top, RED/USE next, details below.
5. **Thou shalt not use pie charts.** Bar gauges and tables always beat them.
6. **Thou shalt use service accounts + tokens, never API keys.** API keys are dead.
7. **Thou shalt compute alerting logic in the query, not in a transformation.** Alerts don't see transforms.
8. **Thou shalt design folders around ownership, not topology.** Teams map to folders, not systems.
9. **Thou shalt keep data in the data source and questions in Grafana.** Grafana is a read model over telemetry; don't try to make it store more.
10. **Thou shalt treat the SQL DB as volatile and the Git repo as durable.** Anything you can't rebuild from Git in 15 minutes is a latent outage.

---

## Further Reading

- Grafana Labs: [Best Practices for Dashboards](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
- Grafana Labs: [Provisioning Grafana](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- Grafana Labs: [Variables and Templating](https://grafana.com/docs/grafana/latest/dashboards/variables/)
- Grafana Labs: [Transform Data](https://grafana.com/docs/grafana/latest/panels-visualizations/query-transform-data/transform-data/)
- Grafana Labs: [Grafana Dashboards Are Now Powered by Scenes](https://grafana.com/blog/2024/10/31/grafana-dashboards-are-now-powered-by-scenes-big-changes-same-ui/)
- Grafana Labs: [Grafana 13 Release -- All the Latest Features (April 2026)](https://grafana.com/blog/grafana-13-release-all-the-latest-features/)
- Grafana Labs: [What's New in Grafana (canonical index)](https://grafana.com/docs/grafana/latest/whatsnew/)
- Alexandre Vazquez: [Alertmanager vs Grafana Alerting](https://alexandre-vazquez.com/alertmanager-vs-grafana-alerting/)
- Grafana Labs: [Terraform Provider for Grafana](https://grafana.com/docs/grafana/latest/as-code/infrastructure-as-code/terraform/)
- This repo, Apr 23: [Alertmanager & Alerting Strategy](../prometheus/2026-04-23-alertmanager-alerting-strategy.md) -- Grafana-managed alerting's routing tree is the same mental model with different storage.
- This repo, Apr 21: [PromQL Deep Dive](../prometheus/2026-04-21-promql-deep-dive.md) -- every Grafana variable that interpolates into a panel query is feeding PromQL you already know.
- This repo, Apr 20: [Prometheus Fundamentals](../prometheus/2026-04-20-prometheus-fundamentals.md) -- `timeInterval` on the data source is just your scrape interval surfaced to Grafana.
