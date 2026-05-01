# Unified Observability — Correlating Signals — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-30
**Difficulty:** Intermediate / Advanced

## Overview

This plan synthesizes a week of individual signal study (Prometheus, Loki, Tempo, OTel, Grafana, Alloy) into the mechanics of pivoting **between** signals: metrics -> traces (exemplars), traces -> logs (`tracesToLogsV2`), logs -> traces (`derivedFields`), and logs -> metrics (LogQL aggregations + recording rules). The focus is the join keys that make correlation work — `trace_id` for per-request pivots and OTel resource attributes (`service.name`, `k8s.*`) for service-level pivots — plus the unified collector (Alloy) that emits all three signals with consistent labels. After completing this plan you should be able to design a Grafana stack where any starting signal links to the other two without copy-pasting IDs across tabs.

Assumes prior mastery of: Prometheus + PromQL, Grafana datasources, OpenTelemetry SDK/Collector, Tempo, Loki 3.x with structured metadata, and Alertmanager. We are not re-teaching any of those individually here.

## Resources

### 1. Honeycomb — "It's Time to Version Observability: Introducing Observability 2.0" ⏱️ 12 min
- **URL:** https://www.honeycomb.io/blog/time-to-version-observability-signs-point-to-yes
- **Type:** Blog (philosophy / framing)
- **Summary:** Sets the conceptual stage for *why* correlation is the hard part of observability. Charity Majors argues the "three pillars" model is broken because each pillar is stored separately with no connective tissue, forcing engineers to copy-paste IDs between tools. Read this first to understand what problem the rest of today's resources are solving — the goal is not "have all three signals" but "be able to pivot between them in one click."
- **Pivot direction covered:** All four (motivational framing).

### 2. OpenTelemetry — Semantic Conventions: Resource Attributes ⏱️ 15 min
- **URL:** https://opentelemetry.io/docs/specs/semconv/resource/
- **Type:** Official Docs
- **Summary:** The universal join key story. Skim to confirm `service.name`, `service.namespace`, `service.instance.id`, `deployment.environment.name`, and the `k8s.*` family. The single most important takeaway: if you emit metrics with `service.name=cart-api` but logs with `app=cart` and traces with `service=cart-svc`, you cannot join them at the service level even though the data is "all there." Resource attributes are the only durable cross-signal correlation strategy beyond per-request `trace_id`.
- **Pivot direction covered:** Foundational — service-level joins for all four pivots.

### 3. Prometheus — Feature Flags + Storage (Exemplars sections) ⏱️ 10 min
- **URL:** https://prometheus.io/docs/prometheus/latest/feature_flags/ and https://prometheus.io/docs/prometheus/latest/storage/
- **Type:** Official Docs
- **Summary:** The minimal Prometheus-side requirements for metrics->traces. `--enable-feature=exemplar-storage` is required, exemplars live in a fixed-size in-memory circular buffer that *also* gets written to the WAL for restart durability, and only OpenMetrics format (not classic Prometheus exposition) carries the `# {trace_id="..."} value timestamp` syntax. Important production sizing detail: exemplar buffer is sized in number-of-exemplars, not bytes — get this wrong and you silently drop the most interesting traces.
- **Pivot direction covered:** Metrics -> Traces (storage layer).

### 4. Grafana — Introduction to Exemplars ⏱️ 12 min
- **URL:** https://grafana.com/docs/grafana/latest/fundamentals/exemplars/
- **Type:** Official Docs
- **Summary:** The visualization side of metrics->traces. Exemplars render as highlighted stars on time-series panels; hover reveals the `traceID`, click "Query with Tempo" opens the trace in a split pane. Pair this with the Prometheus exemplars page to close the loop end-to-end: SDK emits histogram observation with active span -> OpenMetrics scrape carries trace_id -> Prom stores in exemplar WAL -> Grafana panel queries `query_range` with `exemplar=true` -> star renders -> click pivots to Tempo datasource.
- **Pivot direction covered:** Metrics -> Traces (UI + click-through).

