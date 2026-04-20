# Prometheus Fundamentals — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-20
**Difficulty:** Beginner / Intermediate

## Overview

This plan covers the foundations of Prometheus as a monitoring system: its scrape-based pull architecture, the four metric types, instrumentation and exposition format, scrape configs with Kubernetes service discovery, relabeling, local TSDB storage, remote read/write, federation, and when (and when not) to use the Pushgateway. By the end you should be able to reason about what Prometheus is designed for, how metrics flow from an app through a scrape target into the TSDB, and how labels are shaped by relabel rules. PromQL, alerting, and kube-prometheus-stack are intentionally deferred to later days in the week.

## Resources

### 1. Prometheus Overview — What It Is and Isn't (Official) ⏱️ 10 min
- **URL:** https://prometheus.io/docs/introduction/overview/
- **Type:** Official Docs
- **Summary:** The canonical "what is Prometheus" page. Establishes the ecosystem diagram (server, client libs, exporters, Pushgateway, Alertmanager), the pull-based architecture, and — critically — where Prometheus is NOT a good fit (per-request billing, 100% accuracy use cases). Sets the right mental model before anything else.

### 2. Data Model: Metric Names, Labels, Samples (Official) ⏱️ 10 min
- **URL:** https://prometheus.io/docs/concepts/data_model/
- **Type:** Official Docs
- **Summary:** The dimensional data model is the single most important concept in Prometheus. This page defines time series as `metric_name{label="value"} => (timestamp, value)` and explains why "every unique label combination is a new time series" — the cardinality rule that governs all downstream design decisions.

### 3. Metric Types: Counter, Gauge, Histogram, Summary (Official) ⏱️ 15 min
- **URL:** https://prometheus.io/docs/concepts/metric_types/
- **Type:** Official Docs
- **Summary:** Defines the four core metric types and their semantics. Focus on understanding why counters are monotonic (and why `rate()` exists), when a gauge is the wrong choice, and the trade-offs between histograms (aggregable across instances) and summaries (precise but non-aggregable).

### 4. How Does a Prometheus Counter Work? (Robust Perception / Brian Brazil) ⏱️ 10 min
- **URL:** https://www.robustperception.io/how-does-a-prometheus-counter-work/
- **Type:** Blog (co-founder of Prometheus)
- **Summary:** Short, dense blog post by Brian Brazil explaining why counters never decrement, how `rate()` handles resets and scrape failures, and why this design is more resilient than the "calculate delta client-side and push" model used by other systems. Pairs naturally with resource #3 and seeds the mental model for PromQL tomorrow.

### 5. Why is Prometheus Pull-Based? + When to Use the Pushgateway (Official) ⏱️ 15 min
- **URL:** https://prometheus.io/docs/practices/pushing/
- **Type:** Official Docs / Best Practices
- **Summary:** The definitive answer to "pull vs push" and the narrow valid use case for the Pushgateway (service-level batch jobs only). Covers the three big Pushgateway anti-patterns: single point of failure, loss of the `up` health metric, and stale series that "never get forgotten." Essential for interviews — this is a classic gotcha question.

### 6. Life of a Label — Relabeling Flow (Robust Perception / Brian Brazil) ⏱️ 15 min
- **URL:** https://www.robustperception.io/life-of-a-label/
- **Type:** Blog (Prometheus co-founder)
- **Summary:** Walks through the two-phase label lifecycle: service discovery produces `__meta_*` labels which `relabel_configs` transforms into target labels, then scraped samples flow through `metric_relabel_configs`. Short but shows the canonical flowchart that makes the entire relabeling mental model click.

### 7. How Relabeling in Prometheus Works (Grafana Labs Blog) ⏱️ 20 min
- **URL:** https://grafana.com/blog/2022/03/21/how-relabeling-in-prometheus-works/
- **Type:** Blog (high-signal community)
- **Summary:** Practical deep dive into the seven relabel actions (keep, drop, replace, labelkeep, labeldrop, labelmap, hashmod), the three stages where relabeling applies (`relabel_configs`, `metric_relabel_configs`, `write_relabel_configs`), and worked examples for Kubernetes service discovery. This is the page you will want bookmarked when writing real scrape configs.

### 8. Storage: Local TSDB, Retention, Remote Read/Write (Official) ⏱️ 15 min
- **URL:** https://prometheus.io/docs/prometheus/latest/storage/
- **Type:** Official Docs
- **Summary:** How Prometheus stores data on disk: 2-hour block layout, WAL for crash recovery, time- vs size-based retention (15-day default), and the remote_write/remote_read protocol for long-term storage backends like Thanos, Cortex, Mimir, or VictoriaMetrics. Sets up the "why you need federation or remote write for scale" motivation.

### 9. Federation — Hierarchical and Cross-Service (Official) ⏱️ 10 min
- **URL:** https://prometheus.io/docs/prometheus/latest/federation/
- **Type:** Official Docs
- **Summary:** How multiple Prometheus servers compose: hierarchical (global server scrapes aggregated metrics from per-DC/per-cluster servers) and cross-service (one service's Prometheus pulls selected metrics from another's). Short page, but the two patterns are worth knowing by name.

## Study Tips

- **Draw the data flow yourself.** Sketch: app `/metrics` endpoint -> Prometheus scrape -> relabel_configs -> TSDB block -> optional remote_write. If you can draw it from memory, you understand it. Add the Pushgateway as a dotted-line exception, not a first-class component.
- **Obsess over cardinality early.** Every time you see a label in the docs, ask "what's the worst-case value of this label?" User IDs, request paths, and trace IDs are the classic mistakes. This instinct will save you from production blowups later.
- **Keep PromQL out of today's session.** You will be tempted — resist. You need the data model and metric-type semantics cemented first, or PromQL will feel like arbitrary function memorization tomorrow instead of natural consequences of the model.

## Next Steps

Tomorrow is **PromQL Deep Dive** — `rate()`, `irate()`, `increase()`, aggregation operators, `histogram_quantile()`, and subqueries. Then Alertmanager (routing, grouping, inhibition), Grafana fundamentals, and finally kube-prometheus-stack on Friday to tie it all together on Kubernetes. After that, consider exploring Thanos or Mimir for long-term storage, and the OpenMetrics / OpenTelemetry metric convergence story.
