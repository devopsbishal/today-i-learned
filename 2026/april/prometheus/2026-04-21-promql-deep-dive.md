# PromQL Deep Dive -- Thinking in Vectors, Rates, and Labels

> Yesterday we built the warehouse: the milkman's route, the leather ledger, the scrape pipeline, the data model, the TSDB. Today we open the ledger and read it. PromQL is not a query language the way SQL is a query language -- it doesn't return rows, it returns **vectors of labeled time series**, and every operator has to know what to do when two such vectors meet. The single mental habit that separates "I kind of know PromQL" from "I can debug PromQL at 3 AM" is: **at every sub-expression, know whether you're holding an instant vector, a range vector, or a scalar**. That one habit dissolves 80% of the confusing errors Prometheus ever throws at you.
>
> The core analogy for the day: PromQL queries are **photography**. An instant vector is a **polaroid photo** of every series at one moment -- each series in the frame once, wearing its label combination. A range vector is a **time-lapse film strip** -- same series, but instead of one value per series you get a stack of `(timestamp, value)` samples from a window in the past. Functions like `rate()` take a film strip as input and return a polaroid (speed at this instant, averaged across the strip). Aggregations like `sum()` reshape a polaroid by stacking series with matching labels. Operators like `*` match two polaroids side by side and multiply them photo-for-photo, one label combination at a time. The grammar of PromQL is "what shape goes into each function, what shape comes out, and how do labels flow through the join." Everything else -- `rate()` vs `irate()`, `on()`/`ignoring()`, subqueries, recording rules -- is a consequence of that grammar.

---

**Date**: 2026-04-21
**Topic Area**: prometheus
**Difficulty**: Intermediate / Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Instant vector | A polaroid photo of every series at time `t` | One value per series, all at the same instant |
| Range vector | A time-lapse film strip over a window `[5m]` | A stack of samples per series, spanning a time range |
| Scalar | A single number | `42`, `time()`, the output of `scalar(vector)` |
| `rate()` | Average speed across a highway stretch | Per-second average of a counter over a range window, reset-aware |
| `irate()` | Speedometer reading at this instant | Last-two-samples slope; noisy; wrong for alerts and dashboards |
| `increase()` | Odometer delta over the window | `rate() × range_seconds`; same extrapolation math, wears a different hat |
| Rate-then-sum rule | Three pedometers with independent battery resets -- rate each first, then sum | Apply `rate()` per-series BEFORE summing; never sum first |
| Label matching | SQL JOIN `ON (columns)` | Operators pair series by label sets; mismatches silently drop |
| `on()` / `ignoring()` | `JOIN ... USING (col)` vs `JOIN ... USING (all except col)` | Choose which labels participate in the join |
| `group_left` / `group_right` | SQL many-to-one JOIN -- left table's FK, right table's labels copied onto output | Many-to-one enrichment; one-side labels are projected onto the other |
| Aggregation (`sum by`) | GROUP BY in SQL | Reduces labels; without `by`/`without` you collapse to a single series |
| `histogram_quantile()` | Read the p99 from the shoe-rack sizes after counting sales | Linear interpolation inside the bucket that crosses the target quantile |
| Native histogram | A single sparse-bucket jar instead of a rack of jars | One series per histogram; `histogram_quantile()` works directly, no `by (le)` |
| Subquery `[5m:30s]` | Running a second camera over the first camera's output | Materialize a range vector from an instant-vector expression |
| Recording rule | Meal prep on Sunday vs cooking from scratch every guest | Pre-compute expensive queries on a schedule; store as new series |
| `offset 1w` | Looking at the same photo from a week ago | Shifts the query's view of time backwards; negative values (`offset -5m`) shift forward (rare) |
| `@ <ts>` modifier | A postmarked envelope stamped with a specific date | Anchors evaluation to a specific unix timestamp; `@ start()` / `@ end()` in rules |
| `absent()` / `absent_over_time()` | "The house didn't put bottles out today" alert | Returns a series ONLY when the input is empty |
| `label_replace()` | `UPDATE ... SET` run at query time | Rewrites labels using a regex; silent no-op on mismatch |
| `predict_linear()` | "At this rate, when will the gas tank hit zero?" | Linear extrapolation from a range window; the disk-full-in-4h classic |
| RED method | Rate / Errors / Duration -- the service checklist | Three queries per service: traffic, failure %, latency p99 |
| USE method | Utilization / Saturation / Errors -- the resource checklist | For CPU/memory/disk/network: how busy, how queued, how broken |
| `bool` modifier | Turn "is it greater?" into a 1/0 column | Comparison returns a boolean series instead of filtering |
| `absent_over_time` vs `absent` | "Haven't seen the house all week" vs "Not there right now" | Range version guards against transient scrape gaps |

---

## Part 1: The One Mental Model -- Vector Types

If you internalize nothing else today, internalize this: **every PromQL expression has a type, and the type determines what you're allowed to do with it**. Prometheus has four value types. Two you care about constantly, two you rarely think about:

| Type | Shape | Example |
|------|-------|---------|
| **Instant vector** | A set of series, each with exactly **one** value at the evaluation timestamp | `http_requests_total`, `rate(...[5m])`, `sum(...)` |
| **Range vector** | A set of series, each with a **stack** of samples spanning `[range]` | `http_requests_total[5m]` |
| **Scalar** | A single `float64` | `42`, `time()`, `scalar(count(up))` |
| **String** | A literal string (used only in a few functions like `label_replace`) | `"handler"` |

The two big ones -- instant and range -- are the polaroid vs the film strip.

### Why It Matters -- Function Signatures

Every PromQL function has a signature like `rate(range-vector) -> instant-vector` or `sum(instant-vector) -> instant-vector`. If you feed the wrong type in, you get a parser error. The most common error messages you'll see in your life:

```promql
# WRONG -- rate() wants a range vector, you gave it an instant vector
rate(http_requests_total)
# parse error: expected type range vector in call to function "rate",
# got instant vector

# WRONG -- sum() wants an instant vector, you gave it a range vector
sum(http_requests_total[5m])
# parse error: expected type instant vector in aggregation expression,
# got range vector
```

Read those two errors again. Every PromQL mistake you'll ever make when starting out is one of those two, phrased slightly differently. Pin them to your forehead.

### The Canonical Shape

An instant vector at time `t = 1713700000` looks like this internally:

```
http_requests_total{method="GET",  handler="/api/users", status="200"}  @ 47523
http_requests_total{method="GET",  handler="/api/users", status="500"}  @ 12
http_requests_total{method="POST", handler="/api/users", status="201"}  @ 1843
```

One sample per series, all at the same timestamp. A polaroid.

A range vector `http_requests_total[1m]` at the same `t` looks like this:

```
http_requests_total{method="GET",  handler="/api/users", status="200"}  @ [
    (t-60s, 47510), (t-45s, 47514), (t-30s, 47518), (t-15s, 47521), (t, 47523)
]
http_requests_total{method="GET",  handler="/api/users", status="500"}  @ [
    (t-60s, 11),    (t-45s, 11),    (t-30s, 12),    (t-15s, 12),    (t, 12)
]
...
```

Each series carries a full strip of samples from the last 60 seconds at the scrape resolution. On a 15s scrape, that's roughly 4 samples per series for a `[1m]` range, roughly 20 samples for `[5m]`, and so on (missed scrapes reduce the count).

You cannot plot a range vector in Grafana. Grafana issues a `query_range` request -- Prometheus evaluates the *instant-vector expression* at each step across the panel's time range, server-side, and returns a series of points. If you type `http_requests_total[5m]` as your panel query, you'll get an "expected type instant vector" error -- because `[5m]` is itself a range vector and the server can't turn it into a plottable time series until you wrap it in something that reduces it to an instant vector (like `rate(...[5m])`).

### The Type Flow Through a Realistic Query

Take the canonical RED-method p99 latency query:

