# OpenTelemetry Fundamentals -- The Vendor-Neutral Wire Format That Finally Unified Metrics, Logs, and Traces

> The last seven days built a Prometheus-shaped observability stack: pull-based scraping (Apr 20), PromQL (Apr 21), Alertmanager (Apr 23), Grafana (Apr 24), kube-prometheus-stack (Apr 26). Every binary in that stack speaks one signal: Prometheus does metrics, Loki does logs, Tempo does traces, and they barely know each other exists. Each backend has its own scraper, its own agent, its own wire format, its own SDK in your code. Adding a service means instrumenting it three times. Switching backends means rewriting the instrumentation layer. This is the world OpenTelemetry was built to end.
>
> The core analogy for the day, scoped narrowly: **OTLP is USB-C for observability**. Before USB-C, every phone, laptop, camera, and headphone shipped with its own proprietary connector -- mini-USB, micro-USB, Lightning, MagSafe v1, MagSafe v2, barrel jacks of every diameter. Buying a new device meant buying a new charger; switching brands meant a drawer of dead cables. USB-C didn't replace the *power*; it standardized the *connector*. OTLP is the same move applied to the *wire format* between instrumentation and backend: every OTel-aware tool speaks one protocol, and the device manufacturer (Datadog, New Relic, Tempo, Mimir, Splunk, Honeycomb, your own ClickHouse) doesn't dictate the cable anymore. The API/SDK/Collector layering above OTLP gets its own analogy in Part 2 -- forcing one metaphor to span four concepts collapses under its own weight, and OpenTelemetry's whole pitch is that those four concepts are usefully *separate*. The industry already lived through proprietary chargers in the 2010s; it is now reliving the same transition for observability, and the OTLP wire format is the part where the dust has settled first.
>
> Two more analogies anchor today. (1) **OTLP** -- the wire protocol -- is the **metric system in observability**: once everyone agreed on meters and kilograms, instruments became interchangeable across countries, and a French ruler could measure a German bolt without translation. OTLP is the meter stick. (2) The Kubernetes **DaemonSet-plus-Gateway** Collector deployment is **neighborhood mailboxes plus a regional sorting facility**. Every node has its own local mailbox (the agent) so houses don't have to drive to the post office; the mailbox forwards to a central regional sorter (the gateway) which does the heavy work -- routing, redaction, sampling, fan-out to multiple delivery services. You wouldn't ask every house to drive directly to a national hub for every letter; you also wouldn't run the entire postal system out of one mailbox. Both layers exist for a reason.

---

**Date**: 2026-04-27
**Topic Area**: opentelemetry
**Difficulty**: Intermediate

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| OpenTelemetry (OTel) | USB-C for observability | Vendor-neutral standard for metrics, logs, and traces -- one API, one SDK, one wire protocol, swappable backends |
| Three signals (traces/metrics/logs) | Chain-of-custody record (traces), case statistics (metrics), witness statements (logs) | Three telemetry kinds OTel unifies under one model so they can be correlated by trace ID and resource attributes |
| Trace | The full chain-of-custody record for one request | A directed graph of spans showing the path a request takes across services |
| Span | One link in the chain-of-custody record | One unit of work (`name`, `start`, `end`, `attributes`, `events`, `parent_span_id`, `trace_id`) |
| Metric (in OTel) | Same as Prometheus, but with a passport (resource attributes) | Counter/UpDownCounter/Gauge/Histogram emitted via OTLP, carrying `service.name` + `k8s.*` everywhere |
| Log (in OTel) | Same nurse's notes, but stamped with `trace_id` so it joins back to the chain-of-custody record | Structured log record with timestamp, severity, body, attributes, and (if available) trace context |
| API | The USB-C socket on the wall | Vendor-neutral interface your code imports; defines `Tracer`, `Meter`, `Logger` types but does nothing on its own |
| SDK | The actual wiring + transformer inside the wall | Implementation that turns API calls into batched, sampled, exported telemetry |
| Collector | The building's electrical substation | Standalone process that receives, processes, and exports telemetry; lives outside your app |
| OTLP | The metric system of observability | The standard wire protocol (gRPC :4317 or HTTP/protobuf :4318) every OTel-aware tool speaks |
| Receiver | The substation's incoming high-voltage feeders | Collector component that ingests telemetry (otlp, prometheus, filelog, k8s_cluster, kubeletstats) |
| Processor | The substation's transformers + circuit breakers | In-pipeline mutation/filtering/sampling/batching component (memory_limiter, batch, k8sattributes, resource, tail_sampling, filter, transform) |
| Exporter | The outgoing distribution lines to neighborhoods | Collector component that ships telemetry to a backend (otlp, prometheusremotewrite, loki, otlphttp, vendor-specific) |
| Pipeline | The end-to-end electrical path from feeder to neighborhood | `service.pipelines.{traces,metrics,logs}` wiring receivers->processors->exporters together |
| Connector | A jumper cable between two pipelines | Special component that is an exporter to pipeline A and a receiver into pipeline B (e.g., spanmetrics, count) |
| Core distribution | The shrink-wrapped basic toolkit | Curated set of OTel-native components (otlp receiver/exporter, batch, memory_limiter); fits in one binary |
| Contrib distribution | The full hardware store | 100+ components including vendor exporters, prometheus receiver, k8sattributes processor; what production actually uses |
| OpenTelemetry Collector Builder (`ocb`) | The custom-spec hardware store kit | YAML manifest -> compiled Collector binary with only the components you want; the way to ship a slim distro |
| Resource attributes | The address-and-zip-code on every shipping label | `service.name`, `service.version`, `deployment.environment.name`, `k8s.pod.name`, `k8s.namespace.name`, `cloud.provider`, ... |
| Semantic conventions | The international postal-address format | The agreed names for those attributes -- the join keys across signals |
| Auto-instrumentation | A factory-installed dashcam in your car | Language agent that hooks HTTP/DB/gRPC libraries automatically, no source changes |
| Manual instrumentation | A trip diary you write yourself | Code-level `tracer.start_as_current_span("checkout")` for business logic |
| OTel Operator | The K8s mutating webhook that injects USB-C dongles into every pod | Manages `OpenTelemetryCollector` and `Instrumentation` CRDs; auto-instruments via pod annotations |
| `Instrumentation` CRD | The factory dashcam-install policy for the whole fleet | Per-namespace CR defining endpoint/sampler/propagators that injected pods inherit |
| Agent (DaemonSet) | The neighborhood mailbox on every node | One Collector per node; receives from local pods; node-level enrichment |
| Gateway (Deployment) | The regional sorting facility | Cluster-wide Collector pool; tail sampling, fan-out, vendor auth |
| Sidecar Collector | A private courier walking alongside one specific tenant | Per-pod Collector for resource isolation or per-team config; rarely needed |
| Tail sampling | Reading every witness's testimony before deciding whose deposition to keep | Sampling decision made AFTER the full trace assembles -- requires sticky routing |
| Head sampling | Flipping a coin at the front door before letting the witness in | Sampling decision made at trace start, propagated; cheap but loses errors |
| `loadbalancing` exporter | A trace-ID-based courthouse seating chart | Routes spans to the same Collector replica by trace ID, the prerequisite for tail sampling at scale |
| Grafana Alloy | A specialized OTel-Collector distro with a different config language (River) | Built on OTel Collector components; the natural choice when your backend is Mimir/Tempo/Loki |
| Vendor distro (ADOT, Splunk, Honeycomb) | OEM-branded electrical substations | OTel Collector + vendor-supported components, often with vendor-specific exporters and convenience features |

---

## The Big Picture -- How the Pieces Fit

Before any details, look at the entire OpenTelemetry world on one page:

```
OPENTELEMETRY -- END-TO-END VIEW
================================================================================

   YOUR APP POD                                  THE COLLECTOR LAYER (cluster)
+----------------------------+              +----------------------------------+
|                            |              |                                  |
|  Application code          |              |    DaemonSet "agent" Collector   |
|     |                      |              |    (one per node)                |
|     v                      |   OTLP       |                                  |
|  +---------+               |   (gRPC      |   receivers:                     |
|  |   API   |  imports only |    :4317)    |     otlp / kubeletstats /        |
|  | (vendor-|  the API      |   ---------> |     filelog / prometheus         |
|  |neutral) |  surface      |   localhost  |   processors:                    |
|  +----+----+               |   :4317      |     memory_limiter ->            |
|       |                    |              |     k8sattributes ->             |
|       v                    |              |     resource -> batch            |
|  +---------+               |              |   exporters:                     |
|  |   SDK   |  the actual   |              |     otlp -> gateway:4317         |
|  |(sampler,|  exporter     |              |                                  |
|  | batch,  |  ships OTLP   |              +-----------------+----------------+
|  |exporter)|                                                |
|  +---------+                                                | OTLP
|                                                             v
|  Optional: auto-instrumented      +-------------------------------+
|  via OTel Operator init           |   Deployment "gateway"        |
|  container injection              |   Collector (HPA-scaled)      |
|                                   |                               |
+-----------------------------------+   processors:                 |
                                    |     memory_limiter ->         |
                                    |     tail_sampling ->          |
                                    |     transform / filter ->     |
                                    |     batch                     |
                                    |                               |
                                    |   exporters:                  |
                                    |     otlp        -> Tempo      |
                                    |     prom rw     -> Mimir      |
                                    |     loki        -> Loki       |
                                    |     otlphttp    -> Datadog    |
                                    +---------+----+--------+-------+
                                              |    |        |
                                       traces |    |metrics |logs
                                              v    v        v
                                          +-------------------------+
                                          |      BACKENDS           |
                                          |  Tempo / Mimir / Loki   |
                                          |  (or vendor: Datadog,   |
                                          |   New Relic, Honeycomb) |
                                          +-------------------------+
```

Six things to remember about this picture:

1. **The API and the SDK are separate layers, on purpose.** Library authors depend on the API only -- a `requests` library or a database driver can emit OTel spans without dragging in any exporter, sampler, or backend. The application owner picks the SDK and configures it. This is exactly the slf4j/log4j separation the JVM world figured out twenty years ago, ported to telemetry.
2. **The Collector is a separate process, not a library.** It runs on its own, scales independently, restarts independently, and is the natural buffer between your app's instrumentation and your backend's availability. SDK-direct-export exists but is the wrong production answer.
3. **OTLP is the only wire protocol that matters.** App SDK -> agent Collector -> gateway Collector -> backend: every hop is OTLP unless you're talking to a non-OTel-native system (Prometheus remote_write, Loki HTTP). Once everyone speaks OTLP, you can swap any layer without touching the others.
4. **Resource attributes are the join keys.** A trace span carrying `{service.name=checkout, k8s.pod.name=checkout-7d9, deployment.environment.name=prod}` is joinable to a metric and a log line carrying the same triple. This is what unifies the three signals.
5. **The DaemonSet+Gateway split is the production K8s answer.** Agent does node-local enrichment (kubelet stats, pod labels) cheap and parallel; gateway does heavy cluster-wide work (tail sampling, vendor auth, fan-out) on a horizontally scalable pool.
6. **The Operator is what makes auto-instrumentation tractable.** Annotating a pod `instrumentation.opentelemetry.io/inject-python: "true"` adds an init container that drops a Python agent into the pod and sets env vars. No code changes, no Dockerfile changes, no rebuild.

---

## Part 1: The Three Signals -- and What OTel Brings to Each