### 5. Grafana — Configure Trace to Logs (tracesToLogsV2) ⏱️ 18 min
- **URL:** https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/configure-trace-to-logs/
- **Type:** Official Docs
- **Summary:** The Tempo-side configuration for traces -> logs. Memorize the field roles: `datasourceUid` (which Loki to query), `tags` (which span attributes get mapped to log labels — defaults `service.name`, `namespace`, `pod`), `filterByTraceID` (constrain to whole trace), `filterBySpanID` (single span only — needs span_id in logs), `customQuery` (override with LogQL using `${__tags}` and `${__trace.traceId}` variables), and `spanStartTimeShift`/`spanEndTimeShift` (typically `-2s`/`+2s` to handle clock drift between log write and span boundary). Misconfigure `tags` and you get either zero log results or every log in the namespace.
- **Pivot direction covered:** Traces -> Logs.

### 6. Grafana Loki — Structured Metadata + OTLP Ingestion ⏱️ 15 min
- **URL:** https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/ and https://grafana.com/docs/loki/latest/send-data/otel/
- **Type:** Official Docs
- **Summary:** The Loki 3.0+ mechanism that makes trace_id queryable without exploding stream cardinality. Structured metadata is per-line key/value attached at ingest, indexed for fast filtering, but **not** part of the stream identity (so high-cardinality fields like `trace_id` and `user_id` no longer wreck your label budget). When OTLP logs arrive at Loki's `/otlp/v1/logs` endpoint, OTel resource attributes map to stream labels and OTel log-record attributes map to structured metadata. This is the architectural piece that finally makes per-trace log lookup a first-class operation in Loki rather than a regex-on-the-log-body hack. Note the `allow_structured_metadata: true` config requirement.
- **Pivot direction covered:** Traces -> Logs and Logs -> Traces (the trace_id storage substrate for both directions).