```promql
histogram_quantile(
  0.99,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Walk the types outward from the innermost expression:

1. `http_request_duration_seconds_bucket` -- **instant vector** (all buckets, all services, one value each).
2. `http_request_duration_seconds_bucket[5m]` -- **range vector** (5 minutes of samples per bucket series).
3. `rate(...[5m])` -- **instant vector** (per-second average rate per bucket series).
4. `sum by (le, service) (...)` -- **instant vector** (per-service, per-bucket, summed across instances).
5. `histogram_quantile(0.99, ...)` -- **instant vector** (one p99 per service).

The type flow is: instant -> range -> instant -> instant -> instant. If any one of those steps had a type mismatch, the query would fail to parse. The grammar forces correctness.

**The habit**: for any query you write or read, say the type of each sub-expression out loud. Within a week you'll stop making type errors.

---

## Part 2: Selectors & Label Matchers

A **selector** is the base unit of PromQL -- it picks time series out of the TSDB by their metric name and label values. Everything else is built on top.

### The Four Matcher Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Exact equality | `{status="200"}` |
| `!=` | Not equal | `{status!="200"}` |
| `=~` | Regex match (RE2 syntax, anchored automatically) | `{status=~"5.."}` |
| `!~` | Regex does not match | `{handler!~"/health.*"}` |

Important detail: regex matchers are **fully anchored** implicitly. `{status=~"5.."}` is effectively `^5..$`, so it matches "500", "503", "599" but NOT "1500". If you want partial matching, use `.*` explicitly (`status=~".*5.*"`).

### The Empty-Matcher Rule (Memorize This)

There are actually **two separate rules** at work here that most docs conflate. Get them apart and the behavior stops being surprising.

**Rule 1 -- Empty-matching matchers select absent labels too.** From the Prometheus docs verbatim: *"Label matchers that match empty label values also select all time series that do not have the specific label set at all."* Prometheus does not distinguish "label absent" from "label present but empty" -- both look like `""` at query time.

**Rule 2 -- Every vector selector must have at least one matcher that does NOT match the empty string.** This is the "no bare scans" rule; it's why `{}` alone is illegal.

The combination produces these behaviors:

- `{job=""}` -- matches series without the `job` label (or with `job=""`). Legal alone? No -- this matcher matches empty, so you need another non-empty-matching matcher. Combine with a metric name: `up{job=""}`.
- `{job!=""}` -- matches series with a non-empty `job` label. Legal alone because `!=""` by definition does not match empty.
- `{job=~".*"}` -- **also matches absent `job`**, because `.*` matches the empty string and Rule 1 applies. Illegal alone (matches empty). Once combined with a metric name or another non-empty-matching selector, it does include series where `job` is absent.
- `{job=~".+"}` -- requires `job` to be present AND non-empty (`.+` requires at least one character). Legal alone because `.+` does not match empty.
- `{job!~".+"}` -- matches series missing the `job` label (or with empty value). Matches empty, so needs another matcher to be legal alone.

The mental rule in plain English: `=""`, `=~""`, `=~".*"`, and `!~".+"` all match "label absent." `!=""` and `=~".+"` require the label to be present. Selectors need at least one matcher from the second group (or a metric name) to be legal on their own.

### Selecting Across Metric Names

The metric name itself is really just a reserved label called `__name__`. You can match on it:

```promql
# All metrics that start with "http_"
{__name__=~"http_.*"}

# All counters in a specific job
{__name__=~".*_total", job="api"}
```

This is how `topk(10, count by (__name__)({__name__=~".+"}))` -- the cardinality audit query -- works: it counts the number of series per metric name across everything in the TSDB. Tattoo that query on the inside of your wrist; you will run it once a week for the rest of your career.

**Gotcha**: bare `{}` with no matcher at all is forbidden. This is Rule 2 above -- every vector selector needs at least one matcher that does not match the empty string. `{__name__=~".+"}` works because `.+` requires at least one character. The rule isn't technically about "scan prevention" (you can still write queries that touch the whole TSDB), it's a guard against accidentally writing a matcher that matches literally everything in the world.

### Time Range on Selectors

Appending `[duration]` to a selector upgrades it from instant vector to range vector:

```promql
http_requests_total[5m]    # Last 5 minutes of samples
http_requests_total[1h]    # Last 1 hour of samples
```

Durations use `s`, `m`, `h`, `d`, `w`, `y` (years = 365d). Since Prometheus 2.26, compound durations are valid: `1h30m`, `2d12h`, `1w3d` all parse correctly. Earlier versions required single-unit durations; if you're on an old Prometheus, stick to `90m` instead of `1h30m`.

---

## Part 3: Operators -- Arithmetic, Comparison, Logical

### Arithmetic (`+ - * / % ^`)

Element-wise between two vectors (matching labels), or between a vector and a scalar.

```promql
# Convert bytes to gigabytes
node_memory_MemTotal_bytes / 1024 / 1024 / 1024

# Percent of memory used
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
  / node_memory_MemTotal_bytes * 100

# Convert seconds to milliseconds (for dashboards where SI units aren't ideal)
histogram_quantile(0.99, sum by (le) (rate(duration_seconds_bucket[5m]))) * 1000
```

Division by zero returns `NaN` for the affected series, not an error -- the rest of the vector is unaffected.

### Comparison (`== != > < >= <=`) and the `bool` Modifier

By default, comparison operators are **filters**: they return only the series that satisfy the condition.

```promql
# Instances with more than 500 requests per second
rate(http_requests_total[5m]) > 500

# Error ratio above 1%
sum by (service) (rate(errors_total[5m])) / sum by (service) (rate(requests_total[5m])) > 0.01
```

The second query is really what you'd use as the `expr:` of an alerting rule -- Alertmanager fires when the expression returns any series. If the error ratio is below 1%, the expression returns the empty set and no alert fires.

Now the `bool` modifier. If you want comparison to return **1 or 0** instead of filtering, slap `bool` on:

```promql
# Returns 1 if above threshold, 0 if below -- for use in math, not alerts
(rate(http_requests_total[5m]) > bool 500)

# Count of instances over threshold
count(rate(http_requests_total[5m]) > bool 500)
```

`bool` is how you build up SLI math where you want to count "how many minutes was I above threshold" rather than filter to the violating instances.

### Logical/Set Operators (`and`, `or`, `unless`)

These operate on the **intersection** or **union** of label sets, not the values. Think of them as set operations on series identity.

- `a and b` -- series present in both `a` and `b` (with `a`'s values). Same label sets in both.
- `a or b` -- series in either (if present in both, `a`'s value wins).
- `a unless b` -- series in `a` that are NOT present in `b`.

A truth-table example. Suppose:

```
a = { {svc="api"}: 5, {svc="web"}: 3 }
b = { {svc="api"}: 99, {svc="db"}: 7 }
```

Then:

```
a and b    -> { {svc="api"}: 5 }                   # intersection, a's value
a or b     -> { {svc="api"}: 5, {svc="web"}: 3, {svc="db"}: 7 }   # union, a wins ties
a unless b -> { {svc="web"}: 3 }                   # in a, not in b
```

`unless` is what you reach for when you want "alert if X is true but Y is also true" -- `HighErrorRate unless DeploymentInProgress` is the classic. Logical operators support `on()` / `ignoring()` / `group_*` just like arithmetic.

Real pattern:

```promql
# Alert on high error rate BUT suppress during active deploys
sum by (service) (rate(errors_total[5m])) / sum by (service) (rate(requests_total[5m])) > 0.01
  unless on (service)
    kube_deployment_status_replicas_updated != kube_deployment_spec_replicas
```

---

## Part 4: Vector Matching -- The SQL JOIN Moment

This is where PromQL gets genuinely different from anything else you've written, and where every non-trivial query lives or dies. Vector matching is how PromQL decides which series on the left side of an operator talk to which series on the right side.

The mental model: **every binary operation is a JOIN**. The question is always: which label columns are the join keys?

### One-to-One Matching (Default)

By default, PromQL matches two vectors when their **full label sets are identical** (excluding the metric name, which is always dropped by binary operators).

```promql
# Only works cleanly if both sides have the same label combinations
node_filesystem_avail_bytes / node_filesystem_size_bytes
# -> fraction of free space per (instance, mountpoint, device, fstype) combo
```

Both metrics come from `node_exporter`, both have the same labels on each series, so the join is automatic. This is the happy path.

### When Labels Don't Line Up -- `on()` and `ignoring()`

Two metrics rarely have identical label sets in the real world. You need to tell PromQL **which subset of labels to use as the join key**.

- `on(labels...)` -- match only on these labels (like SQL `JOIN ... USING (these_columns)`).
- `ignoring(labels...)` -- match on everything EXCEPT these labels.

Example. Suppose you have:

```
kube_pod_info{pod="api-abc", node="node1", namespace="default"}                      = 1
kube_pod_container_resource_requests{pod="api-abc", container="api", resource="cpu"} = 0.5
```

The label sets differ (`node` exists on `kube_pod_info`, `container` + `resource` exist on `kube_pod_container_resource_requests`). A naive multiplication fails with "no matching series":

```promql
kube_pod_info * kube_pod_container_resource_requests   # FAILS
```

The fix is to say "match on `pod` alone":

```promql
kube_pod_container_resource_requests * on (pod) kube_pod_info
```

This multiplies each container's resource request by the (always-`1`) `kube_pod_info` value with the same `pod` label -- effectively a no-op multiplication whose purpose is to drag a label along (see next section).

### Many-to-One -- `group_left` / `group_right`

A single pod has multiple resource-request series (one per container × resource). But only one `kube_pod_info` entry per pod. That's a **many-to-one** relationship. Default one-to-one matching throws "multiple matches for labels" because multiple left-side series map to the same right-side series.

`group_left` tells PromQL "the left side may have many matches per right side; that's fine, and also copy these labels from the right onto the output":

```promql
kube_pod_container_resource_requests
  * on (pod) group_left (node)
  kube_pod_info
