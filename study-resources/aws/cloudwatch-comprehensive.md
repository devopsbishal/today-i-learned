# AWS CloudWatch Comprehensive -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-05-12
**Difficulty:** Intermediate-to-Advanced

## Overview

CloudWatch in 2026 is no longer just "AWS's metrics dashboard" -- it is a sprawling observability suite spanning metrics (with native PromQL and OpenTelemetry ingestion), logs (now split into Standard and Infrequent Access classes), alarms (metric, composite, anomaly detection), synthetic monitoring (Synthetics canaries), real user monitoring (RUM), container/Lambda/application insights, and a unified trace+metric+log view through ServiceLens and Application Signals. This deep dive focuses on the CloudWatch-specific patterns that matter for production AWS operations -- especially **EMF (Embedded Metric Format)** as the modern way to emit custom metrics, **Logs Insights query syntax** as the biggest day-to-day skill, and the **cost gotchas** that turn a forgotten dimension into a five-figure monthly bill.

## Prerequisites (Already Covered -- Do Not Re-Read)

This is Day 1 of Phase 2.8 (AWS Operations gap-fill) and builds on a very strong observability foundation. The plan assumes you already know:
- **Prometheus + Grafana** (Apr 20-25): pull model, four metric types (counter/gauge/histogram/summary), PromQL, Alertmanager, kube-prometheus-stack -- relevant for the CloudWatch-vs-Prometheus decision framework, do not re-cover metric type basics
- **OpenTelemetry + Distributed Tracing + Loki + SLOs** (Apr 27 - May 1): full observability week including OTel collector pipelines, instrumentation, traces/logs/metrics correlation, error budgets -- today reframes CloudWatch as one possible OTel backend rather than introducing observability concepts
- **EKS** (Dec 2025, 9 entries) and **ECS** (Feb 16-17): cluster anatomy, pod/task lifecycle, control plane -- relevant for Container Insights without re-explaining Kubernetes basics
- **Lambda Deep Dive** (Apr 6) and **API Gateway** (Apr 7): execution model, cold starts, logging behavior -- relevant for Lambda Insights and EMF from Lambda
- **CloudTrail** is TOMORROW (2026-05-13) -- this plan deliberately avoids audit-logging content; CloudWatch Logs is your monitoring/operational log store, CloudTrail is your audit/compliance event store

Frame CloudWatch resources as "which CloudWatch patterns matter when you already understand observability deeply," not "what is monitoring."

## Reading Plan (120 min, time-boxed)

### Block 1: Foundations & Modern Metric Patterns (30 min)

#### 1. CloudWatch Concepts -- AWS Official Docs (10 min)
- **URL:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html
- **Why:** The canonical vocabulary page -- namespaces, metrics, dimensions, statistics, periods, percentiles, resolution (standard 60s vs high-resolution 1s), basic vs detailed monitoring (1-min vs 5-min default for EC2). **Extract specifically:** the precise definition of a "metric" as the unique combination of `namespace + name + dimensions` -- this single concept is the root cause of every cost surprise you will read about later. Skim sections you already know from Prometheus (aggregation, percentiles) and pause on the AWS-specific bits: dimension cardinality math, 15-month metric retention with auto-rollup (1s -> 1m at 3h, -> 5m at 15d, -> 1h at 63d).

#### 2. Embedding Metrics Within Logs (EMF) -- AWS Official Docs (10 min)
- **URL:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format.html
- **Why:** EMF is the single most important "modern CloudWatch" pattern. Instead of calling `PutMetricData` synchronously (latency + API cost + cardinality blindness), you write a structured JSON log line and CloudWatch auto-extracts metrics from it. **Extract:** the `_aws.CloudWatchMetrics` envelope structure, why this is async (no extra latency on your Lambda/container), the cardinality warning at the bottom of the page (high-cardinality dimensions like `requestId` will mint thousands of unique metrics), and the IAM trick -- you only need `logs:PutLogEvents`, not `cloudwatch:PutMetricData`. Skim the linked specification page only if you want exact field-level detail.

#### 3. Lowering Costs with EMF -- AWS Cloud Operations Blog (FireEye Case Study) (10 min)
- **URL:** https://aws.amazon.com/blogs/mt/lowering-costs-and-focusing-on-our-customers-with-amazon-cloudwatch-embedded-custom-metrics/
- **Why:** The "why EMF exists" story. FireEye cut metric collection costs by 65% by switching from `PutMetricData` to EMF, and the blog shows the architectural reasoning (one log line = one metric publish + one log event for forensics, instead of two separate API calls). Older post (2020) but still the canonical narrative. **Extract:** the cost-arithmetic mental model -- when you emit metrics via logs, you pay ingestion ($0.50/GB Standard, $0.25/GB IA) + custom metric fees, not per-API-call PutMetricData costs. This frames the cost optimization block at the end of the session.

