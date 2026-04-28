# Distributed Tracing Deep Dive — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-28
**Difficulty:** Advanced (assumes OpenTelemetry Fundamentals from 2026-04-27 and Prometheus/Grafana fluency)

## Overview

Yesterday's OpenTelemetry Fundamentals doc covered the API/SDK/Collector split, OTLP, receivers/processors/exporters, K8s agent+gateway patterns, and introduced head vs tail sampling. Today goes deep on the trace signal itself: span semantics (kinds, status, events, links), the wire-level mechanics of W3C Trace Context propagation, the math and operational tradeoffs of head vs tail sampling, span processor back-pressure, Tempo's storage and TraceQL query model, the spanmetrics connector for trace-to-metrics correlation via exemplars, and the production failure modes (clock skew, incomplete traces, ingestion cost) that show up in interviews and post-mortems.

Resources are sequenced so you build the data model first (span anatomy -> context propagation), then move into sampling strategy, then storage and query (Tempo + TraceQL), then trace<->metrics correlation, and finish with backend comparison and production gotchas.

## Resources

### 1. OpenTelemetry — Traces Concept Page (Spans, Kind, Status, Events, Links) ⏱️ 15 min  [MUST-READ]
- **URL:** https://opentelemetry.io/docs/concepts/signals/traces/
- **Type:** Official Docs
- **Summary:** The canonical reference for span anatomy. Focus on the five span kinds (SERVER, CLIENT, PRODUCER, CONSUMER, INTERNAL) and how backends use CLIENT/SERVER pairs to build service dependency graphs — getting kind wrong silently breaks your service map. Pay attention to span status (`Unset` / `Ok` / `Error`), span events (timestamped sub-points within a span, e.g. `exception` events with stack traces), and span links (cross-trace references for batch jobs and fan-in scenarios). This is the data-model foundation; every later resource assumes you internalized this vocabulary.

### 2. W3C Trace Context — Specification (HTTP Header Format Section Only) ⏱️ 15 min  [MUST-READ]
- **URL:** https://www.w3.org/TR/trace-context/
- **Type:** Official W3C Specification
- **Summary:** The wire-level standard every modern tracer speaks. Skip the introduction; read sections 3 (`traceparent` header) and 4 (`tracestate`) carefully. Memorize the `traceparent` shape: `version-traceid-parentid-flags` (`00-<32 hex>-<16 hex>-<2 hex>`) and what `flags=01` means (sampled). Understand why `tracestate` is a vendor-multiplexed sidecar capped at 32 entries with most-recent-first ordering. This is what gets you through the interview question "what's actually in the HTTP header that propagates a trace?" — you should be able to reconstruct the format from memory.

### 3. Honeycomb — Sampling Docs (Strategy + Refinery Overview) ⏱️ 20 min  [MUST-READ]
- **URL:** https://docs.honeycomb.io/manage-data-volume/sample/
- **Type:** Vendor Docs (vendor-neutral framing)
- **Summary:** The most practitioner-oriented sampling explainer on the open web. Honeycomb's Refinery is a tail-sampling proxy and the docs walk through dynamic sampling (auto-rate by key cardinality), deterministic sampling (consistent decision across services from `traceID` hash), and the "errors and slow requests at 100%, baseline at 1%" pattern every senior engineer is expected to articulate. Focus on the decision framework: when head sampling is sufficient, when you need tail sampling, and the operational cost of each. Pairs directly with the OTel `tail_sampling` processor you already saw in yesterday's doc.

### 4. OpenTelemetry Blog — Tail Sampling: Why, How, and What to Consider ⏱️ 15 min  [MUST-READ]
- **URL:** https://opentelemetry.io/blog/2022/tail-sampling/
- **Type:** Official OTel Blog
- **Summary:** The definitive write-up of the tail-sampling architecture in OTel Collector. Covers why tail sampling requires the `loadbalancing` exporter (all spans of a trace must reach the same Collector replica for the decision), the memory cost of buffering spans during the decision window, and the policy primitives (`status_code`, `latency`, `string_attribute`, `probabilistic`, `and`, `composite`). Read with yesterday's tail_sampling section open as a side-by-side — this fills in the implementation details that the fundamentals doc deliberately deferred.

### 5. Tail Sampling Processor — README on GitHub ⏱️ 10 min
- **URL:** https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md
- **Type:** Official GitHub README
- **Summary:** The reference for every policy type with config snippets. Skim the policy table (`always_sample`, `rate_limiting`, `latency`, `numeric_attribute`, `boolean_attribute`, `ottl_condition`, `composite`) and study the composite-policy example showing how to AND/OR them — this is what real production configs look like. Bookmark for when you build your own pipeline; the policy ordering and `decision_wait` tuning notes matter operationally.