OpenTelemetry's headline claim is that it unifies the **three pillars of observability** under one project, one wire format, one SDK shape, and one collector. You already know two-thirds of the metrics story from last week. The new material is traces and how OTel reframes logs.

### Metrics -- Same Math, New Passport

OTel metrics are conceptually identical to what you scrape from Prometheus -- counters, up-down counters, gauges, histograms. The differences are operational:

| Aspect | Prometheus (Apr 20) | OpenTelemetry |
|--------|---------------------|---------------|
| Wire format | Plaintext exposition (text/plain) on `/metrics` | OTLP (protobuf over gRPC :4317 or HTTP :4318) |
| Direction | Pull (server scrapes target) | Push (SDK sends to Collector) |
| Resource attributes | Optional via labels; usually job/instance only | Mandatory `service.name` + rich `k8s.*` and `cloud.*` resource set |
| Histograms | Classic (cumulative buckets) + Native (sparse exponential) | Explicit-bucket + Exponential-bucket (1:1 mapping to Prometheus native) |
| Aggregation temporality | Cumulative (counter only goes up; reset on restart) | Cumulative *or* Delta (delta is what statsd-style backends prefer) |
| Cardinality discipline | Same rules apply -- bounded labels only | Same rules apply, but `k8s.pod.name` is unbounded; **drop or replace it before the backend** if you don't have native pod-aware indexing |

The mental shortcut: **OTel metrics = the same thing you've been doing, with OTLP as the wire format and resource attributes attached.** Your existing Prometheus dashboards keep working as long as the Collector exports via `prometheusremotewrite` to Mimir or Prometheus's `--web.enable-remote-write-receiver`.

The one genuine new behavior is **delta temporality**. A Prometheus counter is cumulative -- it always reports the running total since process start. A delta-temporality counter reports "change since last export." Cumulative is friendlier to PromQL's `rate()` (which expects monotonic input); delta is friendlier to AWS CloudWatch and StatsD-style backends. The Collector can convert between the two via the `cumulativetodelta` and `deltatocumulative` processors, but the cleanest answer is: **default to cumulative in the SDK, convert at the gateway only when a downstream backend demands delta.**

### Logs -- The Last Pillar to Get the OTel Treatment

Logs were OTel's last signal to stabilize. The **data model** went stable in 2023; the **SDK** GA'd per-language on different timelines (Java and Go in 2024, .NET shortly after; Python and JavaScript were still in beta as of late 2025 and stabilized through early 2026). Log bridges -- the libraries that route existing logging frameworks (Logback, Serilog, log4net, structlog, winston) into the OTel SDK -- are GA on the JVM and .NET sides, still maturing in dynamic-language ecosystems. The OTel log model is:

```
LogRecord:
  timestamp:        nanos since epoch
  observed_timestamp: when the agent saw it (different if late-arriving)
  severity_number:  1-24 (TRACE=1, DEBUG=5, INFO=9, WARN=13, ERROR=17, FATAL=21)
  severity_text:    "INFO" / "ERROR" / etc.
  body:             AnyValue (string, map, array)
  attributes:       map<string, AnyValue>
  trace_id:         16 bytes (if log was emitted within a span context)
  span_id:          8 bytes
  resource:         {service.name, k8s.pod.name, ...}
```

Two things make the OTel log model better than plain text:

1. **`trace_id` and `span_id` are first-class fields, not attributes.** When your app emits a log line inside an in-progress span, the SDK auto-attaches the IDs. In Grafana Tempo, you can click a span and pivot to the matching Loki log lines via `traceID`. This is **the same exemplar drill-down pattern from the Apr 24 Grafana doc** (where exemplars on a histogram metric pointed at a trace ID), now run in reverse: there you started from a slow-latency bucket on a metric and clicked to a trace; here you start from a span and click to logs. The two click paths plus exemplars (logs ↔ traces ↔ metrics) close the triangle the Apr 26 kube-prometheus-stack could only do for metrics. The unified-observability doc later this week will pull these threads together explicitly.
2. **Logs share the same Resource as your traces and metrics.** `service.name=checkout` shows up identically in all three signals. No more "I have CloudWatch logs tagged `Service` and Prometheus metrics tagged `service`, and now my dashboards can't join them."

In practice, you collect logs through one of three paths:

- **Application-side OTLP log emission** -- your code uses the OTel logging SDK to ship logs alongside traces. Cleanest model, but requires SDK adoption.
- **`filelog` receiver in the Collector** -- the agent (DaemonSet) tails container log files (`/var/log/pods/...`) and converts each line into a LogRecord. This is the Promtail/Fluent Bit replacement.
- **Bridge from existing logging frameworks** -- log4j/SLF4J/zap/winston have OTel "appenders" or "bridges" that convert their output into OTel LogRecords without rewriting the app.

Most production deployments end up with **`filelog` in the agent for stdout/stderr** plus **OTLP-direct from a few critical services that need precise trace-log correlation**.

#### The three-layer trace-to-logs correlation model

When a Grafana "View related logs" button on a Tempo span returns zero results despite logs clearly existing for that service, the failure is on one of three layers, *and all three must be right*:

1. **The log record must carry `trace_id`** -- application/SDK problem. The most common failure: traces went through OTel auto-instrumentation but logs are still being written to stdout and tailed with no trace-context awareness. Fix: wire the logger to OTel context (`OpenTelemetryAppender` for Logback/Log4j2 on JVM, `LoggingHandler` from `opentelemetry-sdk` on Python, `OpenTelemetry.Logs` on .NET, `instrumentation-pino` on Node), or ship logs via the OTLP log exporter so `trace_id` is a first-class top-level field on the `LogRecord`.
2. **The log record must expose `trace_id` as something queryable** -- schema/data-source problem. If `trace_id` is buried inside the message body as text, Loki can't filter on it. Fix: emit logs as structured JSON with `trace_id`/`span_id` as top-level fields, *or* use Loki's **`structuredMetadata`** (Loki 2.9+/3.0) which is the modern non-cardinality-bomb home for trace IDs -- non-indexed, doesn't add label cardinality, queryable via `| trace_id="abc"`. Last-resort fallback: configure a `derivedField` regex on the Loki data source to extract trace_id from the body. Never make `trace_id` a Loki *label* -- that's a cardinality bomb. Note that when logs ship via the OTLP log exporter, this entire layer disappears -- Grafana's Loki data source automatically exposes the OTLP top-level `trace_id` field as queryable, and you can skip JSON-restructuring and derived-field regex.
3. **Grafana must know how to construct the query** -- data-source-config problem. The "View related logs" button silently builds a query against a label/field that doesn't exist. Fix: in the Tempo data source settings configure the linked Loki UID, the tag map (e.g. `service.name` -> `service_name`), and enable "Filter by trace ID."

Layer 1 fails most often (auto-instrumentation handles traces and metrics but not always logs). Layer 2 fails when teams print rather than structure. Layer 3 fails when the data source was set up before logs were even in scope. The diagnostic discipline: when correlation is broken, walk down the three layers in order before guessing.

### Traces -- The Genuinely New Material

A **trace** is the full record of one request as it crosses services. A trace is a tree (sometimes a DAG) of **spans**, each span representing one unit of work:

```
trace_id = a1b2c3...

span: "POST /checkout"               kind=SERVER       (root span; no parent)
  |
  +-- span: "auth.verify"            kind=CLIENT       (child of /checkout)
  |
  +-- span: "inventory.reserve"      kind=CLIENT
  |     +-- span: "DB SELECT"        kind=CLIENT       (downstream service emits this)
  |     +-- span: "redis SETNX"      kind=CLIENT
  |
  +-- span: "payment.charge"         kind=CLIENT
  |     +-- span: "stripe POST"      kind=CLIENT
  |
  +-- span: "kafka publish"          kind=PRODUCER     (async hand-off)
        +-- (in a different service:
              span: "kafka consume"  kind=CONSUMER     -- linked, not child)
```

Each span carries:

| Field | What it holds |
|-------|---------------|
| `name` | A short, low-cardinality label like `GET /api/users/:id` (NOT the full URL with the user ID) |
| `kind` | `SERVER` / `CLIENT` / `INTERNAL` / `PRODUCER` / `CONSUMER` -- shapes how backends visualize it |
| `start_time` / `end_time` | ns-precision timestamps |
| `status` | `OK` / `ERROR` / `UNSET` -- a span being slow doesn't make it an error; an exception does |
| `attributes` | Free-form key-values: `http.status_code`, `db.system`, `messaging.destination` |
| `events` | Timestamped sub-records -- exception stacks, GC pauses, retry markers |
| `links` | Cross-trace pointers (for batch processing, async fan-out) |
| `parent_span_id` | Defines the tree |
| `trace_id` + `span_id` | The 16+8-byte IDs that propagate over the wire |

The **W3C Trace Context** propagation standard defines how trace IDs cross service boundaries: every outbound HTTP/gRPC call carries a `traceparent` header (`00-<trace_id>-<span_id>-<flags>`) that the receiving service uses to make its first span a child of the caller's span. This is the entire reason a distributed trace assembles -- every hop adds its span to the same trace ID. The SDK's auto-instrumentation handles propagation for you on standard libraries; for custom transports (proprietary RPC, message buses without header support) you propagate manually.

**`parent_span_id` vs `links` -- when a span is a *child* vs a *referrer*.** Most spans have exactly one `parent_span_id`, and that defines the tree (a SERVER span is the child of the upstream CLIENT span). `links` are for cases where the parent-child model breaks down:
- **Async fan-out**: a job submits 100 messages to Kafka; the eventual CONSUMER span shouldn't be a child of all 100 producers, but it *should* point back at the producer that emitted its message. Producer span = parent of an INTERNAL span recording "submitted message X"; consumer span = links to that submission span (not parent, because the trace would otherwise have to span hours of queue time).
- **Batch processing**: a batch job processes 1000 input records and emits one output. The output span links to all 1000 input spans -- it has no single parent.
- **Retries**: a retry span links to the original failed attempt rather than parenting it (the original is closed).
- **Cross-trace pointers**: the most legitimate use, e.g. a `cron`-fired workflow links back to the user-initiated trace that originally enqueued the job hours earlier.

The mental rule: **parent = "I am a step inside this work"; link = "I am related to this other work but not nested under it."** If you find yourself making decisions about whether a span has 50 parents, you actually want links.

We will go deeper on traces tomorrow (sampling strategies, span processors, Tempo). For today, the conceptual unlock is: **a trace is the chain-of-custody record of one request, made of spans linked by trace_id and parent_span_id, with `links` as the escape hatch for relationships the tree can't model.**

### What OTel Adds That Pillar-Specific Stacks Don't

Three things show up only when the same project owns all three signals:

1. **Cross-signal correlation by construction.** All three signals carry the same Resource (`service.name`, `k8s.pod.name`, `deployment.environment.name`). Logs and traces share `trace_id`. Metrics histograms can attach **exemplars** (a `trace_id` pointer on a single observation) -- click a slow-latency bucket in Grafana, jump to the trace.
2. **One SDK shape, one config surface.** A Java service configures `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_TRACES_SAMPLER` once and gets traces, metrics, and logs. There is no "Datadog Java agent + Prometheus client + Logback Loki appender" mix.
3. **One Collector pipeline.** The same Collector binary processes all three signals through the same processor catalog. `k8sattributes` enriches all three. `batch` batches all three. `memory_limiter` protects all three.

---

## Part 2: API vs SDK vs Collector -- The Three-Layer Cake