### Block 2: Logs Insights & Log Operations (30 min)

#### 4. CloudWatch Logs Insights Query Syntax -- AWS Official Docs (15 min)
- **URL:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- **Why:** This is the **highest-leverage day-to-day CloudWatch skill** -- every production incident, every cost investigation, every "what just happened" question goes through Logs Insights. **Extract:** the pipeline model (`fields | filter | parse | stats | sort | limit` -- pipe-separated commands left-to-right), the four commands you will use 90% of the time (`fields`, `filter`, `parse`, `stats`), the auto-discovered `@` fields (`@timestamp`, `@message`, `@logStream`, `@requestId` for Lambda), the cost rule that filters should come FIRST in the pipeline (Logs Insights bills on scanned bytes, $0.005/GB), and which commands are NOT supported in Infrequent Access log groups (`pattern`, `diff`, `unmask`). Drill into the `stats` and `parse` subpages if time allows -- `parse @message "* * *" as a, b, c` and `stats count() by bin(5m)` cover most queries.

#### 5. CloudWatch Logs Infrequent Access Log Class -- AWS News Blog (8 min)
- **URL:** https://aws.amazon.com/blogs/aws/new-amazon-cloudwatch-log-class-for-infrequent-access-logs-at-a-reduced-price/
- **Why:** Launched November 2023, this is the **biggest CloudWatch cost lever introduced in the last three years**. **Extract:** IA gives 50% lower ingestion ($0.25/GB vs $0.50/GB Standard), storage costs are identical, Logs Insights still works (minus `pattern`/`diff`/`unmask`), and -- critically -- the log class is set at log-group creation and **cannot be changed afterwards**. What you lose: Live Tail, metric filters/extraction, alarming-from-logs, subscription filters, data protection, Contributor/Container/Lambda Insights enrichment. Ideal targets: vended logs (API Gateway access logs, VPC Flow Logs, ELB logs), verbose application debug logs, compliance archive logs. Pair this with retention tier knowledge: keep hot ops logs in Standard with 7-30d retention, dump verbose/compliance logs into IA with 90d-7yr retention.

#### 6. SigNoz CloudWatch Cost Optimization Playbook (7 min)
- **URL:** https://signoz.io/guides/cloudwatch-cost-optimization/
- **Why:** Concrete tactical checklist updated 2026 covering the gotchas that bite production teams. **Extract:** the retention strategy (7d debug / 30d app / 90d audit as a starting heuristic), the dimension cardinality rule of thumb (~50 unique values per dimension), filtering at source (CloudWatch agent or Fluent Bit `grep`/`exclude` filters) before logs ever hit ingestion, cleaning up `INSUFFICIENT_DATA` orphan alarms ($0.10/alarm/month adds up), and the anti-pattern of using anomaly detection alarms where a static threshold would do (anomaly detection alarms cost 3x more than standard alarms).

### Block 3: Alarms, Insights Suites & Synthetics/RUM (35 min)

#### 7. CloudWatch Alarms Best Practices -- AWS Observability Best Practices (10 min)
- **URL:** https://aws-observability.github.io/observability-best-practices/tools/alarms/
- **Why:** The official AWS observability working group's distilled guidance covering metric alarms, **composite alarms**, and **anomaly detection alarms** in one place. **Extract:** when to use composite alarms (combine multiple metric alarms with AND/OR/NOT boolean logic to suppress noise -- e.g., only page when "high error rate AND traffic is normal" so a deploy that drops traffic to zero doesn't trigger), when anomaly detection wins (metrics with strong daily/weekly seasonality and no fixed threshold -- think traffic, login counts, queue depth in a business-hours app), the two-week training data minimum, and the pattern of **pairing anomaly detection (for "weird") with composite alarms (for "actually-paging-worthy")**. Skim if you already know Prometheus alerting -- focus on the composite-alarm Reason Suppression feature, which has no direct PromQL/Alertmanager equivalent.

#### 8. Container Insights with Enhanced Observability for EKS -- AWS Cloud Ops Blog (10 min)
- **URL:** https://aws.amazon.com/blogs/mt/new-container-insights-with-enhanced-observability-for-amazon-eks/
- **Why:** The Nov-2023 launch that made Container Insights actually competitive with Prometheus/kube-state-metrics for EKS. **Extract:** what "enhanced observability" adds over the original Container Insights -- control-plane component metrics (API server, etcd), per-pod/per-container/Kube-State granularity, the bundled unified pricing model (you pay for storage + ingestion, not per-metric), and the EKS add-on installation path (CloudWatch agent + Fluent Bit DaemonSet). Frame your reading around the **decision question**: given that the user runs kube-prometheus-stack happily on EKS, when is Container Insights actually better? Answer: when AWS-native dashboards, cross-account observability, and zero-maintenance trump PromQL flexibility, and when the team can't afford Prometheus operational overhead. Skim the dashboard screenshots; focus on the architecture diagram and the pricing model section.

