# Prometheus Fundamentals -- The Pull-Based TSDB at the Heart of Cloud-Native Monitoring

> Yesterday closed out the full ArgoCD + Helm production stack -- with one loose thread: `AnalysisTemplate` promising to "gate rollouts on Prometheus metrics," but we had never actually taught Prometheus. Today fixes that. Prometheus is the **observability spine** that Argo Rollouts, HPA/VPA, alerting, SLO dashboards, and half of cloud-native incident response all hang off of. Without it, "metric-gated rollback" is just a YAML comment.
>
> The core analogy for the week: a **milkman on a delivery route**. Prometheus is not the postal service where your application drops letters into a mailbox whenever it wants (that's StatsD, CloudWatch PutMetricData, OpenTelemetry push). Prometheus is the milkman who walks up to your doorstep every 15 seconds, rings the bell marked `/metrics`, collects the full current snapshot of bottles you've got sitting outside, and carries them home to the dairy's ledger (the TSDB). If your house is locked one morning, the milkman writes `up=0` in his notebook and tries again at the next round -- he never panics, never demands you push, never loses data because you forgot to call in. If you have ten thousand houses, he rotates a fleet of milkmen on the route. And if a house keeps moving addresses (Kubernetes pods), he checks the neighborhood directory (service discovery) before each round so he always knows where to show up next.
>
> CloudWatch vs Prometheus fits the analogy perfectly. **CloudWatch is push-to-AWS**: every service pushes metrics to the managed collector, pay per metric, limited cardinality, AWS-only. **Prometheus is pull-from-everywhere**: self-hosted, cardinality-sensitive, protocol-open, polyglot, and owned by you. Neither is strictly better; they answer different questions. Today we learn the pull side deeply so tomorrow's PromQL, Wednesday's Alertmanager, and Friday's kube-prometheus-stack all stand on cement.

---

**Date**: 2026-04-20
**Topic Area**: prometheus
**Difficulty**: Intermediate

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Prometheus server | The dairy's head milkman + ledger book | Single binary: scrapes targets, stores samples in TSDB, evaluates rules, serves queries |
| Pull model (scrape) | Milkman walking the route on a schedule | Prometheus initiates HTTP GET against each target's `/metrics` every `scrape_interval` |
| `/metrics` endpoint | A numbered bottle rack by your front door | App exposes plaintext snapshot of its current metric values; Prometheus just reads the rack |
| Exporter | A translator who reads an analog meter and chalks the numbers on the rack | Sidecar/binary that turns non-Prometheus systems (node, mysql, redis, blackbox) into a `/metrics` endpoint |
| Client library | A metric clipboard your app carries | Go/Python/Java/etc. SDK that lets your code `.Inc()` / `.Observe()` and auto-serves `/metrics` |
| Pushgateway | The drop-off mailbox at the dairy for milkmen who will never get visited | Explicitly only for **service-level batch jobs** -- short-lived jobs that die before any scrape could catch them |
| Alertmanager | The dispatcher who takes the milkman's alerts and routes them to the right on-call pager | Separate binary; dedupes, groups, routes, silences; we cover deeply on Wed |
| TSDB | The dairy's leather-bound ledger | Local time-series database on disk; 2h blocks + WAL; 15-day default retention |
| Time series | One row in the ledger per (metric + label set) | Uniquely identified by `metric_name + {labels}`; each change is a new series |
| Sample | One pencil mark in the ledger | `(timestamp_ms, float64_value)` |
| Data model | "Name + sticker collection + number + time" | `metric_name{label1="v1",label2="v2"} value @ timestamp` |
| Label | A sticker on the bottle | Key=value pair that dimensions a metric; **cardinality-critical** |
| Cardinality | How many different sticker combos exist | Number of unique time series; exponential in label values; the #1 way to kill Prometheus |
| Counter | A car odometer | Monotonically increases, resets to 0 on restart; you never read it directly, you `rate()` it |
| Gauge | A speedometer | Current value, can go up or down (temperature, memory bytes, queue depth) |
| Histogram | A shoe-store rack binned by size | Server-side: pre-defined buckets counted; aggregates across instances; quantiles computed at query time |
| Summary | Each restaurant branch pre-computes its own median tip (you cannot average medians to get the chain's median) | Client-side quantiles via streaming sketch (approximate, configurable error); **cannot aggregate** across instances |
| Exposition format | The handwriting rules for the bottle rack | Simple text format: `# HELP`, `# TYPE`, `metric{labels} value`, newline-delimited |
| OpenMetrics | "CNCF-standardized handwriting rules" | Standardized descendant of the classic exposition format; adds exemplars, `_created` series, required `_total` suffix on counters; mostly backwards-compatible |
| `scrape_configs` | The milkman's route book | YAML defining what to scrape, how often, with what labels |
| `job_name` | The route name ("Maple Street route") | Logical group of targets doing the same thing |
| `static_configs` | A printed hand-written route | Hardcoded target list -- fine for a handful of bare metal |
| Service discovery (SD) | Calling the neighborhood directory before the route | Kubernetes/EC2/Consul/DNS/file SD generate targets dynamically |
| `kubernetes_sd_configs` | Reading the apartment building's tenant board every morning | Queries k8s API for node/pod/endpoints/service/ingress targets, auto-updating |
| `__meta_*` labels | The TSA pre-check tag stuck to your suitcase at the airline counter -- read by every station, stripped before the carousel | Temporary labels attached by SD; dropped before samples hit the TSDB unless you `replace` them into real labels |
| `relabel_configs` | Airport baggage-handling stations (tag, re-route, reject) before the plane loads | Transforms target labels **before** the scrape happens (keep/drop/replace/labelmap/hashmod) |
| `metric_relabel_configs` | Customs inspection after the bag arrives at the destination -- one last chance to filter | Runs on each sample **after** scrape; drop/rename noisy metrics |
| `honor_labels` | "If the bottle has its own label, don't overwrite it" | When scraping Pushgateway or federation -- preserve incoming labels, don't let the server's job/instance labels stomp them |
| Block | A chapter in the leather ledger -- starts as a 2-hour chunk, later compacted into multi-day volumes | Immutable on-disk unit (`chunks/`, `index`, `meta.json`); compaction merges 2h → 6h → 24h over time |
| WAL | The milkman's carbon-copy notepad before writing to the ledger | Write-ahead log flushed to disk; replayed on crash to rebuild the head block; retains ~3h of samples as a safety margin beyond the 2h head window |
| Retention (`--storage.tsdb.retention.time`) | "How many days of ledger we keep in the attic" | Default 15d; older blocks deleted |
| Remote write | The local dairy telegraphing every new ledger entry to the national dairy association's mainframe | Streams samples to Thanos/Cortex/Mimir/VictoriaMetrics/Managed Prom -- the durable, queryable, multi-region tier |
| Remote read | Asking the central archive to fetch old pages | Less common; usually replaced by Thanos Query that federates on top |
| Federation (`/federate`) | A regional manager pulls summary reports from each store manager | One Prometheus scrapes selected time series from another Prometheus; for hierarchical aggregation |
| Recording rule | Pre-computed aggregate saved to the ledger on a schedule | Stored query result as a new time series (e.g. `job:http_requests:rate5m`); we cover deeply on Wed |
| Alerting rule | "If the temperature is above 40 for 5 minutes, page the on-call" | Evaluates a PromQL expression on schedule; fires to Alertmanager when condition holds for `for:` duration |

---

## The Big Picture -- How Prometheus Fits Together

Prometheus is not one thing. It's an **ecosystem of small, single-responsibility binaries** that communicate over HTTP:

```
PROMETHEUS ECOSYSTEM -- THE DAIRY ROUTE
===========================================================================

                   +-------------------------+
                   |   Your Application      |
                   |  (instrumented with     |
                   |   client library)       |
                   |                         |
                   |   GET /metrics    +---->|---- plaintext exposition
                   +-----------^-------------+     counter/gauge/histogram
                               |
                               | HTTP pull every 15s (default)
                               |
    +--------------------------+---------------------------+
    |                                                      |
    |         +-----------------+       +---------------+  |
    |         | kubernetes_sd   |------>| relabel_configs| |    PROMETHEUS SERVER
    |         | (lists pods)    |       | (filter/rename)| |    (single binary)
    |         +-----------------+       +-------+--------+  |
    |                                           |           |
    |                                           v           |
    |                                   +---------------+   |
    |                                   | SCRAPER       |   |
    |                                   | (HTTP client) |   |
    |                                   +-------+-------+   |
    |                                           |           |
    |                                           v           |
    |                                   +---------------+   |
    |                                   | metric_relabel|   |
    |                                   +-------+-------+   |
    |                                           |           |
    |                     +---------------------+           |
    |                     v                                 |
    |             +---------------+   +-----------------+   |
    |             | TSDB (local)  |-->| remote_write    |---|---> Thanos / Mimir /
    |             | blocks + WAL  |   | (queue-sender)  |   |     Cortex / VM / AMP
    |             +-------+-------+   +-----------------+   |
    |                     |                                 |
    |                     v                                 |
    |             +---------------+                         |
    |             | Rule engine   |                         |
    |             | (recording +  |                         |
    |             |  alerting)    |                         |
    |             +-------+-------+                         |
    |                     |                                 |
    |                     v                                 |
    +---------------------+---------------------------------+
                          |
                          | firing alerts
                          v
                 +-----------------+
                 | Alertmanager    |----> PagerDuty / Slack /
                 | (separate bin.) |      email / webhook
                 +-----------------+

                         ^
                         |  PromQL queries
                         |
                 +-----------------+
                 | Grafana / API   |
                 | / promtool      |
                 +-----------------+

    EXCEPTIONS:
      (1) Short-lived batch jobs push to Pushgateway -> Prometheus scrapes
          the gateway. Single-purpose; never for service metrics.
      (2) Non-instrumentable systems (Linux host, MySQL, Redis, HTTP probe)
          use Exporters that translate to /metrics. Same scrape model.
```

Five takeaways before diving in:

1. **The server is one binary.** It scrapes, stores, evaluates rules, and serves PromQL queries. No external DB.
2. **Everything talks HTTP/plaintext.** You can `curl` any `/metrics` endpoint on your laptop and read it.
3. **Pull by default.** Push (Pushgateway) exists but is reserved for one narrow case.
4. **Local storage only out of the box.** Long-term + HA = bolt on remote-write + Thanos/Mimir/Cortex/VictoriaMetrics.
5. **Alertmanager is separate.** The Prometheus server fires; Alertmanager routes/groups/silences. Different binary, different port.

---

## Part 1: The Pull Model -- Why Prometheus Scrapes

The biggest culture-shock concept coming from CloudWatch, StatsD, Datadog Agent, or OpenTelemetry-push is that **Prometheus scrapes you, not the other way around**. You don't send metrics; you expose them. The server initiates.

### Pull vs Push -- Side by Side

| Dimension | Pull (Prometheus) | Push (StatsD / CloudWatch / OTel push) |
|-----------|-------------------|----------------------------------------|
| Who initiates | Monitoring server | Application |
| Target discovery | Server-side SD (k8s/EC2/Consul/DNS) | App needs to know the collector endpoint |
| "Is the app up?" | Free `up{job=...}` metric per scrape attempt | Absence of data is ambiguous (down? or just quiet?) |
| Back-pressure | Server paces itself (scrape_interval) | Server can be overwhelmed by bursty pushers |
| Firewalls | Needs inbound from Prom to app | Outbound from app to collector -- easier across VPCs |
| Short-lived jobs | Problem -- may die before scrape | Natural fit |
| Horizontal scaling | Shard Prometheus by job | Shard the collector, typically by hash |
| Local testing | `curl http://localhost:8080/metrics` just works | Need a mock collector |

### Why Pull Is the Better Default

Brian Brazil (Prometheus co-founder) has written this defense a dozen times; the distilled version:

- **You get `up` for free.** Every scrape attempt produces a time series `up{job, instance} = 1 or 0`. You *always* know whether the target is reachable. In push systems, silence is ambiguous.
- **Operator control.** When you want to debug a service, you can `curl /metrics` from your laptop or run a local Prometheus against it. No deployment needed.
- **No runaway clients.** A misbehaving client can't DDoS the monitoring server by pushing at 10 kHz. The server paces itself.
- **Targets don't need to know the collector.** Deploying a second Prometheus (e.g., for staging or a team-local instance) requires zero changes in the application.
- **Scrape failures are observable.** Every scrape emits `up` (0/1) plus `scrape_duration_seconds`, `scrape_samples_scraped`, `scrape_samples_post_metric_relabeling`, and `scrape_series_added`. For the *reason* a scrape failed (timeout, DNS, 500, parse error), check the `/targets` page's `lastError` field -- it's not a time series, but it's the ground truth.

### When the Push Model Genuinely Wins

There is exactly one use case where pull breaks: **service-level batch jobs** that live less than a scrape interval. Think: a nightly ETL, a DB migration Job, a CI build's test-duration reporter. These jobs will finish and exit before Prometheus ever walks by their house. For this, Prometheus ships the **Pushgateway**.

### The Pushgateway -- Narrowly Useful, Widely Misused

Pushgateway is a **cache**: jobs push to it; Prometheus scrapes it. The trap is assuming you can use it as a general-purpose push collector. You cannot. Here's why:

| Problem | What goes wrong |
|---------|-----------------|
| **No `up` metric per job** | Pushgateway hides individual job health. You lose the "is it alive?" signal. |
| **Metrics stay forever** | Unless you explicitly delete them, old job metrics linger in the gateway, rescraped every interval, looking active. Stale-series nightmare. |
| **Single point of failure** | Every pushing job depends on one gateway; if it's down, you lose data silently. |
| **Label-collision risk** | Multiple jobs with overlapping labels overwrite each other -- classic `instance` label footgun. |
| **Not for service metrics** | Service-level metrics change between scrapes; Pushgateway just caches the last push forever, destroying your rate() accuracy. |

**Use the Pushgateway only for**: **service-level** batch jobs -- jobs whose outcome isn't tied to any specific host (e.g., "delete expired accounts across the whole platform," a one-off CI test reporting pass/fail). **Never** for long-running services, and **not** for host-bound batch jobs -- those have a better option below.

### The Textfile Collector -- The Better Answer for Host-Bound Batch Jobs

Before reaching for the Pushgateway, ask: does this batch job run on a specific host? If yes (nightly DB exports, cron-scheduled maintenance, any job with a sensible `instance` label), **the `node_exporter` textfile collector is almost always the right answer, not the Pushgateway**. The official Prometheus docs explicitly recommend it for this case.

How it works:
- Your job writes metrics in exposition format to a `.prom` file in `/var/lib/node_exporter/textfile_collector/` (atomic: write to `.tmp`, then `rename`).
- `node_exporter` (already running, already scraped) reads every `.prom` file in that directory on each scrape and serves the contents as part of its own `/metrics` response.
- Prometheus scrapes node_exporter normally. Zero new infrastructure.

Why it beats Pushgateway for host-bound jobs:
- **The `up` signal is preserved.** You're scraping node_exporter, which reflects real host liveness. Pushgateway's `up=1` reflects only the Pushgateway itself, not whether the job ran.
- **Lifecycle tied to the filesystem.** Host dies → file dies → no stale ghost metrics.
- **Free staleness signal.** Textfile collector auto-emits `node_textfile_mtime_seconds{file="..."}`, so you can alert on `time() - node_textfile_mtime_seconds{file="nightly_export.prom"} > 90000` (file not updated in 25 hours) as a direct liveness check on the job.
- **No SPOF.** No Pushgateway to deploy, scale, HA, or secure.

**One more correctness note**: a "once-per-run" value (`rows_exported`, `last_success_timestamp`) is semantically a **gauge**, not a counter. The `_total` suffix is a naming lie: `rate()` / `increase()` would treat each day's run as a counter reset and produce garbage. Name these as gauges (`nightly_export_rows_exported`, `nightly_export_last_success_timestamp_seconds`) regardless of the transport.

### Decision: Push, Pull, or Textfile?

```
Is it a batch job?
  YES -> Does it run on a specific host/node?
           YES -> node_exporter textfile collector (write .prom file, node_exporter serves it)
           NO  -> Is it a service-level job with no host context?
                    YES -> Pushgateway (single end-of-run push via HTTP API)

  NO  -> Long-running process:
           Can you add instrumentation to the code?
             YES -> /metrics via client library + pull scrape
             NO  -> Run an exporter alongside it (node, mysqld, redis, blackbox)
```

---

## Part 2: The Data Model -- Metric + Labels + Timestamp + Float

This is the one slide that, once internalized, makes every other Prometheus concept fall into place.

A Prometheus **time series** is uniquely identified by:

```
metric_name{label_key1="value1", label_key2="value2", ...}
```

Each sample on that series is:

```
(timestamp_ms_unix_epoch, float64_value)
```

That's it. No nested objects, no strings, no arrays, no events. Just a name, a label set, and a stream of `(t, v)` pairs.

### Example

```
http_requests_total{method="GET", handler="/api/users", status="200"}  ->  47523 @ 1713456000000
http_requests_total{method="GET", handler="/api/users", status="500"}  ->  12    @ 1713456000000
http_requests_total{method="POST", handler="/api/users", status="201"} ->  1843  @ 1713456000000
```

Each of those lines is a **separate time series**. Same metric name, different label combinations. That last detail is the whole ball game.

### The Cardinality Rule

**Every unique combination of labels is a new time series.** This has a direct operational consequence: if you put a high-cardinality value (user ID, request path with IDs in it, trace ID, timestamp) into a label, you will explode Prometheus memory and die.

The spice-jar analogy: imagine a spice rack where you're only allowed a fixed number of jars, and each jar costs memory. Labels like `method="GET"`, `status="500"`, `handler="/api/users"` are fine -- there are maybe 10 methods, 10 common statuses, 100 handlers. That's `10 x 10 x 100 = 10,000` jars. Fine.

Now someone adds a label `user_id="alice"`. You have 5 million users. That's 50 **billion** jars, each one costing on the order of a few kilobytes of RAM (Prometheus itself budgets roughly ~3 KB per active series across head + WAL + index, quantified in Part 7). Your Prometheus OOMs within minutes. The rack snaps.

### Cardinality Rules of Thumb

| Safe label values | Dangerous label values |
|-------------------|------------------------|
| HTTP method (GET/POST/...) | User ID, account ID, request ID |
| HTTP status code | URL path with IDs baked in (`/api/users/123`) |
| Handler name (the template, not the instantiated URL) | Trace ID / span ID |
| Environment (dev/stg/prod) | Email address |
| Region / AZ | Timestamp |
| Service / job / instance | IP address of individual client |
| Pod name (bounded by pod count) | Session ID |
| Kubernetes namespace | Full SQL query text |

**Target**: keep total active time series per Prometheus under a few million. Beyond that, shard, federate, or use remote write + Thanos/Mimir.

### Active Cardinality vs Churn -- The Kubernetes Trap

"Active series count looks fine, why is Prometheus OOMing?" is usually **churn**, not steady-state cardinality. Churn is the rate at which series are *created and destroyed* over time. A cluster doing a deploy every 15 minutes rotates ReplicaSet hashes through pod names (`api-6f8d9c4b7-abcde` → `api-7a1e2f3c8-xyzab`), which means every deploy creates a fresh generation of time series and deprecates the old one.

A Prometheus with **500K active series but 5M series churned through WAL over the retention window** will OOM on WAL replay even though the steady-state count looks safe. The index has to track every series that existed within retention, not just the ones currently reporting.

Diagnostics:
- `prometheus_tsdb_head_series_created_total` -- cumulative count of series ever admitted to the head. Take `rate()` of this.
- `prometheus_tsdb_head_series_removed_total` -- the other side of the equation.
- `prometheus_tsdb_head_series` -- current live series in the head block.

If `rate(prometheus_tsdb_head_series_created_total[5m])` is roughly matched by `rate(prometheus_tsdb_head_series_removed_total[5m])`, you have churn. Fix it by **dropping the pod name from labels** (use `statefulset_ordinal` or `deployment` instead where possible), or by relying on `kube_pod_info` for pod-level context only when you actually need it.

### Reserved Labels

Prometheus auto-attaches these:

- `job`: the `job_name` from your scrape config.
- `instance`: derived from `__address__`, which defaults to the discovered target's address (`host:port` for most SDs); can be remapped via `relabel_configs`.
- `__name__`: the metric name itself (you can query it: `{__name__=~"http_.*"}`).

Labels starting with `__` are **internal**. They're available during relabeling but stripped before storage -- which is what makes relabeling so powerful (more on this in Part 6).

---

## Part 3: The Four Metric Types

Prometheus has exactly four metric types. Getting the right type for the right measurement is fundamental.

### Counter -- The Odometer

**Monotonically increases, resets to 0 on process restart.** You never read the raw value; you always differentiate it (`rate()`, `increase()`).

- `http_requests_total` -- total requests served since process start
- `errors_total` -- total errors
- `bytes_sent_total` -- total bytes

Why monotonic? Because **rate() is more resilient than deltas**. A counter can be sampled inconsistently, missed for a scrape, or reset, and the math still works out. If the app instead pushed "number of requests in the last second," a dropped scrape would be a dropped second of data forever. With a counter, you just see `rate()` compute across a wider window and the missing second averages cleanly into the neighbors.

The `_total` suffix is a naming convention (officially required by OpenMetrics) that makes it visually obvious you need `rate()`.

```go
// Go client library -- counter
var httpRequests = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests served.",
    },
    []string{"method", "handler", "status"},
)

// responseWriter wrapper captures the status code written by downstream handlers.
type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(code int) {
    r.status = code
    r.ResponseWriter.WriteHeader(code)
}

func instrument(handler string, next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        rec := &statusRecorder{ResponseWriter: w, status: 200}
        next(rec, r)
        // Note: `handler` is the route TEMPLATE (e.g., "/api/users/{id}"),
        // not the instantiated URL -- using r.URL.Path here would explode cardinality.
        httpRequests.WithLabelValues(r.Method, handler, strconv.Itoa(rec.status)).Inc()
    }
}
```

### Gauge -- The Speedometer

**A single value that can go up and down.** You read it directly.

- `process_memory_bytes`
- `queue_depth`
- `temperature_celsius`
- `goroutines_count`

Gauges **do not need `rate()`**. They already represent "the current state." The right aggregation is `max()`, `min()`, `avg()`, `sum()` across instances.

```go
var queueDepth = prometheus.NewGauge(prometheus.GaugeOpts{
    Name: "job_queue_depth",
    Help: "Number of jobs waiting in the queue.",
})

queueDepth.Set(42)        // set absolute
queueDepth.Inc()          // ++
queueDepth.Dec()          // --
queueDepth.Add(5)         // += 5
```

### Histogram -- Binned Observations, Server-Side Aggregation

**Records observations (durations, sizes) into pre-configured buckets.** Exposes three series per histogram:

- `<name>_count` -- total observations (a counter)
- `<name>_sum` -- sum of all observed values (a counter)
- `<name>_bucket{le="<upper>"}` -- cumulative count per bucket (a counter per bucket)

```
http_request_duration_seconds_bucket{le="0.1"}   2847
http_request_duration_seconds_bucket{le="0.5"}   4523
http_request_duration_seconds_bucket{le="1.0"}   4891
http_request_duration_seconds_bucket{le="+Inf"}  4901
http_request_duration_seconds_sum                1832.4
http_request_duration_seconds_count              4901
```

**Quantiles are computed at query time** using `histogram_quantile()`. This means you can aggregate across pods/instances before computing the quantile (e.g., "p99 across the whole service"). This is the killer feature.

```go
var requestDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request latency in seconds.",
        Buckets: prometheus.ExponentialBuckets(0.005, 2, 12), // 5ms -> ~10s
    },
    []string{"method", "handler"},
)

requestDuration.WithLabelValues("GET", "/api/users").Observe(durationSeconds)
```

### Summary -- Client-Side Quantiles, Precise but Non-Aggregable

**Same `_count` and `_sum` as histogram, plus pre-computed quantiles at the client.**

```
http_request_duration_seconds{quantile="0.5"}    0.084
http_request_duration_seconds{quantile="0.9"}    0.312
http_request_duration_seconds{quantile="0.99"}   1.247
http_request_duration_seconds_sum                1832.4
http_request_duration_seconds_count              4901
```

Note the exposition shape: a summary uses the **bare metric name with a `quantile` label**, unlike a histogram which uses `<name>_bucket{le=...}`. Same `_sum` / `_count` conventions.

The quantiles are **not exact** -- they come from a streaming sketch (the Go client uses the CKMS algorithm) with a configurable error bound (default `{0.5: 0.05, 0.9: 0.01, 0.99: 0.001}`). They're precise within a tunable epsilon, typically tighter than a histogram's bucket granularity. But they are fundamentally **per-process**: you cannot average two processes' medians to get the chain's median (that math is broken). Beautiful in isolation. Broken at scale.

### Histogram vs Summary -- The Decision

| Dimension | Histogram | Summary |
|-----------|-----------|---------|
| Quantile accuracy | Approximate, bounded by bucket granularity | Approximate via streaming sketch, typically tighter epsilon than histogram buckets |
| Bucket configuration | Need to pick buckets thoughtfully | No buckets to configure |
| Aggregation across instances | **Yes** -- `histogram_quantile()` on summed buckets | **No** -- you cannot average percentiles |
| Client CPU cost | Low (bucket increment) | Higher (running quantile sketch) |
| Network / storage cost | One series per bucket (10-20 typical) | Three series (0.5, 0.9, 0.99 typical) |

**The rule**: use histograms. Always. The "cannot aggregate" limitation of summaries is catastrophic in microservice architectures -- you cannot compute a service-wide p99 from N pods' local p99s (that math is mathematically nonsense). Histogram's `histogram_quantile(0.99, sum by (le) (rate(foo_bucket[5m])))` just works.

The only time to reach for summary: when you have exactly one instance of something, the observation rate is low, you want the tight sketch epsilon on that single process, and you know you'll never horizontally scale. Rare.

### Native Histograms -- The 2024-Era Answer

Classic histograms have a real cost: 10-20 pre-picked buckets per metric per label combination, multiplied across all your services, is a lot of series. Pick the buckets wrong for your latency distribution and you either get poor accuracy or over-spend on cardinality.

**Native histograms** (introduced experimentally in Prometheus v2.40, stabilizing through v3.x) solve this with **sparse, exponential buckets chosen automatically at scrape time**. Instead of one series per bucket, the entire histogram becomes **one series** whose value carries a compact schema-encoded bucket distribution. The effect:

- 1 series per histogram instead of ~12 -- massive cardinality win, especially in remote_write costs.
- No manual bucket picking; bucket resolution is controlled by a single `NativeHistogramBucketFactor` per metric on the client side (e.g., `1.1` gives ~10% relative error across any latency range).
- Same `histogram_quantile()` query syntax works; the function just operates on the single-series representation.
- Requires **OpenMetrics or Protobuf exposition** (the classic text format can't carry the schema).
- Opt-in per metric on the client (Go and Python clients support it); opt-in on the server via `--enable-feature=native-histograms`.

The decision in 2026: for greenfield services, use native histograms. For existing classic histograms, migrate on next touch. The rule "always histogram over summary" remains -- native histograms just sharpen the gap further.

#### Migrating from Classic to Native Histograms

You won't do this big-bang; dashboards, alerts, and recording rules live too long. The safe rollout leverages the fact that Go and Java clients can **emit both flavors simultaneously** if you set both `Buckets` and `NativeHistogramBucketFactor` on the same metric.

**Expected series reduction** is roughly the bucket count: a classic histogram with 12 buckets exposes ~14 series per label combo (`_bucket × 12` + `_sum` + `_count`); a native histogram exposes **1 series** per label combo. 200 services × 3 histograms × 10 label combos: **~84K classic series → ~6K native series (~14×)**. Per-sample size grows (native samples carry a schema-encoded bucket distribution), but series count -- the Mimir cost driver -- collapses.

**Phase 0 -- server-side first (upstream before downstream):**
- Mimir: `native_histograms_ingestion_enabled: true` per tenant.
- Prometheus: `--enable-feature=native-histograms`.
- Canary one service end-to-end before proceeding.

**Phase 1 -- dual-emit rollout:**
- Update Go/Java client code: keep existing `Buckets` AND add `NativeHistogramBucketFactor: 1.1` (~10% relative error). Both flavors emit side by side. Series count rises ~7% temporarily. Existing PromQL queries keep working.

**Phase 2 -- migrate PromQL:**
- Classic: `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`
- Native: `histogram_quantile(0.99, sum(rate(http_request_duration_seconds[5m])))` -- no `by (le)`, no `_bucket` suffix.
- Programmatic access uses `histogram_count()`, `histogram_sum()`, `histogram_fraction()` in place of the `_sum` / `_count` suffix patterns.
- Run old and new queries in parallel across dashboards, alerts, and recording rules -- verify results match before cutting over.

**Phase 3 -- flip clients to native-only:**
- Remove `Buckets` from client code, service-by-service. Rollback path: restore `Buckets`.

**Phase 4 -- cleanup:**
- Delete dead `_bucket` / `_sum` / `_count` references from repos.

**Prerequisites that will bite if missed:**
- **Remote_write 2.0** required -- RW 1.0 does not carry native histograms.
- **Grafana 10+** required for native-histogram heatmap rendering.
- **Federation** needs native-histogram scrape options explicitly enabled.
- **Library support**: Go and Java are most mature in 2026; Python is catching up; some ecosystems (Ruby, PHP) lag. Mixed fleets need longer dual-emit windows.

**The single silent-failure risk**: alerts referencing `_bucket` / `_count` going empty after Phase 3 on migrated services. Empty PromQL expressions don't fire. **Dashboards going empty is visible; alerts going quiet is invisible.** Mitigate with a complete query inventory before starting, parallel alert rules during Phase 2, and synthetic alert-fire tests during the cutover.

### Exemplars -- The Metrics-to-Traces Bridge

An **exemplar** is a `(traceID, timestamp [, additional labels])` pointer attached to a single histogram bucket observation. It's the glue between your metrics and your traces.

How it works:
1. Your instrumented code observes a latency while a trace is active. The client library grabs the current `traceID` from the span context and attaches it as an exemplar on the histogram bucket that observation fell into.
2. The exposition format (OpenMetrics, not classic) carries exemplars alongside bucket values.
3. Prometheus stores them in a ring buffer when started with `--enable-feature=exemplar-storage`.
4. Grafana visualizes exemplars as clickable dots overlaid on heatmaps and quantile panels. Click the dot → jump to the exact Tempo/Jaeger trace that generated that slow request.

This is why "metrics, traces, logs converge" is not marketing handwaving. An exemplar is the two-line YAML change that turns "our p99 just spiked" into "here's the actual trace that caused it." Friday's kube-prometheus-stack doc sets this up; Tuesday's Distributed Tracing doc (next week) closes the loop.

### Naming and Unit Conventions

- Counters end in `_total`. (`http_requests_total`, not `http_requests`.)
- Units use SI base units: **seconds**, **bytes**, **meters**. Not milliseconds, not kilobytes.
  - `http_request_duration_seconds`, not `http_request_duration_ms`.
  - `memory_bytes`, not `memory_mb`.
- Name starts with the subsystem or library name: `node_cpu_seconds_total`, `mysql_global_status_queries`.

---

## Part 4: Instrumentation and the Exposition Format

Every metric Prometheus ever sees arrives as plaintext over HTTP. The format is dead-simple -- you could literally `cat` a file and serve it.

### The Exposition Format

```
# HELP http_requests_total Total HTTP requests served by this process.
# TYPE http_requests_total counter
http_requests_total{method="GET",handler="/api/users",status="200"} 47523
http_requests_total{method="GET",handler="/api/users",status="500"} 12
http_requests_total{method="POST",handler="/api/users",status="201"} 1843

# HELP process_memory_bytes Resident memory in bytes.
# TYPE process_memory_bytes gauge
process_memory_bytes 134217728

# HELP http_request_duration_seconds HTTP request latency in seconds.
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{handler="/api/users",le="0.1"} 2847
http_request_duration_seconds_bucket{handler="/api/users",le="0.5"} 4523
http_request_duration_seconds_bucket{handler="/api/users",le="+Inf"} 4901
http_request_duration_seconds_sum{handler="/api/users"} 1832.4
http_request_duration_seconds_count{handler="/api/users"} 4901
```

Rules:
- `# HELP` and `# TYPE` metadata lines are optional but conventional.
- Each sample is one line: `name{labels} value [timestamp]`.
- Timestamp is almost always **omitted** -- Prometheus attaches its own scrape-time timestamp, which is what you want.
- Empty labels OK: `up 1` is valid.

### Client Libraries and What They Do

Official clients: **Go, Java, Python, Ruby**. Third-party but production-grade: **Node.js, Rust, .NET, C++, PHP**. They all do the same thing:

1. Provide `Counter`, `Gauge`, `Histogram`, `Summary` primitives.
2. Auto-serve a `/metrics` HTTP handler that renders the current state in exposition format.
3. Auto-instrument some default process metrics (CPU time, memory, fds, Go GC, JVM heap, etc.).

### Python Client -- Minimal FastAPI Example

```python
from fastapi import FastAPI, Request
from prometheus_client import Counter, Histogram, make_asgi_app
import time

app = FastAPI()

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "handler", "status"],
)
REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["method", "handler"],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0),
)

@app.middleware("http")
async def record_metrics(request: Request, call_next):
    start = time.monotonic()
    status_code = 500  # default if call_next raises
    try:
        response = await call_next(request)
        status_code = response.status_code
        return response
    finally:
        elapsed = time.monotonic() - start
        # Use the TEMPLATED route path ("/users/{id}"), not the instantiated URL.
        # request.url.path would explode cardinality -- every user ID = new series.
        route = request.scope.get("route")
        handler = route.path if route else request.url.path
        REQUEST_DURATION.labels(request.method, handler).observe(elapsed)
        REQUEST_COUNT.labels(request.method, handler, str(status_code)).inc()

# Mount /metrics for Prometheus to scrape
app.mount("/metrics", make_asgi_app())
```

### Exporters -- Translating Non-Instrumented Systems

For things you can't modify (Linux, PostgreSQL, Redis, nginx, a closed third-party service), use an **exporter**. An exporter is a separate process or sidecar that:

1. Talks to the target system in its native protocol.
2. Translates the state into Prometheus exposition format.
3. Exposes a `/metrics` endpoint Prometheus can scrape.

Common exporters:

| Exporter | What it monitors |
|----------|-----------------|
| `node_exporter` | Linux host metrics (CPU, memory, disk, network, filesystem) |
| `cadvisor` | Container-level CPU/memory/IO (usually kubelet-embedded) |
| `kube-state-metrics` | Kubernetes object state (deployment replicas, pod phase, etc.) |
| `blackbox_exporter` | HTTP/TCP/ICMP/DNS probes ("is this URL up?") |
| `mysqld_exporter` | MySQL `SHOW STATUS`, `SHOW GLOBAL VARIABLES` |
| `redis_exporter` | `INFO` command of a Redis server |
| `postgres_exporter` | `pg_stat_*` views |
| `snmp_exporter` | Network devices via SNMP |
| `statsd_exporter` | Bridges push-based StatsD metrics into pull model |

---

## Part 5: Scrape Configs -- The Route Book

The full `prometheus.yml` has four top-level sections most of the time:

```yaml
global:
  scrape_interval: 15s           # default; override per job
  scrape_timeout: 10s            # must be < scrape_interval
  evaluation_interval: 15s       # how often to evaluate rules
  external_labels:
    cluster: "use1-prod"         # attached to remote_write samples only
    replica: "0"                 # distinguishes HA replicas in Thanos/Mimir
    # NB: external_label VALUES must be strings -- always quote them, especially numerics

rule_files:
  - "rules/*.yml"

scrape_configs:
  - job_name: prometheus          # Prometheus scrapes itself
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node                # bare-metal hosts
    scrape_interval: 30s
    static_configs:
      - targets:
          - "host1.example.com:9100"
          - "host2.example.com:9100"
        labels:
          env: prod
          role: worker

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager.monitoring.svc:9093"]
```

### Scrape Interval and Timeout

- `scrape_interval`: typically **15-30s**. Below 10s, storage costs grow quickly and sample-level noise often exceeds signal; sub-15s is reserved for genuinely latency-sensitive workloads where the cost is justified. Above 60s is too coarse for most alerting.
- `scrape_timeout`: must be **less than scrape_interval**. 10s is a common default. Misconfiguration here is a classic bug -- if timeout > interval, you can have overlapping scrapes.
- Prefer **consistency across jobs**. Different intervals per job make `rate()` windows awkward.

### `static_configs` vs Service Discovery

- `static_configs` is fine for a dozen stable hosts.
- Anything dynamic (Kubernetes, EC2 ASGs, Consul, Nomad) should use service discovery. Hand-maintaining target lists in production is an anti-pattern.

---

## Part 6: Service Discovery and Relabeling

This is the "aha" part of Prometheus. If you don't internalize relabeling, Prometheus feels like magic. If you do, it's Lego.

### Kubernetes Service Discovery

Prometheus ships with native Kubernetes SD. You give it credentials (or run it in-cluster and use the pod's service account), and it queries the API server for targets. Five **roles**:

| Role | Discovers | Typical use |
|------|-----------|-------------|
| `node` | All cluster nodes | Scrape `node_exporter` / kubelet per host |
| `endpoints` | All endpoints of all services | Scrape apps behind a Service (pre-EndpointSlice default) |
| `endpointslice` | EndpointSlices (modern, scalable) | Same as above, better at scale |
| `pod` | Every pod in the cluster | Scrape pods directly without needing a Service |
| `service` | Service ClusterIPs | Blackbox-probe services |
| `ingress` | Ingress hosts | Blackbox-probe external URLs |

Each discovered target arrives with a big bag of **`__meta_*` labels** describing it:

```
__meta_kubernetes_namespace
__meta_kubernetes_pod_name
__meta_kubernetes_pod_node_name
__meta_kubernetes_pod_label_<key>        # each pod label becomes one
__meta_kubernetes_pod_annotation_<key>   # each pod annotation becomes one
__meta_kubernetes_pod_container_name
__meta_kubernetes_pod_container_port_number
__meta_kubernetes_pod_ip
__meta_kubernetes_pod_ready
__meta_kubernetes_service_name
... (30+ more)
```

These are **temporary**. They exist during relabeling and are **dropped before samples hit the TSDB** -- unless you `replace` them into regular labels via `relabel_configs`.

**Footnote on annotation names**: Kubernetes annotation keys like `prometheus.io/scrape` contain `/` and `.`, both of which are illegal in Prometheus label names. Prometheus SD replaces them with `_` when turning the annotation into a `__meta_*` label, so `prometheus.io/scrape` becomes `__meta_kubernetes_pod_annotation_prometheus_io_scrape`.

### Relabeling -- The Airport Baggage System

**Think of `__meta_*` labels as the TSA pre-check tag stuck to your suitcase at the airline counter.** Every station -- security, gate, baggage loader -- reads that tag to decide what happens next, and it's stripped off before the bag hits the carousel at your destination. `relabel_configs` is where a station copies useful information from that temporary tag onto the permanent luggage label (or rejects the bag outright). At each station, a worker can:

- **`keep`**: let the part continue only if it matches a pattern -- otherwise eject.
- **`drop`**: eject the part if it matches.
- **`replace`**: stamp a new label on the part based on existing labels (the most common action).
- **`labelmap`**: copy a bunch of labels at once based on a regex on names.
- **`labeldrop`**: remove labels whose names match a regex.
- **`labelkeep`**: remove all labels except those matching a regex.
- **`hashmod`**: compute `hash(label) mod N` and use it to shard targets across N Prometheus replicas.

### Two Phases: `relabel_configs` vs `metric_relabel_configs`

| Phase | When | Acts on | Typical use |
|-------|------|---------|-------------|
| `relabel_configs` | **Before scrape**, on each discovered target | Target labels (`__address__`, `__meta_*`, `instance`, `job`) | Filter which pods to scrape, set the scrape address/port/scheme, map pod labels to series labels |
| `metric_relabel_configs` | **After scrape**, on each sample | The sample's labels (metric name + labels) | Drop noisy metrics, rename metric names, strip high-cardinality labels |

A third phase, `write_relabel_configs`, runs on samples being sent to remote_write -- useful for downsampling to long-term storage.

### Canonical Kubernetes Pod Scrape Job

Here is a realistic `pod` role job: scrape every pod that has annotation `prometheus.io/scrape=true`, respect `prometheus.io/port`, and pull pod labels into Prometheus labels.

```yaml
- job_name: kubernetes-pods
  kubernetes_sd_configs:
    - role: pod

  relabel_configs:
    # 1) Only scrape pods annotated prometheus.io/scrape=true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: "true"

    # 2) If pod annotates prometheus.io/path, use it; else default /metrics
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)

    # 3) If pod annotates prometheus.io/port, use pod_ip:that_port as scrape address
    - source_labels:
        - __meta_kubernetes_pod_ip
        - __meta_kubernetes_pod_annotation_prometheus_io_port
      action: replace
      target_label: __address__
      regex: (.+);(\d+)
      replacement: $1:$2

    # 4) Copy ALL pod labels (app, version, team, ...) into series labels
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)

    # 5) Promote namespace and pod name to first-class labels
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: pod

  # Second phase: drop a noisy histogram bucket series
  metric_relabel_configs:
    - source_labels: [__name__]
      action: drop
      regex: go_gc_heap_allocs_by_size_bytes_bucket
```

The pattern to internalize: **`__meta_*` is the temporary baggage tag; `replace` is how you copy information from it onto the permanent luggage label before the tag gets stripped.**

### The kube-prometheus-stack / ServiceMonitor Connection (Preview -- Friday)

In production Kubernetes, you almost never write scrape configs like the above by hand. Instead, you install the **Prometheus Operator** (bundled in the `kube-prometheus-stack` Helm chart), which adds CRDs: `ServiceMonitor`, `PodMonitor`, `PrometheusRule`, `Probe`, and `Prometheus`. Each `ServiceMonitor` is a declarative abstraction that the Operator compiles down to raw `scrape_configs` + `relabel_configs` entries in a generated `prometheus.yml`. We cover this in depth on **Friday, Apr 24**. For today, the mental model is: writing a `ServiceMonitor` is writing today's relabel config with training wheels.

### Hashmod -- Horizontal Sharding

When one Prometheus can't scrape everything, shard:

```yaml
relabel_configs:
  - source_labels: [__address__]
    modulus: 4                          # 4 shards
    target_label: __tmp_hashmod
    action: hashmod
  - source_labels: [__tmp_hashmod]
    regex: "2"                          # this replica scrapes shard 2 (of 0..3)
    action: keep
```

Each replica keeps only its slice. Pair with per-replica `external_labels: { replica: "2" }` and remote_write to a deduplicating backend (Thanos Receive, Mimir) to reconstruct a unified view.

### Debugging Targets -- The State Lifecycle

When something "isn't scraping," walk the `/targets` page in this order:

| Signal | What it means |
|--------|--------------|
| Target not listed at all | SD filter ejected it. Check `relabel_configs` `keep`/`drop` rules. |
| Target listed, state `down` | SD found it, scrape is failing. Check `lastError` column -- DNS, timeout, TLS, 404 on `/metrics`. |
| Target `up=1`, `scrape_samples_scraped > 0` but `scrape_samples_post_metric_relabeling == 0` | **All samples were dropped by `metric_relabel_configs`.** Classic foot-gun: an overzealous `drop` regex matched every metric. |
| Target `up=1`, samples present, but no series in PromQL | Labels were relabeled out of existence (e.g., you `labeldrop`ed the `job` label). |

The two series to watch per target: `scrape_samples_scraped` (raw count off the wire) vs `scrape_samples_post_metric_relabeling` (what survived the post-scrape filter). A big gap = over-aggressive metric_relabel rules.

---

## Part 7: Storage -- The Local TSDB

Prometheus stores all samples locally on disk. No MySQL, no Cassandra, no S3 out of the box. The format is purpose-built for time-series:

### On-Disk Layout

```
/prometheus/
  wal/                              # write-ahead log
    00000001
    00000002
    ...
  chunks_head/                      # head chunks spilled from RAM and mmap'd
                                    # (the in-flight head appender stays in memory)
  01HXK5.../                        # finished 2-hour block (ULID-named)
    meta.json
    chunks/
      000001
    index
    tombstones                      # for deletes
  01HXK6.../
  01HXK7.../                        # older blocks get compacted into bigger blocks
```

Key mechanics:

- **WAL (Write-Ahead Log)**: every sample is first written to WAL, then held in the in-memory head block. Prometheus keeps enough WAL segments to cover ~3h of data (slightly more than one head-block window), so a crash replay can always reconstruct the head cleanly across a block boundary.
- **Head block**: the current (in-flight, mutable) 2-hour window. The latest samples sit in RAM; once head chunks reach ~120 samples they spill to disk as mmap'd files under `chunks_head/`. The head as a whole is what remote_write and queries read from for recent data.
- **2-hour immutable blocks**: head flushes to disk every 2 hours. Each block is immutable.
- **Compaction**: blocks get merged into larger blocks over time (2h -> 6h -> 24h -> ...). Reduces index overhead.
- **Retention**: when blocks fall outside `--storage.tsdb.retention.time` (default **15d**), they're deleted.

### Sizing Math

Rough rule of thumb for planning:

```
bytes_per_sample ≈ 1-2 (with compression; was ~3.5 older Prometheus)
storage_needed ≈ retention_seconds * samples_per_second * bytes_per_sample

Example: 1M active series, 15s scrape, 15d retention
  = 1,000,000 / 15 = ~66,666 samples/sec
  = 66,666 * 86,400 * 15 * 1.5 bytes
  ≈ 130 GB
```

Memory is typically dominated by active series count, WAL, and index. Rule of thumb: **~3 KB of RAM per active series** across head + WAL + index combined (not 3 KB each). 1M series ≈ 3 GB baseline, but real Prometheus instances at this scale need **8-16 GB** to absorb WAL replay spikes, query-time fan-out, and compaction overhead.

### Retention Configuration

```
--storage.tsdb.retention.time=30d           # time-based (default 15d)
--storage.tsdb.retention.size=500GB         # size-based cap (default: disabled)
--storage.tsdb.path=/prometheus             # storage root
```

If both are set, whichever limit hits first wins.

### Remote Write -- Long-Term and HA

Local TSDB is a weak story for:

- **Retention > 30-60d.** You'll run out of disk.
- **HA with deduplication.** Two Prometheus replicas scraping the same targets produce duplicate-but-not-identical series.
- **Horizontal query across shards.** When you shard, you lose unified PromQL.

The answer is `remote_write` -- a protocol that streams samples from Prometheus to an external store:

```yaml
remote_write:
  - url: https://mimir.example.com/api/v1/push
    queue_config:
      max_samples_per_send: 2000
      capacity: 10000
      max_shards: 200
    write_relabel_configs:
      # drop high-cardinality metrics from LTS to save cost
      - source_labels: [__name__]
        regex: go_gc_.*
        action: drop
```

Common remote-write targets:

| Backend | What it is | Sweet spot |
|---------|-----------|-----------|
| **Thanos** | Sidecar uploads 2h blocks to S3; Thanos Query federates multiple Prometheuses | Long-term on S3, global view, downsampling |
| **Cortex** | Horizontally-scalable multi-tenant Prometheus | Multi-team SaaS-like monitoring |
| **Mimir** | Grafana Labs' Cortex fork | Same space, more polished, Grafana Cloud's backend |
| **VictoriaMetrics** | Single-binary or cluster Prometheus replacement | Higher compression, lower TCO |
| **AWS Managed Prometheus (AMP)** | AWS-hosted remote_write endpoint | Managed, auth via SigV4, pairs with AMG |
| **Grafana Cloud / Chronosphere** | SaaS remote_write | No ops burden |

You rarely use `remote_read` directly anymore. Thanos Query / Mimir Query replace it with a smarter federated query layer.

### Staleness Markers -- How Prometheus Forgets

One of the most quietly elegant parts of the TSDB. When a target disappears from service discovery, or a previously-returned series is no longer in a scrape response, Prometheus **writes a staleness marker** on the next evaluation -- a special NaN value (`0x7ff0000000000002`) that PromQL treats as "this series is intentionally gone, do not carry the last value forward."

Why this matters for the mental model:

- **`absent()` works correctly.** Without staleness markers, `absent(up{job="api"})` would return false for up to 5 minutes after the target dies, because the last `up=1` would still be considered "current."
- **`rate()` doesn't hallucinate.** If a pod is deleted mid-5m window, staleness truncates its series cleanly instead of extrapolating the last value for the rest of the window.
- **The 5-minute lookback.** PromQL's instant queries look back up to 5 minutes for the most recent sample. Staleness markers short-circuit this lookback explicitly, so vanished series don't appear alive for 5 minutes post-death.
- **Target disappearance triggers it too.** When service discovery drops a target (pod deleted, instance terminated), Prometheus stamps staleness on every series from that target on the next scrape cycle.

The headline: staleness is why Prometheus can handle Kubernetes pod churn without dashboards lighting up with ghost metrics. It's invisible 99% of the time, and when you need to explain an interview question about "why doesn't my `absent()` alert fire immediately when a pod dies?" -- this is the answer.

---

## Part 8: Federation

Federation is **Prometheus scraping another Prometheus**. It answers "how do I aggregate per-cluster metrics into a global view?" -- the regional-manager-consolidating-store-reports problem.

Every Prometheus exposes a special endpoint: **`/federate`**. You `GET /federate?match[]=<series-selector>` and get back the current snapshot of every matching time series.

### Hierarchical Federation

```
                      GLOBAL PROMETHEUS
                      (queries, dashboards)
                             ^
                             |  scrapes /federate every 60s,
                             |  pulls only pre-aggregated rollups
                  +----------+------------+
                  |                       |
            US-EAST-1 PROM          EU-WEST-1 PROM
            (scrapes k8s,            (scrapes k8s,
             ec2, rds, lambda)        ec2, rds, lambda)
                 |                      |
           thousands of              thousands of
           per-pod series            per-pod series
```

Config on the **global** Prometheus:

```yaml
scrape_configs:
  - job_name: federate-regional
    honor_labels: true                 # keep cluster/region labels from child
    metrics_path: /federate
    params:
      match[]:                         # what series to pull up
        - '{__name__=~"job:.*"}'       # recording rules (the pre-aggregates)
        - '{__name__=~"node_cpu:.*"}'
        - 'up'
    scrape_interval: 60s
    static_configs:
      - targets:
          - prom-use1.monitoring.example.com
          - prom-euw1.monitoring.example.com
```

Two rules to engrave:

1. **Always pull only recording-rule outputs, not raw metrics.** Federating raw series is how federation turns into a cardinality bomb -- you just duplicated a million series.
2. **Set `honor_labels: true`.** Without it, the global Prometheus's `job` label would stomp the child's. You'd lose the `cluster` / `region` dimension.

### Cross-Service Federation

The other use: one team's Prometheus pulls a few selected metrics from another team's Prometheus. Small scale, pragmatic. Same mechanism, different intent.

### Federation Limitations

- **Not HA.** If the child is down during a scrape, you get a gap; no replication.
- **Not for long-term storage.** You can't backfill from a child; you only ever get the current snapshot.
- **Cardinality inherited.** If the child has 10M series and you match broadly, global gets 10M series.
- **For long-term + global**, use **remote_write to Thanos / Mimir / VictoriaMetrics / AMP**, not federation. Federation is for live aggregation; remote_write is for durability.

### `honor_labels` vs `honor_timestamps`

Two scrape-config knobs that matter specifically for federation (and for the Pushgateway):

- **`honor_labels: true`** -- already covered above. When the scraped exposition ships its own `job` / `instance` / `cluster` labels, the scraper does **not** overwrite them with its own. Without this, the global Prometheus's `job="federate-regional"` would stomp the child's `job="api"`. Essential when the source is itself an aggregator.
- **`honor_timestamps: true`** (the default) -- when the scraped exposition includes **explicit sample timestamps**, Prometheus honors them rather than stamping its own scrape-time timestamp. The `/federate` endpoint emits timestamps by default, so without `honor_timestamps` the global view would silently skew times. Set `honor_timestamps: false` to force the scrape-time timestamp, which is useful when you do **not** trust the source's clock (e.g., some exporters emit stale timestamps for cached data, making `rate()` lie).

In practice: for federation, leave both at their sensible defaults (`honor_labels: true`, `honor_timestamps: true`). For Pushgateway, `honor_labels: true` is non-negotiable.

---

## Part 9: Deploying Prometheus -- The Terraform + Helm Stub

In production Kubernetes, nobody installs Prometheus from raw binaries or hand-written manifests. The path is **kube-prometheus-stack**, wrapped in a Terraform `helm_release`. **Friday's doc covers the chart values, Operator CRDs, and ServiceMonitor wiring in depth** -- here is only the stub so you know what the Terraform shape looks like:

```hcl
resource "helm_release" "kube_prom" {
  name             = "kube-prom"
  namespace        = "monitoring"
  create_namespace = true

  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  version    = "65.1.0"   # check latest: artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack

  values = [file("${path.module}/values.yaml")]
}
```

Keep the `values.yaml` out of Terraform for anything non-trivial -- it's a few hundred lines in production. The one Terraform-local concern today is pinning the chart `version` so `terraform apply` is reproducible. Everything else (retention, storageSpec, externalLabels, ServiceMonitor selectors, remoteWrite endpoints) belongs to Friday's coverage of the chart itself.

---

## Part 10: Production Gotchas -- What Bites the First Time

### Gotcha 1: Cardinality Explosion From a Single Bad Label

Symptom: Prometheus RAM climbs, scrape intervals stretch, query latency balloons, then OOM-kill. A single bad commit can do this in minutes.

Cause: someone added `user_id`, `request_id`, `email`, or a path without templating (`/users/12345` instead of `/users/{id}`) to a metric label.

Prevention:
- **Code review any new label.** The question to ask: "what's the worst-case cardinality of this label?"
- **Cardinality analysis**: `topk(10, count by (__name__)({__name__=~".+"}))` shows your heaviest metric names; `count by (job) ({__name__=~".+"})` shows which job is the culprit.
- **Kill switch**: `metric_relabel_configs` with `labeldrop` regex on the offending label, deployed fast while the developer fixes the root cause.

### Gotcha 2: Counter Resets Treated as Drops

Counters reset to 0 on process restart. If you naively compute `current - previous`, you get a huge negative number.

Prometheus's `rate()` / `increase()` handle this correctly -- they detect a reset (sample_t < sample_{t-1}) and compensate. **Never hand-roll "deltas" over counter data in PromQL.** Always use `rate()` / `increase()`.

### Gotcha 3: `scrape_timeout >= scrape_interval`

If timeout equals or exceeds interval, scrapes overlap; memory and CPU double; the `up` metric starts flapping. Always keep timeout strictly less than interval (common: `10s` timeout, `15s` interval).

### Gotcha 4: Forgetting `honor_labels` in Federation / Pushgateway

Without `honor_labels: true`, the scraper's `job` and `instance` labels overwrite the ones from the scraped source. In federation, you lose the `cluster` / `region` labels from child Prometheuses. In Pushgateway scraping, you lose the `instance` label from the pushing job.

Rule: `honor_labels: true` whenever scraping a source that is itself an aggregator.

### Gotcha 5: Using Pushgateway for Services

Any long-running service put behind the Pushgateway produces:
- Metrics that stick around after the service dies ("zombie metrics").
- No `up` signal.
- Broken `rate()` because the gateway just caches the latest push forever.

Rule: Pushgateway is **only** for service-level batch jobs. If it's a long-running process, expose `/metrics` directly and let Prometheus scrape it.

### Gotcha 6: No HA -- One Prometheus, One Single Point of Failure

A single Prometheus server is a single point of failure. During an upgrade or crash, you have a gap in your metrics and possibly your alerting.

Solution: **run two identical Prometheus replicas** with identical scrape configs, different `replica` external labels (e.g., `replica: "0"` and `replica: "1"`). Both scrape the same targets on the same schedule; both produce near-identical series whose timestamps differ slightly due to scrape jitter. Deduplication happens at two layers:

- **Alertmanager dedupes alerts by fingerprint** (the alert's full label set, with the `replica` external label stripped). Replicas A and B firing the same alert collapse to one notification -- no double-paging. This works because Prometheus strips the `replica` external label from outbound alerts before sending them to Alertmanager (configured by `--alertmanager.notification-queue-capacity` and Alertmanager's `cluster` settings in modern deployments, but the replica-label-stripping behavior is built-in).
- **Thanos Query / Mimir dedupe samples on read.** You mark `replica` as a "deduplication label" in the query layer (Thanos: `--query.replica-label=replica`; Mimir: `replica_label` in tenant config). At query time, the engine picks one replica's series per timestamp window, silently falling through to the other replica on gaps. Dashboards and PromQL see one smooth series.

Never rely on a single Prom for alerting in prod. And always set distinct `replica` external labels -- without them, the dedup math cannot tell replicas apart and you get double-counted rates.

**Debugging 2× inflation in dashboards**: when a Grafana panel shows exactly 2× the expected rate despite replicas being labeled correctly, dedup isn't running *for that query*. Before editing the PromQL, check the **query path**:
- Is Grafana pointing at the query-frontend / Thanos Querier, or bypassing it to hit the ingester / store-gateway directly?
- Is `replica_label` configured in the Mimir tenant (or `--query.replica-label=replica` in Thanos Querier)?
- Is the query hitting a recording rule whose pre-aggregated series was written **before** dedup and still carries `replica`?

Defensive PromQL: `sum without (replica) (rate(...))` is a no-op when dedup works and protects against the 2× when it doesn't. Cheap insurance.

### Gotcha 7: Ignoring the `for:` Duration in Alerting Rules

(Preview of Wednesday.) An alerting rule without `for:` fires on the very first evaluation where the condition is true. Any single-scrape blip (network hiccup, brief spike) pages your on-call. Always use `for: 5m` or similar to require the condition to persist before firing.

### Gotcha 8: Storage Retention vs Disk Size Mismatch

Setting `retention.time=30d` on a 100 GB disk, then cardinality grows, and now 30d of data = 200 GB. Prometheus doesn't retroactively shorten; it just crashes on full disk. Always pair `retention.time` **and** `retention.size` so the size cap provides the safety valve.

### Gotcha 9: `rate()` Graph Dips During Rolling Deploys

Symptom: `rate(http_requests_total[5m])` drops sharply to near-zero for 30-60 seconds during a rolling deploy, then snaps back. ALB / ingress logs show traffic was steady throughout. The graph is lying.

Cause: a rolling deploy rotates pod identities. Every new pod name produces a **new time series** -- counter values do not carry across identity changes. Two mechanics combine to produce the visible dip:
- Dying pods fall out of service discovery; Prometheus stamps staleness markers on their series, cleanly terminating them. These series correctly stop contributing.
- New pods are fresh -- `rate()` needs **≥2 samples** in its window before it can compute anything. For the first 15-30 seconds after pod-ready, a new pod's series returns nothing from `rate()`.

During the overlap, dying-pod series are ending while new-pod series haven't crossed the 2-sample threshold. Per-series `rate()` values sum to less than steady-state. The shorter your `rate()` window, the sharper and deeper the artifact.

Prevention:
- **Aggregate away the churning label**: `sum by (service) (rate(http_requests_total[5m]))`. Per-series gaps get averaged across the many pods that weren't affected by this particular deploy step.
- **Back dashboards and alerts with recording rules** that query pre-aggregated stable series, not raw churning ones.
- **Keep the `rate()` window comfortably wider than `scrape_interval`** -- the `4 × scrape_interval` rule of thumb. A `[1m]` window on a 15s scrape = only 4 samples, so a single churn event can wipe `rate()` to zero.

The one-line takeaway: **deploy-time dips in `rate()` graphs are a label-churn artifact, not a traffic event**. Diagnose by checking whether the dip correlates with pod-name rotation, not with real traffic.

---

## Decision Frameworks Recap

### Histogram vs Summary

```
Do you need to compute quantiles across multiple instances?
  YES -> Histogram (native histogram preferred in 2026+)
  NO (exactly one process, low rate, tight sketch epsilon on that single process, no horizontal scale ever) -> Summary
```

In 2026 with Native Histograms generally available, the answer is even more strongly "histogram" -- they solve most of summary's bucket-picking and cardinality objections.

### Pushgateway or Pull?

```
Is it a long-running service?                   -> Pull with /metrics endpoint
Is it a batch job longer than a scrape interval?-> Pull with /metrics (add a short-lived HTTP server at the end)
Is it a batch job shorter than a scrape interval?-> Pushgateway (single end-of-run push)
```

### Static or Service Discovery?

```
< 10 stable hosts, you control them all?        -> static_configs
Kubernetes?                                      -> kubernetes_sd_configs (or ServiceMonitor via Operator)
Dynamic cloud infra (EC2 ASG, ECS, Fargate)?    -> ec2_sd_configs / ecs_sd_configs
Third-party inventory system?                    -> consul_sd_configs / file_sd_configs
```

### Local TSDB or Remote Write?

```
Retention < 15-30 days, single Prometheus, no HA requirement?  -> Local TSDB only
HA required (two replicas with dedup)?                          -> Remote write to Thanos/Mimir (dedup on read)
Retention > 30 days, global query?                              -> Remote write to Thanos/Mimir/Cortex/VM/AMP
Multi-tenancy (separate teams, isolated data)?                  -> Mimir / Cortex (built for this)
```

### Federation or Remote Write for Multi-Cluster?

```
Need live operational dashboards aggregating clusters? -> Federation (with recording rules only)
Need long-term, historical, unified query?             -> Remote write to Thanos/Mimir
Need both?                                             -> Remote write (covers federation's use case too)
```

---

## Mental Models to Remember

- **Prometheus is the milkman, not the mailbox.** Pull by default; push (Pushgateway) is the narrow exception for short-lived batch jobs.
- **Every unique label combination is a new time series.** Cardinality is the single biggest operational risk. Think hard about labels before adding them; reject user IDs, paths with IDs, trace IDs, emails, timestamps.
- **Four metric types, one that matters most: histogram.** Counters for "how many," gauges for "how much right now," histograms for everything distribution-shaped. Summaries exist but rarely the right answer.
- **Counters are monotonic; you always `rate()` them.** The `_total` suffix is a signal to the reader. Use SI base units (seconds, bytes).
- **The data model is the whole ball game.** `metric_name{labels} value @ timestamp` -- that's all Prometheus stores. Everything else (PromQL, alerting, rules, federation) is a function over that shape.
- **Relabeling is the Lego mortar.** Service discovery produces `__meta_*` scratch labels; `relabel_configs` transforms them into real labels or filters targets; `metric_relabel_configs` runs per sample after scrape. Master the seven actions (keep, drop, replace, labelmap, labeldrop, labelkeep, hashmod) and you master production Prometheus configuration.
- **Local TSDB is the default, remote_write is the scale answer.** 15-day local retention is fine for most teams; for long-term or HA, bolt on Thanos/Mimir/Cortex/VictoriaMetrics/AMP via `remote_write`.
- **Federation is for live aggregation, not durability.** Pull only recording-rule outputs; never federate raw high-cardinality series. Use `honor_labels: true` to preserve `cluster`/`region` labels.
- **CloudWatch and Prometheus answer different questions.** CloudWatch is the AWS-integrated, push, pay-per-metric, low-cardinality-bounded story. Prometheus is the self-hosted, pull, open-protocol, cardinality-flexible story. Most production stacks run both.
- **Pushgateway is a foot-gun outside its narrow use case.** Metrics stick around forever, no `up` signal, single point of failure. Only for batch jobs with an end-of-run snapshot.
- **Run Prometheus as two replicas, not one.** Different `replica` external labels, same scrape config. Alertmanager dedupes alerts; Thanos/Mimir dedupes samples on remote_write.

Today seeded the model. Tomorrow is **PromQL Deep Dive** -- `rate()`, `irate()`, `increase()`, aggregation operators (`sum by`, `avg without`), `histogram_quantile()`, subqueries, and the common PromQL gotchas. Wednesday's **Alerting + Alertmanager** teaches how the rule engine evaluates PromQL on a schedule and how Alertmanager groups, routes, silences, and inhibits alerts. Thursday covers **Grafana** -- the visualization layer that makes these metrics legible to humans. Friday ties it all together with **kube-prometheus-stack and the Prometheus Operator**, where ServiceMonitor/PodMonitor/PrometheusRule replace the hand-written YAML you saw today. By week's end, the full observability spine -- from `/metrics` endpoint to paged on-call engineer -- is assembled.
