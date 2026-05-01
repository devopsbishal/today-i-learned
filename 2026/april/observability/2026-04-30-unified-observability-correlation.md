# Unified Observability -- Correlating Signals -- The Three Witnesses to One Incident

> The last ten days laid every individual brick of the observability stack. Apr 20-23 stood up the metrics pillar (Prometheus TSDB, PromQL, exemplars, Alertmanager). Apr 24 hung the dashboards on the wall (Grafana). Apr 27-28 wired up traces (OTel SDK, Collector pipelines, W3C Trace Context, Tempo, tail sampling). Apr 29 added the third pillar (Loki, structured metadata, LogQL, Alloy as the unified collector). Each of those days had a strong analogy and went deep on one signal in isolation. **Today is the day we stop talking about pillars and start talking about the *seams between them*.** Because the dirty secret of "three pillars of observability" is that having all three signals doesn't help you a bit if your only way to move between them is to copy-paste a trace_id out of one tab and into another at 3 AM.
>
> **The core analogy for today: the three signals are three witnesses to the same incident.** Metrics is the **security camera** -- it watched the whole building and knows precisely *when* the alarm tripped, what the rate of foot traffic was, and which door the heatmap lit up at. It does not know any individual person's name. Traces is the **GPS tracker** -- it followed one specific person room-by-room through the building, timing every door they opened, but it has no idea what that person was thinking or saying along the way. Logs is the **diary entry** -- the witness's own words, scrawled at every step, full of motivation and detail and the exact error message that was printed on the screen, but with no map of the building and no clock you can fully trust. Each of the three knows something the others don't. Each of the three is **useless on its own** for solving a real incident -- the camera tells you *something happened at 14:03* but not *what*; the GPS tells you *they were in the basement for 3 seconds too long* but not *why*; the diary tells you *"DB query timed out"* but not *which request* or *whether it's still happening*. **A real investigation needs all three witnesses pointing at the same incident with the same case-file number.** That case-file number, for per-request investigation, is the **trace_id**. For per-service investigation, it's the set of **OTel resource attributes** (`service.name`, `service.namespace`, `deployment.environment.name`, the `k8s.*` family). Without the case-file number, you have three witnesses telling unrelated stories. With it, you have one click from "p99 spike at 14:03" -> "this exact request is slow" -> "and here is the WARN log line written by that request, mid-flight."
>
> Two corollaries the analogy carries that show up everywhere below. **(1) The witnesses must agree on the suspect's name.** A camera that tags the person as "Subject 472", a GPS tracker that tags them as "user-id-payments-svc", and a diary that calls them "checkout-api" can never be cross-referenced even though all three witnessed the same event. This is why `service.name=checkout` MUST be byte-identical across metrics, traces, and logs -- and why the OTel semantic conventions are not bureaucratic clutter but the *literal* mechanism by which the join works. **(2) Some witnesses don't see every event.** A scheduled job that runs at 2 AM has no incoming HTTP request, therefore no `trace_id`, therefore *its log lines have no per-request join key*. The only join key those logs share with metrics and traces is the resource attribute set. This is why `service.name` is the *durable* correlation strategy and `trace_id` is the *opportunistic* one -- one always exists, the other only exists during a request lifetime. Production observability hinges on getting both right.

---

**Date**: 2026-04-30
**Topic Area**: observability
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Metrics signal | The security camera | Knows when and how often, doesn't know who or why |
| Traces signal | The GPS tracker | Knows the path one request took, doesn't know the prose |
| Logs signal | The diary entry | Knows what was happening in the request's head, doesn't know the map |
| Correlation | Giving every witness the same case-file number | A shared join key (trace_id, resource attributes) so all three stories line up |
| `trace_id` | The case-file number for one request | 32-hex W3C identifier; the per-request join key across all three signals |
| Resource attributes | The suspect's standard ID badge | `service.name` + `service.namespace` + `deployment.environment.name` + `k8s.*`; the per-service durable join key |
| Exemplars | Photographs stapled to the camera footage | A `trace_id` + value attached to a histogram bucket sample; the metrics->traces pivot |
| `tracesToLogsV2` | The "show me this person's diary entries during this trip" button | Tempo data source config that builds the LogQL query from a span's resource attributes + trace_id |
| `derivedFields` | The "click any case number in the diary to open its file" link | Loki data source config that turns a regex match in a log line into a clickable link to Tempo |
| Grafana Correlations | The cross-references in the front of every binder | Grafana 10+ Explore-level link API that lives above the data sources, source/target/transform model |
| LogQL metric query | "Count diary entries that say 'fire'" | `rate({service="api"} |= "ERROR" [5m])` -- generates a metric on the fly from log streams |
| Loki ruler | The librarian who tallies fire-count once a minute and writes it on the board | Server-side recording rules that materialize LogQL aggregations into real Prometheus series |
| Structured metadata (Loki 3.0+) | The case-file number written on the inside cover of the diary | Indexed-but-not-stream-defining; the *correct* home for `trace_id` in logs |
| Grafana Alloy | The single forensic team that interviews all three witnesses with the same form | One agent emitting metrics+logs+traces with byte-identical resource attributes; eliminates name mismatch |
| Head sampling | "We only kept GPS tracks for 1% of suspects" | SDK-side sampling decision; cheap but the dropped traces still produced log diary entries -- *orphan logs* |
| Tail sampling | "We kept GPS tracks only for the suspects who looked suspicious at the end" | Collector-side decision after seeing whole trace; preserves errors/slow but adds memory cost |
| Orphan log | A diary entry with a case number nobody can find a file for | A log line carrying a `trace_id` whose trace was head-sampled-out -- looks broken, isn't |
| `service.name` mismatch | Three witnesses calling the same suspect three different names | The single most common correlation killer; metrics says `cart-api`, logs says `cart`, traces says `cart-svc` |
| `deployment.environment.name` (semconv 1.27+) | The renamed badge field after the 2025 reissue | Replaces older `deployment.environment`; mixed-SDK fleets see both keys until everyone upgrades |
| Promtail (EOL Mar 2, 2026) | The retired forensic interviewer who used a different form | Cannot emit OTel resource attributes the same way Alloy can; mixed-fleet = mismatch |

---

## The Whole Picture in One Diagram

Before diving in, here is the entire correlation surface on one page. Every config knob below ties to an edge in this graph.

```
THE THREE WITNESSES AND THE FOUR PIVOTS
================================================================================

                       +--------------------+
                       | App with OTel SDK  |
                       | (active span ctx)  |
                       +---------+----------+
                                 |
                                 | emits all three signals with the same
                                 | resource attributes + trace_id+span_id
                                 v
                       +--------------------+
                       |   Grafana Alloy    |     <- the unified forensic team
                       |  (single agent,    |        OTel Collector core +
                       |   River syntax)    |        Prometheus + Loki blocks
                       +--+--------+-----+--+
                          |        |     |
                  metrics |   logs |     | traces (OTLP)
                  (scrape |  (OTLP |     |
                  or RW)  |   or push)   |
                          v        v     v
                  +-------+--+  +--+----+ +-+----------+
                  | Prom /   |  | Loki  | |  Tempo     |
                  | Mimir    |  |       | |            |
                  | (TSDB +  |  | (TSDB | | (block     |
                  | exemplar |  |  + SM | |  storage)  |
                  | ring buf)|  |  + S3)| |            |
                  +----+-----+  +---+---+ +----+-------+
                       ^            ^         ^
                       |            |         |
                       |    (the four pivots) |
                       |            |         |
                  +----+------------+---------+----+
                  |          GRAFANA               |
                  |                                 |
                  |  metrics ---(exemplars)--> trace|
                  |  trace --(tracesToLogsV2)-> log |
                  |  log --(derivedFields)----> trace
                  |  log --(LogQL/ruler)--> metric  |
                  |                                 |
                  |  also: Correlations API (10+)   |
                  +---------------------------------+

   THE TWO JOIN KEYS
   ---------------------------------
   per-request:  trace_id (32-hex W3C)  -- only exists during a request
   per-service:  service.name + service.namespace + deployment.environment.name
                 + k8s.namespace.name + k8s.pod.name + service.instance.id
                 -- exists on EVERY signal, EVERY time, including batch jobs
                    and startup logs that have no trace_id
```

Six things to keep in mind as we go:

1. **The trace_id is opportunistic; resource attributes are durable.** Build correlation on resource attributes first, trace_id second.
2. **The exemplar buffer is in-memory but WAL-backed.** A normal Prometheus restart replays exemplars from the WAL up to retention (~2h). What you actually lose is exemplars that exceeded WAL retention, or that aged out of the buffer because the buffer is sized smaller than the WAL window. Mimir/Thanos via remote_write is the durable home.
3. **Head sampling kills correlation.** A trace dropped at the SDK still produces logs with that trace_id -- those logs become orphans pointing at nothing.
4. **The unified collector is the cardinality enforcer.** One Alloy emitting all three signals with one resource attribute set is the only way to *guarantee* `service.name` matches across signals.
5. **trace_id belongs in structured metadata, not labels.** Loki 3.0+ made this a first-class slot; Loki 2.x users putting trace_id in labels are paying for it in stream cardinality every day.
6. **Grafana Correlations (10+) is the modern API.** `derivedFields` and `tracesToLogsV2` still work and still ship, but the strategic direction is the per-data-source link configs collapsing into the central Correlations editor over time.