### 7. Grafana Labs Blog — From Agent to Alloy: Why we transitioned ⏱️ 15 min
- **URL:** https://grafana.com/blog/2024/04/09/grafana-agent-to-grafana-alloy-opentelemetry-collector-faq/
- **Type:** Official Blog
- **Summary:** Why "one collector for all three signals" matters for correlation. Alloy is an OTel Collector distribution with native Prometheus pipelines built in — it replaces Promtail (EOL Feb 28, 2026), Grafana Agent (EOL Nov 1, 2025), and standalone OTel Collector deployments with a single binary using `.alloy` syntax (the successor to Flow's River). The correlation angle: when one agent emits metrics, logs, and traces, it can attach **identical** resource attributes to all three signals automatically — eliminating the "logs say `app=cart`, traces say `service.name=cart-api`" mismatch that breaks join queries. Skim the migration tooling section (`alloy convert --source-format=promtail`).
- **Pivot direction covered:** Infrastructure that makes all four pivots actually work in practice.

### 8. Grafana Blog — OpenTelemetry log correlation with traces (signal correlation) ⏱️ 13 min
- **URL:** https://opentelemetry.io/docs/concepts/signals/logs/
- **Type:** Official Docs
- **Summary:** How `trace_id`/`span_id` end up on log records in the first place. Log appenders/bridges (logback for Java, structlog for Python, winston for Node, OTel logging for .NET) hook into the active span context and inject `trace_id`/`span_id` as top-level fields. Critical maturity caveat: Java/.NET/C++/PHP logging is **stable**, Go/Rust are **beta**, Python/Ruby/JS are **development** — for unstable languages, you may still need to inject trace IDs via a logger formatter rather than relying on the OTel logs SDK. Also note: head-sampling decisions made in the Collector affect *traces* but NOT *logs*, so a sampled-out trace will still produce orphan log lines with a trace_id that resolves to nothing — a confusing correlation gotcha.
- **Pivot direction covered:** Traces -> Logs (the SDK side that produces the data).

### 9. (If time permits) Medium — Using Prometheus Exemplars to jump from metrics to traces in Grafana ⏱️ 10 min
- **URL:** https://vbehar.medium.com/using-prometheus-exemplars-to-jump-from-metrics-to-traces-in-grafana-249e721d4192
- **Type:** Walkthrough article (publicly accessible, not paywalled)
- **Summary:** A concrete end-to-end walkthrough: instrument a Go app to emit histogram exemplars with the active span's trace_id, configure Prometheus scrape with `honor_labels` and OpenMetrics `Accept` header, configure the Grafana Prometheus datasource exemplars block with the Tempo datasourceUid. Useful as a "wire it together" reference after the conceptual reading. Skip if you are tight on time and have already done a hands-on exemplar setup.
- **Pivot direction covered:** Metrics -> Traces (practical glue).

## Total: ~110 min core + 10 min optional = 120 min

## Coverage Map

| Pivot direction | Resources |
|---|---|
| Metrics -> Traces (exemplars) | #3 Prometheus storage, #4 Grafana exemplars, #9 walkthrough |
| Traces -> Logs (tracesToLogsV2) | #5 Tempo trace-to-logs, #6 Loki structured metadata, #8 OTel logs SDK |
| Logs -> Traces (derivedFields) | #6 Loki structured metadata (trace_id as queryable field) |
| Logs -> Metrics (LogQL aggregations) | Implicit in prior Loki/PromQL docs — no new resource needed today |
| Service-level joins (resource attributes) | #2 OTel semconv, #7 Alloy unified emission |
| Why bother (philosophy) | #1 Honeycomb Observability 2.0 |

## Study Tips

- **Read #1 and #2 first as a pair.** The Honeycomb piece tells you *why* you need a join key; the OTel semconv page tells you *what* the join keys are. Everything after that is mechanics.
- **Build a mental wiring diagram as you go.** Sketch boxes for App SDK / Alloy / Prometheus / Loki / Tempo / Grafana, and draw the trace_id flow on it. By the end you should be able to point at any edge and name the configuration knob (exemplar storage, OTLP endpoint, structured metadata, tracesToLogsV2, derivedFields).
- **Anti-pattern watch list — note these as you read.** (a) Different `service.name` values across signals — breaks every service-level pivot. (b) Exemplars enabled but no Tempo configured — stars render but click does nothing. (c) `derivedFields` regex without anchors (`^...$`) — Grafana panels hang when log lines are long. (d) Head-sampled-out traces produce orphan logs with dangling trace_ids. (e) Dashboards that visualize but never link out — every panel should have a "where do I go from here" path.
- **Format compatibility gotcha.** W3C trace IDs are 32 hex chars (128-bit); legacy Jaeger uses 16 hex chars (64-bit). If you have a mixed fleet, log injection and Tempo lookup may mismatch. Verify your SDK is configured for W3C.

## Next Steps

After this plan, the natural follow-ons are:

- **Tail-based sampling in the OTel Collector** — head sampling is cheap but loses the interesting traces; tail sampling preserves errors/slow requests but adds memory cost. Worth a dedicated session.
- **SLO-driven alerting from logs and traces** — using Loki recording rules and Tempo metrics-generator to derive RED metrics from raw signals, closing the loop back to Alertmanager.
- **Grafana Correlations feature (Explore-level, not datasource-level)** — the newer cross-datasource linking that goes beyond `tracesToLogsV2` and lets you define arbitrary pivots (e.g., Loki -> Postgres, Prom -> CloudWatch). Worth a follow-up day once the basics here feel automatic.
- **Hands-on lab:** stand up a local docker-compose with Alloy, Mimir/Prom, Loki 3.x, Tempo, and Grafana; instrument a sample app with OTel SDK; verify all four pivot directions work end-to-end. This is the only way to internalize the configuration shape.