This separation is the single most common point of confusion. Internalize it now.

```
+-----------------------------+
|        Application          |
+-----------------------------+
              |
              | calls e.g. tracer.start_span("checkout")
              v
+-----------------------------+         "the USB-C socket on the wall"
|            API              |          - vendor-neutral
|   (opentelemetry-api)       |          - defines abstract Tracer/Meter/Logger
|                             |          - default impls are no-ops
+-----------------------------+         - safe for libraries to depend on
              |
              | the application wires in an SDK at startup
              v
+-----------------------------+         "the actual wiring inside the wall"
|            SDK              |          - real implementation
|   (opentelemetry-sdk)       |          - samplers, processors, batchers
|                             |          - exporters (OTLP, console, ...)
+-----------------------------+         - decides how often to send and where
              |
              | OTLP gRPC :4317 or HTTP :4318
              v
+-----------------------------+         "the building's electrical substation"
|         Collector           |          - separate process
|   (otelcol-contrib binary)  |          - receivers, processors, exporters
|                             |          - independent scaling and lifecycle
+-----------------------------+         - the buffer between app and backend
              |
              v
        Tempo / Mimir / Loki / vendor backend
```

### Why the API/SDK split matters

A library author -- say the `requests` HTTP client maintainer or the `psycopg` Postgres driver maintainer -- wants to emit OTel spans for every HTTP call or query. If they had to depend on the SDK, they'd be dragging exporters, samplers, batchers, and a Collector address into every transitive dependency. Instead, the library depends only on **the API package**, which is small, has no transitive deps, and *does nothing* at runtime if no SDK is configured.

The application owner -- the team running the service -- chooses the SDK at the entry point of `main()`. If they configure the OTLP exporter, the library's spans suddenly start flowing. If they don't configure anything, the library code still runs, the API calls turn into no-ops, and there's zero overhead. **The API is the contract; the SDK is the engine.**

This is identical to the JVM logging story: SLF4J is the API, Logback/Log4j2 are SDKs. If your library prints to SLF4J and the app provides no logger, nothing happens. If the app wires Logback, you suddenly get logs.

### Why the Collector is its own process

You could, in principle, configure the SDK to export OTLP directly to your backend (Tempo/Mimir/vendor cloud). For a development laptop, that's fine. For production, never do it:

- **Backpressure.** If the backend is slow or down, the SDK's export buffer fills up and either drops telemetry or starts adding latency to your application's request path. A Collector buffer is in a separate process; backend pressure stays in the Collector.
- **Network partitions.** A Collector with a persistent file-based queue can survive a half-hour backend outage and replay everything. An in-process SDK buffer typically holds 30 seconds of data.
- **Per-pod connection storm.** Imagine 500 pods each holding a gRPC connection to your backend. Now imagine restarting them. The Collector funnels 500 pods into 5 gateway connections.
- **Cross-cutting policy.** Sampling, redaction, cost controls, multi-tenant routing belong in one place where ops owns them, not duplicated in 200 microservices' SDK configs.
- **Vendor independence.** Switching from Datadog to New Relic should be a Collector exporter change, not a deploy of every service to swap an SDK.

The rule: **App SDK -> Collector (always) -> Backend.** No exceptions in production.

---

## Part 3: OTLP -- The Metric System of Observability

OTLP (OpenTelemetry Protocol) is the wire format. It is to telemetry what TCP is to networking: boring infrastructure that everyone agreed on.

### Two transports, one schema

| Transport | Default port | Format | When to use |
|-----------|-------------|--------|-------------|
| gRPC | **4317** | protobuf over HTTP/2 | Default. Lower overhead, streaming, multiplexing. Use this everywhere unless something blocks gRPC. |
| HTTP/protobuf | **4318** | protobuf over HTTP/1.1 POST | Locked-down environments, FaaS without persistent connections, debugging with `curl`. |
| HTTP/JSON | **4318** (same listener) | JSON-encoded protobuf shape over HTTP/1.1 POST | **The only transport viable from browsers** (no protobuf decoder in JS by default, gRPC blocked by CORS) and the readable-with-`jq` option for debugging. ~3-5x larger payloads than protobuf -- not for production volume. |

Same protobuf schema in all three cases -- only the framing/encoding differs. The HTTP listener serves both protobuf and JSON on the same port; content-type negotiation (`application/x-protobuf` vs `application/json`) selects the encoding.

### Why OTLP matters strategically

Before OTLP, every backend had its own ingest format: Datadog DogStatsD, New Relic Insights JSON, Jaeger Thrift, Zipkin JSON v2, Prometheus exposition text, Loki HTTP push API, OpenCensus, OpenTracing wire, AWS X-Ray JSON. To send telemetry to two backends, you needed two SDK configurations and two agents. Switching backends required rewriting the instrumentation layer.

With OTLP:
- Your **app SDK speaks OTLP** -- only OTLP.
- Your **Collector speaks OTLP** to the SDK and again OTLP to other Collectors.
- Your **backend ingests OTLP natively** (Tempo, Honeycomb, Datadog, New Relic, Grafana Cloud, AWS CloudWatch / AMP / X-Ray, Splunk Observability Cloud, Dynatrace -- all of them as of 2024-2026). Note that **AWS Distro for OpenTelemetry (ADOT)** is a *Collector distribution*, not a backend; the AWS-native backends behind it are CloudWatch, AMP (Managed Prometheus), and X-Ray.
- Switching backends is a Collector exporter swap, not a code change.

This is the unlock that makes vendor-neutrality real. The market has moved here decisively in the last three years -- 2026 is the first year you can confidently say "we will instrument our entire fleet with OTel and pick a backend later."

### Confusing port gotcha

`OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317` is gRPC. `OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318` is HTTP. The error message when you mismatch them is something like `failed to parse protobuf: unexpected EOF`, which is unhelpful. Some SDKs require `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` or `=http/protobuf` explicitly. Always set both endpoint and protocol to avoid the silent failure mode where one app uses gRPC and the listener is HTTP-only.

---

## Part 4: The Collector Pipeline -- Receivers -> Processors -> Exporters

The Collector is the workhorse of the OTel stack. Its config follows a clean three-stage shape.

### Anatomy of a Collector config

```yaml
# otel-collector-config.yaml
# Define every component, then wire them in service.pipelines

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  prometheus:                          # the Collector can SCRAPE Prometheus targets
    config:
      scrape_configs:
        - job_name: my-app
          scrape_interval: 30s
          static_configs:
            - targets: ['app:9090']
  filelog:                             # tail container log files
    include: [/var/log/pods/*/*/*.log]
    start_at: end
    operators:
      - type: container                # parses the K8s log format
  kubeletstats:                        # node-level kubelet metrics
    collection_interval: 30s
    auth_type: serviceAccount
    endpoint: ${env:K8S_NODE_NAME}:10250
    insecure_skip_verify: true
    metric_groups: [pod, container, node]
  k8s_cluster:                         # cluster-level k8s object events
    auth_type: serviceAccount
    collection_interval: 30s

processors:
  # ORDER MATTERS in service.pipelines below; declaration order here is irrelevant
  memory_limiter:                      # ALWAYS first in pipelines -- prevents OOM
    check_interval: 1s
    limit_percentage: 80               # GC kicks at 80% of process memory
    spike_limit_percentage: 25
  k8sattributes:                       # enriches with k8s.pod.*, k8s.namespace.*, etc.
    auth_type: serviceAccount
    extract:
      metadata:
        - k8s.pod.name
        - k8s.pod.uid
        - k8s.deployment.name
        - k8s.namespace.name
        - k8s.node.name
        - k8s.pod.start_time
      labels:
        - tag_name: app.kubernetes.io/component
          key: app.kubernetes.io/component
          from: pod
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip
      - sources:
          - from: connection           # falls back to source IP of OTLP request
  resource:
    attributes:
      - key: deployment.environment.name
        value: prod
        action: upsert
      - key: cloud.provider
        value: aws
        action: upsert
      - key: k8s.cluster.name
        value: prod-us-east-1
        action: upsert
  resourcedetection:                   # auto-detects EC2/EKS/GKE/AKS metadata
    detectors: [env, eks, ec2]
    timeout: 5s
    override: false                    # don't overwrite values you set explicitly
  filter/dropnoisy:                    # drop the noisy /healthz spans
    traces:
      span:
        - 'attributes["http.target"] == "/healthz"'
        - 'attributes["http.target"] == "/metrics"'
  transform/sanitize:                  # redact a sensitive header
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - delete_key(attributes, "http.request.header.authorization")
  tail_sampling:                       # gateway-only; needs sticky routing!
    decision_wait: 30s                 # buffer spans this long before deciding
    num_traces: 50000
    expected_new_traces_per_sec: 1000
    policies:
      - name: errors-100pct
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow-100pct
        type: latency
        latency: { threshold_ms: 1000 }
      - name: baseline-1pct
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }
  batch:                               # ALWAYS last before exporter
    timeout: 5s
    send_batch_size: 8192
    send_batch_max_size: 16384

exporters:
  otlp/tempo:
    endpoint: tempo-distributor.observability.svc:4317
    tls:
      insecure: true
  prometheusremotewrite:
    endpoint: http://mimir-distributor.observability.svc/api/v1/push
    headers:
      X-Scope-OrgID: prod-us-east       # Mimir tenant header
    resource_to_telemetry_conversion:
      enabled: true                    # turn resource attrs into metric labels
  loki:
    endpoint: http://loki-gateway.observability.svc/loki/api/v1/push
    headers:
      X-Scope-OrgID: prod-us-east
  otlphttp/datadog:                    # vendor backup path -- swap in/out without code changes
    endpoint: https://otlp.us3.datadoghq.com
    headers:
      dd-api-key: ${env:DD_API_KEY}
  debug:                               # console pretty-print for troubleshooting
    verbosity: detailed
    sampling_initial: 5

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:                               # go pprof for debugging hot Collectors
    endpoint: 0.0.0.0:1777
  zpages:                              # in-cluster span/metric inspector at :55679
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, pprof, zpages]
  telemetry:
    metrics:
      level: detailed
      address: 0.0.0.0:8888            # Collector self-metrics; SCRAPE separately!
    logs:
      level: info
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, k8sattributes, resource, resourcedetection,
                   filter/dropnoisy, transform/sanitize, tail_sampling, batch]
      exporters:  [otlp/tempo, otlphttp/datadog]
    metrics:
      receivers:  [otlp, prometheus, kubeletstats, k8s_cluster]
      processors: [memory_limiter, k8sattributes, resource, resourcedetection, batch]
      exporters:  [prometheusremotewrite]
    logs:
      receivers:  [otlp, filelog]
      processors: [memory_limiter, k8sattributes, resource, resourcedetection, batch]
      exporters:  [loki]
```

A few things this YAML earns its keep on:

- **Defining a component is necessary but not sufficient.** It must be referenced in `service.pipelines.{traces,metrics,logs}` to actually run. Forgetting the wiring is the most common debug session.
- **Processor order is the order in `service.pipelines`, not the order under top-level `processors:`.** The top-level block is just a registry of named instances.
- **Same component, multiple instances.** `filter/dropnoisy` and `transform/sanitize` use the `<type>/<name>` pattern -- you can have many `filter` instances with different names.
- **Self-monitoring goes out a different door.** `:8888` is the Collector's own Prometheus metrics endpoint. It must be scraped *externally* (your Prometheus or another Collector) -- if you scraped it through this same Collector, a brief Collector outage would erase the evidence of itself going down.