---

## Part 1: The Universal Join Keys -- What "Correlation" Actually Means

Correlation across signals is just a join. The question is *what column do you join on*. There are exactly three answers.

### Join key #1: `trace_id` (per-request)

The W3C Trace Context (`traceparent` header, see Apr 28) gives every request a 32-hex 128-bit `trace_id` that propagates through every service touching the request. If your SDK injects that trace_id into every metric exemplar, every span, and every log line written during the request, then `WHERE trace_id = X` is a single-column join across all three signals.

**Strengths:** dead-simple, works at the granularity of "the one request the user complained about."
**Weaknesses:** (a) only exists during an active request -- batch jobs, scheduled tasks, startup code, and out-of-band background work have no trace_id; (b) dies the moment a span gets sampled out -- the dropped trace is gone but the log line isn't; (c) cardinality bomb if you treat it as a label.

### Join key #2: OTel resource attributes (per-service / per-workload)

Every signal that comes out of an OTel-instrumented app carries a fixed set of *resource attributes* -- the durable identity of the workload. The OTel semantic conventions standardize the canonical set:

| Attribute | Example | Notes |
|-----------|---------|-------|
| `service.name` | `checkout-api` | THE primary service identity. Must be identical across signals. |
| `service.namespace` | `payments` | Logical grouping above service; pairs with name to disambiguate "two services named `api`." |
| `service.instance.id` | `checkout-api-7df8c-xyz12` | Per-replica identity. High-cardinality -- belongs in structured metadata, NOT a Prometheus label. |
| `service.version` | `2026.04.30-a3f2c1` | Build identity. |
| `deployment.environment.name` | `prod` | Environment. **Renamed from `deployment.environment` in semconv v1.27 (Sept 2024)** -- mixed-SDK fleets see both keys; standardize during upgrade. |
| `k8s.namespace.name` | `default` | Stabilized in recent semconv releases -- the `k8s.*` namespace is now a permanent contract. |
| `k8s.pod.name` | `checkout-api-7df8c-xyz12` | Bounded-but-churning; structured metadata, not a label. |
| `k8s.cluster.name` | `prod-us-east-1` | Top-of-tree disambiguator across fleets. |
| `host.name` | `ip-10-0-3-47.ec2.internal` | Node identity. |

**Strengths:** every signal has them, including signals that never touched a request (startup logs, batch metrics, cron-job spans). They survive sampling, restarts, and out-of-band emission.
**Weaknesses:** they only narrow the join to "this service in this environment," not to one request -- you usually combine them with a time window.

The K8s semconv stabilization matters because the `k8s.*` namespace is now a permanent contract. SDKs and Collectors emit the same K8s attribute keys with the same semantics, which means a Helm value of `k8s.namespace.name=payments` set on metrics, logs, and traces is finally guaranteed to mean the same thing to all three backends. Pre-stabilization, vendor differences caused subtle key-name drift.

The `deployment.environment` -> `deployment.environment.name` rename in v1.27 (Sept 2024) is the gotcha to watch out for in any heterogeneous fleet today. A pre-v1.27 Java SDK still emits `deployment.environment=prod`; a 2026 Go SDK emits `deployment.environment.name=prod`. Your dashboards must account for both keys until every instrumentation is upgraded -- typically by aliasing in the Collector's resource processor:

```yaml
processors:
  resource:
    attributes:
      - key: deployment.environment.name
        from_attribute: deployment.environment       # backfill new key from old
        action: insert
      - key: deployment.environment
        action: delete                                # then drop the old one
```

### Join key #3: Time window (the last-resort join)

When you have neither a `trace_id` nor matching resource attributes, you join on overlapping time windows -- "show me the logs from this namespace during the 5 minutes the metrics spiked." This is squishy (clock skew, log flush delays, log batching), but it's the join key of last resort and the reason `tracesToLogsV2` defaults to a `spanStartTimeShift: -2s` / `spanEndTimeShift: +2s` cushion.

The mental model going forward: **prefer trace_id when it exists; always have resource attributes; fall back to time windows only when both fail.**

---

## Part 2: Pivot 1 -- Metrics -> Traces (Exemplars)

The first pivot: you're staring at a histogram bucket in a dashboard, you see one bucket light up red, you want a *representative trace* from that bucket -- not "any slow trace in the same time range," but specifically *one of the requests that contributed to this bucket*.

This is what an **exemplar** is. An exemplar is a `(trace_id, value, timestamp)` triple attached to a single histogram bucket sample at scrape time. Prometheus stores exemplars alongside the metric data and Grafana renders them as little dots on top of histogram time series.

### What gets emitted on the wire

Exemplars piggyback on the **OpenMetrics exposition format** -- not classic Prometheus exposition, which has no syntax for them. The wire format is a `# {key=value,...} value timestamp` suffix on each bucket sample:

```
# HELP http_request_duration_seconds Request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 12345
http_request_duration_seconds_bucket{le="0.5"} 12678 # {trace_id="4bf92f3577b34da6a3ce929d0e0e4736"} 0.234 1714492800.000
http_request_duration_seconds_bucket{le="1.0"} 12690
http_request_duration_seconds_bucket{le="+Inf"} 12695
http_request_duration_seconds_sum 8742.3
http_request_duration_seconds_count 12695
```

The crucial piece: the exemplar attaches to the *bucket sample*, not to the histogram. When you click a heatmap cell or a histogram-quantile time series in Grafana, the click resolves to "the bucket sample at this timestamp," which carries the trace_id, which becomes the link target.

### Prometheus side: enable the feature, size the buffer

Exemplar storage is **off by default** in Prometheus. You enable it with a feature flag:

```yaml
# prometheus container args
- --enable-feature=exemplar-storage
- --storage.exemplars.exemplars-limit=200000   # ring buffer size; default 100k
```

The buffer is **in-memory ring**, fixed size. Once full, the oldest exemplars get evicted. Two consequences:

- **The buffer is in-memory, but exemplar records are appended to the WAL.** A normal Prometheus restart replays them up to WAL retention (~2h default), so recent exemplars survive. What you actually lose is anything that exceeded WAL retention or aged out of the buffer because the buffer is sized smaller than the WAL window covers. If you remote-write to Mimir/Thanos, exemplars travel along (~Prometheus 2.39+ / Mimir 2.6+) and the receiver becomes the durable home.
- **Sizing matters at high traffic.** A 100k buffer at 10k req/s is ~10 seconds of coverage. Bump it to 1-2M for production fleets emitting heavy exemplars; the memory cost is small (~80 bytes/exemplar).

### SDK side: emit exemplars

OTel SDKs in Java/Python/Go/Node emit exemplars automatically when a histogram observation occurs inside an active span. The behavior is governed by the `OTEL_METRICS_EXEMPLAR_FILTER` environment variable:

| Value | Meaning |
|-------|---------|
| `trace_based` (default) | Only emit an exemplar if the active span is *sampled* |
| `always_on` | Emit an exemplar for every histogram observation regardless of sampling |
| `always_off` | Disable exemplars entirely |

The `trace_based` default is usually right -- it skips emission when the trace was head-sampled-out, so you don't get exemplars pointing at non-existent traces. The `always_on` mode is useful when you intentionally drop most spans but still want every exemplar to point somewhere (you'd combine it with an unsampled-trace storage tier, which is rare).

### Grafana side: enable the click-through

Two things to configure on the Prometheus data source:

```yaml
# /etc/grafana/provisioning/datasources/prom.yaml
datasources:
  - name: prom-prod-use1
    uid: prom-prod-use1
    type: prometheus
    url: http://prometheus.monitoring.svc:9090
    jsonData:
      exemplarTraceIdDestinations:
        - name: trace_id              # MUST match the exemplar label key on the wire
          datasourceUid: tempo-prod   # <- where to jump on click
          urlDisplayLabel: "View trace"
```

That single block does two things: (a) tells Grafana that an exemplar with label `trace_id=...` should be linked to the data source with UID `tempo-prod`, and (b) controls the button text in the popover. Without this block, exemplars still render as dots on the panel but the click does nothing -- the most common "I enabled exemplars but they don't work" gotcha.

**The `name` field is not arbitrary** -- it must byte-match the exemplar label key Prometheus stored. OTel SDKs emit `trace_id` (snake_case); some custom emitters use `traceID` or `traceid`. A mismatch produces silent broken click-through (dot renders, click does nothing). Verify with `curl prom:9090/api/v1/query_exemplars?query=...` and read the actual label key.

The PromQL pattern that reveals exemplars:

```promql
# Histogram quantile -- exemplars surface on every bucket sample
histogram_quantile(0.99,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

**Critical rule: exemplars only render on queries that select bucket samples.** An exemplar is attached to a histogram bucket sample, so the query must reach those samples -- directly (`rate(..._bucket[5m])`) or via `histogram_quantile`. A query like `rate(http_requests_total[5m])` over a plain counter has no buckets, so no exemplars render no matter how many you've stored. Recording rules that pre-aggregate the histogram quantile (e.g. `histogram:job:p99`) also strip exemplars, because the recorded series is a regular sample, not a bucket sample. If your dashboards use recording rules and exemplars don't show up, this is why -- query the raw histogram bucket series directly for exemplar surface.

In Grafana's panel editor, the data source has an **"Exemplars" toggle** under the query options. Enable it. The panel re-renders with diamond/star markers on top of the time series, each with a trace_id tooltip.

### Production gotchas (metrics->traces edition)

- **Cumulative-to-delta conversion has ambiguous exemplar semantics.** Exemplars on delta points don't have a clean defined meaning, and the OTel `cumulativetodelta` processor's behavior here has varied across Collector versions. If you're temporality-converting cumulative histograms to delta for a delta-only backend, **verify your specific Collector version preserves exemplars through this processor before relying on the click-through.** Cleanest answer: keep cumulative end-to-end and use a backend that natively accepts cumulative exemplars (Mimir, Prometheus).
- **`honor_labels: true` on the Prometheus scrape job** -- the OpenMetrics exposition includes the trace_id as an exemplar label; without `honor_labels` the scrape may rewrite labels in surprising ways. Less common in 2026 since most paths are remote-write rather than scrape, but still bites in classic Prometheus deployments.
- **`Accept` header for OpenMetrics** -- some older client libraries default to classic Prometheus exposition and never emit the exemplar suffix. Confirm the scrape negotiates `application/openmetrics-text` (Prometheus 2.45+ does this automatically; older versions require explicit config).
- **Native histograms strip exemplars on conversion.** If you're using Prometheus native histograms (Apr 21) and convert from classic histograms via the `histogram_quantile` rewrite, double-check exemplars survive the conversion -- they do in 2.50+, but pre-2.50 versions have edge cases.
- **Time-range mismatch breaks exemplar overlay.** A panel showing 30 days at 1m step has exemplars only on data points where any exemplar was scraped; gaps render as no-dots even though the underlying buckets had traffic. Zoom in to verify, or use the "show all exemplars in time range" data-source option.

---

## Part 3: Pivot 2 -- Traces -> Logs (`tracesToLogsV2`)

The second pivot: you're inside Tempo's trace view, looking at a 3-second span on `payment-service`, you want the *log lines that span wrote*. Two requirements have to hold for this pivot to work:

1. Every log line your app writes must carry the active `trace_id` (and ideally `span_id`).
2. Loki must store that trace_id somewhere queryable -- in the log line body, in structured metadata (Loki 3.0+, the right answer), or as a label (the WRONG answer, see below).

### Requirement 1: trace_id injection into logs

Three injection paths, in order of how clean they are:

**(a) OTel logs SDK with auto-correlation.** The OTel logs API automatically reads the active span context and emits `trace_id`+`span_id` as top-level fields on every `LogRecord`. This is the cleanest path -- zero application code changes, propagation handled by the SDK. **Caveat:** the OTel logs SDK maturity matrix changes monthly -- check `opentelemetry.io/docs/languages/<lang>/instrumentation/` for current status of your language. For any language not yet stable, fall back to path (b).

**(b) Logging-framework bridges.** Every major logging framework has an OTel bridge that reads the active span context out of thread-local / context-local storage and injects trace_id into the structured log output:

- **Java -- Logback MDC**: the `OpenTelemetryAppender` injects `trace_id`, `span_id`, `trace_flags` into the MDC, which your Logback layout pattern can include.
- **Python -- structlog**: a custom processor calling `trace.get_current_span().get_span_context()` and merging the trace_id into the log dict.
- **Node -- winston / pino**: the OTel logs API SDK's bridge automatically annotates the log object.
- **Go -- slog**: `otelslog.NewHandler` wraps any `slog.Handler` and adds trace fields.

```python
# structlog example -- the trace-id processor
import structlog
from opentelemetry import trace

def add_trace_context(_, __, event_dict):
    span = trace.get_current_span()
    ctx = span.get_span_context()
    if ctx.is_valid:
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        add_trace_context,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)
```

**(c) Manual injection.** When neither (a) nor (b) works, drop down to `trace.get_current_span().get_span_context()` and stuff the trace_id into your log message yourself. Avoid if possible -- it's easy to forget the call on one log path and create silent gaps.

### Requirement 2: where to store trace_id in Loki

This is where Loki 3.0+ structured metadata earned its keep (Apr 29). Three storage options, ordered correctly:

| Storage | Queryable? | Cardinality safe? | Verdict |
|---------|-----------|-------------------|---------|
| **Structured metadata** (Loki 3.0+) | yes, indexed | yes, doesn't define stream | The right answer in 2026 |
| Log line body (e.g. JSON `message.trace_id`) | yes via `| json` parser | yes, never a label | Fine; line filter `|= "<trace_id>"` is fast for needle queries |
| **Loki label** | yes | NO, cardinality bomb | Wrong. Will explode stream count |

The 2026 rule: **if you're on Loki 3.0+, put trace_id in structured metadata.** If you're on 2.x, put it in the log line and use `|= "<trace_id>"` line filters. Never put it in a label.

### Trace ID format compatibility -- W3C vs Jaeger

W3C Trace Context (Apr 28) uses 32-hex-character (128-bit) trace IDs. Legacy Jaeger and some older OpenTracing implementations use 16-hex-character (64-bit) trace IDs. If you have a mixed fleet:

- **Pad the 16-hex on output** -- prepend `0000000000000000` when ingesting from a 64-bit-source so the stored trace_id is always 32-hex.
- **Or do a contains-match in `tracesToLogsV2`** -- search for the trailing 16 hex chars regardless of leading zeros.

The former is cleaner; the latter is a stopgap. Either way, *pick one* across your fleet -- mixing the two formats in the same Loki tenant guarantees lookup misses.

### The `tracesToLogsV2` config in Tempo data source

This is the click-through configuration on the Tempo data source side -- the button in Tempo's UI that builds and runs a LogQL query against Loki for the span you clicked.

```yaml
# /etc/grafana/provisioning/datasources/tempo.yaml
apiVersion: 1
datasources:
  - name: tempo-prod
    uid: tempo-prod
    type: tempo
    url: http://tempo.observability.svc:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki-prod
        # which span attributes/resource attrs become Loki label filters
        tags:
          - key: service.name
            value: service          # span attr key -> Loki label key
          - key: k8s.namespace.name
            value: namespace
          - key: k8s.pod.name
            value: pod
        filterByTraceID: true        # constrain to whole-trace logs
        filterBySpanID: false        # span-level too narrow for most cases
        spanStartTimeShift: "-2s"    # clock-drift cushion
        spanEndTimeShift: "+2s"
        # Override the auto-built query with custom LogQL
        customQuery: true
        query: '{${__tags}} |= "${__trace.traceId}"'
```

Field-by-field:

- **`datasourceUid`** -- which Loki to query.
- **`tags`** -- the mapping table from span attributes (left side) to Loki label keys (right side). The default list is `service.name`, `namespace`, `pod`. If your span attributes use OTel canonical names (`k8s.namespace.name`) but your Loki labels use short names (`namespace`), this mapping is essential -- without it the query says `{service.name="checkout"}` and Loki returns nothing because no Loki stream has that label key.
- **`filterByTraceID`** -- when true, the resulting query is anchored to logs containing the trace_id. Almost always `true`.
- **`filterBySpanID`** -- when true, narrows further to logs from one specific span. Usually `false` -- you want the whole request's logs, not just the leaf span's.
- **`spanStartTimeShift` / `spanEndTimeShift`** -- pad the time window by ±2s to handle clock skew between the log writer and the span boundary (logs commonly flush after the span closes).
- **`customQuery` + `query`** -- when set, override Grafana's auto-generated query. Useful when your trace_id lives in structured metadata vs the log body and you need to hint the query: `{${__tags}} | trace_id="${__trace.traceId}"` for structured metadata, vs `{${__tags}} |= "${__trace.traceId}"` for line-search.

### The orphan-trace mechanic (the single most-misunderstood correlation failure)

The trace_id was minted by the SDK at request entry, *before* the head-sampling decision. Logs got written with that trace_id all through request handling. Then the SDK's head-sampling decision said "drop this trace" -- so no spans were exported to Tempo. Result: the log lines exist with a real-looking trace_id, but Tempo returns "trace not found" when the click-through fires. **The logs are not broken -- they're just pointing at a trace that never made it to storage.** This is why head-sampling is fundamentally hostile to log-trace correlation: at 1% head sampling, ~99% of trace_ids in your logs are orphans. Tail sampling is the fix because the sampling decision happens *after* the spans exist, so any trace you care about (errors, slow, baseline) is preserved. There is no client-side code change that fixes head-sampled orphans.

### Production gotchas (traces->logs edition)

- **Head-sampled-out spans produce orphan logs.** See the section above. Switch to tail sampling, or accept that most trace_ids in logs are unfollowable.
- **Wrong tags = wrong logs.** A `tracesToLogsV2.tags` entry of `service.name -> service` builds `{service="checkout"} |= "<trace_id>"`. If your Loki labels are actually keyed `app="checkout"`, the query returns zero results and the user thinks the pivot is broken.
- **Tags too broad = too many logs.** If `tags` includes only `service.name` for a service that runs in 30 namespaces, the resulting query scans all 30 namespaces' streams, decompresses them, and grep-searches for the trace_id. Slow and expensive. Always include enough `tags` to narrow to the namespace or pod.
- **trace_id in pod_name is a category error.** Some teams try to "expose" the trace_id by labeling the pod with it -- this is a confused mental model and a stream-cardinality bomb. trace_id goes in the log line or structured metadata; never in pod metadata.

---

## Part 4: Pivot 3 -- Logs -> Metrics (LogQL Metric Queries + Loki Ruler)

The third pivot is different in shape: you're not jumping UI tabs, you're *deriving a metric from log streams*. Two reasons you'd want to:

1. **The app you're observing isn't instrumented for metrics**, but it logs structured events. You can't ship a code change. LogQL lets you compute "error rate per service" from the existing log stream.
2. **The metric you care about doesn't exist in your metrics backend** -- audit-event rate, business metric extraction (`payment.amount` aggregated from log lines), 4xx-by-route counts.

### LogQL metric queries -- on the fly

LogQL's metric mode (Apr 29) wraps a log-stream selector in a range aggregation and produces a Prometheus-shaped vector:

```logql
# Error log rate per service
sum by (service) (
  rate({app="api", level="error"}[5m])
)

