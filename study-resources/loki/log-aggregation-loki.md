# Log Aggregation with Loki — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-29
**Difficulty:** Intermediate

## Overview

This plan builds on your existing Prometheus, Grafana, and OpenTelemetry knowledge to master Grafana Loki — the label-indexed, object-storage-backed log aggregation system that takes a fundamentally different approach than ELK. By the end you'll understand Loki's architecture (distributors / ingesters / queriers / compactor on top of S3), write efficient LogQL, avoid the cardinality landmines that kill production Loki clusters, and know why Grafana Alloy has replaced Promtail as the canonical 2026 collector.

## Resources

### 1. Loki Architecture & Deployment Modes (Official Docs) ⏱️ 25 min

- **URL:** [Loki Architecture](https://grafana.com/docs/loki/latest/get-started/architecture/)
- **Companion URL:** [Deployment Modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/)
- **Type:** Official Docs
- **Summary:** The canonical source for Loki's read path / write path, the roles of distributor, ingester, querier, query-frontend, compactor, and index-gateway, plus the three deployment modes (monolithic up to ~20 GB/day, Simple Scalable Deployment up to ~1 TB/day with read/write/backend separation, and full microservices). Read these two pages back-to-back — the architecture page explains the components, the deployment-modes page explains how they get packaged in production. This is the foundation everything else builds on.

### 2. Loki Label Best Practices + Cardinality (Official Docs) ⏱️ 20 min

- **URL:** [Label Best Practices](https://grafana.com/docs/loki/latest/get-started/labels/bp-labels/)
- **Companion URL:** [Cardinality](https://grafana.com/docs/loki/latest/get-started/labels/cardinality/)
- **Type:** Official Docs
- **Summary:** The single most important conceptual lesson in all of Loki: "labels are for streams, log line is for searchable content." Covers the bounded-vs-unbounded label rule, the 100k-active-streams-per-tenant guideline, the `requestId`-as-label disaster example, and the `logcli series '{}' --analyze-labels` diagnostic command. If you only remember one thing from today, it's that putting `user_id` / `trace_id` / `request_id` in labels is the #1 way to OOM your ingesters — those go in structured metadata or the log line itself.

### 3. LogQL Log Queries Reference (Official Docs) ⏱️ 25 min

- **URL:** [LogQL Log Queries](https://grafana.com/docs/loki/latest/query/log_queries/)
- **Type:** Official Docs
- **Summary:** The authoritative LogQL reference covering stream selectors (`{app="api"}`), the four line filter operators (`|=`, `!=`, `|~`, `!~`), label filters with comparison operators, every parser (`| json`, `| logfmt`, `| pattern`, `| regexp`, `| unpack`), and formatters (`| line_format`, `| label_format`, `| drop`, `| keep`). Pay special attention to the "place filters early for performance" guidance — line filters before parsers is a 10x query speedup. PromQL knowledge from your April 21 deep dive transfers directly to LogQL's metric query syntax.

### 4. Loki 3.0 Release: Bloom Filters + Native OTel + Structured Metadata (Grafana Blog) ⏱️ 15 min

- **URL:** [Loki 3.0 Release](https://grafana.com/blog/2024/04/09/grafana-loki-3.0-release-all-the-new-features/)
- **Type:** Blog (Official)
- **Summary:** The headline 3.x release announcement covering the three pillars of modern Loki: bloom filters that skip 70-90% of chunks during needle-in-haystack queries, native `/otlp/v1/logs` ingestion that removes the OTel-Collector → Loki-exporter hop, and structured metadata that lets you store high-cardinality fields (trace IDs, request IDs) outside the index. This directly connects to your April 27 OTel work — Loki is now a first-class OTLP backend.

### 5. Loki 3.3: Blooms Pivot to Structured Metadata (Grafana Blog) ⏱️ 10 min

- **URL:** [Loki 3.3 Blooms](https://grafana.com/blog/2024/11/21/grafana-loki-3.3-release-faster-query-results-via-blooms-for-structured-metadata/)
- **Type:** Blog (Official)
- **Summary:** The crucial follow-up to the 3.0 release: Grafana abandoned n-gram blooms over raw log content in favor of blooms over structured-metadata keys and key-value pairs. The new blooms are "orders of magnitude smaller, faster to build, download, and query," and align with how OTel data naturally arrives. Important gotcha: the new bloom format is incompatible with 3.0/3.2 — you must delete existing bloom blocks on upgrade. Read this immediately after #4 so you have the current (not the historical) mental model.

### 6. Migrating from Promtail to Grafana Alloy (Official Docs + 2025 Field Report) ⏱️ 20 min

- **URL:** [Alloy from Promtail](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/)
- **Companion URL:** [Promtail to Alloy 2025 field report](https://developer-friendly.blog/blog/2025/03/17/migration-from-promtail-to-alloy-the-what-the-why-and-the-how/)
- **Type:** Official Docs + Blog
- **Summary:** Promtail entered LTS in Feb 2025 and reaches EOL March 2, 2026 — so as of today (April 2026) Promtail is dead and Alloy is the only supported path. The official guide covers the `alloy convert --source-format=promtail` command and the three-component pattern that replaces a Promtail config: `local.file_match` → `loki.source.file` → `loki.write`. The companion 2025 blog post adds production-grade Helm values, journal scraping, Kubernetes events ingestion, and the metric-name-rename gotcha (your `promtail_*` alerts all break — grep your alerting rules before cutover). This is the convergence story you saw in OTel on April 27, now applied to logs.

### 7. Loki at Scale — Navigating High-Volume Logging Challenges (Video) ⏱️ 25 min

- **URL:** [Loki at Scale video](https://www.youtube.com/watch?v=sDI6h51s8KA)
- **Type:** Video (Conference talk, 2024)
- **Summary:** A practitioner-focused talk on running Loki at multi-TB-per-day scale: stream-explosion war stories, query-frontend caching, the role of the compactor in retention, multi-tenancy via `X-Scope-OrgID`, distributor rate limiting, and how to right-size ingesters so they don't checkpoint themselves to death. Watch this last — once you have the architecture, LogQL, and cardinality fundamentals from #1-#3, this talk's lessons will land much harder than if you watched it cold.

## Study Tips

- **Pair this with your April 28 distributed tracing doc.** Loki's killer feature in Grafana is the trace-to-logs pivot via the `traceID` field in structured metadata. As you read the Loki 3.0 blog, sketch out how a span ID flows from your application → OTel SDK → Tempo for the trace → Loki via OTLP for the logs → Grafana derived field for the click-through. This mental loop is what makes the whole observability stack pay off.
- **Resist the urge to model logs like Prometheus metrics.** Your PromQL muscle memory will tempt you to put everything in labels because that's how Prometheus selects series. In Loki, labels select *streams*, and a stream is a physical chunk file on S3 — every new label value materializes new files. The stream cap is ~100k per tenant; Prometheus routinely runs millions of series. Different ceiling, different rules.
- **Run `logcli series '{}' --analyze-labels --since=1h` against any Loki cluster you touch.** It prints a sorted list of which labels have which cardinality — instant detection of the "someone shoved `pod_name` or `request_id` into a label" anti-pattern. Make it the first command you run during any Loki incident.

## Next Steps

- **Day 4 of Observability week** is likely AWS-native logging (CloudWatch Logs Insights, log subscription filters to Kinesis/Firehose, OpenSearch Service) — the managed-service contrast to today's self-hosted Loki story will sharpen your "build vs buy" intuition.
- **Hands-on lab:** Deploy `loki-stack` or the `grafana/loki` Helm chart in SSD mode against MinIO or S3, point Alloy at it via `loki.write`, then deliberately blow up an ingester by pushing logs with `request_id` as a label. Recover by moving `request_id` to structured metadata. Nothing teaches cardinality like watching an OOM in real time.
- **Deeper dives worth bookmarking:** the Loki `operations/` docs section (specifically `bloom-filters/`, `query-frontend/`, and `multi-tenancy/`) and the Alloy `reference/components/loki/` pages for production tuning. Save these for when you actually operate a cluster, not for today.