```

Now the output has one series per container × resource, **carrying the `node` label from `kube_pod_info`**. You've just enriched container-level metrics with node information -- the classic pattern for building "CPU requests per node" dashboards:

```promql
sum by (node) (
  kube_pod_container_resource_requests{resource="cpu"}
    * on (pod) group_left (node) kube_pod_info
)
```

**`group_left` vs `group_right`**: same idea, mirrored. `group_left` means "left side has the many, right side has the one, project right's extra labels onto the output." `group_right` is the reverse. SQL analogy: `group_left` is "left JOIN where left has FK to right."

**Gotcha**: if you forget `on()` / `ignoring()`, or list the wrong labels, you'll get one of two classic errors:
- `many-to-many matching not allowed` -- PromQL can't figure out which side is "the one."
- `multiple matches for labels: grouping labels must ensure unique matches` -- your join key doesn't uniquely identify series on the "one" side; add more labels to `on()` or fewer to `ignoring()`.

### The Label Projection Rule

Binary arithmetic operators **drop the metric name** from the output. The output is labeled but nameless. That's why `a / b` doesn't produce a metric called "a_divided_by_b" -- it produces series with just labels. If you want a name back, use `label_replace(..., "__name__", "my_metric", "", "")` or (better) put the expression into a recording rule with an explicit name.

---

## Part 5: Aggregation Operators

Aggregations collapse a vector along label dimensions. The operators:

| Operator | What it does |
|----------|--------------|
| `sum` | Add values |
| `avg` | Arithmetic mean |
| `min`, `max` | Element-wise min/max |
| `count` | Number of series in each group |
| `stddev`, `stdvar` | Standard deviation / variance across series |
| `topk(k, expr)` | Top `k` series by value (per group, if grouped) |
| `bottomk(k, expr)` | Bottom `k` |
| `quantile(φ, expr)` | The φ-quantile across series values (NOT over time!) |
| `count_values(label, expr)` | Count series per distinct value; emit new series labeled by `label` |
| `group` | Collapse to one series per group, value `1`; used for label-set math |

### `by` vs `without` -- Choosing What to Keep

An aggregation without a `by` or `without` clause collapses ALL labels -- you get one series total. Usually you want to keep some dimensions:

```promql
# Total rate summed across EVERYTHING -- rarely what you want
sum(rate(http_requests_total[5m]))

# Rate per service and status -- usually what you want
sum by (service, status) (rate(http_requests_total[5m]))

# Everything except instance and pod -- the "without" idiom
sum without (instance, pod) (rate(http_requests_total[5m]))
```

**Official Prometheus guidance** (and Brian Brazil's preference): prefer `without` over `by` in production. Why? `by (service, status)` silently drops any NEW labels someone adds to the metric later. `without (instance, pod)` explicitly lists the labels you want to erase and keeps everything else -- future-proof against schema evolution. In practice, dashboard authors often use `by` because it's more explicit about what shows up in the legend; use your judgment.

### `topk` and `bottomk` -- Read the Leaderboard

```promql
# The 10 metric names with the most active series
topk(10, count by (__name__) ({__name__=~".+"}))

# The 5 pods with the highest CPU usage right now
topk(5, rate(container_cpu_usage_seconds_total[5m]))