# p99 of an extracted latency field
quantile_over_time(0.99,
  {app="api"} |= "request" | json | unwrap duration [5m]
) by (service)

# 5xx ratio derived from request logs
sum(rate({app="api"} | json | status >= 500 [5m]))
/
sum(rate({app="api"} | json | status != 0 [5m]))
```

Two big drawbacks vs instrumented metrics:

- **Query cost scales with log volume.** A LogQL query over a 24-hour window decompresses every chunk in every matching stream over those 24 hours, *then* computes the aggregation. A Prometheus query over the same period reads pre-aggregated samples. The cost ratio is often 100-1000x against Loki -- run these on small time windows or convert to recording rules.
- **No histogram buckets.** You can `quantile_over_time` on an extracted latency field, but you don't get a Prometheus histogram. Quantile-over-extracted-latency is a *single quantile per query*, not a percentile distribution you can aggregate further across labels.

### Loki ruler -- the bridge to a real time series

The fix for both drawbacks: the **Loki ruler** evaluates LogQL queries on a schedule (just like Prometheus recording rules) and remote-writes the result into Prometheus or Mimir. The expensive log-stream scan happens once per minute; every dashboard query against the resulting time series is cheap.

```yaml
# loki-ruler-rules.yaml
groups:
  - name: api-derived-metrics
    interval: 1m
    rules:
      # Recording rule: error rate per service
      - record: api:error_rate:5m
        expr: |
          sum by (service) (
            rate({app="api", level="error"}[5m])
          )

      # Recording rule: extracted business metric
      - record: payments:successful_amount:5m
        expr: |
          sum by (currency) (
            sum_over_time(
              {app="payments"} |= "settlement_confirmed"
                | json
                | unwrap amount [5m]
            )
          )

      # Alerting rule: log-derived alert
      - alert: AuthServiceSilent
        expr: |
          absent_over_time({app="auth"}[5m]) == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Auth service has logged nothing in 5 minutes"
```

The ruler config points at a Prometheus-compatible remote_write endpoint (Mimir, Cortex, AMP, plain Prometheus with `--enable-feature=remote-write-receiver`):

```yaml
# loki config (Loki 3.x ruler schema -- map of named clients)
ruler:
  rule_path: /loki/rules
  storage:
    type: local
    local:
      directory: /etc/loki/rules
  ring:
    kvstore: { store: inmemory }
  enable_api: true
  evaluation_interval: 1m
  alertmanager_url: http://alertmanager.observability.svc:9093
  remote_write:
    enabled: true
    clients:
      mimir-prod:
        url: http://mimir.observability.svc:9009/api/v1/push
        headers:
          X-Scope-OrgID: tenant-default
```

> **Schema note:** Loki 2.x used a single `remote_write.client` block; Loki 3.x switched to `remote_write.clients.<name>` (a map keyed by client name) so a ruler can fan out to multiple Prometheus-compatible backends. If you copy a 2.x snippet onto 3.x, the parser silently ignores the unknown `client` key and the ruler emits no remote writes -- a quiet failure.

### When to derive metrics from logs vs instrument the app

| Use case | Right answer |
|----------|--------------|
| You own the app and can ship a code change | Instrument with OTel metrics SDK |
| You don't own the app but it logs structured events | LogQL recording rule |
| You need histogram buckets / percentile distributions | Instrument; logs->metrics gives you single-quantile only |
| Audit / compliance event rate (must come from logs anyway) | LogQL recording rule |
| Silent-failure alert ("service stopped logging") | LogQL `absent_over_time` -- the killer use case |
| Business metric from existing log fields, low cardinality | LogQL recording rule |

### Production gotchas (logs->metrics edition)

- **`{}` matcher in a ruler rule = DoS.** A LogQL rule with no stream selector scans every stream for the tenant every minute. Always specify at least one bounded label. Loki refuses unbounded matchers in queries (`-querier.unaligned-error` config), but rules sometimes slip through if the validator is permissive.
- **Recording rules over huge time windows starve the ruler.** A `[1h]` rate window evaluated every minute has to read 1h of logs every minute -- that's a 60x scan-amplification. Keep rule windows at `[5m]` and let your alert thresholds handle the smoothing.
- **`unwrap` on a string field silently returns nothing.** If you `unwrap duration` against a field that's actually a string `"250ms"`, you get an empty series with no error. Use `unwrap duration(field)` for Go-style duration strings or `unwrap bytes(field)` for size strings.
- **`__error__=""` filter is mandatory after parsers.** A `| json` parser stage marks lines that failed to parse with an `__error__` label; without `| __error__=""` after the parser, broken-JSON lines leak into the aggregation as zero-valued samples.

---

## Part 5: Pivot 4 -- Logs -> Traces (Grafana `derivedFields`)

The fourth pivot: you're inside a Loki query result (Explore view), staring at a WARN log line that contains a trace_id, you want to *click that trace_id* and jump straight to Tempo's trace view.

This is the **`derivedFields`** mechanism on the Loki data source. It's a regex-based link extractor: Grafana scans every rendered log line for a regex match, and turns each match into a clickable link that builds a URL into a configured target data source.

```yaml
# /etc/grafana/provisioning/datasources/loki.yaml
apiVersion: 1
datasources:
  - name: loki-prod
    uid: loki-prod
    type: loki
    url: http://loki-gateway.observability.svc:3100
    jsonData:
      derivedFields:
        # Trace ID -> Tempo
        - name: trace_id
          # ANCHORED regex: \b boundaries prevent runaway matches
          matcherRegex: '\btrace_id=([0-9a-f]{32})\b'
          url: '$${__value.raw}'
          datasourceUid: tempo-prod
          urlDisplayLabel: 'View trace'

        # Trace ID inside JSON body -- alternate match
        - name: trace_id_json
          matcherRegex: '"trace_id"\s*:\s*"([0-9a-f]{32})"'
          url: '$${__value.raw}'
          datasourceUid: tempo-prod

        # Request ID -> internal request-tracking system
        - name: request_id
          matcherRegex: '\brequest_id=([0-9a-f-]{36})\b'
          url: 'https://internal-rt.example.com/req/$${__value.raw}'
          # external link, no datasourceUid

        # User ID -> user database (for support escalations)
        - name: user_id
          matcherRegex: '\buser_id=([0-9]+)\b'
          url: 'https://admin.example.com/users/$${__value.raw}'