#### 9. Synthetics Canaries + RUM End-to-End -- Simon Ireilly Blog (10 min)
- **URL:** https://blog.simonireilly.com/posts/cloudwatch-rum-end-to-end-monitoring/
- **Why:** The single best community walkthrough that pairs **Synthetics** (server-side golden-path probing) with **RUM** (browser-side real-user telemetry) into a coherent end-to-end monitoring story. **Extract:** the conceptual split -- Synthetics canaries are scheduled headless-browser scripts (Puppeteer/Playwright) testing critical user journeys (login -> checkout -> success page) and alerting on failure, while RUM injects a JavaScript snippet that captures real-user page load times, JS errors, route changes, and core web vitals. The complementary pattern: Synthetics catches "is the site up?" before users complain, RUM tells you "for whom and where is it slow?" after the fact. **Skip** the AWS news-blog announcement for RUM (older, marketing-heavy) -- this blog covers the same ground with CDK code.

#### 10. CloudWatch Application Signals + ServiceLens (Skim) -- AWS Feature Page (5 min)
- **URL:** https://aws.amazon.com/cloudwatch/features/application-observability-apm/
- **Why:** Quick orientation only -- Application Signals (GA 2024) is AWS's APM-equivalent on top of CloudWatch, auto-instrumenting Java/Python/Node apps with the AWS Distro for OpenTelemetry (ADOT) to emit RED metrics (Rate, Errors, Duration) per service operation, SLOs, and a service map. **Extract:** Application Signals is essentially "AWS's opinionated take on OpenTelemetry semantic conventions for APM, with auto-discovered SLO dashboards." ServiceLens is the older console feature that stitches X-Ray traces + CloudWatch metrics + logs into a service map -- still useful for non-Application-Signals workloads. **Skip the deep dive** -- you already know SLOs and distributed tracing from May 1; this is just the AWS-managed version. Note that Application Signals is the natural answer to "I want APM on AWS without running Datadog or Honeycomb."

### Block 4: Decision Framework & Cost Wrap-Up (25 min)