### 6. Grafana Tempo — TraceQL Documentation (Language + Query Editor) ⏱️ 25 min  [MUST-READ]
- **URL:** https://grafana.com/docs/tempo/latest/traceql/
- **Type:** Official Docs
- **Summary:** TraceQL is to Tempo what PromQL is to Prometheus — and Grafana modeled the syntax intentionally. Read the language reference: span selectors (`{ .http.status_code = 500 }`), structural operators (`>>` descendant, `>` child, `&&`/`||` siblings within a trace), aggregates (`count()`, `avg()`, `max()` over span attribute values), and intrinsics (`name`, `kind`, `status`, `duration`, `rootName`). Practice writing queries like `{ resource.service.name = "checkout" && .http.status_code >= 500 } | count() > 3` mentally. This is THE query language you will use in Grafana to drill from a Prometheus alert into the offending traces, and interview-relevant for any role with Tempo in the stack.

### 7. Grafana Labs Blog — TraceQL: A Powerful Query Language for Distributed Tracing ⏱️ 10 min
- **URL:** https://grafana.com/blog/2023/02/07/get-to-know-traceql-a-powerful-new-query-language-for-distributed-tracing/
- **Type:** Vendor Blog (Grafana Labs engineering)
- **Summary:** The narrative companion to the TraceQL reference docs — explains the *why* of structural queries ("show me traces where the auth service called the user service which then errored") that you simply can't express in tag-based search systems like classic Jaeger. Read after the docs to consolidate intent vs syntax. Skim if short on time; this is "nice-to-have" but the structural-query motivation is the strongest argument for Tempo over Jaeger in 2026.

### 8. Span Metrics Connector — README on GitHub ⏱️ 10 min  [MUST-READ]
- **URL:** https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/connector/spanmetricsconnector/README.md
- **Type:** Official GitHub README
- **Summary:** The connector that makes the trace<->metrics correlation story real. It consumes spans from a traces pipeline and emits RED metrics (`calls_total`, `duration_seconds_bucket`) into a metrics pipeline, with exemplars pointing back to trace IDs. This is what powers "click the latency spike on a Grafana panel and jump to a representative trace" — and it's how you cheaply get service-graph metrics without instrumenting them by hand. Focus on the `dimensions` config (which span attributes become metric labels — beware cardinality explosion) and the `exemplars.enabled` flag. Pairs with yesterday's "connector = jumper cable between pipelines" analogy.

## Study Tips

- Before starting, re-skim yesterday's `2026-04-27-opentelemetry-fundamentals.md` Sampling section and "OpenTelemetry vs Vendor Agents" decision framework. Today's content slots directly into those gaps — knowing where it fits prevents the "everything is new" feeling.
- For TraceQL, open Grafana Play (https://play.grafana.org/) in another tab and run two or three queries against the public Tempo data source. Reading the syntax is half-effective; running a real query against real data and seeing the trace waterfall light up is what makes it stick.
- The W3C Trace Context spec is short — actually open Chrome DevTools on any modern site (Stripe, Shopify, AWS console) and inspect the `traceparent` header on outgoing requests. Seeing one in the wild for the first time is a small but durable "ah, this is real" moment.
- Sampling is the topic interviewers love because it has no single right answer. Practice articulating the tradeoff out loud: "head sampling is cheap and stateless but loses errors; tail sampling catches errors but needs sticky routing via the loadbalancing exporter and N seconds of span buffering — pick based on whether your error budget can tolerate missed traces during incidents."

## Next Steps

After this plan you will have the trace-side mental model to match yesterday's collector-side model. Natural follow-ups in the syllabus:
- **Grafana Tempo deep dive** — storage internals (object-storage-backed columnar blocks), the metrics-generator subsystem (Tempo's built-in spanmetrics + service-graph), and TraceQL Metrics for ad-hoc trace analytics.
- **Trace-based testing / SLO from traces** — using traces as the source of truth for latency SLOs instead of pre-aggregated histograms.
- **AWS X-Ray vs OTel** — when ADOT (AWS Distro for OpenTelemetry) makes sense, X-Ray's sampling rules engine, and the OTLP-to-X-Ray exporter path.
- **eBPF-based tracing** (Beyla, Pixie, Coroot) — zero-instrumentation tracing alternatives and where they complement vs replace OTel SDK instrumentation.