```

Field-by-field:

- **`name`** -- becomes the link label in the rendered log line.
- **`matcherRegex`** -- the regex applied to every rendered line. **Always anchor with `\b` word boundaries or `^...$` line anchors.** A greedy unanchored regex will grind Grafana's UI to a halt over millions of log lines -- the most common derivedFields production gotcha.
- **`url`** -- the URL template. `${__value.raw}` is the captured group from the regex (note the doubled `$$` to escape Grafana's own templating).
- **`datasourceUid`** (optional) -- if set, the link is *internal* (opens Grafana split-pane Explore on that data source); if absent, the link is *external* (opens in a new tab).
- **`urlDisplayLabel`** (optional) -- the visible button text.

The reverse round-trip: a log line with `trace_id=4bf92f3577b34da6a3ce929d0e0e4736 ... DB query timeout 2.9s` becomes a clickable trace_id that opens Tempo's trace view, which itself has a "Logs for this trace" button (the `tracesToLogsV2` from Part 3) -- close the loop and you can navigate logs <-> traces freely.

### Production gotchas (logs->traces edition)

- **Greedy unanchored regex hangs the panel.** A `matcherRegex: '[0-9a-f]{32}'` (no anchors) over 50,000 lines does ~50,000 regex evaluations *per render*. Browser tab freezes. Always anchor.
- **Regex matches an unrelated 32-hex string.** A 32-hex commit SHA in a log line gets caught by an unanchored trace_id regex and links to a non-existent trace. Use `\btrace_id=` or `"trace_id":` prefixes in the regex to disambiguate.
- **`${__value.raw}` not escaped.** Grafana's own dashboard variable syntax uses single `$`; data-source provisioning YAML requires `$$` to pass a literal `$` through to the data source. Single-`$` variants work in some places and not others -- always use `$$` in YAML provisioning.
- **External link without target=_blank.** Grafana opens external links in the same tab by default. The user clicks, loses their Loki tab, swears at you. Configure the link to open in a new tab via the Grafana org settings.

---

## Part 6: Grafana Correlations -- The Modern API

`derivedFields` and `tracesToLogsV2` are both **per-data-source** link configs. They live on the Loki data source and the Tempo data source respectively. This has three drawbacks:

1. **Coupling.** A Loki data source's `derivedFields` knows about Tempo's UID. Move Tempo to a different cluster, every Loki data source needs an update.
2. **Asymmetry.** Some pivots have a dedicated config (`tracesToLogsV2`); others are generic regex (`derivedFields`); others (Loki -> Postgres for "look up this user") have no config at all.
3. **Discovery.** A new engineer has to know to look at the data source config to understand what links exist.

Grafana introduced **Correlations** as the replacement (beta in 10.x, GA in **Grafana 11.3, October 2024**) -- a centralized, source-agnostic link API that lives at the org level, not on individual data sources. Both old configs still work and ship, but new pivots should be authored as Correlations.

### The Correlations model

A Correlation is a tuple:

```
Correlation:
  source:
    datasourceUid: loki-prod
    field: trace_id              # field in the source result (parsed or matched)
  target:
    datasourceUid: tempo-prod
    query: '${trace_id}'         # query to execute, with var interpolation
    queryType: traceql
  transformations:
    - type: regex
      expression: '\btrace_id=([0-9a-f]{32})\b'
      mapValue: trace_id         # captured group becomes ${trace_id}
  label: 'View trace'            # button text
  description: 'Open Tempo trace from log line'
```

Provisioning Correlations via YAML at boot:

```yaml
# /etc/grafana/provisioning/correlations/loki-to-tempo.yaml
apiVersion: 1

correlations:
  - uid: loki-trace-id-to-tempo
    sourceDatasourceUid: loki-prod
    targetDatasourceUid: tempo-prod
    label: 'View trace in Tempo'
    description: 'Click to open the matching trace'
    config:
      type: query
      target:
        query: '${trace_id}'
        queryType: traceql
      field: 'Line'                                  # raw log line
      transformations:
        - type: regex
          expression: '\btrace_id=([0-9a-f]{32})\b'
          mapValue: 'trace_id'

  - uid: tempo-span-to-loki
    sourceDatasourceUid: tempo-prod
    targetDatasourceUid: loki-prod
    label: 'Logs for this span'
    config:
      type: query
      target:
        query: '{service="${service_name}"} |= "${trace_id}"'
        queryType: range
      field: 'traceID'
      transformations:
        - type: regex
          expression: '([0-9a-f]{32})'
          mapValue: 'trace_id'
```

### When to use Correlations vs derivedFields vs tracesToLogsV2

| Scenario | Reach for |
|----------|-----------|
| Loki -> Tempo (trace_id pivot) | Correlations *or* derivedFields. Greenfield: Correlations. |
| Tempo -> Loki (span -> logs) | `tracesToLogsV2` *still* the most fully-featured for this specific pivot (handles `spanStartTimeShift` etc); Correlations works but lacks some niceties as of G11 |
| Loki -> Postgres (`user_id` -> user record) | Correlations -- this never had a per-data-source equivalent |
| Prom -> Tempo (exemplars) | `exemplarTraceIdDestinations` on the Prometheus data source -- exemplars don't fit the Correlations model (they're emitted by the data source itself) |
| Anything cross-organization-cross-tenant | Correlations -- it's the only API that handles the cross-tenant case |

The strategic direction: Correlations is the long-term home for everything except exemplars (which are data-source-native). Don't migrate working `derivedFields` for the sake of it, but author new ones in Correlations.

### Cross-tenant correlation (the platform-team gotcha)

In any platform-team setup, Loki / Tempo / Mimir each shard by `X-Scope-OrgID` -- team-a's logs live in tenant `team-a`, team-a's traces in a separate Tempo tenant. A trace_id pivot from tenant-a logs to tenant-a traces is fine because the data sources are scoped at the data-source level (each Grafana data source has a fixed `X-Scope-OrgID` header in `httpHeaderValue1`). But the moment you need to follow a request that crossed *two teams* -- say team-a calls team-b's API -- the trace_id exists in two tenants and a single click-through can only land in one of them.

Solutions, in order of practicality:

1. **Multi-tenant data sources.** Loki and Mimir support multi-tenant queries via `|` separator in the header (`X-Scope-OrgID: team-a|team-b`). Configure a "platform-wide" data source with the union header for cross-team traces. Authorize narrowly.
2. **Per-team correlation pairs.** Configure one `tracesToLogsV2` per (tempo-tenant, loki-tenant) pair. The Tempo data source's UI shows multiple "Logs in tenant X" buttons.
3. **Correlations across data source UIDs.** The Correlations API handles target-data-source switching natively -- one Correlation per cross-tenant pivot, each landing in the correct destination data source. This is the cleanest answer in 2026 and the case where Correlations *clearly* beats `tracesToLogsV2`.

Whatever you pick, document it -- platform engineers debugging at 3 AM cannot guess which tenant the next click goes to.

---

## Part 7: Resource Attributes as the Universal Join Key (Deeper)

A worked example of why `service.name` consistency matters more than any single feature:

> **Scenario.** A Prometheus alert fires: `checkout p99 latency > 500ms for 5m`. The alert label set is `{service="checkout", env="prod", cluster="prod-use1"}`. The on-call engineer clicks the alert, lands on the Grafana panel, hits "Explore Logs" -- they need the logs for the same `service=checkout` over the alert window.
>
> **The break.** The metrics pipeline emitted `service=checkout` (because the Java app's `Micrometer` config said `management.metrics.tags.service=checkout`). The logging pipeline emitted `app=checkout-svc` (because the K8s pod label was `app=checkout-svc` and the Alloy `discovery.relabel` block promoted `__meta_kubernetes_pod_label_app` to `app`). The traces pipeline emitted `service.name=Checkout API` (because the OTel SDK's `OTEL_SERVICE_NAME` env var was set in title case for log readability).
>
> Three different keys (`service`, `app`, `service.name`), three different values (`checkout`, `checkout-svc`, `Checkout API`), zero possible joins. The on-call engineer copy-pastes a timestamp into Loki, manually filters down by guessing label names, eventually finds the logs. Five minutes lost; in a real incident, fifteen.

The fix is **mandatory naming discipline enforced at the collector**. The OTel Collector's resource processor is the chokepoint:

```yaml
# In Alloy or OTel Collector
processors:
  resource:
    attributes:
      # Backfill OTel canonical names from K8s metadata if missing
      - key: service.name
        from_attribute: app                          # promote K8s "app" label
        action: insert                                # only if missing
      - key: service.namespace
        from_attribute: k8s.namespace.name
        action: insert
      # Enforce the env name
      - key: deployment.environment.name
        from_attribute: deployment.environment       # backfill from old key
        action: insert
      - key: deployment.environment
        action: delete

  # Case normalization needs OTTL via the transform processor --
  # the resource processor's `extract` action *creates* new attributes
  # from named regex groups; it does NOT mutate the existing key.
  transform:
    metric_statements:
      - context: resource
        statements:
          - set(attributes["service.name"], ConvertCase(attributes["service.name"], "lower"))
    log_statements:
      - context: resource
        statements:
          - set(attributes["service.name"], ConvertCase(attributes["service.name"], "lower"))
    trace_statements:
      - context: resource
        statements:
          - set(attributes["service.name"], ConvertCase(attributes["service.name"], "lower"))
```

In Loki specifically, the OTLP ingestion path lets you control which resource attributes become labels vs structured metadata vs dropped:

```yaml
# loki config
distributor:
  otlp_config:
    default_resource_attributes_as_index_labels:
      - service.name                                  # PROMOTE these to labels
      - service.namespace
      - deployment.environment.name
      - k8s.namespace.name
      - cluster
    # everything else (k8s.pod.name, service.instance.id, etc) stays as structured metadata