# Top 3 services by error rate
topk(3, sum by (service) (rate(errors_total[5m])))
```

**Gotcha**: `topk` returns series, not values, so in Grafana it renders as a graph with up to `k` lines. If your dataset has churn (pod names rotating), the "top 5" changes over time and you'll see disappearing/appearing lines -- disorienting. For stable leaderboards, feed stable series to `topk` (aggregate up to `service` or `deployment` first).

### `quantile` vs `histogram_quantile` -- Don't Mix Them Up

- `quantile(0.99, expr)` -- the 99th-percentile VALUE across a set of SERIES. E.g., "what's the 99th-percentile-ranked instance's CPU right now?" Operates on the current value of each series at evaluation time. Takes a single instant vector.
- `histogram_quantile(0.99, ...)` -- the 99th-percentile OBSERVATION from a HISTOGRAM. Takes a histogram's `_bucket` series and interpolates inside the bucket to estimate the p99 latency.

Different tools for different questions. Using `quantile` instead of `histogram_quantile` on histogram buckets is meaningless -- you'd be computing "the 99th-percentile-ranked bucket count," which is nonsense.

---

## Part 6: The `rate()` Family -- The Most Important Section

This is the engine room. A confident PromQL author is someone who can explain exactly what `rate()` does, when it's wrong, and what to use instead.

### The Three Main Functions

| Function | Input | Output | Use for |
|----------|-------|--------|---------|
| `rate(v range-vector)` | Range vector of a counter | Per-second average rate across the window | **Almost everything** (alerts, dashboards, SLIs) |
| `irate(v range-vector)` | Range vector of a counter | Per-second rate from the **last two samples** | High-resolution graphs of fast-moving counters; rarely useful |
| `increase(v range-vector)` | Range vector of a counter | Total increase across the window (= `rate() × seconds`) | "How many requests in the last hour?" |

All three handle **counter resets** -- if the value decreases from one sample to the next, the function assumes a reset (process restart) happened and sums the delta from zero (see yesterday's counter section for the mechanics). You never write counter-reset logic yourself; the function does it for you. This is why you must never sum counters before rating (more below).

### The Extrapolation Math -- What `rate()` Actually Does

This is the bit that surprises people. `rate(http_requests_total[5m])` does NOT simply compute `(last_sample - first_sample) / 300` where `300 = 5m`. If it did, any partial window would give wrong results.

The actual algorithm (as documented by Julius Volz, Prometheus co-founder):

1. Collect all samples in the `[5m]` range ending at the evaluation timestamp.
2. Compute the raw delta: `last_sample - first_sample`, accounting for counter resets along the way.
3. Determine the time span actually covered by samples: `last_sample_time - first_sample_time`. This is usually LESS than 5 minutes -- the first sample might be 4m 45s before evaluation instead of exactly 5m.
4. **Extrapolate** the delta outward to the true window boundaries:
   - Forward extrapolation: from `last_sample_time` to the evaluation time `t`.
   - Backward extrapolation: from the window start `t - 5m` to `first_sample_time`.
5. Divide by `5m` (300 seconds) to get the per-second rate.

There's a cap on the extrapolation: if the gap at either boundary is more than `1.1 × average_sample_interval`, PromQL only extrapolates to halfway into the gap (to avoid spurious rates when a scrape has been missing for a long time). There's also a minimum-sample requirement: `rate()` needs **at least 2 samples within the range** to produce output -- 0 or 1 sample returns no series.

**The operational consequence**: `rate(counter[5m])` on a 15s scrape interval gives you 20 samples to work with, the math is stable, and partial windows at the start of a graph don't show spikes. `rate(counter[30s])` on the same scrape gives you 2 samples, any jitter is directly visible as a big swing, and you're one missed scrape away from no data.

**The 4× rule of thumb**: your range should be at least 4× your scrape interval. On a 15s scrape, minimum `[1m]`; prefer `[5m]` for dashboards and alerts. This gives you enough samples for the extrapolation to smooth over jitter and still respond promptly to real changes.

### `rate()` vs `irate()` -- The Difference in Practice

`irate()` uses ONLY the last two samples in the range, ignoring everything else. For a counter that increments quickly (thousands of requests per second), this gives you a high-resolution graph that tracks the latest scrape cycle.

For a counter that increments slowly (a `logins_total` that ticks up a few times a minute), `irate()` is nearly always wrong:

- If the last two samples are from the same minute's traffic: huge spike.
- If the last two samples have no increment: flat zero.
- Either way, the graph jitters wildly, and alerts on `irate()` fire on literally any momentary burst.

**Rule**: never put `irate()` in an alerting rule or on a production dashboard. Use it only when graphing high-cardinality, fast-moving internal metrics where you specifically want the instantaneous behavior. In the last five years of code review on real Prometheus stacks, "someone used `irate()` where they meant `rate()`" is the #2 bug after sum-then-rate.

### `increase()` -- Not A Separate Function, Just `rate() * seconds`

Under the hood, `increase(x[5m])` is identically `rate(x[5m]) * 300`. Same extrapolation logic, same counter-reset handling. The difference is semantic:

- `rate()` gives you a per-second figure suitable for graphing, alerting on thresholds, SLI computation.
- `increase()` gives you a window total suitable for "how many X happened in the last Y."

Example: "How many 500s in the last hour?":

```promql
sum by (service) (increase(http_requests_total{status=~"5.."}[1h]))
```

Note the same extrapolation rule: `increase()` over a small window with only 2 samples can return fractional values ("0.93 requests"). That's because the extrapolation is extending the delta across the time gap, and fractional increases are the honest answer when samples are sparse. If you need integer counts, use `round()` -- but question the usefulness of a window so small first.

### `irate()` Danger Zone: Alerts and Dashboards

```promql
# DON'T DO THIS IN AN ALERT
alert: HighErrorRate
expr: irate(http_errors_total[5m]) > 1
# Fires and resolves constantly; alert fatigue in 24h
```

Why it's broken: `irate()` looks at only the last two samples. If your error count jumped from 0 to 1 between scrapes, `irate()` reports `1/15s = 0.067 per second`. That's fine. But if the next scrape sees another increment, `irate()` now reports the rate between THOSE two samples. Every transient blip of traffic manifests as a spike. Alerts flap. Use `rate([5m])` and you get a smoothed-over average that only sustained error behavior crosses.

### `resets(v range-vector)` -- Counting Counter Resets

Sometimes you want to count how many times a counter restarted (i.e., how many times a process crashed). `resets()` returns that count over the window.

```promql
# Processes that restarted in the last hour
resets(process_start_time_seconds[1h]) > 0
```

This is also the tool for detecting missed counter data -- if `resets()` jumps unexpectedly, something's restarting or you're confusing two series with the same labels.

### THE RULE: Rate-Then-Sum, Never Sum-Then-Rate

This is the #1 PromQL mistake in code review. Brian Brazil has blogged about it at least three times. Internalize it.

**Wrong**:

```promql
rate(sum by (service) (http_requests_total)[5m:])
```

**Right**:

```promql
sum by (service) (rate(http_requests_total[5m]))
```

Why? Consider a service with 3 pods. Each pod is a separate counter. At time `t1`, all three pods have been running a while -- the sum is, say, 30,000 requests. At `t2`, pod 2 restarts; its counter goes from 10,000 back to 0. The summed series drops from 30,000 to 20,000.

- **Sum-then-rate** sees the summed series go 30,000 -> 20,000, interprets the drop as a counter reset on the *summed* series, and adds the post-drop value (20,000) back on top of the pre-drop total as if the whole aggregate had restarted from 30,000. The result is a fake 20,000-request spike that propagates through every downstream dashboard and alert.
- **Rate-then-sum** computes the rate per pod first. Pod 2's `rate()` correctly identifies a counter reset (value decreased) and handles it -- Pod 2 contributes its delta from zero. Then we sum three well-behaved per-pod rates and get the correct service-wide rate.

The rule generalizes: **any transformation that knows about counter semantics (rate, increase, irate, deriv, resets) must run on a single per-series counter before aggregation**. Aggregating counters directly destroys the per-series reset signal.

The analogy: three friends each wear a pedometer. Each pedometer resets to 0 when its battery dies and gets swapped. To get the group's steps-per-hour correctly, you compute each person's hourly rate *per pedometer* -- each device's battery swap is visible in that device's raw counter -- and THEN add the three rates. If you instead sum the three displays first (5,000 + 7,000 + 3,000 = 15,000) and one pedometer resets between readings, the summed number drops to 12,000 and you have no way to tell whether the group walked backward or someone's battery died. The per-device reset signal is only legible on the per-device counter; collapsing first destroys it. That is exactly what sum-then-rate does with a service's per-pod counters.

---

## Part 7: Histograms in PromQL

Yesterday's doc covered what a histogram IS (bucket counts, `_sum`, `_count`, aggregable across instances). Today covers how to query them.

### The Canonical p99 Query -- Every Piece Explained

```promql
histogram_quantile(
  0.99,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Walk it outward:

1. **`http_request_duration_seconds_bucket`** -- the raw bucket counters. Every instance exposes one series per bucket (per other-label-combo). See yesterday's cardinality section: 10 buckets × 5 services × 20 instances = 1000 bucket series, which balloons fast once you add handler/method/status labels on a busy cluster.
2. **`[5m]`** -- 5-minute range window to feed `rate()`. Must exceed scrape interval by at least 4×.
3. **`rate(...[5m])`** -- per-second bucket increment rate. This is where counter-reset handling happens, and it's per-series (not after aggregation). The rate-then-sum rule applies here too.
4. **`sum by (le, service) (...)`** -- CRUCIAL: `le` is the bucket-upper-bound label. Summing without `le` would collapse all buckets into one series; `histogram_quantile()` needs them separate. `service` is kept so we get a per-service p99. `instance` and all other labels are summed away -- we're aggregating across all pods of a service.
5. **`histogram_quantile(0.99, ...)`** -- linearly interpolates inside the bucket that crosses the 0.99 quantile line and returns an estimated p99.

**The single most common mistake**: forgetting `le` in the `by` clause. Without it, all buckets collapse into one series and `histogram_quantile()` returns `NaN` or nonsense. "Why is my p99 always 0 or +Inf?" -- almost always a missing `le`.

### Why `avg(histogram_quantile(...))` Is Broken

If you have three regions each computing their own p99 latency, you might be tempted to:

```promql
# WRONG
avg by (service) (histogram_quantile(0.99, sum by (le, region) (rate(bucket[5m]))))
```

This is statistically meaningless. Quantiles don't aggregate arithmetically. The average of three regions' p99s is NOT the global p99 -- you could have two regions at 100ms p99 and one at 500ms, averaging to 233ms, while the true global p99 might be 450ms or 200ms depending on the distributions. You cannot average percentiles. (This is the same reason summaries can't aggregate across instances -- their pre-computed per-process quantiles have the same problem.)

The correct move: aggregate the BUCKETS (which ARE additive), THEN compute the quantile.

```promql
# RIGHT
histogram_quantile(0.99, sum by (le, service) (rate(bucket[5m])))
```

This is why histograms win over summaries: buckets are additive across instances, so you can compute a single global quantile from the full distribution. Summaries lock you into per-instance quantiles with no valid way to aggregate.

### Other Useful Histogram Functions

- `histogram_count(v instant-vector)` -- total observation count (native histograms only; for classic, use `rate(<metric>_count[5m])`).
- `histogram_sum(v instant-vector)` -- sum of observation values (native histograms only).
- `histogram_fraction(lower, upper, v)` -- fraction of observations in `[lower, upper]`. Native-histogram-only. Good for "what fraction of requests were under 100ms?" without computing a quantile.
- `histogram_stddev(v)`, `histogram_stdvar(v)` -- standard deviation / variance of the distribution. Native-only.

For **classic histograms**, the equivalents are:

```promql
# Average latency (classic)
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])

# Fraction of requests under 100ms (classic)
sum(rate(http_request_duration_seconds_bucket{le="0.1"}[5m]))
  / sum(rate(http_request_duration_seconds_count[5m]))
```

The first query is the "average latency" idiom: rate-of-sum divided by rate-of-count. **Average is not median** -- a long-tail distribution's average can look fine while p99 is terrible, which is why you graph p50/p90/p99 percentiles, not averages.

### Native Histograms -- Simpler Queries

Native histograms (v2.40+) collapse all those buckets into one series, and the query surface gets much cleaner:

```promql
# Native histogram p99 -- no `by (le)`, no `_bucket` suffix
histogram_quantile(0.99, sum by (service) (rate(http_request_duration_seconds[5m])))

# Native histogram avg latency (using histogram_sum / histogram_count)
histogram_sum(rate(http_request_duration_seconds[5m]))
  / histogram_count(rate(http_request_duration_seconds[5m]))

# Fraction under 100ms
histogram_fraction(0, 0.1, rate(http_request_duration_seconds[5m]))
```

The key syntactic change is **no `by (le)`** -- native histograms carry their bucket schema inside the single-series value, so PromQL handles bucket-preservation automatically. The migration from classic to native means rewriting dashboards and alerts to drop `_bucket` and `le`, which is exactly why dual-emit mode during migration matters (yesterday's doc).

---

## Part 8: Subqueries, `offset`, and `@` Modifiers

### Subqueries -- Range Vectors From Instant-Vector Expressions

Built-in `[5m]` range selectors only work on bare metrics. But sometimes you want a range vector of `rate(x[5m])` itself -- "the rate, at each of the last 60 evaluations, so I can take a max." For that you use a **subquery**:

```
<instant-vector-expression>[<range>:<resolution>]
```

Example: **peak 5-minute error rate over the last hour**.

```promql
max_over_time(
  (sum by (service) (rate(errors_total[5m])))[1h:1m]
)
```

Reading the types: the inner `sum by (service) (rate(errors_total[5m]))` is an instant vector (a single sum-of-rate evaluated at `t`). The `[1h:1m]` says "run this expression once per minute across the last hour, producing a range vector with 60 samples per series." `max_over_time()` takes that range vector and returns the max per series.

**Alignment**: the `:1m` step is a query-time resolution, NOT the scrape resolution. Subqueries re-evaluate the inner expression at *subquery step boundaries* -- which almost never coincide with your actual scrape samples. That's the real reason subqueries are expensive: PromQL has to evaluate the inner instant-vector expression at each synthetic step, and those steps can be offset from real samples (e.g., `[5m:15s]` on a 15s scrape can land 14.8s away from the nearest real sample). Three footguns:

1. **Step too coarse**: `[1h:5m]` gives only 12 samples per series; you may miss short spikes.
2. **Step too fine**: `[1h:1s]` is wildly expensive -- it forces PromQL to evaluate the inner expression 3600 times per outer evaluation.
3. **Step misaligned with scrape interval**: the subquery step is purely synthetic; it does NOT snap to sample timestamps. `rate(x[5m])[1h:15s]` on a 15s scrape looks like "read the actual samples," but the inner `rate()` still runs its own extrapolation math at each synthetic step.

The step should be comparable to your scrape interval (for most production Prometheus deployments, `:30s` or `:1m` is the sweet spot).

**The meta-rule**: subqueries are expensive. If you find yourself writing `[1h:1m]` inside a production dashboard, convert it to a recording rule instead. Record `sum by (service) (rate(errors_total[5m]))` as `service:errors:rate5m` at the normal evaluation interval; then the subquery becomes `max_over_time(service:errors:rate5m[1h])` -- same answer, but the recording rule has pre-computed the inner piece and the subquery degenerates into a native range vector with no re-evaluation cost.

### `offset` -- Look Back in Time

`offset <duration>` shifts the query's view of time BACKWARDS by that duration.

```promql
# Current request rate
sum(rate(http_requests_total[5m]))

# Request rate one week ago
sum(rate(http_requests_total[5m] offset 1w))

# Week-over-week change
sum(rate(http_requests_total[5m]))
  - sum(rate(http_requests_total[5m] offset 1w))

# Percent change
(sum(rate(http_requests_total[5m]))
  - sum(rate(http_requests_total[5m] offset 1w)))
  / sum(rate(http_requests_total[5m] offset 1w)) * 100
```

Offset can go forward too (Prometheus 2.33+): `offset -5m` evaluates 5 minutes IN THE FUTURE relative to the current evaluation. This is rare in practice but occasionally useful for week-over-week comparisons where you want to line up future boundaries relative to a historical reference point (e.g., "how does today's peak compare to what we projected from last week's curve?"). Most people will go their entire careers without writing it.

### The `@` Modifier -- Pinned Timestamps

Introduced experimentally in Prometheus 2.25 (behind `--enable-feature=promql-at-modifier`) and promoted to stable in 2.33 (January 2022), `@` pins a sub-expression to an absolute unix timestamp (or a keyword).

```promql
# At a specific unix timestamp
http_requests_total @ 1609746000

# At the query's START / END -- crucial for alerting
http_requests_total @ start()
http_requests_total @ end()
```

The alerting use case is the killer one. Suppose you have an alert:

```promql
increase(http_errors_total[1h]) > 100
```

Concretely, this is how the non-determinism manifests and why it erodes trust in alerts:

- **3:00 AM** -- alert engine evaluates `increase(http_errors_total[1h])` over the window `02:00 -> 03:00` and gets 150. `150 > 100` -> alert FIRES. Page goes out.
- **3:07 AM** -- on-call engineer gets the page, clicks the dashboard link. Grafana evaluates the same expression at *its* "now" (3:07). The `[1h]` window has now shifted to `02:07 -> 03:07`. Seven minutes of the original incident have fallen out of the window; seven minutes of (hopefully calmer) new data have entered. The panel returns 87.
- Engineer reads: **alert says 150 errors, dashboard says 87. Which one is right? Did the alert lie?**

Neither "lied" -- they evaluated identical PromQL, faithfully, at different moments. But the human narrative ("the monitoring is inconsistent, maybe the alerting is broken") is exactly how you lose on-call trust. This is what "determinism" means in PromQL-land: cross-evaluator reproducibility of the window the alert fired on.

With `@ start()`:

```promql
increase(http_errors_total[1h] @ start()) > 100
```

The `[1h]` range is now pinned to the alert engine's evaluation timestamp, not to whoever is looking. When the dashboard is templated with the same anchored timestamp (Grafana supports this via `$__to` / variable-templated `@` modifiers), Grafana renders exactly the window the alert fired on. Both evaluators agree. No "did the alert lie" confusion. This is the idiomatic pattern for any alert whose dashboard will be visited post-fire, which is, in practice, every alert.

**The rule of thumb for production alerting queries: if the expression contains a range selector (`[...]`), pin it with `@ start()`.** Especially burn-rate alerts, which use very long windows (`[1h]`, `[6h]`, `[24h]`, `[30d]`) where even small evaluation-time drift produces visibly different numbers.

### `start()`, `end()` -- Rule Context

In recording and alerting rules, `@ start()` and `@ end()` give the rule engine's view of the evaluation window (which is usually "now" for instant rule evaluations, but can differ for replays or backfills). Cheap to add; rules become reproducible.

---

## Part 9: Essential Functions You'll Use Constantly

### `absent()` and `absent_over_time()` -- Alerting on Silence

```promql
# Alert when a metric is completely missing right now
absent(up{job="api"})

# Alert when a metric has been missing for the last 10 minutes
absent_over_time(up{job="api"}[10m])
```

`absent()` returns an empty vector when the input has ANY series, and returns a single series with value `1` when the input is empty. That value-of-1-when-empty is how you write "page me if this thing is gone" alerts.

**The label-inheritance trick**: `absent()`'s output series **inherits any label equality matchers from the input selector**. `absent(up{job="api", instance="prod-us-east-1:9100"})` returns a series with `{job="api", instance="prod-us-east-1:9100"}` as labels -- which lets Alertmanager route the alert to the right team via its `group_by` clause. Omit this and your "something is missing somewhere" alert arrives with no routing labels and pages the whole org.

`absent_over_time()` is the range version -- it returns `1` only if the input has been empty the ENTIRE window. This is more useful in production because a single missed scrape shouldn't fire an alert -- a 10-minute outage should.

**Gotcha**: `absent()` is tricky on multi-label series. `absent(http_requests_total)` fires only if there are ZERO series for the whole metric -- including all pods, all services, etc. More often you want `absent(http_requests_total{service="api"})` or similar to alert on one-specific-thing-is-gone.

### `changes()` -- Count How Many Times a Value Changed

```promql
# Pods that have restarted more than 5 times in the last hour
changes(kube_pod_container_status_restarts_total[1h]) > 5
```

`changes()` counts distinct value changes in the window. Different from `resets()` (which counts only counter *decreases*). A gauge that toggles `0 -> 1 -> 0 -> 1` over a window shows `changes() = 3` (every transition counted) but `resets() = 2` (only the `1 -> 0` decreases). Useful for gauges that shouldn't flip frequently.

### `delta()` and `deriv()` -- Gauges' Answer to `rate()`

- `delta(v range-vector)` -- the difference between the first and last sample in the range, with the same extrapolation to boundaries that `rate()` does. For gauges, not counters; no reset handling. Useful for "how much did memory change in the last hour?"
- `deriv(v range-vector)` -- the per-second derivative via simple linear regression across samples. Useful for trend analysis on a gauge.

```promql
# Memory pressure trend -- how fast is available memory dropping?
deriv(node_memory_MemAvailable_bytes[1h])   # bytes/sec; negative if decreasing
```

**Critical gotcha**: `rate()` on a gauge returns garbage. Gauges aren't counters; they go up AND down. `rate()` interprets every decrease as a counter reset and produces spurious spikes. Use `deriv()` on gauges, always.

### `predict_linear()` -- Extrapolate Into the Future

```promql
# Predict when the disk will fill up (in seconds from now)
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0
```

This alerts when "if the current trend continues, in 4 hours the available bytes will be negative." The 4-hour disk-full alert is the canonical example. The `1h` window gives enough history to compute a stable linear slope; the `4 * 3600` is "how many seconds into the future to extrapolate." `< 0` means "will the disk be exhausted by then?"

Why this is better than a threshold alert:
- A threshold of "alert at 10% free" fires right when the disk is about to blow up, giving no lead time.
- `predict_linear` fires when the TREND indicates imminent doom, giving you hours to respond.

It's also wrong sometimes -- bursty writes break linear extrapolation, and holt_winters()-style seasonal projection handles periodicity better. But for most "is the trend bad?" questions, linear is enough.

**The negative-slope caveat**: `predict_linear` fits a linear slope across the window. If your `1h` window happens to contain a brief cleanup burst (e.g., a scheduled log-rotation job that frees 20 GB in one minute), the slope flips positive -- "disk is growing, no alert" -- even though the baseline trend is still downward. The alert goes silent exactly when you'd want to be paged. Bursty workloads should use a longer window (`6h`) or pair `predict_linear` with a floor check: `predict_linear(...) < 0 AND node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.2`. The AND gate means "trend is bad AND we're already below 20% free" -- resilient to short upward blips.

### `holt_winters()` -- Seasonal Smoothing

```promql
holt_winters(metric[24h], 0.8, 0.3)
```

Analogy: Holt-Winters is what weather forecasters do when they account for "this is a Tuesday in summer" rather than just "it's been sunny all week" -- the model has separate knobs for level, trend, and seasonality, so it doesn't mistake a predictable daily dip for an emerging problem.

Double-exponential smoothing with configurable trend smoothing (`sf`) and trend-growth smoothing (`tf`). Useful for predicting traffic patterns that have daily cycles. Rare in practice; most folks use recording rules + `offset 1w` for year-over-year comparison instead.

### `clamp()`, `round()`, `floor()`, `ceil()` -- Arithmetic Hygiene

```promql
# Cap percentage at 100 (defensive against >100% metrics)
clamp_max(cpu_percent, 100)

# Round p99 latency to nearest millisecond
round(histogram_quantile(0.99, sum by (le) (rate(bucket[5m]))) * 1000)

# Clamp values to a range
clamp(metric, 0, 100)
```

Rare but useful when you need clean dashboard labels or alert thresholds.

### `label_replace()` -- SQL `UPDATE ... SET` at Query Time

```promql
# Rename "device" label to "disk" for joining with another metric
label_replace(node_disk_read_bytes_total, "disk", "$1", "device", "(.*)")
```

Signature: `label_replace(v, dst_label, replacement, src_label, regex)`. Matches `regex` against the `src_label`; if it matches, sets `dst_label` to the interpolated `replacement`. If it doesn't match, the series passes through UNCHANGED (no error!).

**The silent-failure gotcha**: a non-matching regex returns the input unchanged. If your regex has a typo, the label rewrite silently doesn't happen and your join further up the pipeline produces no results. When debugging "my query returns nothing," start with: "did my `label_replace` actually match?" Add the new label to a `by ()` clause or inspect the table view to verify.

### `label_join()` -- Concatenate Labels Into One

```promql
# Create a "namespaced_pod" label combining namespace and pod
label_join(kube_pod_info, "namespaced_pod", "/", "namespace", "pod")
# pod="api-abc", namespace="default" -> namespaced_pod="default/api-abc"
```

Useful for creating unique identifiers across label dimensions. Same silent-pass-through behavior as `label_replace` if inputs don't exist.

### `sort()`, `sort_desc()` -- Order Series for Display

```promql
# Ordered for dashboard legend
sort_desc(sum by (service) (rate(http_requests_total[5m])))
```

No-op in math; only useful for display in Grafana/promtool.

### `time()`, `vector()`, `scalar()` -- Time and Type Coercion

- `time()` -- returns the evaluation timestamp as unix seconds. Scalar.
- `vector(s scalar)` -- promote a scalar to an instant vector (with no labels). For mixing scalars into vector math.
- `scalar(v instant-vector)` -- demote a single-series instant vector to a scalar. Returns `NaN` if the input has 0 or more than 1 series.

### `timestamp()` -- When Was This Sample Taken?

```promql
# Time since last sample of `up` was written -- NOT "time since scrape succeeded"
time() - timestamp(up)

# Time since last SUCCESSFUL scrape (filter on up == 1 first)
time() - timestamp(up == 1)
```

**Gotcha**: `time() - timestamp(up)` measures time since the last `up` sample was written, not time since the last *successful* scrape. When scrapes fail, Prometheus still writes `up=0` samples -- so `timestamp(up)` keeps updating even while the target is down. If you want a real freshness signal, filter to `up == 1` first (as above), or combine with an `up == 1` check in the alert expression.

Useful for data-freshness alerts: "has this metric not been scraped *successfully* for more than 10 minutes?"

---

## Part 10: Recording Rules -- The Meal Prep of Observability

A **recording rule** is a PromQL expression whose result is stored as a new time series on a schedule. Instead of every dashboard and alert re-computing `sum by (service) (rate(http_requests_total[5m]))` from scratch, you compute it once per `evaluation_interval` and store it as `service:http_requests:rate5m`. Readers just read that cheap series.

The analogy: instead of cooking from scratch every time a guest arrives, you meal-prep on Sunday. Dashboards become "heat the pre-made serving" fast.

### When to Use Recording Rules

- The query is expensive and runs frequently (appears on many dashboards or in alerts that evaluate every minute).
- The query uses a subquery (`[...:step]`). Recording rules are almost always the right answer instead.
- You're federating to an upper-tier Prometheus or Thanos -- pre-aggregate at the source to cut cardinality in transit.
- Multiple alerts share the same base expression -- record it once, reference it N times.

### The Naming Convention -- `level:metric:operation`

Prometheus has a semi-official naming convention documented by the project maintainers:

```
<level>:<metric>:<operation>
```

- **`level`** -- the aggregation level. `instance`, `job`, `service`, `cluster`, `namespace`, `global`. What labels are still present on the output.
- **`metric`** -- the underlying metric name, often with label qualifiers stripped.
- **`operation`** -- what function was applied. Common: `rate5m`, `rate1m`, `sum`, `avg`, `count`, `quantile99`.

Examples:

```
job:http_requests:rate5m
namespace_service:kube_pod_info:sum
cluster:node_cpu_seconds:irate1m
service_handler:http_request_duration_seconds:p99
```

The colons are visual separators, not syntactically significant -- Prometheus doesn't enforce them, but following the convention makes dashboards and alerts self-documenting.

### Rule Group Structure

```yaml
groups:
  - name: http.rules
    interval: 30s          # How often the whole group evaluates
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))

      - record: job:http_error_ratio:rate5m
        expr: job:http_errors:rate5m / job:http_requests:rate5m
```

**Crucial property**: within a group, rules evaluate **sequentially**. Rule 3 (`job:http_error_ratio:rate5m`) can reference Rules 1 and 2 because they've already produced output by the time Rule 3 runs. Across groups, evaluation is **scheduled concurrently** -- groups are dispatched in parallel, though each group still runs its own rules sequentially internally. So if Rule A in Group 1 depends on Rule B in Group 2, you have a race condition -- Rule A might see Rule B's output from the previous cycle.

**Operational rule**: put dependent rules in the SAME group. Put independent rule categories in separate groups so they parallelize at dispatch time (but don't expect parallelism *within* a group).

### Staleness in Recording Rules

When the source metric disappears (a pod dies), its recording-rule-derived series also disappears -- via the same staleness-marker mechanism we covered yesterday. The recording rule's last sample is followed by a staleness marker in the TSDB, and `rate()` / `absent()` / aggregations all treat the series as "gone" for subsequent evaluations. You don't need to special-case anything; the staleness short-circuit propagates through recording rules cleanly.

### The Forgotten-`by`-Clause Cardinality Bomb

```yaml
- record: http_requests:rate5m
  expr: rate(http_requests_total[5m])      # DANGER: no aggregation!
```

This records ONE new series per original series -- basically doubling cardinality for no benefit. Recording rules should always reduce cardinality (aggregation via `sum by`) or enrich with derived values (a ratio, a quantile). If your recording rule has as many output series as input series, it's wrong.

---

## Part 11: The RED Method -- Services Checklist

**RED** = Rate, Errors, Duration. Coined by Tom Wilkie (Grafana Labs), the three questions you should be able to answer for every HTTP-facing service:

### R -- Rate: How much traffic am I serving?

```promql
sum by (service) (rate(http_requests_total[5m]))
```

### E -- Errors: What fraction is failing?

```promql
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
  /
sum by (service) (rate(http_requests_total[5m]))
```

Note the structure: `sum/sum` of rates. NOT `avg` of per-instance error ratios -- that's the same category of mistake as avg-of-quantiles. The correct service-wide error ratio is total-errors divided by total-requests, computed from sums.

### D -- Duration: How slow is it?

```promql
histogram_quantile(
  0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)
```

With three queries, one recording-rule group, and one Grafana dashboard, you've got the SLI-level view of every service in your org. Recording rules to make them cheap:

```yaml
groups:
  - name: red.rules
    interval: 30s
    rules:
      - record: service:http_requests:rate5m
        expr: sum by (service) (rate(http_requests_total[5m]))

      - record: service:http_errors:rate5m
        expr: sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))

      - record: service:http_error_ratio:rate5m
        expr: service:http_errors:rate5m / service:http_requests:rate5m

      - record: service:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

---

## Part 12: The USE Method -- Resources Checklist

**USE** = Utilization, Saturation, Errors. Coined by Brendan Gregg. The three questions for every infrastructure resource (CPU, memory, disk, network):

### U -- Utilization: How busy is it?

```promql
# CPU utilization per node (1 - idle time)
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Memory utilization per node
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Disk utilization per mount
1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
```

### S -- Saturation: Is it queuing up?

```promql
# Load average vs CPU count (>1 means saturated)
node_load1 / count by (instance) (node_cpu_seconds_total{mode="idle"})

# Pressure-stall information (PSI) -- seconds of CPU pressure per sec
rate(node_pressure_cpu_waiting_seconds_total[5m])

# Memory saturation via swap usage
1 - (node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes)
```

### E -- Errors: Is it broken?

```promql
# Disk I/O errors
rate(node_disk_io_errors_total[5m])

# Network drops
rate(node_network_receive_drop_total[5m])
  + rate(node_network_transmit_drop_total[5m])

# TCP retransmits
rate(node_netstat_Tcp_RetransSegs[5m])
```

USE dashboards pair perfectly with RED dashboards: RED answers "are my services OK?" and USE answers "are my hosts OK?" Together, when a service goes bad, you can correlate quickly: is it request-side (traffic spike), logic-side (error rate up), or infrastructure-side (CPU maxed, disk slow, network dropping)?

---

## Part 13: Production Gotchas (Ranked by How Often They Bite)

### 1. Sum-Then-Rate vs Rate-Then-Sum

Covered above. #1 PromQL mistake in code review. Always rate per series first.

### 2. `rate()` on Gauges Returns Nonsense

Gauges go up AND down. `rate()` treats every decrease as a counter reset, producing spurious spikes. Use `deriv()` on gauges.

```promql
# WRONG
rate(node_memory_MemAvailable_bytes[5m])

# RIGHT
deriv(node_memory_MemAvailable_bytes[5m])
```

### 3. `histogram_quantile` Over Too-Small a Window

On a 15s scrape, `[1m]` is only 4 samples -- too few to estimate quantiles reliably. Quantiles are second-moment statistics; they need volume. At minimum use `[5m]`, prefer `[10m]` for high-quantile alerting (p99.9).

The analogy: trying to read the mood of a crowd from 4 random faces. Four is not enough.

### 4. `avg` Over Rates vs `sum/sum` For Ratios

```promql
# WRONG -- average of per-instance ratios
avg by (service) (
  rate(errors_total[5m]) / rate(requests_total[5m])
)

# RIGHT -- global ratio from summed rates
sum by (service) (rate(errors_total[5m]))
  / sum by (service) (rate(requests_total[5m]))
```

Why? Same reason as avg-of-quantiles. Imagine 2 pods, one with 100 requests/0 errors, one with 1 request/1 error. Per-instance ratios: 0% and 100%. Average: 50%. Real error ratio: 1/101 ≈ 1%. The sum-over-sum gives the truth; the avg-of-ratios is dominated by the low-traffic pod.

### 5. Subqueries Evaluate at Subquery Step, Not Scrape Resolution

`foo[5m:30s]` runs the expression every 30 seconds for 5 minutes' worth of steps -- that's 10 evaluations per outer step, at boundaries that may not align with actual scrape samples. You get alignment artifacts where the subquery's samples don't match what scrape samples exist. For high-quality alerts, convert to recording rules.

### 6. `@` Modifier Locks to Absolute Time -- `@ start()` in Alerts

Without `@ start()`, an alert's `expr:` evaluates against the alert engine's "now" AND Grafana's "now" when users click through -- different answers. Use `@ start()` in alert queries for deterministic behavior.

### 7. `label_replace()` With Non-Matching Regex Silently Returns Original

If your regex has a typo or the label you're replacing doesn't match, you get back the unmodified series. Downstream join fails silently. Always preview `label_replace` output in Grafana's table view before wrapping it in larger expressions.

### 8. Empty Matcher Semantics

`{job=""}` matches series WITHOUT the `job` label. `{job!=""}` matches series WITH a non-empty `job` label. Mixing these up gives opposite results. Memorize the rules.

### 9. `increase()` Less Than 2× Scrape Interval Can Return 0 or Fractions

`increase(x[30s])` on a 15s scrape gives you 2 samples -- barely enough. If one is missed, you get nothing. Always use range >= 2× scrape; prefer 4× (the same 4× rule as yesterday's doc).

### 10. Recording Rule Cardinality Explosion from Missing `by` Clause

```yaml
# DANGER: one output series per input series, no reduction
- record: http_requests:rate5m
  expr: rate(http_requests_total[5m])
```

Recording rules should always reduce cardinality. If your output series count is close to your input series count, you've made the TSDB bigger for no reason.

### 11. `irate()` Flapping Alerts

`irate()` uses only the last 2 samples. Alerts on `irate()` flap with every scrape jitter. Use `rate([5m])` for alerts; reserve `irate()` for ad-hoc high-res graphs.

### 12. Classic-to-Native Histogram Query Compatibility

During migration (dual-emit phase), you have both `_bucket` series AND native histogram samples. Classic queries `sum by (le) (rate(..._bucket[5m]))` keep working; native queries `histogram_quantile(0.99, sum by (service) (rate(...[5m])))` use the native form. A query that reaches for `_bucket` on a native-only metric fails silently (empty vector) -- exactly the "alerts go quiet" risk we flagged yesterday.

### 13. `absent()` on Multi-Label Series Needs Selector Pinning

`absent(http_requests_total)` only fires when the ENTIRE metric is gone. Usually you want `absent(http_requests_total{job="api", handler="/healthz"})` -- specific enough that ONE missing target fires.

### 14. `bool` Modifier Quietly Changes Semantics

`rate(x[5m]) > 0` returns series where the rate is positive (filter). `rate(x[5m]) > bool 0` returns EVERY series with a 1/0 value. Using `bool` in an alert's `expr:` makes the alert fire constantly (every series has a 1 or 0, both truthy for alert purposes depending on engine). Be careful with `bool` in alerts.

### 15. Scrape-Interval-Range Mismatch Across Jobs

If `job=api` scrapes every 15s and `job=database` scrapes every 60s, a single `rate(...[1m])` applied to both gives reasonable output for `api` (4 samples) and degenerate output for `database` (0-1 samples). Range choice should respect each job's scrape cadence. Use `$__rate_interval` in Grafana or recording rules per job.

### 16. `$__rate_interval` -- Grafana's Built-In Answer to the Range-Choice Problem

Grafana computes `$__rate_interval` as `max($__interval + scrape interval, 4 × scrape interval)` where `$__interval` is the dashboard's current step (itself derived from panel width and time range). In practice it's usually `4 × scrape interval`, auto-adjusted per panel.

```promql
# In Grafana, this auto-respects the 4x rule
sum by (service) (rate(http_requests_total[$__rate_interval]))
```

Use it instead of hardcoded `[5m]` in Grafana panels that span both short zooms (where `[5m]` wastes data) and long zooms (where `[5m]` produces too few points per pixel). Not available outside Grafana -- for Prometheus rules, stick with explicit ranges.

### 17. `histogram_quantile` Returning `+Inf` -- Top Bucket Too Low

If your p99 falls in the `+Inf` bucket (i.e., your histogram's top `le` is lower than the 99th-percentile latency), `histogram_quantile` either returns `+Inf` or caps at the upper bound of the last finite bucket -- neither is a real latency. See yesterday's histogram section for bucket design. The fix is to raise the top `le` in instrumentation; there's no query-side workaround.

---

## Part 14: Decision Frameworks

### Rate vs iRate vs Increase -- Which Do I Use?

```
Am I aggregating / alerting / dashboarding?
  YES -> rate([5m]) or greater (per-second avg, stable)

Am I computing a window total (how many X in the last Y)?
  YES -> increase([Y])

Am I zoomed into a high-res graph of a fast-moving counter,
specifically wanting the last-two-sample rate for debugging?
  YES -> irate([1m]) -- be aware of the jitter

In any doubt -> rate([5m]).
```

### Recording Rule vs Subquery

The real decision boundary: **is the inner expression SHARED across queries, or is this a ONE-OFF?**

```
Will the inner expression be re-used (dashboards/alerts/other rules)?
  YES -> recording rule (pre-compute once, reference many times)
  NO  -> subquery is fine; don't over-engineer

Does the query use a subquery inside a production alert?
  YES -> recording rule (subqueries re-evaluate the inner every tick;
         alerts evaluate often; multiply them and the cost adds up)

Is this an ad-hoc investigation in Explore?
  YES -> subquery is fine; recording rules aren't worth the PR

Is the query cheap (couple of series, short range, runs rarely)?
  YES -> don't bother with a recording rule
```

### Classic Histogram vs Native Histogram In Queries

```
Is the metric emitting in native format (Protobuf or OpenMetrics)?
  YES -> use native form (no `by (le)`, no `_bucket`)
  NO  -> use classic form (with `by (le)` on `_bucket` series)

Are we in a dual-emit migration window?
  YES -> run both queries in parallel, verify equivalence; cut over after
         Phase 3 of yesterday's migration plan; delete classic queries last
```

### Aggregation -- `by` vs `without`

```
Do I know every label that this metric will ever have?
  NO  -> `without (labels-I-want-gone)` -- future-proof
  YES -> `by (labels-I-want-kept)` -- explicit

For dashboard legends: usually `by`, because it clarifies what dimensions
show up in the rendered panel. For production recording rules: `without`
is more resilient to instrumentation changes.
```

### When to Use `@ start()` / `@ end()`

```
Is this an alerting rule expression?
  YES -> consider `@ start()` for deterministic evaluation across
         alert engine and Grafana click-throughs. Essential for
         `increase([window] @ start())` in burn-rate alerts.

Is this a recording rule?
  YES -> `@ start()` pins the window to the rule's evaluation
         time, making replays reproducible.

Is this an ad-hoc dashboard query?
  NO  -> user expects "now" behavior; don't add @ modifier.
```

---

## Part 15: Quick Reference Cheat Sheet

### Vector Types Cheat Sheet

```
Instant vector      -> one value per series at time t
Range vector        -> stack of samples per series over [range]
Scalar              -> single float
String              -> literal (label_replace et al only)

Selector            -> instant vector       http_requests_total
Selector [range]    -> range vector         http_requests_total[5m]
rate(range)         -> instant              rate(x[5m])
sum(instant)        -> instant              sum by (service) (x)
instant [r:s]       -> range (subquery)     rate(x[5m])[1h:1m]
```

### Label Matcher Cheat Sheet

```
=       exact         {status="200"}
!=      not equal     {status!="200"}
=~      regex         {status=~"5.."}     # fully anchored
!~      not regex     {handler!~"/health.*"}

Special (two rules):
  Rule 1: matchers that match "" also match absent labels
  Rule 2: every selector needs one matcher that does NOT match ""

{foo=""}       matches absent-label series (matches ""; needs companion)
{foo!=""}      matches series WITH label foo non-empty (legal alone)
{foo=~".*"}    ALSO matches absent-label series (matches ""; needs companion)
{foo=~".+"}    matches series WITH label foo non-empty (legal alone)
{__name__=~"http_.*"}   matches metric names
```

### Operator Cheat Sheet

```
Arithmetic:  + - * / % ^
Comparison:  == != > < >= <=  (filter by default; bool for 1/0)
Logical:     and  or  unless   (set operations on label sets)

Matching:
  on(labels...)         match on ONLY these labels
  ignoring(labels...)   match on EVERYTHING BUT these labels
  group_left(labels)    left has many, right has one; copy right's labels
  group_right(labels)   right has many, left has one; copy left's labels

Aggregation:
  sum / avg / min / max / count / stddev / stdvar
  topk(n, expr) / bottomk(n, expr)
  quantile(phi, expr)  -- quantile OVER SERIES, not over time
  count_values(label, expr)
  group
  [by (labels) | without (labels)]
```

### Function Cheat Sheet (Most Common)

```
Counters:
  rate(c[r])          per-sec avg rate over r
  irate(c[r])         per-sec rate from last 2 samples
  increase(c[r])      total increase over r  (= rate*r)
  resets(c[r])        count of counter resets in r

Gauges:
  delta(g[r])         first-to-last diff over r
  deriv(g[r])         per-sec slope via linear regression
  predict_linear(g[r], s)   extrapolate s seconds ahead
  holt_winters(g[r], sf, tf) exponential smoothing

Histograms:
  histogram_quantile(phi, v)          quantile from buckets
  histogram_count(native)             observation count (native)
  histogram_sum(native)               observation sum (native)
  histogram_fraction(lo, hi, native)  fraction in range (native)

Time:
  time()              eval timestamp (scalar)
  timestamp(v)        each sample's timestamp
  offset d            shift query time back by d
  @ t                 evaluate at absolute timestamp t
  @ start() / @ end() evaluate at rule/query boundaries

Labels:
  label_replace(v, dst, repl, src, regex)
  label_join(v, dst, sep, src1, src2...)
  sort / sort_desc

Presence:
  absent(v)                 1 when v is empty, else empty
  absent_over_time(v[r])    1 when v empty throughout r
  changes(v[r])             number of value changes in r
  last_over_time(v[r])      last sample's value in r
  <op>_over_time(v[r])      avg/min/max/sum/count/stddev/stdvar/quantile over time

Math hygiene:
  clamp(v, min, max) / clamp_min / clamp_max
  round(v, precision) / floor(v) / ceil(v)
  abs(v)
  vector(s) / scalar(v)
```

### RED & USE Cheat Sheet

```
RED (per service):
  R: sum by (service) (rate(http_requests_total[5m]))
  E: sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
      / sum by (service) (rate(http_requests_total[5m]))
  D: histogram_quantile(0.99,
       sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))

USE (per resource):
  U: 1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
  S: node_load1 / count by (instance) (node_cpu_seconds_total{mode="idle"})
  E: rate(node_disk_io_errors_total[5m])
```

### The 10 Commandments of PromQL

1. **Know the type of every sub-expression.** Instant, range, scalar, string.
2. **Rate then sum, never sum then rate.** Counter math is per-series.
3. **`rate()` on counters; `deriv()` on gauges.** Never `rate()` on a gauge.
4. **Range must be at least 4× scrape interval.** `[5m]` is the safe default.
5. **Preserve `le` when aggregating histograms.** `sum by (le, ...)`.
6. **Never average quantiles.** Aggregate buckets; THEN compute quantile.
7. **Ratios are sum/sum, not avg-of-ratios.** Weight by volume.
8. **Subqueries are expensive; recording rules are cheap.** Pre-compute.
9. **Recording rules reduce cardinality.** Always `sum by`; never pass-through.
10. **`@ start()` in alerts.** Deterministic evaluation across tools.

---

## Key Takeaways

- **PromQL is a vector language, not a row language.** Every expression returns labeled time series, and every operator is implicitly a JOIN on label sets. Think "SQL with columns as dimensions" and you'll navigate most of it.
- **The instant-vs-range-vector distinction is THE mental model.** Every parse error traces back to a type mismatch. Saying the type of each sub-expression out loud is the single highest-leverage habit.
- **`rate([5m])` handles counter resets, extrapolates to window boundaries, and smooths jitter.** It is almost always the right choice over `irate()`. Use `irate()` only for high-resolution debugging graphs, never alerts or production dashboards.
- **Rate-then-sum is the law.** Aggregating counters before taking the rate destroys per-series reset information and produces silent data corruption. Always rate per-series first.
- **Histograms aggregate, summaries don't** (see April 20's bucket-additivity section). PromQL's canonical p99 query reflects this: `histogram_quantile(0.99, sum by (le, service) (rate(..._bucket[5m])))` preserves `le`, sums buckets across instances, THEN computes the quantile. Native histograms simplify this to one series per metric, dropping the `_bucket` and `by (le)`. If `histogram_quantile` returns `+Inf`, your top bucket is too low -- fix it in instrumentation.
- **Vector matching is a SQL JOIN.** Default is one-to-one on identical labels. `on()` / `ignoring()` choose the join key. `group_left` / `group_right` enable many-to-one enrichment, copying labels from the "one" side onto the output. The `kube_pod_container_resource_requests * on(pod) group_left(node) kube_pod_info` pattern is the template for 90% of Kubernetes label enrichment.
- **Recording rules are meal prep.** Expensive queries (especially those with subqueries) should be pre-computed on a schedule and stored as new series. The `level:metric:operation` naming convention keeps them discoverable. Same-group rules run sequentially; different groups run in parallel.
- **RED for services, USE for resources.** Three queries each, recording-ruled, dashboarded, alerted. Together they answer "is this service healthy?" and "is this host healthy?" with minimal query complexity.
- **The 4× rule from yesterday carries over.** Range windows should be at least 4× the scrape interval. `[5m]` on a 15s scrape is the sweet spot for most `rate()` / `histogram_quantile` / `increase()` queries.
- **Silent failures are the real enemy.** `label_replace` with a bad regex, `absent()` on the wrong label selector, alerts referencing classic `_bucket` on a migrated metric, recording rules with missing `by` clauses -- all return plausible-looking emptiness rather than errors. The defense is disciplined query review and parallel-run validation during migrations.
- **`@ start()` makes alerts deterministic.** Without it, alert engine's "now" and Grafana's "now" disagree, producing the "the alert says errors happened but my dashboard looks fine" confusion that kills trust in monitoring.

---

## Further Reading

- Yesterday's companion: `2026/april/prometheus/2026-04-20-prometheus-fundamentals.md` -- covers the metric types, TSDB, staleness markers, and exemplars that PromQL queries operate on.
- **Alerting rules (tomorrow)** -- PromQL expressions become alert bodies; `for:`, `ALERTS` / `ALERTS_FOR_STATE` series, and `absent_over_time()` patterns for missing-data alerts.
- **Grafana Fundamentals (Thursday)** -- dashboard variables (`label_values()`), `$__rate_interval`, the Explore pane as the primary PromQL development environment.
- **Kubernetes Metrics Stack (Friday)** -- kube-prometheus-stack deploys all the recording rules for RED/USE out of the box via PrometheusRule CRDs. Good reference for production rule shapes.
- PromLabs cheat sheet -- https://promlabs.com/promql-cheat-sheet/ -- bookmark and never unbookmark.
- Robust Perception -- "Rate Then Sum, Never Sum Then Rate" -- https://www.robustperception.io/rate-then-sum-never-sum-then-rate/ -- the original authoritative post on the #1 PromQL bug.
- PromLabs -- "How Exactly Does PromQL Calculate Rates?" -- https://promlabs.com/blog/2021/01/29/how-exactly-does-promql-calculate-rates/ -- Julius Volz's extrapolation-math walkthrough.
- Prometheus docs -- Querying Basics -- https://prometheus.io/docs/prometheus/latest/querying/basics/ -- the canonical reference for selectors, empty-matcher rules, and the vector-selector rule.