### The Canonical Processor Order

This is muscle memory. Memorize it:

```
memory_limiter -> k8sattributes -> resource(/detection) -> filter/transform -> tail_sampling -> batch -> exporter
```

Why this order:

1. **`memory_limiter` first.** If incoming load spikes, the limiter rejects new data with a recoverable error before downstream processors allocate buffers. A `memory_limiter` placed third can OOM the process while spans are queued ahead of it.
2. **`k8sattributes` early.** Subsequent processors (filter, transform, sampling) often want to make decisions based on `k8s.namespace.name` or `k8s.deployment.name`. Enrich first, decide later.
3. **`resource` and `resourcedetection` next.** These set Resource attributes that the exporters and downstream Collectors will rely on for `service.name`, environment, cluster, region.
4. **Filter/transform.** Drop noise, redact, mutate. Cheaper to drop a span before sampling considers it.
5. **`tail_sampling`** -- gateway only -- decides last among processors because it needs the full enriched and filtered span population.
6. **`batch` is always immediately before the exporter.** Batching reorders nothing; it just amortizes the network cost of OTLP gRPC calls.

Skipping `memory_limiter` or putting `batch` early is the canonical "Collector OOMs at 3 AM under burst load" failure pattern.

### Receivers, Processors, Exporters -- the Component Catalog

A non-exhaustive but high-value subset:

| Receivers (data in) | What it does |
|---------------------|--------------|
| `otlp` | The standard inbound -- gRPC :4317 + HTTP :4318. Always present. |
| `prometheus` | The Collector becomes a Prometheus scraper. Drops `/metrics` exposition text into the metrics pipeline. Replaces having a separate Prometheus binary in many cases. |
| `filelog` | Tails files. Used for K8s container stdout/stderr at `/var/log/pods/.../*.log`. |
| `kubeletstats` | Talks to the kubelet's `:10250/stats/summary`. Replaces `cAdvisor` scraping. |
| `k8s_cluster` | Watches the K8s API for object-level events and metrics. Pairs with `kube-state-metrics`-style use cases. |
| `hostmetrics` | CPU, memory, disk, net at the host level. Replaces a node_exporter-only flow. |
| `jaeger`, `zipkin` | Backwards compatibility with legacy tracing fleets during migration. |

| Processors (in-flight transform) | What it does |
|----------------------------------|--------------|
| `memory_limiter` | Backpressure / OOM protection. Always first. |
| `batch` | Coalesces telemetry into bigger OTLP requests. Always last. |
| `k8sattributes` | Adds `k8s.pod.*`, `k8s.namespace.name`, `k8s.deployment.name`, `k8s.node.name` by IP / by connection / by pod_uid. Requires RBAC. |
| `resource` / `resourcedetection` | Set or auto-detect Resource attributes (cloud, host, env). |
| `attributes` / `transform` | Mutate any attribute via OTTL (the OTel Transformation Language). |
| `filter` | Drop spans/metrics/logs by predicate. Cheap noise reduction. |
| `tail_sampling` | The "keep all errors and slow traces, sample 1% of the rest" decision. **Stateful** -- buffers spans for `decision_wait` before deciding. **Sticky-routed** -- all spans of one trace must land on the same Collector instance. |
| `probabilistic_sampler` | Stateless head-based sampling -- cheap, decides before buffering. Cannot keep all errors. |
| `groupbyattrs` | Reorganizes batches by an attribute -- useful before `batch` to keep one tenant per batch. |
| `cumulativetodelta` / `deltatocumulative` | Convert metric temporality. |
| `metricstransform` / `metricsgeneration` | Rename, scale, derive metrics inside the pipeline. |

| Exporters (data out) | What it does |
|----------------------|--------------|
| `otlp` | Forward to another Collector or an OTLP-native backend. The standard inter-Collector hop. |
| `otlphttp` | Same, over HTTP/protobuf. Used when the destination requires HTTP (Datadog, Honeycomb, browsers). |
| `prometheusremotewrite` | Pushes metrics to Mimir/Prometheus/VictoriaMetrics/AMP. The bridge between OTLP metrics and the Prometheus universe. |
| `loki` | Pushes logs to Loki via its native HTTP push API. |
| `prometheus` | EXPOSES `/metrics` for someone else to SCRAPE. Wildly different operational model from `prometheusremotewrite` -- the Collector becomes a target instead of a sender. |
| `file` | Writes to disk for offline analysis or backfill. |
| `debug` | Prints to stdout. Indispensable during config development; never in prod pipelines. |
| Vendor: `datadog`, `splunk_hec`, `awsxray`, `googlecloud`, `dynatrace`, `newrelic` | Vendor-specific exporters. Mostly redundant in 2026 since vendors accept OTLP natively, but they expose vendor-specific features (e.g., Datadog APM trace flame graphs). |

### Connectors -- Pipelines Talking to Pipelines

Connectors are a relatively new component type that act as an exporter for one pipeline and a receiver for another:

```yaml
connectors:
  spanmetrics:                         # generate RED metrics from traces
    namespace: spanmetrics
    histogram:
      explicit:
        buckets: [10ms, 50ms, 100ms, 200ms, 500ms, 1s, 2s, 5s]
    dimensions:
      - name: http.status_code
      - name: http.method

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp/tempo, spanmetrics]   # spans go out AND into the connector
    metrics/from-spans:
      receivers: [spanmetrics]               # connector emits metrics here
      exporters: [prometheusremotewrite]
```

Two especially useful connectors:
- **`spanmetrics`** -- compute RED metrics (rate/errors/duration) from spans without instrumenting metrics separately.
- **`servicegraph`** -- emit service-to-service edge metrics from traces, powering Grafana's service map.

---

## Part 5: Core vs Contrib -- Which Distribution to Run

The OpenTelemetry Collector ships in two upstream flavors plus the option to roll your own:

| Distribution | Components | Use it for |
|--------------|------------|------------|
| **Core** (`otelcol`) | ~25 OTel-native components | Slim Collectors. Works for OTLP-only flows. Doesn't include `prometheus` receiver, `k8sattributes`, `loki` exporter, or any vendor exporter. |
| **Contrib** (`otelcol-contrib`) | 100+ components, including everything above | What 95% of production deployments use. Larger binary (~250 MB), but you almost certainly need at least one contrib component (k8sattributes, prometheusremotewrite, vendor exporter, ...). |
| **Custom via OpenTelemetry Collector Builder (`ocb`)** | Whatever you list in a YAML manifest | Slim binary with only the components you want. Use when binary size matters or for security audits ("we ship only signed, audited components"). |
| **Vendor distros** (ADOT from AWS, Splunk Distribution, Honeycomb's Refinery-paired distro, Grafana Alloy) | Contrib subset plus vendor add-ons | Pick when you want vendor support contracts and curated upgrade paths. |

`ocb` lets you write:

```yaml
# builder-manifest.yaml
dist:
  name: my-otelcol
  output_path: ./dist

receivers:
  - gomod: go.opentelemetry.io/collector/receiver/otlpreceiver v0.103.0
processors:
  - gomod: github.com/open-telemetry/opentelemetry-collector-contrib/processor/memorylimiterprocessor v0.103.0
  - gomod: github.com/open-telemetry/opentelemetry-collector-contrib/processor/batchprocessor v0.103.0
  - gomod: github.com/open-telemetry/opentelemetry-collector-contrib/processor/k8sattributesprocessor v0.103.0
exporters:
  - gomod: go.opentelemetry.io/collector/exporter/otlpexporter v0.103.0
```

Then `ocb --config builder-manifest.yaml` produces a binary one-fifth the size of `otelcol-contrib`, vetted to exactly the components you listed. Most teams don't bother and just run Contrib; the savings only matter at FaaS scale or in regulated environments.

---

## Part 6: Auto-Instrumentation vs Manual Instrumentation

OTel's value proposition collapses without instrumentation. There are two flavors.

### Auto-instrumentation -- the factory-installed dashcam

A language **agent** that hooks into the runtime and emits spans/metrics for popular libraries with zero code changes.

| Language | Auto-instrumentation mechanism | What you get for free | Maturity |
|----------|--------------------------------|----------------------|----------|
| **Java** | `-javaagent:opentelemetry-javaagent.jar` JVM agent | Servlet, Spring MVC, JDBC, JMS, Kafka clients, gRPC, async frameworks (~150 libraries) | Excellent (oldest, most coverage) |
| **Python** | `opentelemetry-instrument` wrapper or import-time `opentelemetry.instrumentation.auto_instrumentation` | Flask, Django, FastAPI, requests, urllib3, redis-py, psycopg, sqlalchemy, kafka-python | Excellent |
| **Node.js** | `--require @opentelemetry/auto-instrumentations-node/register` | Express, Fastify, Koa, http(s), gRPC, mongodb, mysql2, redis, ioredis | Excellent |
| **.NET** | Profiler-based agent (CLR profiler API) or `OpenTelemetry.AutoInstrumentation` package | ASP.NET Core, HttpClient, SqlClient, gRPC, MassTransit | Good |
| **Ruby** | Add `opentelemetry-instrumentation-all` to Gemfile + call `OpenTelemetry::SDK.configure { \|c\| c.use_all }` at boot | Rails, ActiveRecord, Net::HTTP, Sidekiq, Faraday, Redis | Good |
| **Go** | **No runtime injection possible** -- requires either build-time instrumentation (`otelbuild`) or eBPF-based instrumentation (`opentelemetry-go-instrumentation`) | What's available varies; eBPF agent traces HTTP/gRPC at kernel level | Improving (eBPF agent in beta) |
| **PHP** | Zend extension (`opentelemetry-php`) | Laravel, Symfony, PDO, curl | Limited |
| **Rust** | Manual only -- no auto-instrumentation | n/a | Manual |

**What is "auto-instrumentation" actually doing at runtime?** The mechanism varies dramatically by language family, and understanding it is what separates "I added a flag" from "I can debug why no spans are appearing":

- **JVM (Java, Kotlin, Scala, Clojure)**: the agent uses **Byte Buddy / ASM bytecode rewriting** at class-load time. The `-javaagent` flag registers a `java.lang.instrument` hook; before any application class is loaded by the classloader, the agent inspects it, matches against a registry of "instrumented library" patterns (Spring's `DispatcherServlet`, JDBC's `PreparedStatement`, Kafka's `Producer.send`, etc.), and rewrites methods in-place to wrap them with span-creation logic. The original `.class` file on disk is unchanged; the in-memory representation is patched. This is why JVM auto-instrumentation has the deepest coverage -- bytecode rewriting can reach private methods, async callbacks, and library internals no source-level wrapper could touch.
- **.NET**: the agent hooks the **CLR Profiler API** -- a low-level Microsoft profiling interface designed for APM tools. The profiler intercepts JIT compilation events, modifies IL (intermediate language) for matched methods, and forces re-JIT. Same idea as the JVM agent, different runtime; functionally equivalent reach.
- **Python**: the `opentelemetry-instrument` wrapper sets `PYTHONPATH` to point at a bootstrap module, which **monkey-patches imports at startup** -- when your code does `import flask`, the bootstrap intercepts the import and substitutes a wrapped version that calls the original under a span context. Less powerful than bytecode rewriting (can't reach private methods or already-imported modules), but trivial to ship -- no JVM agent dance, no profiler-API permissions.
- **Node.js**: the `--require` hook installs **monkey-patches at module load time** via `require-in-the-middle`. Same playbook as Python -- the runtime supports import interception cheaply, so the SDK leans on it.
- **Ruby**: the `use_all` API loads instrumentation gems that **call `Module.prepend` on the target classes** at startup. Same monkey-patch family, slightly different mechanism (Ruby's prepend chain is the idiomatic way to wrap a method without breaking inheritance).
- **Go**: see below -- Go is the genuine outlier.
- **PHP**: a **Zend extension** loaded by the PHP interpreter, which intercepts function calls at the engine level. Similar to how Xdebug works.

The shape of the takeaway: bytecode rewriting (JVM/.NET) > import monkey-patching (Python/Node/Ruby) > extension hook (PHP) > nothing (Go). The deeper the runtime hook, the more coverage you get for free, but also the higher the chance of breaking when the underlying library changes its internals.

The Go situation is genuinely different: Go binaries are statically compiled with no JIT, no class loader, no dynamic linker hooks. There is nothing to hook into at startup. The two answers are:

- **`opentelemetry-go-instrumentation`** -- an eBPF agent that traces uprobes in Go binaries from outside the process. Requires running as a separate DaemonSet with `CAP_BPF` and `CAP_SYS_ADMIN`. Coverage is narrower than JVM auto-instrumentation but expanding.
- **Build-time wrapping** -- libraries like `otelhttp`, `otelgrpc`, `otelpgx` provide drop-in middlewares that you import explicitly. Closer to manual instrumentation than auto.

### Manual instrumentation -- the trip diary

For business-meaningful operations -- "checkout submitted," "payment authorized," "inventory reserved" -- you write spans yourself:

```python
# Python example -- manual span around a business operation
from opentelemetry import trace, metrics

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)
checkout_counter = meter.create_counter("checkout.completed", unit="1")

def submit_checkout(cart, user):
    with tracer.start_as_current_span(
        "checkout.submit",
        attributes={
            "checkout.cart_size": len(cart.items),
            "checkout.cart_value_usd": cart.total_usd,
            "user.tier": user.tier,             # low-cardinality: free / pro / enterprise
            # NOT user.id -- unbounded cardinality on an attribute
        },
    ) as span:
        try:
            charge = charge_payment(cart, user)
            span.set_attribute("payment.provider", charge.provider)
            checkout_counter.add(1, {"status": "ok", "user.tier": user.tier})
            return charge
        except PaymentDeclined as e:
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            span.record_exception(e)
            checkout_counter.add(1, {"status": "declined", "user.tier": user.tier})
            raise
```

Three habits make manual instrumentation pay off:

1. **Span names should be low-cardinality.** `checkout.submit`, not `checkout submitted at 2026-04-27T13:42:11`. The variable detail goes in attributes.
2. **Attributes are subject to the same cardinality discipline as Prometheus labels.** `user.tier` is bounded; `user.id` is not. Putting `user.id` in a span attribute is fine for traces (Tempo doesn't index attributes by default), but if `spanmetrics` later derives a metric from it, you've created a cardinality bomb.
3. **`set_status(ERROR)` is what flips the span red in Grafana, not a slow duration.** A 30-second-but-successful operation has `status=OK`. A 100ms-but-threw operation has `status=ERROR`. Don't conflate latency with errors.

### The decision: auto, then manual

**Start with auto-instrumentation. Add manual spans for business operations.** The auto agent gives you HTTP, DB, and external-call coverage on day one, which buys you the framework of the trace tree. Manual spans fill in the business-meaningful nodes that no library can know about.

---

## Part 7: Resource Attributes & Semantic Conventions

The single most consequential decision in your OTel rollout is: **which resource attributes do you set, with what values, on every signal.** This is the Resource block on every Span, Metric, and LogRecord:

```
resource:
  service.name:                  checkout
  service.namespace:             retail
  service.version:               2.7.4
  service.instance.id:           checkout-7d9f-x9k4z
  deployment.environment.name:   prod         # was "deployment.environment" pre-v1.27
  k8s.cluster.name:              prod-us-east-1
  k8s.namespace.name:            retail
  k8s.deployment.name:           checkout
  k8s.pod.name:                  checkout-7d9f-x9k4z
  k8s.pod.uid:                   abcd-1234-...
  k8s.node.name:                 ip-10-0-3-4.ec2.internal
  k8s.container.name:            app
  cloud.provider:                aws
  cloud.region:                  us-east-1
  cloud.availability_zone:       us-east-1a
  host.name:                     ip-10-0-3-4.ec2.internal
  host.arch:                     amd64
  os.type:                       linux
```

These attributes are **not just labels.** They are the join keys that make a Tempo trace, a Mimir metric, and a Loki log line provably about the same workload. If a trace has `service.name=checkout` but the metric has `service=checkout` (different attribute name), the cross-signal correlation breaks even though a human can read both.

Two rules:

1. **Use the standard names from the [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/).** Don't invent `app.team` if `service.namespace` already exists. Don't shorten `k8s.deployment.name` to `deployment`. Stable, agreed names are non-negotiable.
2. **Set them on every signal, identically.** Easiest: set them on the Resource at the SDK level (auto-detection + `OTEL_RESOURCE_ATTRIBUTES`) and again at the Collector level via `resource` and `k8sattributes` processors as a defense-in-depth.

In 2026, the K8s semantic conventions graduated from experimental to release-candidate -- meaning the names above are *stable* and safe to commit dashboards and alerts against. In semantic conventions v1.27, `deployment.environment` was **deprecated alongside** the new `deployment.environment.name` -- both are still in the spec, but `.name` is the future-proof choice. There is no automatic aliasing; you have to write a `transform` processor rule (`set(attributes["deployment.environment.name"], attributes["deployment.environment"]) where attributes["deployment.environment.name"] == nil`) during the rollout, otherwise dashboards filtering on one name will miss services emitting the other.

### Where these attributes come from

There are three injection sites, layered:

1. **SDK auto-detection** -- the SDK's resource detector reads env vars, EC2 metadata, EKS metadata, GCP metadata, host info. Set `OTEL_RESOURCE_ATTRIBUTES=service.name=checkout,service.version=2.7.4,deployment.environment.name=prod` once on the pod and the SDK picks it up.
2. **The `resourcedetection` processor in the Collector** -- the agent Collector running as DaemonSet auto-detects EC2/EKS/GCE metadata and adds `cloud.*` and `host.*`.
3. **The `k8sattributes` processor** -- the agent Collector queries the K8s API by source IP of the OTLP packet (or by `k8s.pod.uid` if the SDK provided it) and adds `k8s.*` attributes.

The OTel Operator's auto-instrumentation injection sets `OTEL_RESOURCE_ATTRIBUTES` automatically with the K8s downward-API values, so you usually get this layered defense for free.

---

## Part 8: Deployment Patterns on Kubernetes

The eternal question: where does the Collector run? Five answers, in increasing sophistication:

### Pattern 0: SDK direct export, no Collector

```
[App SDK] --OTLP--> [Backend]
```

- Simplest. Works in dev/laptop scenarios.
- **Production: never.** Network blip = lost telemetry. Backend slow = app slow. Per-pod connections to backend storm at restart. No central policy for sampling / redaction / multi-tenancy.

### Pattern 1: Agent only (DaemonSet)

```
[App SDK] --OTLP localhost:4317--> [Agent (DaemonSet, one per node)] --> [Backend]
```

- Each node runs one Collector. Pods send OTLP to `node-ip:4317` (typically discovered via the K8s downward API: `OTEL_EXPORTER_OTLP_ENDPOINT=http://$(NODE_IP):4317`).
- Agent does node-local enrichment: `k8sattributes`, `kubeletstats`, `filelog` for container logs, `hostmetrics`.
- **OK for small clusters.** Each agent must hold a connection to the backend; each agent is its own back-pressure unit.

### Pattern 2: Gateway only (Deployment)

```
[App SDK] --OTLP gateway-svc:4317--> [Gateway (Deployment, replicas=N)] --> [Backend]
```

- One central horizontally-scaled pool of Collectors behind a Service. Pods send to a stable DNS name.
- Gateway can do tail sampling, fan-out exports, vendor auth.
- **OK for small clusters with no per-node enrichment needs.** Loses the cheap node-local processing.

### Pattern 3: Agent + Gateway (the production answer)

```
[App SDK] --OTLP localhost:4317-->
        [Agent DaemonSet] --OTLP gateway-svc:4317-->
                [Gateway Deployment, HPA-scaled] --> [Backends]
```

- DaemonSet agents do node-local work (k8sattributes, kubeletstats, filelog) and forward via OTLP to a Gateway pool.
- Gateway pool does heavy work: tail sampling, vendor exporters, multi-tenant routing, redaction, fan-out.
- **The right answer for almost any cluster above 20 nodes.**

### Pattern 4: Sidecar Collector

```
[App + Sidecar Collector in same pod] --OTLP localhost:4317--> [Sidecar] --> [Gateway]
```

- One Collector per pod. Resource isolation per workload; per-team config when sharing infra.
- **Niche.** Useful for: multi-tenant clusters where each team needs its own redaction policy; latency-sensitive workloads where the localhost hop matters; services that need a per-pod credential the cluster-wide agent shouldn't have.
- The OTel Operator can inject a sidecar Collector via an `OpenTelemetryCollector` CR with `mode: sidecar` plus the pod annotation `sidecar.opentelemetry.io/inject: "name"`.

### The Tail-Sampling Stickiness Gotcha

Tail sampling decides which traces to keep *after* seeing the entire trace. That requires the Collector instance making the decision to have **all spans of the trace.** In a multi-replica gateway with normal load balancing, span A of trace T might land on replica 1 and span B might land on replica 2 -- neither replica has the full trace, so neither can correctly tail-sample.

Three answers:

1. **Single Gateway replica.** Trivial; doesn't scale.
2. **`loadbalancing` exporter on a front-of-gateway Collector.** A first tier of Collectors hashes by trace ID and routes to the same gateway replica every time. Now replica 1 always sees all spans of trace T. This is the standard pattern at scale.
3. **Push tail sampling to the backend.** Tempo, Honeycomb, and Grafana Cloud Traces can do tail-sampling-equivalent work at ingest. This avoids the sticky-routing complexity at the Collector layer, at the cost of vendor-specific features.

The architecture for option 2 looks like:

```
[Apps] -> [Agent DaemonSet] -> [Front Gateway: loadbalancing exporter, hash by trace ID]
                                        -> [Tail-sampling Gateway replica 0]
                                        -> [Tail-sampling Gateway replica 1]
                                        -> [Tail-sampling Gateway replica 2]
                                        -> ... -> [Tempo]
```

The front gateway is stateless and trivially scales; the tail-sampling pool is what carries the trace ID buffers.

**Two architectural details that bite teams in production:**

1. **The tail-sampling pool should be a `StatefulSet`, not a `Deployment`.** Tail sampling needs stable pod identity so the `loadbalancing` exporter's `dns` resolver returns predictable endpoints (point it at a *headless* Service for the StatefulSet so DNS returns all pod IPs, not the Service VIP). A Deployment behind a headless Service technically works, but pod IPs churn on every rollout and the consistent-hash ring rebuilds every time -- defeating the stickiness you went to all this trouble for.

2. **`decision_wait` must exceed your DNS-rebalance window, not just your slowest p99.9.** The `loadbalancing` exporter re-resolves DNS on a configurable interval (`resolver.dns.interval`, default 5s). During pool scale events -- HPA scale-up, rolling restart, node drain -- there's a brief window where new pods enter the consistent-hash ring and a small fraction of in-flight traces have their *later* spans routed to a different replica than their *earlier* spans. If `decision_wait` is shorter than `dns.interval + propagation_time + the slowest service's p99.9`, the trace is decided before its late spans land on the new replica, and you get the same span-fragment-evaluation problem the `loadbalancing` exporter was supposed to prevent. Tail sampling's "wait for the whole trace" is also "wait for routing to converge."

---

## Part 9: The OpenTelemetry Operator -- K8s-Native Management

The OTel Operator is a Kubernetes controller that manages two CRDs: `OpenTelemetryCollector` and `Instrumentation`.

### `OpenTelemetryCollector` -- declarative Collector deployments

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: gateway
  namespace: observability
spec:
  mode: deployment                     # or daemonset, sidecar, statefulset
  image: otel/opentelemetry-collector-contrib:0.103.0
  replicas: 3
  resources:
    requests: { cpu: 200m, memory: 1Gi }
    limits:   { cpu: 2,    memory: 4Gi }
  serviceAccount: otel-gateway
  config:                              # the same YAML as the file-based config
    receivers:
      otlp:
        protocols:
          grpc: { endpoint: 0.0.0.0:4317 }
          http: { endpoint: 0.0.0.0:4318 }
    processors:
      memory_limiter: { check_interval: 1s, limit_percentage: 80 }
      batch: { timeout: 5s, send_batch_size: 8192 }
      tail_sampling:
        decision_wait: 30s
        policies:
          - name: errors
            type: status_code
            status_code: { status_codes: [ERROR] }
          - name: baseline
            type: probabilistic
            probabilistic: { sampling_percentage: 1 }
    exporters:
      otlp/tempo:
        endpoint: tempo-distributor.observability.svc:4317
        tls: { insecure: true }
    service:
      pipelines:
        traces:
          receivers:  [otlp]
          processors: [memory_limiter, tail_sampling, batch]
          exporters:  [otlp/tempo]
  ports:
    - name: otlp-grpc
      port: 4317
      protocol: TCP
    - name: otlp-http
      port: 4318
      protocol: TCP
```

What the Operator does:
- Creates the Deployment/DaemonSet/StatefulSet, ConfigMap, Service, ServiceAccount, and (optionally) Ingress.
- Validates the `config` against the included component set.
- Rolls out config changes with a careful update strategy.
- Wires `targetallocator` for the Prometheus receiver in HA setups (see below).

### `targetallocator` -- the bridge that lets multi-replica Collectors replace single-replica Prometheus

The right way to think about migrating from kube-prometheus-stack to OTel is **disaggregation**: Prometheus was doing **four jobs in one process** -- scrape discovery, scraping, storage/query, and rule evaluation -- and OTel splits those across purpose-built components, with the headline win being that scraping itself becomes horizontally shardable. The `targetallocator` solves the discovery + scrape-distribution piece. Storage moves to a Prometheus-compatible TSDB (Mimir, Cortex, Thanos, Grafana Cloud) via `prometheusremotewrite`. **Rule evaluation moves to Mimir's ruler** (or Cortex/Thanos ruler) -- the OTel Collector has no built-in rules evaluator, so `PrometheusRule` CRDs are read by the ruler, which evaluates against Mimir's query engine and pushes alerts to the same Alertmanager you already run. Application `/metrics` endpoints, `ServiceMonitor`/`PodMonitor` CRs, `PrometheusRule` CRs, dashboards, and Grafana data sources all stay byte-for-byte unchanged -- the only thing teams *might* notice is faster recovery from cardinality spikes, which now hit one Collector shard instead of taking the whole Prometheus down.

The `targetallocator` is the single most operationally important Operator feature for teams migrating from the kube-prometheus-stack (Apr 26). Native Prometheus is single-replica because two Prometheus instances each scraping the same target produces duplicate data with no easy dedup story (the Apr 20 fundamentals doc covered the HA-via-`replica`-external-label fingerprint dedup hack). That's why kube-prometheus-stack defaults to one replica and pushes durability via remote_write.

`targetallocator` solves this differently: it watches `ServiceMonitor` and `PodMonitor` CRDs, computes the full scrape target list, then **shards the targets across N Collector replicas** -- each replica scrapes a disjoint subset. Add a Collector replica, the allocator rebalances; lose one, the allocator reallocates the orphaned targets to surviving replicas.

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: prom-otel
spec:
  mode: statefulset                  # required for sticky pod identity during rebalance
  replicas: 3
  targetAllocator:
    enabled: true
    serviceAccount: ta-sa
    prometheusCR:
      enabled: true                  # watch ServiceMonitor / PodMonitor CRDs
      serviceMonitorSelector: {}     # match all
      podMonitorSelector: {}
  config: |
    receivers:
      prometheus:
        config:
          scrape_configs: []         # empty -- the allocator injects the targets
    # ... processors + exporters
```

This is the migration path from kube-prometheus-stack: keep your `ServiceMonitor` CRDs (the Apr 26 doc's primary scrape-target abstraction), drop Prometheus, replace it with N Collector replicas + targetallocator, and you've gained horizontal scrape-side scalability without rewriting a single team's monitor. The `prometheusremotewrite` exporter then pushes to Mimir/Cortex/AMP exactly as Prometheus did.

### `Instrumentation` CRD -- annotation-driven auto-injection

The Operator has a mutating webhook. When a pod is created with the right annotation, the webhook mutates the pod spec to add an init container that drops a language agent into a shared volume and sets env vars.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: default
  namespace: retail
spec:
  exporter:
    endpoint: http://otel-agent.observability.svc:4317
  propagators:
    - tracecontext
    - baggage
    - b3                               # if you have legacy B3 services in the mix
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"                    # 10% head-based; tail sampling on the gateway tightens further
  resource:
    addK8sUIDAttributes: true
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.46b0
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.32.0
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.49.0
  dotnet:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:1.2.0
  go:
    image: ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:v0.13.0
```

Then on the Deployment that needs auto-instrumentation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  namespace: retail
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
        # OR: "default" to reference the Instrumentation by name
        # OR: "retail/default" for cross-namespace
```

What the webhook does on pod creation:
1. Adds an `initContainer` running the language-specific image; it copies `/otel-auto-instrumentation/` into a shared `emptyDir` volume.
2. Mounts the volume into the app container at `/otel-auto-instrumentation/`.
3. Sets `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_PROPAGATORS`, `OTEL_SERVICE_NAME` (= deployment name), and the language-specific bootstrap variable (`PYTHONPATH`, `JAVA_TOOL_OPTIONS=-javaagent:...`, `NODE_OPTIONS=--require ...`).

Result: a pod that previously emitted nothing now emits OTLP traces and metrics on every request, with zero source code or Dockerfile changes.

Language nuances:
- **JVM, Python, Node.js, .NET** -- works smoothly, has been production-ready for 18+ months.
- **Go** -- the `go` instrumentation is the eBPF agent variant; it runs as a separate process injected via init container, requires `CAP_BPF`/`CAP_SYS_ADMIN` on the pod, and traces only the workloads explicitly opted in. Coverage is narrower than JVM auto-instrumentation but improving.

---

## Part 10: OTel vs Vendor Agents -- The Strategic Argument

Vendor-specific agents are the incumbent: Datadog Agent, New Relic Infrastructure Agent, Dynatrace OneAgent, AppDynamics, Splunk SignalFx Smart Agent. Each has been polished for 5-15 years. They are still ahead of OTel on:

- **One-click custom dashboards** ("here's your service map without configuration")
- **Proprietary signals** -- RUM (real-user monitoring in browsers), session replay, code-level profiling integrated with traces, deployment markers, synthetic monitoring
- **AI-assisted root-cause analysis** (the "why is checkout slow right now" feature)
- **Polish** -- the agent is tuned, the UI is tuned, the dashboards are tuned

OTel wins on:

- **Vendor neutrality.** Your instrumentation outlives your contract.
- **Zero per-host or per-metric pricing on the agent itself.** OSS Collector is free; you pay only the backend.
- **One agent and one wire format across the fleet** instead of N vendor agents.
- **Source-code instrumentation that doesn't change** when you switch backends. Manual spans you wrote two years ago still work after migrating from Datadog to Honeycomb.
- **Multi-vendor fan-out.** Send traces to Tempo and Datadog simultaneously. Useful during migrations.

The 2024-2026 market reality:

- **Every major APM vendor now ingests OTLP natively.** Datadog, New Relic, Dynatrace, Splunk, Honeycomb, AppDynamics. Datadog's official position is "send us OTLP, we accept it." This is the change that made the bet worth taking.
- **Vendors increasingly contribute to OTel itself.** Splunk, Honeycomb, Lightstep (now ServiceNow), and Grafana are major Collector contributors.
- **Vendor distros are common.** ADOT (AWS), Splunk Distribution, Grafana Alloy. Same Collector, vendor-curated component selection.

The strategic rule: **Instrument once with OTel. Run a Collector. Pick the backend that fits this quarter's budget, regulatory constraints, and feature needs. Switch when those constraints change.**

### The vendor-vs-OTel binary is itself softening

Two-camp framing -- "you're either dd-trace or you're OTel" -- was real in 2022-2023 and is less true in 2026:

- **dd-trace libraries can emit OTLP.** Newer dd-trace versions support OTLP export, meaning you can write spans against the dd-trace API but ship them through an OTel Collector pipeline. The data-model differences (Continuous Profiler ↔ trace stitching, DBM correlation, ASM signal linking, RUM->APM session stitching) still bind you to Datadog's ingest for those *features*, but the wire protocol no longer does.
- **The Datadog Agent is OTLP-capable.** OTel SDKs in your apps -> Datadog Agent (acting as the local Collector) -> Datadog backend is a fully supported path.
- **Dynatrace, New Relic, Splunk, Grafana Cloud, Honeycomb, Lightstep all ingest OTLP natively** with feature parity for the table-stakes APM features.

What this means in practice: **the migration cost between vendor-agent-instrumentation and OTel-instrumentation is dramatically lower in 2026 than it was 18 months ago.** The pragmatic hybrid that real mid-size orgs land on -- OTel SDKs across the fleet plus dd-trace specifically for the one or two services where Continuous Profiler or DBM matters most -- is supported by both ecosystems and no longer requires rewriting instrumentation if priorities shift.

**The diagnostic question to ask leadership** when this debate comes up: *"What would have to be true for us to migrate off our current vendor?"* If the answer is "almost nothing would convince us," the optionality argument for OTel is performative -- pick the vendor SDK, optimize cost via the Collector and contract negotiation, ship value. If the answer is "we'd switch if X happened," write down what X is, and you're now paying the OTel adoption cost specifically to keep that option real and measurable. The worst outcome is the one most teams default to: nominally choosing OTel for "vendor-neutrality" while never building the operational discipline (Collector ownership, semantic-convention adherence, query-usage telemetry) that makes the option actually exercisable.

---

## Part 11: Convergence with Grafana Alloy

The Apr 24 Grafana doc mentioned **Grafana Alloy** as the unified telemetry collector replacing Promtail, Grafana Agent, and the OTel Collector for Grafana-stack users. The key fact:

> Alloy is *built on* the OTel Collector. It uses the upstream Collector's component framework under the hood, including the receivers, processors, and exporters. The visible difference is the **config language**: Alloy uses **River** (a HCL-flavored, expression-rich syntax) instead of OTel's YAML.

What Alloy adds on top of bare Contrib:

- **Native Prometheus pipelines** as first-class blocks (`prometheus.scrape`, `prometheus.remote_write`) -- arguably nicer than the Collector's `prometheus` receiver for shops that came from a Prometheus-first world.
- **Loki-native blocks** (`loki.source.file`, `loki.process`, `loki.write`) -- replaces Promtail.
- **Pyroscope profiling support** -- continuous profiling integrated as another signal.
- **River expressions** -- variables, conditionals, and component references resolved at config-eval time, instead of YAML's strict-template approach.

What Alloy doesn't change:
- OTLP is still OTLP. Alloy receives and emits OTLP; it interoperates with any OTel Collector.
- Component types (receivers, processors, exporters) are the same Collector components, often just wrapped by Alloy's River names.

**When to pick which:**

| Tool | Pick when |
|------|-----------|
| OTel Collector (Contrib) | Vendor-neutral pipeline, mixed backends, want the upstream config syntax, want maximum component coverage. |
| Grafana Alloy | All-in on Grafana stack (Mimir + Tempo + Loki + Pyroscope), want first-class Prometheus and Loki pipelines, prefer River over YAML, value Grafana's enterprise support. |
| Vendor distro (ADOT, Splunk Distribution) | Want a vendor support contract, run on that vendor's cloud (e.g., ADOT on EKS), value pre-curated component lists. |

The migration path either way is shallow because everything speaks OTLP -- you can run Alloy at the agent tier and OTel Collector at the gateway tier, or vice versa.

---

## Part 12: Production Helm + Terraform Examples

### Agent Collector via the upstream Helm chart

> **Version note**: the chart version (e.g. `0.95.0`) and the Collector image tag (`0.103.0`) move on different cadences. The chart's `appVersion` typically tracks the Collector minor closely; if you pin a Collector newer than the chart's default, set `image.tag` explicitly as below. In production, consider pinning the chart to the matching Collector release (chart `0.103.x` for Collector `0.103.x`) once it's published.

```yaml
# helm install opentelemetry-collector-agent open-telemetry/opentelemetry-collector \
#   --version 0.95.0 -n observability -f agent-values.yaml

# agent-values.yaml
mode: daemonset
image:
  repository: otel/opentelemetry-collector-contrib       # ALWAYS contrib in prod
  tag: 0.103.0
presets:
  kubernetesAttributes:                                  # auto-enables k8sattributes processor
    enabled: true
    extractAllPodLabels: false
    extractAllPodAnnotations: false
  kubeletMetrics:
    enabled: true                                        # enables kubeletstats receiver
  logsCollection:
    enabled: true                                        # enables filelog receiver for /var/log/pods
    includeCollectorLogs: false                          # avoid log feedback loops
clusterRole:
  create: true                                           # k8sattributes needs pod/ns list+watch RBAC
  rules:
    - apiGroups: [""]
      resources: [pods, namespaces, nodes]
      verbs: [get, list, watch]
    - apiGroups: [apps]
      resources: [deployments, replicasets, statefulsets, daemonsets]
      verbs: [get, list, watch]
config:
  exporters:
    otlp:
      endpoint: opentelemetry-collector-gateway.observability.svc.cluster.local:4317
      tls: { insecure: true }
  processors:
    memory_limiter:
      check_interval: 1s
      limit_percentage: 80
      spike_limit_percentage: 25
    resource:
      attributes:
        - key: deployment.environment.name
          value: prod
          action: upsert
        - key: k8s.cluster.name
          value: prod-us-east-1
          action: upsert
    batch:
      timeout: 5s
      send_batch_size: 8192
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, k8sattributes, resource, batch]
        exporters: [otlp]
      metrics:
        receivers: [otlp, kubeletstats]
        processors: [memory_limiter, k8sattributes, resource, batch]
        exporters: [otlp]
      logs:
        receivers: [otlp, filelog]
        processors: [memory_limiter, k8sattributes, resource, batch]
        exporters: [otlp]
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 1,    memory: 1Gi }
tolerations:                                              # run on every node, including tainted ones
  - operator: Exists
priorityClassName: system-node-critical                  # don't get evicted before user pods
```

### Gateway Collector (Deployment, HPA-scaled)

```yaml
# helm install opentelemetry-collector-gateway open-telemetry/opentelemetry-collector \
#   --version 0.95.0 -n observability -f gateway-values.yaml

mode: deployment
replicaCount: 3
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 12
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
image:
  repository: otel/opentelemetry-collector-contrib
  tag: 0.103.0
service:
  type: ClusterIP                                         # internal-only; agents send to it
config:
  receivers:
    otlp:
      protocols:
        grpc: { endpoint: 0.0.0.0:4317 }
        http: { endpoint: 0.0.0.0:4318 }
  processors:
    memory_limiter:
      check_interval: 1s
      limit_percentage: 80
    tail_sampling:                                        # gateway-tier tail sampling
      decision_wait: 30s
      num_traces: 50000
      policies:
        - name: errors-100pct
          type: status_code
          status_code: { status_codes: [ERROR] }
        - name: slow-100pct
          type: latency
          latency: { threshold_ms: 1000 }
        - name: baseline-1pct
          type: probabilistic
          probabilistic: { sampling_percentage: 1 }
    transform/sanitize:
      trace_statements:
        - context: span
          statements:
            - delete_key(attributes, "http.request.header.authorization")
            - delete_key(attributes, "http.request.header.cookie")
    batch: { timeout: 5s, send_batch_size: 8192 }
  exporters:
    otlp/tempo:
      endpoint: tempo-distributor.observability.svc:4317
      tls: { insecure: true }
    prometheusremotewrite:
      endpoint: http://mimir-distributor.observability.svc/api/v1/push
      headers: { X-Scope-OrgID: prod-us-east }
      resource_to_telemetry_conversion: { enabled: true }
    loki:
      endpoint: http://loki-gateway.observability.svc/loki/api/v1/push
      headers: { X-Scope-OrgID: prod-us-east }
  service:
    telemetry:
      metrics: { level: detailed, address: 0.0.0.0:8888 }
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, transform/sanitize, tail_sampling, batch]
        exporters: [otlp/tempo]
      metrics:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [prometheusremotewrite]
      logs:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [loki]
resources:
  requests: { cpu: 500m, memory: 1Gi }
  limits:   { cpu: 4,    memory: 4Gi }
podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

### The same gateway via the OTel Operator's CRD

```yaml
# kubectl apply -f gateway-collector.yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: gateway
  namespace: observability
spec:
  mode: deployment
  replicas: 3
  image: otel/opentelemetry-collector-contrib:0.103.0
  config:
    # ... same config block as above ...
```

### `Instrumentation` CR for Python auto-injection

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: default
  namespace: retail
spec:
  exporter:
    endpoint: http://opentelemetry-collector-agent.observability.svc:4317
  propagators: [tracecontext, baggage]
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"                     # 100% from SDK; let gateway tail-sample
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.46b0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  namespace: retail
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
    spec:
      containers:
        - name: app
          image: ghcr.io/example/checkout:2.7.4
          env:
            - name: OTEL_SERVICE_NAME
              value: checkout
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: service.namespace=retail,service.version=2.7.4,deployment.environment.name=prod
```

### Terraform `helm_release` wrapper

```hcl
resource "helm_release" "otel_agent" {
  name       = "otel-agent"
  namespace  = "observability"
  repository = "https://open-telemetry.github.io/opentelemetry-helm-charts"
  chart      = "opentelemetry-collector"
  version    = "0.95.0"

  values = [templatefile("${path.module}/values/agent.yaml.tmpl", {
    cluster_name = var.cluster_name
    environment  = var.environment
    gateway_svc  = "otel-gateway.observability.svc.cluster.local:4317"
  })]

  depends_on = [helm_release.otel_operator]   # CRDs must exist first if using Operator
}

resource "helm_release" "otel_gateway" {
  name       = "otel-gateway"
  namespace  = "observability"
  repository = "https://open-telemetry.github.io/opentelemetry-helm-charts"
  chart      = "opentelemetry-collector"
  version    = "0.95.0"

  values = [templatefile("${path.module}/values/gateway.yaml.tmpl", {
    tempo_endpoint = "tempo-distributor.observability.svc:4317"
    mimir_endpoint = "http://mimir-distributor.observability.svc/api/v1/push"
    loki_endpoint  = "http://loki-gateway.observability.svc/loki/api/v1/push"
    tenant         = var.environment
  })]
}
```

---

## Production Gotchas (Pinned to the Wall)

A non-exhaustive list of the things that bite teams in their first six months of OTel:

1. **`memory_limiter` must be the FIRST processor in every pipeline.** If it isn't, sustained load can OOM the Collector before backpressure kicks in. Symptom: pods restart-loop under traffic spikes; metrics show no rejection events.
2. **Tail sampling requires sticky routing or a `loadbalancing` exporter front tier.** Multi-replica gateways with naive load balancing will randomly distribute spans of the same trace across replicas and tail-sampling decisions become incoherent. Symptom: "we sample 1% but I can never find the trace I'm looking for."
3. **The `batch` processor before retry can lose data on Collector restart unless persistent queue is enabled.** Default in-memory queue evaporates on pod restart. Set `sending_queue.storage` to a `file_storage` extension for survival.
4. **SDK direct export is fragile under network partitions.** If your backend is across the internet (vendor cloud), every blip is dropped telemetry. Always front it with a Collector that has retry + persistent queue.
5. **`resourcedetection` processor defaults to `override: false`, so SDK-set Resource attributes win over Collector auto-detection.** If your SDK pins `cloud.region=us-east-1` at app startup and the pod fails over to `us-west-2`, the *Collector* sees the pod is in `us-west-2` but the *attribute on the spans* is still `us-east-1` -- because the SDK got there first and the processor refuses to overwrite. Symptom: DR-failover region attribute is permanently stale, regional dashboards lie. Set `override: true` on Collector-authoritative attributes, or stop setting them in the SDK.
6. **Collector version skew between agent and gateway.** New OTLP fields (e.g., new histogram bucket types) added in 0.103 won't deserialize on a 0.92 gateway and may be silently dropped. Pin both layers to the same minor version and roll them together.
7. **`k8sattributes` processor needs cluster-wide RBAC for `pods`, `namespaces`, and (for ownership-chain resolution) `replicasets`/`deployments`/`statefulsets`/`daemonsets` `list` + `watch`.** Without it, the processor silently passes through with no enrichment. Symptom: `k8s.deployment.name` is missing on every span. Set `clusterRole.create=true` in Helm and grant the ownership-chain rules.
8. **Auto-instrumentation language version mismatch.** The Java agent's compatibility matrix lists supported JDK versions; mounting it into a JDK 8 container with a JDK 21-only agent silently fails (no traces, no errors in the agent log unless you crank verbosity). Pin agent versions explicitly per Instrumentation CR per language.
9. **OTLP gRPC retries `UNAVAILABLE` but not `INVALID_ARGUMENT`, so a Collector that 400s a malformed batch silently drops the whole batch.** The SDK's success counter looks healthy because the request *completed*; it's the Collector's `otelcol_exporter_send_failed_spans_total` (and `_metric_points_total`, `_log_records_total`) that records the loss. Watch the Collector's exporter-failed counters, not just the SDK side. Common triggers: a span with an invalid `trace_id` length, an attribute key that violates the spec, a histogram with negative bucket counts after a buggy aggregation. Add the `otelcol_processor_dropped_*` counters and `otelcol_exporter_send_failed_*` to your default Prometheus scrape so this loss is visible.
10. **The Collector emits its own telemetry on `:8888`, and you must scrape it externally -- never through itself.** If you route the Collector's self-metrics through the very same Collector that's reporting them, an outage erases the evidence you need to debug the outage. The right pattern is the inverse of the Apr 26 kube-prometheus-stack: there, Prometheus scrapes everything *via* `ServiceMonitor` CRDs that the Operator generates; here, you treat each Collector pod as just another scrape target and drop a `ServiceMonitor` matching its `:8888` port. Everything you learned about `ServiceMonitor` lifecycle (label selectors, namespace selectors, the `enforced*` capacity overrides) applies unchanged. The Collector self-metrics worth alerting on: `otelcol_processor_dropped_*`, `otelcol_exporter_send_failed_*`, `otelcol_processor_refused_*`, `otelcol_processor_batch_batch_send_size`, and the queue length on persistent-queue exporters.
11. **Sampling decisions don't propagate retroactively.** Once a parent service head-samples out (`parentbased_traceidratio` decides "drop"), the trace is gone from every downstream service that respects parent context. You can never get those spans back. Conservative head sampling + tail sampling on the gateway is the safer combo.
12. **`prometheus` exporter and `prometheusremotewrite` exporter are NOT the same operational model.** `prometheus` exposes a `/metrics` endpoint for someone else to scrape -- the Collector becomes a target. `prometheusremotewrite` actively pushes to Prometheus/Mimir. Pick the wrong one and you'll wonder why no metrics are arriving (target wasn't added to Prometheus's scrape configs) or why the push is failing (no remote-write endpoint exists).
13. **Span attribute cardinality is your responsibility.** Tempo and most trace backends don't enforce limits the way Prometheus does, but `spanmetrics`-derived metrics absolutely do. A `user.id` attribute that becomes a metric label = a million-series cardinality bomb. Audit your attribute choices.
14. **K8sattributes pod_association by IP fails when pods share a node IP (host-network pods).** Set multiple `pod_association` rules: by `k8s.pod.uid` if the SDK provides it, by `k8s.pod.ip`, and finally by `connection` source IP. Symptom: random spans missing K8s metadata.
15. **`tail_sampling.decision_wait` floor is bound by the slowest service in scope, not the average.** Set it to 30s and a service whose p99.9 is 45s routinely has its trace decision *finalized* before the slow span arrives -- so the slow span lands in a different decision window, with the rest of its trace already exported or dropped. The trace looks fine in 99.9% of cases and silently incoherent in the tail you most wanted to keep. Set `decision_wait > p99.9` of every service in the cluster, and instrument the Collector's `otelcol_processor_tail_sampling_late_span_age` to detect drift. The accidental upper bound this puts on Collector memory (`decision_wait * spans_per_second`) is the real reason multi-pool tail-sampling tiers exist.
16. **Telemetry changes are production changes.** Dropping a metric, raising sampling thresholds, or filtering log severities looks like config tuning but is functionally equivalent to deleting a database column -- if a Grafana panel, an alert rule, or someone's runbook depends on what you removed, you've just silently broken it. The discipline: **(1)** pull query-usage telemetry from Grafana/Mimir/Loki/Tempo before dropping anything (which metrics are actually queried? which logs are actually filtered?), **(2)** dry-run with the `debug` exporter and compare cardinality/volume before-and-after, **(3)** announce in a platform changelog with a per-team opt-out, **(4)** stage the rollout with a 7-day "everything still flowing to S3 archive" safety net so you can backfill if you cut wrong. Symptom of skipping this: an on-call who finds that the burn-rate alert silently stopped firing two weeks ago because the recording rule's input metric was filtered out in the last cardinality cleanup. Cost-reduction work goes wrong here far more often than it goes wrong at the YAML level.
17. **Native histograms are a 10-50x cardinality cost lever you're probably not using.** Classic histograms emit a series per bucket boundary (`_bucket{le=...}`), so a 20-bucket histogram with 10 label combinations = 200 series; native (sparse-exponential) histograms collapse that to 1 series per label combination by storing the buckets internally as a sparse exponential distribution. The Apr 20 Prometheus doc covered the migration path; on the OTel side the SDK config is `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=DELTA` plus `OTEL_HISTOGRAM_AGGREGATION=base2_exponential_bucket_histogram` (or per-language equivalent), and on the Collector side `prometheusremotewrite.send_native_histograms: true` ships them through. The reason this is worth flagging as a gotcha: most teams roll out OTel with classic histograms (the SDK default in early Java/Python releases), then discover six months later that 60% of their Mimir series count is histogram bucket noise. One config change, then a backend that supports them (Mimir 2.10+, Prometheus 2.40+), and the cardinality bill drops without losing any quantile precision.

---

## Decision Frameworks

### 1. Auto-instrumentation vs Manual

| Situation | Use |
|-----------|-----|
| Net-new service or fleet-wide rollout | **Auto first** -- get HTTP, DB, gRPC coverage on day one |
| Business operations (checkout, payment, fraud check) | **Manual** -- the libraries can't see your business logic |
| Custom transports (proprietary RPC, internal message bus) | **Manual** -- write spans + propagation yourself |
| Go services | **Manual + middleware** (otelhttp, otelgrpc) -- eBPF auto-instrumentation is improving but narrower |
| Production answer | **Both** -- auto for breadth, manual for the spans that matter |

### 2. DaemonSet vs Gateway vs Both vs Sidecar

| Cluster shape | Recommended |
|---------------|-------------|
| Dev / single-node / <10 nodes | Gateway only is fine |
| Production K8s cluster, 20-200 nodes | **Agent + Gateway** -- the canonical answer |
| Multi-tenant cluster, per-team config | Add **sidecar** for the tenants that need their own redaction/auth |
| Tail sampling at scale | Agent + **front load-balancing tier** + tail-sampling Gateway pool |
| Want zero per-node Collector cost | Gateway only, but accept losing node-local enrichment |

### 3. OTel Collector vs Grafana Alloy vs Vendor Distro

| You are | Pick |
|---------|------|
| Vendor-neutral, multi-backend, max component coverage | **OTel Collector Contrib** |
| All-in on Grafana stack (Mimir + Tempo + Loki + Pyroscope) | **Grafana Alloy** |
| All-in on AWS, want IRSA-integrated defaults | **AWS Distro for OpenTelemetry (ADOT)** |
| Want vendor support contracts and curated upgrades | The vendor distro for that vendor (Splunk, Datadog DDOT, Honeycomb, Dynatrace) |

### 4. Head-Based vs Tail-Based Sampling

| Goal | Sampling |
|------|----------|
| Predictable cost, simple architecture | **Head-based** (`parentbased_traceidratio: 0.1`) |
| Catch all errors and slow traces, accept Collector complexity | **Tail-based** at gateway, with sticky routing |
| Best of both | **Conservative head sampling at SDK** (e.g., 100%) **+ tail sampling at gateway** |
| Highest scale | Push tail sampling to backend (Honeycomb, Tempo TraceQL search) |

### 5. SDK Direct Export vs Collector

Default: **always Collector.** The Collector is cheap; SDK-direct is fragile under network partition, has no persistent queue, can't enrich with K8s metadata, and binds your apps to one backend's auth model.

| Situation | Honest answer |
|-----------|---------------|
| Long-running services on K8s/VMs | **Collector.** No exceptions. |
| Lambda / Cloud Run / FaaS with no sidecar surface | SDK-direct to a **regional Gateway Collector** (still a Collector, just not co-located). The Lambda OTel layer and ADOT Lambda layer ship a small in-process exporter, but you front them with a remote gateway -- never the vendor backend. |
| Edge / IoT devices (memory < 64 MB) | SDK-direct, accept the fragility, add a vendor-side ingest gateway with idempotent retry. The Collector's footprint is too large here. |
| Very-low-volume admin tools / cron jobs | SDK-direct is acceptable -- the operational surface of a Collector exceeds the value of the telemetry. Keep an eye out for the day this stops being low-volume. |
| Local dev | SDK-direct to a local OTLP debug exporter or a `docker run` Collector -- whichever is faster to iterate on. |

The rule: **direct-export only when there is genuinely no place for a Collector to live.** Even Lambda has a place for a Collector (the layer); you almost never want the SDK talking directly to a vendor's public ingest.

---

## Why OpenTelemetry Matters Strategically

OpenTelemetry is the first vendor-neutral observability standard with real industry adoption. It is the second-largest CNCF project by activity (after Kubernetes itself). Every major APM vendor accepts OTLP natively. Every cloud provider has an OTel-compatible managed service. Every large engineering org has either migrated or is migrating.

The bet is simple: **instrument once with OTel, swap backends as your needs change.** That bet was speculative in 2021, plausible in 2023, and effectively settled in 2026. The drawer of proprietary chargers is being thrown away. What you get back is twenty years of accumulated instrumentation that doesn't have to be rewritten when the procurement contract changes -- a luxury this industry has never had.

---

## The 10 Commandments of OpenTelemetry

1. **Thou shalt put a Collector between thy SDK and thy backend.** SDK direct export is for laptops, not production.
2. **Thou shalt run agent + gateway on Kubernetes.** Agent for node-local cheapness, gateway for cluster-wide policy.
3. **Thou shalt set `service.name` and the K8s attributes on every signal.** They are the join keys; without them, no cross-signal drill-down works.
4. **Thou shalt use the standard semantic-convention names.** `service.name`, not `app`. `k8s.deployment.name`, not `deployment`. Stable names are non-negotiable.
5. **Thou shalt place `memory_limiter` first and `batch` last in every pipeline.** This is muscle memory; deviation is debugging.
6. **Thou shalt ensure k8sattributes has the RBAC it needs.** `pods`, `namespaces`, and the ownership-chain resources, with `list` and `watch`. Silent failure otherwise.
7. **Thou shalt not deploy tail sampling without sticky routing.** A `loadbalancing` exporter front tier or a single replica -- pick one.
8. **Thou shalt instrument auto first, then add manual spans for business operations.** The auto agent gives you the skeleton; manual fills in the meaning.
9. **Thou shalt scrape the Collector's self-telemetry from somewhere other than that Collector.** Otherwise an outage erases its own evidence.
10. **Thou shalt OTLP everywhere, and never lock thyself into a vendor's wire format.** That is the entire reason this project exists.

---

## Further Reading

- [OpenTelemetry Concepts](https://opentelemetry.io/docs/concepts/) -- the canonical primer
- [OpenTelemetry Collector Architecture](https://opentelemetry.io/docs/collector/architecture/) -- receivers/processors/exporters
- [Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/) -- the agreed attribute names
- [OTel Operator on Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/operator/) -- the auto-injection mechanic
- [Grafana Alloy vs OTel Collector](https://grafana.com/blog/grafana-alloy-opentelemetry-collector-with-prometheus-pipelines/) -- when to pick which
- Repo cross-references: [Prometheus Fundamentals](../prometheus/2026-04-20-prometheus-fundamentals.md), [Grafana Fundamentals](../grafana/2026-04-24-grafana-fundamentals.md), [Alertmanager & Alerting Strategy](../prometheus/2026-04-23-alertmanager-alerting-strategy.md), [Kubernetes Metrics Stack](../prometheus/2026-04-26-kubernetes-metrics-stack.md)