```

The contract: **if it's bounded enough to be a label and you want service-level joins on it, promote it. Everything else stays as structured metadata.**

The Tempo side has no such config -- spans carry the full resource-attributes set always, and TraceQL can filter on any of them. The asymmetry is fine because Tempo's storage model doesn't have a cardinality bottleneck on resource attributes.

### The naming-discipline checklist

Before you ship any new instrumentation:

- [ ] `service.name` is byte-identical across metrics, traces, and logs.
- [ ] `service.namespace` is set if you have multiple services with the same `service.name`.
- [ ] `deployment.environment.name` (NOT `deployment.environment`) is set on every signal.
- [ ] `k8s.*` attributes are emitted by the Collector's `k8sattributes` processor.
- [ ] The Loki `default_resource_attributes_as_index_labels` list is explicit (not the unsafe default).
- [ ] At least one Grafana dashboard has been confirmed to filter all three signals by the same `service.name` value successfully.

### The framing that matters

**Observability is a schema problem before it's a tooling problem.** The LGTM stack (and any vendor stack) is just plumbing -- the contract that lets the plumbing connect is `service.name` and the rest of the OTel resource conventions. Every individual tool can be configured perfectly and the correlation surface still breaks the moment three pipelines spell the workload's identity differently. The hardest cross-team correlation bugs are silent-zero-results that look like missing data; they're actually failed joins on misspelled identity. Get the schema contract right at the pipeline layer (Collector / Alloy), enforce it at the edge (reject signals with missing/non-conformant `service.name`), and every correlation feature works for free.

---

## Part 8: Grafana Alloy as the Unified Collector

The Apr 29 Loki doc and the Apr 27 OTel doc both pointed at **Alloy** as the convergence point. From a correlation perspective, Alloy is the answer to "how do I make sure all three signals carry the same resource attributes?" -- because *all three signals come out of the same agent process*, the resource attributes get attached once and stay consistent by construction.

The 2026 status:

- **Promtail: dead.** EOL March 2, 2026 (one month and two days ago). Receives no updates; `alloy convert --source-format=promtail` is the migration path.
- **Grafana Agent: dead.** EOL November 1, 2025. Same migration story.
- **OTel Collector contrib: alive.** Still the right answer when you need a custom processor that Alloy hasn't pulled in.

### The single-Alloy-three-pipelines pattern

```alloy
// === RESOURCE ATTRIBUTES (applied to all signals) ===
otelcol.processor.resource "common" {
  attribute {
    key    = "deployment.environment.name"
    value  = "prod"
    action = "upsert"
  }
  attribute {
    key    = "k8s.cluster.name"
    value  = "prod-use1"
    action = "upsert"
  }

  output {
    metrics = [otelcol.exporter.prometheus.mimir.input]
    logs    = [otelcol.exporter.otlphttp.loki.input]
    traces  = [otelcol.exporter.otlp.tempo.input]
  }
}

// === K8s METADATA ENRICHMENT ===
otelcol.processor.k8sattributes "default" {
  extract {
    metadata = ["k8s.namespace.name", "k8s.pod.name", "k8s.deployment.name",
                "k8s.node.name", "k8s.container.name"]
  }
  output {
    metrics = [otelcol.processor.resource.common.input]
    logs    = [otelcol.processor.resource.common.input]
    traces  = [otelcol.processor.resource.common.input]
  }
}

// === RECEIVERS ===
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }
  output {
    metrics = [otelcol.processor.k8sattributes.default.input]
    logs    = [otelcol.processor.k8sattributes.default.input]
    traces  = [otelcol.processor.k8sattributes.default.input]
  }
}

// === EXPORTERS ===
otelcol.exporter.prometheus "mimir" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "https://mimir.observability.svc/api/v1/push"
    headers = { "X-Scope-OrgID" = "tenant-default" }
  }
  // exemplars survive remote_write since Mimir 2.6
  send_exemplars = true
}

otelcol.exporter.otlphttp "loki" {
  client {
    endpoint = "https://loki-gateway.observability.svc/otlp"
    headers  = { "X-Scope-OrgID" = "tenant-default" }
  }
}

otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo-distributor.observability.svc:4317"
    tls { insecure = true }
    headers = { "X-Scope-OrgID" = "tenant-default" }
  }
}
```

What this earns:

1. **One `resource` block sets `deployment.environment.name` and `k8s.cluster.name` on all three signals at once.** No way for the three signals to drift.
2. **`k8sattributes` enriches all three signals from the same K8s API watch.** No way for one signal to have `k8s.namespace.name` and another to be missing it.
3. **One OTLP receiver feeds all three pipelines.** Apps emit OTLP once and the Alloy fans out.
4. **`send_exemplars = true`** preserves exemplars on remote_write to Mimir -- correlation surface still works.

### When you still pick OTel Collector contrib over Alloy

- Custom processors not yet in Alloy (e.g., `loadbalancing` exporter for tail sampling on a separate Collector tier was a long-time gap; check the current Alloy component list before assuming).
- Heavily customized OTel Collector deployments where you've already invested in the pure-OTel YAML config -- migration cost exceeds the consolidation benefit.
- Edge cases where you need OTel Collector's full extension ecosystem (some auth extensions, some health probes).

For greenfield in 2026: Alloy.

---

## Part 9: End-to-End Click-Through Walkthrough

The whole point of correlation is the click-through. Here is the canonical "alert -> root cause" navigation, with the exact UI interaction at each step and the exact config that makes each step work.

```
1. ALERT FIRES
   ----------------
   Alertmanager sends to Slack:
     "[CRITICAL] checkout p99 latency > 500ms in prod-use1"
   Alert payload includes a "View in Grafana" URL pointing at the
   pre-built `checkout-slo` dashboard, time range pre-zoomed to the
   alert window.

   Config that makes this work:
     - Alertmanager `external_url` set to grafana.example.com
     - Alert annotation: { dashboard_uid: "checkout-slo", panel_id: 5 }
     - Slack receiver template builds URL from those annotations
```

```
2. CHECKOUT-SLO DASHBOARD
   ----------------
   Engineer lands on dashboard. The histogram-quantile panel shows
   p99 spiking from 200ms to 700ms. Histogram is rendered with the
   "Show exemplars" toggle ON. Diamond-shaped dots appear on the
   spike, each with a trace_id tooltip on hover.

   Config that makes this work:
     - Prometheus: --enable-feature=exemplar-storage
     - SDK: OTEL_METRICS_EXEMPLAR_FILTER=trace_based (default)
     - Mimir remote_write: send_exemplars=true (Apr 21)
     - Grafana data source: exemplarTraceIdDestinations -> tempo-prod
     - Panel option: "Exemplars" enabled on the query
```

```
3. CLICK AN EXEMPLAR
   ----------------
   Engineer clicks one of the dots in the slow bucket. Popover shows
   "trace_id: 4bf92f3577b34da6a3ce929d0e0e4736" + "View in Tempo"
   button. They click it.

   Config that makes this work:
     - exemplarTraceIdDestinations.datasourceUid = tempo-prod
     - urlDisplayLabel: "View in Tempo"
```

```
4. TEMPO TRACE VIEW
   ----------------
   Split-pane opens. Left side: previous panel still visible. Right
   side: trace waterfall with ~12 spans. Engineer sees a 3.0-second
   span on `payment-service` -> `checkout-db` (the rest of the trace
   was 80ms). They click that span.

   The span attributes panel shows:
     - service.name = payment-service
     - k8s.namespace.name = payments
     - k8s.pod.name = payment-svc-7df8c-xyz12
     - db.statement = "SELECT ..." (truncated)
```

```
5. CLICK "LOGS FOR THIS SPAN"
   ----------------
   In the span detail panel, a button reads "Logs for this span"
   (or "Logs for this trace" -- the wording depends on
   filterByTraceID vs filterBySpanID config). They click it.

   Config that makes this work (Tempo data source):
     tracesToLogsV2:
       datasourceUid: loki-prod
       tags:
         - { key: service.name,        value: service }
         - { key: k8s.namespace.name,  value: namespace }
       filterByTraceID: true
       spanStartTimeShift: "-2s"
       spanEndTimeShift: "+2s"
```

```
6. LOKI QUERY RESULT
   ----------------
   Loki Explore opens with auto-built query:
     {service="payment-service",namespace="payments"}
       |= "4bf92f3577b34da6a3ce929d0e0e4736"

   Three log lines come back:
     14:03:01.234 INFO  Starting payment processing
     14:03:01.456 INFO  Acquiring DB connection
     14:03:04.123 WARN  db_query_duration_ms=2900 db.statement="SELECT..."
                        slow_query=true

   The WARN line tells the whole story: the DB query took 2.9s.
   That's the root cause.

   Config that makes this work:
     - service.name=payment-service consistent in metrics+traces+logs
     - trace_id stored in structured metadata in Loki 3.0+ via
       Alloy's loki.process.parse_json -> stage.structured_metadata
     - Loki labels include `service`, `namespace` (matching the
       tracesToLogsV2 mapping)
```

```
7. CLICK "VIEW METRICS FOR THIS SERVICE"
   ----------------
   Optional: from the WARN log line, derivedFields detected the
   service name and offers a link back to the metrics dashboard
   for `payment-service`. Engineer clicks it to verify the DB-side
   metrics also show the spike.

   Config that makes this work (Loki data source):
     derivedFields:
       - name: service
         matcherRegex: '\bservice="?([a-z0-9-]+)"?'
         url: '/d/service-overview/${__value.raw}'
         urlDisplayLabel: 'Service dashboard'