#### 11. Prometheus vs CloudWatch for Cloud-Native Applications -- InfraCloud (15 min)
- **URL:** https://www.infracloud.io/blogs/prometheus-vs-cloudwatch/
- **Why:** **The single most important article in this plan for your situation.** You have deep Prometheus and OTel knowledge and need an explicit, numbers-backed comparison of when CloudWatch wins, when Prometheus wins, and when the answer is "both via OTel collector fan-out." **Extract:** the architectural contrast (push/sink vs pull/scrape), the cardinality philosophy gap (the article's 100-node EKS benchmark: ~19,000 CloudWatch metrics vs ~1.5M Prometheus time series -- CloudWatch is engineered for low-to-medium cardinality, Prometheus for high), the cost benchmark numbers (self-managed Prometheus ~$3K/month vs CloudWatch ~$8.6K/month at the article's scale -- the inflection point is roughly "above 50 nodes, Prometheus wins on cost, below 20 nodes, CloudWatch wins on ops overhead"), and the alerting comparison (PromQL expressivity vs CloudWatch's native AWS-service-integration). The mental model to lock in: **CloudWatch = managed AWS-integrated sink with cardinality discipline; Prometheus = self-managed flexible pull with high-cardinality tolerance; OTel = the universal collector that lets you send to either or both**.

#### 12. Vantage: CloudWatch Metrics Pricing Explained in Plain English (10 min)
- **URL:** https://www.vantage.sh/blog/cloudwatch-metrics-pricing-explained-in-plain-english
- **Why:** The clearest English-language pricing breakdown. **Extract specifically these gotchas:** (1) every unique `namespace + metric_name + dimension_combination` is a separate billed metric at $0.30/month (declining tier to $0.02/month above 1M), so adding a `userId` dimension to a 100K-user app instantly creates 100K metrics = $30K/month -- the canonical cardinality-trap example; (2) high-resolution metrics (1s/5s/10s/30s periods) cost 4x because PutMetricData runs 4x per minute; (3) high-res granularity rolls up to 1-minute after only 3 hours, so paying for sub-minute resolution beyond short-term burst monitoring is wasted money; (4) the tiered pricing curve means custom metrics are cheap at low counts but get cheap-per-unit at scale -- removing 100 useless metrics from a 5K-metric account saves more than removing 100 from a 5M-metric account. Pair this with the dimension hygiene rule from resource 6.

## What to Skip

Given your existing observability depth, **explicitly skip** these common CloudWatch resources -- they will waste your two-hour budget:
- **"What is CloudWatch?" / Getting Started guides** -- you already know what monitoring is; the concepts page (Resource 1) gives you AWS-specific vocabulary faster
- **AWS Skill Builder CloudWatch Fundamentals course** -- 4-6 hours of basics you already understand from Prometheus
- **X-Ray getting started** -- you already know distributed tracing from the OTel week; X-Ray is just one possible tracing backend and Application Signals abstracts it
- **CloudWatch agent installation tutorials** -- read the agent config schema when you implement, not now
- **The original 2018 EMF announcement blog** -- superseded by the docs page and FireEye case study
- **CloudWatch Logs subscription filters deep dive** -- only relevant if you build a log-pipeline to OpenSearch/Kinesis Firehose; mention-only for this session
- **CloudWatch Evidently** -- feature flagging service, being deprecated; not worth time
- **Pre-2023 CloudWatch blog posts** -- log classes, Application Signals, enhanced Container Insights, and PromQL support all post-date them; the operating model has materially changed
- **CloudTrail content** -- that is tomorrow's topic, intentionally deferred

## Concepts to Nail by End of Session

By the time you finish this plan and write up your TIL doc, you should be able to answer each of these without looking anything up:

- [ ] **EMF mental model:** Why writing a structured JSON log line is cheaper and lower-latency than calling `PutMetricData`, and how the `_aws.CloudWatchMetrics` envelope tells CloudWatch which fields to extract as metrics vs leave as log context
- [ ] **The cardinality math:** A metric is `namespace + name + dimension_combination`; adding a dimension with N unique values multiplies your metric count by N; the practical ceiling is ~50 unique values per dimension before costs explode
- [ ] **Logs Insights pipeline:** Be able to write a query end-to-end -- `fields @timestamp, @message | filter @message like /ERROR/ | parse @message "user=* path=*" as user, path | stats count() by path | sort count desc | limit 10` -- and explain why filter comes before stats (scanned-bytes billing)
- [ ] **Log class decision:** When to use Standard vs Infrequent Access (the loss list: Live Tail, metric filters, alarming-from-logs, subscription filters, data protection) and the irreversibility gotcha
- [ ] **Alarm taxonomy:** Metric alarm (single metric + threshold), composite alarm (boolean of other alarms, used to suppress noise), anomaly detection alarm (ML-trained band, requires 2 weeks of data, 3x cost) -- and the composite-over-anomaly pattern for paging discipline
- [ ] **Insights suite map:** Container Insights = ECS/EKS infra metrics; Lambda Insights = Lambda runtime telemetry; Application Insights = AWS-managed app-stack monitoring; Application Signals = APM/RED/SLOs via ADOT; ServiceLens = X-Ray + metrics + logs unified view; Synthetics = scheduled probes; RUM = browser-side telemetry
- [ ] **CloudWatch vs Prometheus vs OTel decision:** CloudWatch wins on AWS-native zero-ops + cross-account; Prometheus wins on cardinality + PromQL expressivity + per-dashboard cost; OTel is the universal transport letting you send to both -- and CloudWatch now natively ingests OTel metrics with PromQL query support
- [ ] **Top three cost levers:** (1) Cap dimension cardinality on custom metrics, (2) Set retention on every log group + use IA class for vended/audit logs, (3) Filter logs at the source (CloudWatch agent / Fluent Bit) before ingestion

## Next Steps

- **Implementation (tonight):** Write a Lambda function that emits EMF metrics (use AWS Lambda Powertools for Python or TypeScript -- it has built-in EMF support); create a log group in Infrequent Access class, configure a Logs Insights query saved-query, set up a composite alarm combining an error-rate metric alarm with a low-traffic suppression alarm, and create a Synthetics canary hitting one of your existing API Gateway endpoints from earlier this study plan
- **Tomorrow (2026-05-13): CloudTrail** -- the audit/compliance counterpart to CloudWatch monitoring. Frame the contrast clearly: CloudWatch Logs = operational logs (what your app printed), CloudTrail = API audit logs (who called what AWS API and when). They sometimes share infrastructure (CloudTrail can write to a CloudWatch Logs group for alerting) but they answer fundamentally different questions
- **Day 3 of Phase 2.8: AWS Config** -- the configuration drift/compliance state engine, complementing CloudTrail (events) with Config (state)
- **End of week: Cost & Performance Insights** -- Cost Explorer, Compute Optimizer, Trusted Advisor, the GenAI-driven Application Signals SLO recommendations