```

Round-trip complete. Total clicks: 4. Total IDs copy-pasted by hand: 0. That's what correlation gets you when it's wired correctly.

---

## Part 10: Production Decision Frameworks

### Framework 1: Exemplars vs separate trace search

| Need | Reach for |
|------|-----------|
| "I want a representative trace from this metric spike" | Exemplars |
| "I want to find all slow traces in the last 24h matching some criteria" | Tempo TraceQL search |
| "I want to find a trace by trace_id (someone gave it to me)" | Tempo search by ID directly |
| "I want spanmetrics-style RED metrics derived from spans" | Tempo metrics-generator (Apr 28) -- separate story |

Exemplars are NOT a trace search engine. They're a pointer to *one* representative trace per histogram bucket sample. Don't try to use them for search.

### Framework 2: Correlations vs derivedFields vs tracesToLogsV2

```
                +--------------------------------+
                |  Is this a Tempo->Loki span->  |
                |  logs pivot specifically?      |
                +-----+-----------------------+--+
                      | yes                   | no
                      v                       v
              tracesToLogsV2          +-------+--------+
              (richer config:         | Greenfield     |
               span time shifts,      | pivot? Or new  |
               filterByTraceID,       | data source    |
               filterBySpanID)        | combo?         |
                                      +---+--------+---+
                                          | yes    | no
                                          v        v
                                   Correlations  Use existing
                                   (modern API)  derivedFields
                                                 (don't migrate
                                                  for migration's
                                                  sake)
```

### Framework 3: Where to store trace_id in Loki

```
Loki version >= 3.0 ?
   +-- yes -> Structured metadata. Always. Use loki.process.stage.structured_metadata
   +-- no  -> Loki version >= 2.0 ?
              +-- yes -> Log line body (JSON or `trace_id=` k=v). Filter via |= line filters
              +-- no  -> upgrade Loki

Never put trace_id in a label. Never.
```

### Framework 4: Head sampling vs tail sampling (correlation impact)

| Approach | Where | Cost | Correlation impact |
|----------|-------|------|--------------------|
| **Head sampling** (e.g., 1%) | SDK -- decision made on first span | Cheap (drops at source) | 99% of trace_ids in logs are orphans (trace doesn't exist) |
| **Tail sampling** (errors + slow + 1% baseline) | Collector -- decision after seeing whole trace | Expensive (Collector buffers all spans + memory) | All interesting trace_ids have traces; baseline 1% covers happy path |
| **`always_on` exemplar filter** | Per-metric exemplar emission | No cost beyond exemplar buffer | Exemplars exist even for sampled-out traces -> dead links |

The correlation-friendly default in 2026: **tail sampling with errors + slow + 1% baseline** + exemplar filter `trace_based`. This gives you traces for every interesting case, exemplars only for traces that exist, and correlation that works at the precise moment you need it.

#### Tail sampling is a deployment topology change, not a config flag

Adopting tail sampling means a **two-tier Collector deployment**, not a single-line YAML edit:

- **Tier 1 (front):** receives OTLP from apps. Runs a `loadbalancing` exporter that consistent-hashes by `trace_id`, ensuring every span of a given trace lands on the same downstream Collector replica.
- **Tier 2 (back):** the tail-sampling Collectors. Each instance buffers spans by trace_id, applies the policy after a 5-30s decision window, and forwards the kept traces to Tempo.

Without the `loadbalancing` exporter in tier 1, spans for the same trace get sprayed across tier-2 replicas, each replica only sees a fragment, and the policy decision is made on incomplete data -- you sample-out half the spans of a trace you meant to keep.

**Capacity planning** for tail sampling: tier-2 memory scales with `(spans/sec * decision_window_seconds * avg_span_size)`. At 50K spans/s with a 30s window and 1KB/span, that's ~1.5GB/replica steady-state -- bigger than people expect. Plan for the rebalance edge case too: a tier-2 pod restart shifts trace_id ownership across the consistent-hash ring, in-flight traces get split, decisions get made on partial data.

**Treat the sampler as a critical-path component:** monitor tier-2 buffer pressure, sample-decision latency, dropped-traces-per-policy, hash-ring rebalance events. If the sampler is unhealthy, traces silently disappear -- worse than head-sampling because you don't even know what you lost.

### Framework 5: Correlation join key by signal source

| If your signal is from... | Join on... |
|---------------------------|-----------|
| App with active OTel span (HTTP request) | trace_id (per-request) + service.name (per-service) |
| App background worker / cron | service.name + service.instance.id + time window |
| App startup logs (no span yet) | service.name + service.instance.id + time window |
| Infra logs (kubelet, kube-apiserver) | k8s.node.name / k8s.cluster.name + time window |
| Cloud-provider events (CloudTrail) | resource ARN + time window (no trace_id at all) |

Always have the resource-attribute join available. Trace_id is the icing.

---

## Part 11: Production Gotchas (the field-tested list)

| # | Problem | Cause | Fix |
|---|---------|-------|-----|
| 1 | Service name mismatch across signals | Three pipelines emit `service`, `app`, `service.name` with different values | Enforce `service.name` in OTel resource processor; use Alloy as single agent |
| 2 | Exemplars enabled but Tempo not linked | `exemplarTraceIdDestinations` not configured | Add the block to the Prometheus data source provisioning |
| 3 | Exemplars stripped by cumulative-to-delta | OTel `cumulativetodelta` processor drops exemplars by default | Keep cumulative end-to-end, or use a backend that preserves exemplars on delta |
| 4 | Head-sampled traces produce orphan logs | SDK dropped the trace; logs still wrote the trace_id | Switch to tail sampling, or `always_off` exemplar filter |
| 5 | Trace ID format mismatch (W3C vs Jaeger) | Mixed-fleet 32-hex vs 16-hex IDs | Pad on ingestion; pick one format fleet-wide |
| 6 | derivedFields greedy regex hangs panel | Unanchored `[0-9a-f]{32}` over millions of lines | Anchor with `\btrace_id=` prefix |
| 7 | derivedFields matches unrelated 32-hex | Commit SHA caught by trace_id regex | Use anchoring + key prefix; never rely on length alone |
| 8 | trace_id in Loki labels | Misconfigured `discovery.relabel` promoting `trace_id` | Move to structured metadata via `loki.process.stage.structured_metadata` |
| 9 | LogQL metric query without recording rule = expensive | Dashboard re-runs the LogQL aggregation on every load | Convert to Loki ruler recording rule; query the materialized series |
| 10 | tracesToLogsV2 with too-broad tags returns wrong logs | `tags: [service.name]` only, no namespace narrowing | Add `k8s.namespace.name` (or equivalent) to tags |
| 11 | Exemplar feature flag missing on Prometheus | No `--enable-feature=exemplar-storage` arg | Add the flag; restart |
| 12 | `deployment.environment` vs `deployment.environment.name` | OTel semconv v1.27 renamed the key | Use resource processor to backfill new key from old, then delete old |
| 13 | OTel logs SDK still in development for Python/Ruby/JS | Maturity gap as of Apr 2026 | Use logging-framework bridges (structlog, winston) until SDK stabilizes |
| 14 | Mixed Promtail + Alloy = duplicate logs | Both agents tail the same files; different label sets | Promtail dead Mar 2 2026 -- finish migration; verify single agent on each node |
| 15 | Time-range mismatch breaks exemplar overlay | 30-day view + 5m bucket sparsity | Zoom in; use "Show all exemplars in time range" data source option |
| 16 | OTLP-to-Loki promotes too many resource attrs to labels | Vanilla Loki helm install uses unsafe default list | Override `default_resource_attributes_as_index_labels` to the bounded set |
| 17 | Grafana panel `$__interval` too long for trace_id needle queries | LogQL line filter scans every chunk in the interval | Narrow the time range manually or precompute with recording rule |
| 18 | Alloy stale resource attributes after K8s pod relocation | k8sattributes processor cache lag | Tune `k8sattributes.delete_pod_cache_after`; verify pod-event watch is healthy |

---

## Part 12: Production Helm + Terraform Wiring

The realistic, runnable end-to-end stack: kube-prometheus-stack with exemplars, Tempo with `tracesToLogsV2`, Loki with `derivedFields`, Alloy emitting all three, and a Grafana Correlation provisioned via YAML.

### kube-prometheus-stack values (excerpt)

```yaml
# kube-prometheus-stack values.yaml
prometheus:
  prometheusSpec:
    enableFeatures:
      - exemplar-storage                   # <-- enable exemplars
    additionalArgs:
      - name: storage.exemplars.exemplars-limit
        value: "1000000"                   # <-- 1M ring buffer
    remoteWrite:
      - url: https://mimir.observability.svc:9009/api/v1/push
        sendExemplars: true                # <-- preserve on remote_write
        headers:
          X-Scope-OrgID: tenant-default

grafana:
  additionalDataSources:
    - name: tempo-prod
      uid: tempo-prod
      type: tempo
      url: http://tempo-query-frontend.observability.svc:3200
      jsonData:
        tracesToLogsV2:
          datasourceUid: loki-prod
          tags:
            - { key: service.name,       value: service }
            - { key: k8s.namespace.name, value: namespace }
            - { key: k8s.pod.name,       value: pod }
          filterByTraceID: true
          filterBySpanID: false
          spanStartTimeShift: "-2s"
          spanEndTimeShift: "+2s"

    - name: loki-prod
      uid: loki-prod
      type: loki
      url: http://loki-gateway.observability.svc:3100
      jsonData:
        derivedFields:
          - name: trace_id
            matcherRegex: '\btrace_id=([0-9a-f]{32})\b'
            url: '$${__value.raw}'
            datasourceUid: tempo-prod
            urlDisplayLabel: 'View trace'
          - name: trace_id_json
            matcherRegex: '"trace_id"\s*:\s*"([0-9a-f]{32})"'
            url: '$${__value.raw}'
            datasourceUid: tempo-prod

    - name: prom-prod
      uid: prom-prod-use1
      type: prometheus
      url: http://prometheus-server.monitoring.svc:9090
      jsonData:
        exemplarTraceIdDestinations:
          - name: trace_id
            datasourceUid: tempo-prod
            urlDisplayLabel: "View trace"
        timeInterval: "15s"
        prometheusType: Mimir
```

### Loki values for OTLP correlation

```yaml
# loki values.yaml (excerpt)
loki:
  limits_config:
    allow_structured_metadata: true       # <-- mandatory for trace_id in SM
  distributor:
    otlp_config:
      default_resource_attributes_as_index_labels:
        - service.name
        - service.namespace
        - deployment.environment.name
        - k8s.namespace.name
        - cluster
      # everything else stays as structured metadata
  ruler:
    enable_api: true
    evaluation_interval: 1m
    alertmanager_url: http://alertmanager.monitoring.svc:9093
    remote_write:
      enabled: true
      clients:                              # Loki 3.x map syntax
        mimir-prod:
          url: http://mimir.observability.svc:9009/api/v1/push
          headers:
            X-Scope-OrgID: tenant-default
```

### Grafana Correlation YAML (provisioned at boot)

```yaml
# /etc/grafana/provisioning/correlations/loki-tempo.yaml
apiVersion: 1

correlations:
  - uid: loki-trace-id-to-tempo
    sourceDatasourceUid: loki-prod
    targetDatasourceUid: tempo-prod
    label: 'View trace in Tempo'
    description: 'Open the matching trace from a log line'
    config:
      type: query
      target:
        query: '${trace_id}'
        queryType: traceql
      field: 'Line'
      transformations:
        - type: regex
          expression: '\btrace_id=([0-9a-f]{32})\b'
          mapValue: 'trace_id'

  - uid: tempo-span-to-loki
    sourceDatasourceUid: tempo-prod
    targetDatasourceUid: loki-prod
    label: 'Logs for this trace'
    config:
      type: query
      target:
        query: '{service="${service_name}"} | trace_id="${trace_id}"'
        queryType: range
      field: 'traceID'
```

### Terraform for the data sources + correlations

```hcl
resource "grafana_data_source" "prom_prod" {
  type = "prometheus"
  name = "prom-prod"
  uid  = "prom-prod-use1"
  url  = "http://prometheus-server.monitoring.svc:9090"

  json_data_encoded = jsonencode({
    timeInterval = "15s"
    exemplarTraceIdDestinations = [{
      name             = "trace_id"
      datasourceUid    = grafana_data_source.tempo_prod.uid
      urlDisplayLabel  = "View trace"
    }]
  })
}

resource "grafana_data_source" "tempo_prod" {
  type = "tempo"
  name = "tempo-prod"
  uid  = "tempo-prod"
  url  = "http://tempo-query-frontend.observability.svc:3200"

  json_data_encoded = jsonencode({
    tracesToLogsV2 = {
      datasourceUid       = grafana_data_source.loki_prod.uid
      tags = [
        { key = "service.name",       value = "service" },
        { key = "k8s.namespace.name", value = "namespace" },
      ]
      filterByTraceID     = true
      spanStartTimeShift  = "-2s"
      spanEndTimeShift    = "+2s"
    }
  })
}

resource "grafana_data_source" "loki_prod" {
  type = "loki"
  name = "loki-prod"
  uid  = "loki-prod"
  url  = "http://loki-gateway.observability.svc:3100"

  # NOTE on $-escaping: Grafana provisioning YAML expects `$${__value.raw}`
  # (literal "$$" so Grafana renders it as "$"). HCL collapses `$$` to a
  # single `$` during interpolation, so the literal `$$` we want in the
  # final JSON requires `$$$$` here in Terraform source.
  json_data_encoded = jsonencode({
    derivedFields = [
      {
        name           = "trace_id"
        matcherRegex   = "\\btrace_id=([0-9a-f]{32})\\b"
        url            = "$$$${__value.raw}"     # renders as $${__value.raw}
        datasourceUid  = grafana_data_source.tempo_prod.uid
        urlDisplayLabel = "View trace"
      }
    ]
  })
}
```

---

## Part 13: The 10 Commandments of Signal Correlation

1. **Thou shalt make `service.name` byte-identical across metrics, traces, and logs.** Not similar. Not close. Identical.
2. **Thou shalt prefer resource attributes over `trace_id` for durable correlation.** Trace_id only exists during a request; resource attributes always exist.
3. **Thou shalt never put `trace_id` in a Loki label.** It belongs in structured metadata (3.0+) or the log line body (2.x). Labels are the cardinality bomb.
4. **Thou shalt enable `--enable-feature=exemplar-storage` before configuring exemplar destinations.** The other order produces silent broken links.
5. **Thou shalt anchor `derivedFields` regexes with word boundaries.** A greedy regex over millions of log lines is a panel-freezing footgun.
6. **Thou shalt use one Alloy instance per node, not Promtail and Alloy and OTel Collector all at once.** Convergence is the entire point.
7. **Thou shalt prefer tail sampling over head sampling when log-trace correlation matters.** Head sampling produces orphan logs; tail sampling preserves correlation for interesting traces.
8. **Thou shalt convert expensive LogQL metric queries into Loki ruler recording rules.** A 24h dashboard refresh that scans 24h of logs every load is a bill you pay forever.
9. **Thou shalt use Grafana Correlations for new cross-data-source pivots.** Per-data-source link configs (`derivedFields`, `tracesToLogsV2`) still ship and still work for their canonical pivots, but Correlations is the strategically blessed long-term path.
10. **Thou shalt walk the round-trip click-through end-to-end after every config change.** Alert -> exemplar -> trace -> logs -> dashboard. If any link breaks, the whole correlation surface is broken until it's fixed.

---

The ten commandments above are really one rule, restated ten times: **get all three witnesses to call the suspect by the same name, and give every witness the same case-file number.** That's it. The camera (metrics) sees when something happened; the GPS (traces) sees the path it took; the diary (logs) sees what they were thinking. If you've enforced `service.name` consistency at the collector and put a trace_id in every log line, a 3 AM incident is four clicks: alert -> exemplar -> trace -> logs. Skip either discipline and the same incident is an hour of grep across three uncorrelated piles of evidence. The ROI on getting correlation right is measured in mean-time-to-resolution, and it compounds every incident forever.

---

## Cheat Sheet

```promql
# EXEMPLARS -- the metrics->traces query pattern
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)
# Then enable "Exemplars" toggle on the panel.
```

```yaml
# tracesToLogsV2 (Tempo data source)
tracesToLogsV2:
  datasourceUid: loki-prod
  tags:
    - { key: service.name,       value: service }
    - { key: k8s.namespace.name, value: namespace }
  filterByTraceID: true
  spanStartTimeShift: "-2s"
  spanEndTimeShift: "+2s"
```

```yaml
# derivedFields (Loki data source)
derivedFields:
  - name: trace_id
    matcherRegex: '\btrace_id=([0-9a-f]{32})\b'
    url: '$${__value.raw}'
    datasourceUid: tempo-prod
```

```yaml
# Grafana Correlation (provisioned)
correlations:
  - uid: loki-to-tempo
    sourceDatasourceUid: loki-prod
    targetDatasourceUid: tempo-prod
    config:
      type: query
      target: { query: '${trace_id}', queryType: traceql }
      field: 'Line'
      transformations:
        - type: regex
          expression: '\btrace_id=([0-9a-f]{32})\b'
          mapValue: 'trace_id'
```

```yaml
# OTel Collector resource processor -- enforce naming
processors:
  resource:
    attributes:
      - { key: deployment.environment.name, from_attribute: deployment.environment, action: insert }
      - { key: deployment.environment, action: delete }
      - { key: service.name, from_attribute: app, action: insert }
```

```yaml
# Loki OTLP attribute promotion (cardinality discipline)
distributor:
  otlp_config:
    default_resource_attributes_as_index_labels:
      - service.name
      - service.namespace
      - deployment.environment.name
      - k8s.namespace.name
      - cluster
```

```bash
# Diagnostics
logcli series '{}' --analyze-labels --since=1h          # Loki cardinality check
curl prom:9090/api/v1/query_exemplars?query=...         # Prom exemplar query
curl tempo:3200/api/traces/<trace_id>                    # Tempo trace fetch
```

---

## Further Reading

- [Honeycomb -- It's Time to Version Observability](https://www.honeycomb.io/blog/time-to-version-observability-signs-point-to-yes) -- the philosophy
- [OpenTelemetry semconv -- Resource Attributes](https://opentelemetry.io/docs/specs/semconv/resource/) -- the join key contract
- [Grafana -- Configure Trace to Logs](https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/configure-trace-to-logs/) -- `tracesToLogsV2` reference
- [Grafana -- Correlations](https://grafana.com/docs/grafana/latest/administration/correlations/) -- the modern API
- [Grafana Loki -- Structured Metadata](https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/) -- the trace_id storage substrate
- Apr 20-29 in this journal -- every individual signal that this doc stitches together
