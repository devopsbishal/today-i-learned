# AWS CloudWatch Comprehensive -- The Factory Instrumentation Room

> Phase 2.7 closed with the **bank vault** -- securing the CI/CD pipeline that builds and ships your software. Phase 2.8 opens with the question that comes next: once your software is running in production, **how do you know it's running well?** That is CloudWatch's job, and CloudWatch in 2026 is not the small "AWS metrics console" of 2018. It is a sprawling observability suite -- managed metrics with OpenTelemetry (OTLP) ingestion via the OTel Collector's `awsemf` exporter and Metric Streams for export to Prometheus-compatible backends, Logs split into two storage classes (Nov 2023), three flavours of alarm (metric / composite / anomaly detection), synthetic probes, real-user monitoring, four overlapping Insights products, and an AWS-managed APM (Application Signals, GA 2024). For first-class PromQL you reach for **AWS Managed Prometheus (AMP)**, which sits alongside CloudWatch rather than replacing it. The price of this surface area is operator complexity: the difference between a $300/month CloudWatch bill and a $30,000/month one is usually a single dimension on a single custom metric.
>
> **The core analogy for today: a factory's instrumentation room.** Walk it through. On the factory floor, **every machine has a gauge** -- temperature, pressure, RPM, output rate. The gauges are **metrics**. The cluster of gauges for a particular machine, with a label for what's being measured, is a **metric namespace + name + dimensions**. The floor manager keeps a **logbook** -- everything the machine operators write down through the shift, every error message, every warning sticker on the side of a machine. That's **CloudWatch Logs**. Red lights mounted above the gauges that trip when a needle crosses the line are **alarms**. The factory employs a **dummy walker** who pretends to be a customer, walks the showroom every 5 minutes, and confirms the doors open -- that's **Synthetics**. They also poll **actual customers** about their showroom experience afterward -- that's **RUM**. Specialist inspectors come in for **specific machine families** -- the container-line inspector (Container Insights), the welding-station inspector (Lambda Insights), the assembly-line specialist with a service map (Application Signals). And -- the part nobody warns you about until you've already been burned -- the **cost auditor** notices that the factory is paying for a separate gauge on every individual screw passing through the line, because someone added `userId` as a dimension and the factory has 100,000 users. The instrumentation room is full of gauges that nobody reads, alarms in `INSUFFICIENT_DATA` that nobody owns, and logbooks the size of warehouses with retention set to "forever" because somebody once said "logs are cheap."
>
> Three corollaries to bolt on. **(1) The gauge, the logbook, and the red light are three different products that bill three different ways**, and confusing them is the most common cost surprise in AWS. Metrics bill by *unique combination*; logs bill by *ingested GB plus stored GB plus scanned GB*; alarms bill *flat-per-month-per-alarm*. Each of these has its own cardinality trap, its own retention lever, and its own way of silently inflating to a five-figure bill. **(2) "Modern CloudWatch" is mostly EMF.** Embedded Metric Format -- write a structured JSON log line, CloudWatch auto-extracts metrics from it -- is the single largest architectural shift since 2020, and it is what every Lambda Powertools / OpenTelemetry exporter / production custom-metrics pipeline uses today. If you are still calling `PutMetricData` synchronously on a hot path in 2026, you are paying for latency, API calls, and (worst) cardinality blindness that EMF would have given you for free. **(3) CloudWatch is one possible observability sink, not the only one.** You already know Prometheus, OpenTelemetry, Loki, Tempo. The decision is not "CloudWatch or not" -- it is "for which signal type, at which scale, with which lock-in cost, sent via which transport." The honest answer at ~50 EKS nodes is usually "OTel collector fanning out to both CloudWatch (for the AWS-integrated alarms) and Prometheus/Loki (for the high-cardinality stuff CloudWatch will refuse to bill you reasonably for)."

---

**Date**: 2026-05-12
**Topic Area**: aws
**Difficulty**: Intermediate-to-Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Namespace | The wing of the factory | Top-level metric container (`AWS/Lambda`, `AWS/EC2`, your custom `MyApp/Orders`) |
| Metric name | The label on the gauge | What is being measured (`Invocations`, `Errors`, `CheckoutLatency`) |
| Dimension | A sticker on the gauge | Key/value pair narrowing the metric (`FunctionName=checkout`, `Environment=prod`) |
| Unique billed metric | One specific gauge in the room | `namespace + name + dimension_combo` -- the unit of cost ($0.30/month tiered down) |
| Basic vs detailed monitoring | Hourly check vs every-minute check | EC2 default 5-min vs $2.10/instance/mo for 1-min; Lambda is always 1-min built-in |
| Standard resolution | Once-a-minute gauge reading | 60-second granularity, default for everything |
| High-resolution | Stopwatch-on-the-gauge | 1/5/10/30-second; costs 4x; **rolls up to 1-min after 3 hours anyway** |
| Retention tiers | Old gauge readings auto-summarized | 1s -> 1m@3h, -> 5m@15d, -> 1h@63d, gone @15mo |
| PutMetricData | Calling the gauge-recording clerk on the phone | Synchronous API, costs $0.01/1000 calls, adds latency, no cardinality visibility |
| EMF | Writing the gauge reading in the logbook in code-readable format | Structured JSON log line; CloudWatch auto-extracts metrics; async, single-write, lower-cost |
| `_aws.CloudWatchMetrics` envelope | The "extract these as metrics" highlight in the logbook | Tells CloudWatch which JSON fields are metrics, which are dimensions, which are context |
| Lambda Powertools | The clerk's pre-printed metrics-logbook template | Python/TypeScript library that emits valid EMF for you |
| Cardinality trap | Paying for a gauge on every screw | Adding `userId` to a 100K-user app = 100K unique metrics = $30K/month |
| CloudWatch agent | A field technician installed on the machine | On-host daemon collecting OS-level + custom metrics + logs |
| OpenTelemetry Collector | The universal translator | Receives any format, exports to CloudWatch + Prometheus + others |
| Log group | The logbook for one product line | Collection of log streams for one app/function/service |
| Log stream | One operator's shift entries | One sequential stream of log events (one container, one Lambda invocation cohort) |
| Standard log class | The fireproof on-site logbook | $0.50/GB ingestion, all features (Live Tail, metric filters, alarming, subscriptions) |
| Infrequent Access (IA) | The off-site archive | $0.25/GB ingestion, **loses** Live Tail / metric filters / alarming / subscriptions / Insights enrichment |
| Log class irreversibility | Once labeled archive, always archive | Class is set at log-group creation; cannot be changed -- delete and recreate |
| Vended logs | Standard manufacturer paperwork | API Gateway / VPC Flow / ELB logs -- ideal IA targets |
| Logs Insights | The librarian who can grep the logbook | SQL-ish pipeline language; bills $0.005/GB scanned |
| `fields \| filter \| parse \| stats` | The librarian's standard query template | Pipeline order matters: filter first, scan less, pay less |
| `@timestamp`, `@message`, `@requestId` | Auto-discovered logbook columns | Standard `@` fields you can always reference |
| Metric alarm | One red light on one gauge | Single metric crossing a threshold for N of M evaluation periods |
| Composite alarm | A red light driven by a logic gate | Boolean combination (AND/OR/NOT) of other alarms via `ALARM_RULE` |
| Anomaly detection alarm | A red light that learns the gauge's normal rhythm | ML-trained band; 2-week training; **3x the cost** of a standard alarm |
| Orphan alarm | A red light wired to nothing | `INSUFFICIENT_DATA` alarms still bill $0.10/month each |
| Synthetics canary | The dummy walker on the showroom floor | Scheduled Puppeteer/Playwright probe of golden-path URLs |
| RUM | Customer feedback cards in the showroom | JS snippet on your frontend capturing real-user page loads + JS errors + Core Web Vitals |
| Container Insights | The container-line specialist inspector | EKS/ECS metrics + control plane (since enhanced observability Nov 2023) |
| Lambda Insights | The Lambda-runtime specialist | Memory / CPU / network telemetry from the Lambda extension layer |
| Application Insights | The auto-discovery generalist | AWS-managed app-stack monitoring with pattern recognition |
| Application Signals | AWS's APM (GA 2024) | Auto-instrumentation via ADOT; RED metrics + SLOs + service map |
| ServiceLens | The older trace+metric+log stitcher | X-Ray + CloudWatch fused into a service map; pre-Application-Signals |
| `cloudwatch:PutMetricData` IAM | Permit to phone the clerk | Only needed if you use the API; **EMF only needs `logs:PutLogEvents`** |
| Subscription filter | The auto-forward rule on the logbook | Real-time fan-out of log events to Lambda / Kinesis / Firehose |
| CloudWatch vs CloudTrail | Operational logbook vs auditor's ledger | CloudWatch = "what your app printed"; CloudTrail = "who called what AWS API" |

---

## The Big Picture in One Diagram

```
THE FACTORY INSTRUMENTATION ROOM -- WHAT EACH PRODUCT IS
==============================================================================

    THE FACTORY FLOOR (your workloads)                  THE INSTRUMENTATION ROOM
    ----------------------------------                  -------------------------

    +--------------+   PutMetricData (phone)
    |   Lambda     |--------------------------+
    +--------------+                          |
                                              v
    +--------------+   EMF JSON log line   +--------------------+
    |   Container  |---------------------->|  CloudWatch Logs   |    METRICS (gauges)
    +--------------+   (logs:PutLogEvents) |  (Standard or IA)  |        ^
                                           +---------+----------+        |
    +--------------+   StatsD / OTel                 |    auto-extract   |
    |   EC2 host   |-------+                         |    from EMF       |
    +--------------+       |                         v                   |
                           v                 +----------------+   +-----------------+
                    +--------------+         | Logs Insights  |   |  CloudWatch     |
                    | CloudWatch   |         | (librarian)    |   |  Metrics TSDB   |
                    | Agent / OTel |---------+----------------+   | (15mo, rollup)  |
                    | Collector    |         | $0.005/GB scan |   +-------+---------+
                    +--------------+         +----------------+           |
                                                                          |
    +--------------+   ADOT auto-instr                                    v
    |  APM (Java/  |--------------------+                          +--------------+
    |  Py/Node)    |                    |                          | Alarms       |
    +--------------+                    |  +------------------+    | (metric +    |
                                        +->| Application      |    |  composite + |
                                           | Signals (APM)    |    |  anomaly)    |
                                           +------------------+    +------+-------+
                                                                          |
    +--------------+   Lambda Insights ext layer                          v
    |  EKS / ECS   |--------------------+                          +--------------+
    +--------------+                    |                          | SNS / Slack /|
                       +-------------+  |                          | PagerDuty /  |
                       | Container   |<-+                          | Lambda /     |
                       | Insights    |                             | Auto Scaling |
                       +-------------+                             +--------------+

    +--------------+                                              +-----------------+
    |  Browser     |----- RUM JS snippet ------------------------>| RUM console     |
    +--------------+                                              | (separate plane)|
                                                                  +-----------------+

    +--------------+                                              +-----------------+
    |  Synthetics  |----- Puppeteer/Playwright probe ------------>| Canary results  |
    |  canary      |     (scheduled cron from CW Synthetics)      | -> alarms       |
    +--------------+                                              +-----------------+

    CARDINAL RULES:
      1. METRICS bill by unique (namespace + name + dimension_combo). One bad
         dimension can mint 100,000 metrics in an afternoon.
      2. LOGS bill by ingested GB + stored GB + scanned GB on Insights queries.
         Retention discipline + IA class are the two biggest levers.
      3. ALARMS bill flat per month (and 3x for anomaly detection). Orphans in
         INSUFFICIENT_DATA still bill.
      4. EMF is the modern way to emit metrics. PutMetricData is the legacy way.
      5. Five Insights products overlap; people conflate them constantly --
         memorize what each one is FOR.
```

---

## Part 1: The Vocabulary You Cannot Skip -- Metrics, Dimensions, Resolution, Retention

Everything in CloudWatch starts from one definition: a **metric is the unique combination of `namespace + metric_name + dimension_combination`**. Internalize this single sentence and every cost story below makes sense. Miss it, and you will mint thousands of metrics by accident and not understand why your bill quadrupled.

### Namespace, metric name, dimension -- the gauge anatomy

A **namespace** is the wing of the factory. Top-level container: `AWS/Lambda`, `AWS/EC2`, `AWS/ApplicationELB`, or your own `MyApp/Orders`. Custom namespaces should be `Org/Service` style. Namespaces are organizational only -- they do not bill on their own.

A **metric name** is the label printed on the gauge. `Invocations`, `Errors`, `Duration`, `CheckoutLatency`. Just a string.

A **dimension** is a sticker stuck to the gauge that narrows what it measures. `FunctionName=checkout-service`, `Environment=prod`, `Region=us-east-1`. Dimensions are key/value pairs. A metric can have up to **30 dimensions** in 2026 (was 10 for years -- raised in 2022).

**The unique-metric definition:** every distinct combination of `(namespace, name, dimension_set)` is a separate billed metric. The same metric name with two different dimension values is **two metrics**. Internalize the math:

```
MyApp/Orders.CheckoutLatency {Env=prod,Region=us-east-1}     <- one metric
MyApp/Orders.CheckoutLatency {Env=prod,Region=us-west-2}     <- second metric
MyApp/Orders.CheckoutLatency {Env=staging,Region=us-east-1}  <- third metric
MyApp/Orders.CheckoutLatency                                  <- FOURTH metric (no dimensions = a distinct entry)
```

The "no-dimensions" version is its own unique metric. CloudWatch does **not** auto-aggregate across dimensions; if you want `CheckoutLatency` across all regions, you have to either publish it dimensionless yourself or query with metric math.

### The cardinality trap (the canonical $30K-bill example)

The bank-vault analogy from yesterday warned about supply-chain attacks; today's equivalent is the **dimension cardinality bomb**. Suppose your app processes orders for 100,000 unique users, and somebody innocently adds `UserId` as a dimension to the `CheckoutLatency` metric "for debugging."

```
MyApp/Orders.CheckoutLatency {UserId=u-001}    <- metric 1
MyApp/Orders.CheckoutLatency {UserId=u-002}    <- metric 2
...
MyApp/Orders.CheckoutLatency {UserId=u-100000} <- metric 100,000
```

At CloudWatch's standard custom-metric pricing (us-east-1: $0.30/metric/month for the first 10K, $0.10/metric for the next 240K (10K-250K), $0.05/metric for the next 750K (250K-1M), $0.02/metric above 1M), 100,000 unique metrics cost roughly **$12,000/month** -- $3,000 for the first 10K tier plus $9,000 for the next 90K at $0.10 each -- and that's just the storage; you've also paid for every `PutMetricData` call along the way. The gauge cabinet is full of 100,000 gauges and nobody reads any of them; you have just paid five figures for a debug field.

The rule of thumb is **~50 unique values per dimension before costs explode**. `Environment` (prod/staging/dev) is safe -- three values. `Region` is safe -- 30ish. `FunctionName` is safe -- maybe 100. `UserId`, `RequestId`, `OrderId`, `IpAddress`, `SessionId` are **never safe as dimensions** -- they are unbounded. These belong in log lines (queryable with Insights), not in metric dimensions.

> **The principle worth tattooing**: *Dimensions are for things you'd alarm on. Log fields are for things you'd investigate.* ID-shaped values almost always belong in the second bucket.

### The VIP-allowlist pattern -- when you do need per-entity alarming

The naive fix when someone says "but I need a per-merchant SLO dashboard!" is to put `merchantId` in dimensions and watch the bill explode. The senior fix is the **allowlist pattern**: emit **two EMF metric blocks** from the same log event.

- **Block A** is the base metric -- low cardinality, no ID-shaped dimensions, used for global dashboards and aggregate alarms. Cost: ~15 series per "metric."
- **Block B** is gated by a check against an SSM Parameter Store list or a config file of "VIP entities" (top 20 enterprise merchants, regulated tenants, your largest 5 customers). It emits the same metric name *with* the high-cardinality dimension -- but only for the 20 entities that actually warrant per-entity paging. Cost: ~20 series, not 12,000.

The cost arithmetic on a payment metric with `Environment` (3), `PaymentMethod` (5), and `MerchantId` (12,000):

- Naive: 3 x 5 x 12,000 = **180,000 series ≈ $20,000/month** (10K x $0.30 + 170K x $0.10).
- With allowlist: 15 base series + 20 VIP-merchant series = **35 series ≈ $11/month**.

The non-VIP merchants still get full investigation surface via Logs Insights (`stats count(*) by merchantId` queryable on the same logs where the metric was emitted -- since the EMF JSON carries `merchantId` as a root-level field even when it's *not* in the `Dimensions` array). You only pay for per-entity *alarming*, which is a much smaller set of entities than per-entity *investigation*.

This is the **#1 lesson of the entire doc**, repeated three more times before the end.

### Basic vs detailed monitoring -- two confusing axes

**Basic vs detailed monitoring** is an EC2-and-some-other-services concept: how often AWS *itself* publishes the metric to CloudWatch.

| Service | Basic (default) | Detailed (opt-in) | Cost of detailed |
|---------|-----------------|-------------------|------------------|
| EC2 | 5-minute | 1-minute | $2.10/instance/month |
| RDS | 1-minute (always) | -- | Built-in |
| Lambda | 1-minute (always) | -- | Built-in |
| ELB / ALB / NLB | 1-minute (always) | -- | Built-in |
| Auto Scaling group | 1-minute aggregations | -- | Built-in |

EC2 is the place this still matters. By default, AWS publishes EC2 CPU/disk/network metrics every 5 minutes. Turning on detailed monitoring drops it to every minute and you pay per instance. For autoscaling-sensitive workloads (HPA-style behaviour, fast-scaling fleets), detailed monitoring is usually worth it. For an idle dev instance, it isn't.

### Standard vs high-resolution -- the per-metric axis

**Standard vs high-resolution** is a *per-metric* thing for *custom metrics you publish yourself*. Standard is 60-second granularity. High-resolution is 1, 5, 10, or 30 seconds. The big gotcha:

- **High-res metrics cost 4x** because `PutMetricData` runs 4x/minute under the hood.
- **High-res metrics roll up to 1-minute after only 3 hours.** After 3 hours, your sub-minute samples are gone; only the 1-minute aggregate remains.

The implication: high-res is for burst monitoring of latency-sensitive systems where you genuinely want sub-minute visibility *for an active incident or short experiment*. It is not a "set and forget" default. If you set high-res across your whole fleet and never look at sub-minute data within the 3-hour window, you are paying 4x for nothing.

### The retention/rollup ladder (the gauge readings get summarized)

CloudWatch keeps metric data for **15 months total**, but it doesn't keep it at original resolution -- it auto-summarizes:

| Age of data | Resolution kept |
|-------------|-----------------|
| < 3 hours | 1-second (only if you published high-res) |
| 3 hours -- 15 days | 1-minute |
| 15 days -- 63 days | 5-minute |
| 63 days -- 15 months | 1-hour |
| > 15 months | Gone |

You cannot opt out of this rollup or pay to retain finer granularity longer. If you need to keep raw 1-minute data for a year for compliance, you have to ship it elsewhere (S3 via metric streams, or an OTel pipeline writing to Mimir/Prometheus long-term storage).

### Statistics, periods, percentiles -- the librarian's reading tools

CloudWatch can summarize a metric over a time period using a **statistic**: `Sum`, `Average`, `Minimum`, `Maximum`, `SampleCount`, `p50`, `p90`, `p95`, `p99`, `p99.9`, or any custom `pNN.N` percentile. The **period** is the bucket width (typically 60s for standard metrics).

Two gotchas the Prometheus crowd should pay attention to:

1. **Percentiles in CloudWatch are computed from the raw published values** (or, for EMF / built-in service metrics, from the histogram of values per period). They are *not* the average of pre-computed quantiles per shard, so unlike Prometheus summaries, they aggregate correctly across instances.
2. **`Average` of latency is almost always the wrong stat**; reach for `p95`/`p99`. Average smooths out the long tail and hides the actual user experience.

---

## Part 2: PutMetricData vs EMF -- The Modern Way to Emit Custom Metrics

If you take only one section away from today, take this one. EMF is the **single largest architectural shift in CloudWatch since 2020**, and most production AWS estates today either use it directly or have an OTel / Lambda Powertools wrapper that emits it under the hood.

### The old way: PutMetricData (calling the clerk on the phone)

Before EMF, the only way to publish a custom metric was the CloudWatch API:

```python
import boto3
cw = boto3.client('cloudwatch')

# In the hot path of an HTTP handler:
cw.put_metric_data(
    Namespace='MyApp/Orders',
    MetricData=[{
        'MetricName': 'CheckoutLatency',
        'Dimensions': [
            {'Name': 'Environment', 'Value': 'prod'},
            {'Name': 'Region', 'Value': 'us-east-1'},
        ],
        'Value': 0.247,
        'Unit': 'Seconds',
        'Timestamp': datetime.utcnow(),
    }]
)
```

Three problems:

1. **It's synchronous.** Your hot path now includes an HTTP call to `monitoring.amazonaws.com`. That's 10-50 ms of added latency per request, every request. On a Lambda function this is even worse -- you can hit `PutMetricData` only after the response, which means you either delay the response or fire-and-forget without back-pressure.
2. **It costs per API call.** $0.01 per 1,000 `PutMetricData` requests. A million requests/day = $300/month just in API calls, before you store a single metric.
3. **It's cardinality-blind.** Nothing tells you that you just emitted `UserId=u-12345` as a dimension and minted metric #100,000. You find out next month on the bill.

The factory analogy: every time the machine wants to record a gauge reading, the operator literally **picks up the phone, dials the recording office, waits for someone to answer, dictates the reading, and hangs up**. It works. It is slow. It costs per call. And the recording office doesn't tell you "you're spelling your machine name wrong; you've created 100,000 separate filing cabinets this month."

### The new way: EMF (writing in the logbook in code-readable format)

Embedded Metric Format inverts the model: **you write a structured JSON log line, and CloudWatch auto-extracts metrics from it on the way in**. You never call `PutMetricData`. You just `logger.info(json_blob)`.

A minimal EMF log line looks like this:

```json
{
  "_aws": {
    "Timestamp": 1715472000000,
    "CloudWatchMetrics": [
      {
        "Namespace": "MyApp/Orders",
        "Dimensions": [["Environment", "Region"]],
        "Metrics": [
          { "Name": "CheckoutLatency", "Unit": "Seconds" },
          { "Name": "OrdersProcessed", "Unit": "Count" }
        ]
      }
    ]
  },
  "Environment": "prod",
  "Region": "us-east-1",
  "CheckoutLatency": 0.247,
  "OrdersProcessed": 1,
  "RequestId": "abc-123-def-456",
  "UserId": "u-987",
  "OrderTotal": 49.99
}
```

The `_aws.CloudWatchMetrics` envelope is the **highlighter mark** that tells CloudWatch which fields are gauges, which are stickers (dimensions), and which are just notes (log context). Specifically:

- **`Namespace`** -- which wing of the factory.
- **`Dimensions`** -- a list-of-lists. Each inner list is one **dimension set** that will be applied to the metrics. The example above declares ONE dimension set: `[Environment, Region]`. CloudWatch will pull the values of those fields from the same JSON object (so this log line emits the metric tagged `Environment=prod, Region=us-east-1`). You can declare multiple dimension sets to publish the same metric at multiple aggregation levels in a single log write.
- **`Metrics`** -- the gauge names plus units. Each name must match a field in the same JSON object.

The other top-level fields (`RequestId`, `UserId`, `OrderTotal`) are **just log context**. They go into the log line for forensic query-ability via Logs Insights, but they do NOT become metrics or dimensions because they aren't named in the envelope. **This separation is the killer feature** -- you can keep `userId` and `requestId` in the log for debugging without minting per-user metrics.

### Why EMF wins on every axis except one

| Property | PutMetricData | EMF |
|----------|---------------|-----|
| Latency on hot path | 10-50ms synchronous API call | One `logger.info()`, ~microseconds |
| Cost per emit | $0.01/1000 API calls + metric cost | $0.50/GB ingestion + metric cost (negligible per emit) |
| IAM permission | `cloudwatch:PutMetricData` | Only `logs:PutLogEvents` |
| Log context alongside metric | None -- separate API call | Free -- same JSON line is searchable in Logs Insights |
| Cardinality safety | Blind | **Same blindness** (see below) |
| Best client library (Python) | boto3 | aws-lambda-powertools |
| Best client library (Node) | aws-sdk | @aws-lambda-powertools/metrics |

The "one axis" where EMF doesn't help: **cardinality**. EMF still mints a new unique metric for every new dimension-value combination it sees. If you put `UserId` in `Dimensions`, you still get 100,000 metrics. EMF is not magic; it is a more efficient *transport*. The cardinality discipline is on you.

### EMF batch limits to know

EMF has hard limits enforced by the log-ingestion path -- Powertools auto-flushes when you hit them, but knowing the underlying ceiling matters for high-throughput services:

- **100 metric values per log event** (one event can carry many metric *values*, but bounded).
- **256 KB max log event size** -- if your JSON blob exceeds this, ingestion drops or truncates it. High-cardinality dimension sets or large embedded payloads will hit this first.
- **30 dimensions per metric** -- AWS API hard cap. The practical cap is much lower (50 unique *values* per dimension).
- **9 unique dimension sets per metric envelope** (the `Dimensions` array entries -- each entry is a different way to slice the same metric, e.g., `[["service"], ["service","region"], ["region"]]`).

If you breach any of these from an EMF emitter that doesn't auto-flush (i.e., not Powertools), the log event silently fails ingestion -- you lose the data and won't be paged for it. Powertools handles the boundaries for you (it splits envelopes when you exceed 100 metrics, for example), which is the main reason it's the production default rather than hand-rolled JSON.

### The FireEye 65% case study

The canonical motivation for EMF is the **FireEye case study** AWS published in 2020. FireEye switched their metric emission from `PutMetricData` to EMF and cut their **overall CloudWatch monitoring cost by 65%**. The math:

- Before: every metric was a `PutMetricData` call -- one HTTP request per metric. They emitted millions per day.
- After: each metric is part of a JSON log line, **multiple metrics share a single log emit**, and the log line *also* carries forensic context (request ID, user ID) that was previously a *second* observability path through CloudWatch Logs.

Net: one network write replaces two (one metric API call + one log write), API-call costs disappear, log-ingestion costs include forensic context "for free." Note: EMF still mints the same number of unique custom metrics as `PutMetricData` would -- the 65% savings come from consolidating API calls and folding the forensic log path into the same emit, not from cheaper metrics themselves. This is what every modern AWS workload should be doing in 2026.

### IAM trick: only `logs:PutLogEvents` (not `cloudwatch:PutMetricData`)

A small but underrated benefit: your Lambda role / ECS task role / EC2 instance profile only needs `logs:PutLogEvents` to emit EMF metrics. You can entirely revoke `cloudwatch:PutMetricData` across your account if you're EMF-only, which is a tidy least-privilege win. In some compliance regimes this is the difference between "we have a custom-metrics blast radius" and "we don't."

### Python with AWS Lambda Powertools (the production library)

You should never hand-roll the JSON envelope. AWS Lambda Powertools is the production-grade EMF library. Python example:

```python
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
import os

# Namespace + default dimensions configured once
metrics = Metrics(namespace="MyApp/Orders", service="checkout-service")
metrics.set_default_dimensions(environment=os.environ["ENVIRONMENT"])

@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event, context):
    metrics.add_metric(name="OrdersReceived", unit=MetricUnit.Count, value=1)

    start = time.monotonic()
    try:
        result = process_order(event)
        metrics.add_metric(name="OrderSuccess", unit=MetricUnit.Count, value=1)
    except PaymentError:
        metrics.add_metric(name="PaymentFailure", unit=MetricUnit.Count, value=1)
        raise
    finally:
        duration = time.monotonic() - start
        metrics.add_metric(
            name="CheckoutLatency", unit=MetricUnit.Seconds, value=duration
        )
        # forensic context -- NOT a dimension, just log context
        metrics.add_metadata(key="request_id", value=context.aws_request_id)
        metrics.add_metadata(key="user_id", value=event.get("user_id"))

    return result
```

The `@metrics.log_metrics` decorator flushes the EMF JSON to stdout at the end of the function invocation, where the Lambda runtime forwards it to CloudWatch Logs, where the EMF parser extracts metrics. `capture_cold_start_metric=True` adds a free `ColdStart` count metric, which is one of the most useful Lambda metrics that doesn't ship by default.

**Dimensions in Powertools** are set via `set_default_dimensions()` (applied to every metric in the invocation) or `metrics.add_dimension()` for invocation-specific dimensions. **Metadata** (`add_metadata`) is log context only -- it goes into the JSON line but never becomes a dimension. This is the API that prevents you from accidentally adding `user_id` as a dimension and minting 100,000 metrics.

### TypeScript / Node.js equivalent

```typescript
import { Metrics, MetricUnit } from '@aws-lambda-powertools/metrics';
import { logMetrics } from '@aws-lambda-powertools/metrics/middleware';
import middy from '@middy/core';

const metrics = new Metrics({
  namespace: 'MyApp/Orders',
  serviceName: 'checkout-service',
  defaultDimensions: { environment: process.env.ENVIRONMENT! },
});

const lambdaHandler = async (event: any, context: any) => {
  metrics.addMetric('OrdersReceived', MetricUnit.Count, 1);

  const start = process.hrtime.bigint();
  try {
    const result = await processOrder(event);
    metrics.addMetric('OrderSuccess', MetricUnit.Count, 1);
    return result;
  } catch (err) {
    metrics.addMetric('PaymentFailure', MetricUnit.Count, 1);
    throw err;
  } finally {
    const durationSec = Number(process.hrtime.bigint() - start) / 1e9;
    metrics.addMetric('CheckoutLatency', MetricUnit.Seconds, durationSec);
    metrics.addMetadata('request_id', context.awsRequestId);
    metrics.addMetadata('user_id', event.user_id);
  }
};

// middy middleware calls metrics.publishStoredMetrics() at end of invocation
export const handler = middy(lambdaHandler).use(logMetrics(metrics, {
  captureColdStartMetric: true,
}));
```

Same shape, same semantics, same cardinality discipline. The Powertools libraries are the de-facto standard across Lambda runtimes; for non-Lambda workloads (containers, EC2), the equivalent is to either use the **CloudWatch agent's EMF support**, write the JSON line yourself, or use the **OTel collector's `awsemf` exporter** which translates OTel metrics into EMF on the way to CloudWatch Logs.

---

## Part 3: PutMetricData vs EMF vs Agent vs OTel -- The Decision Framework

You actually have four ways to get a custom metric into CloudWatch in 2026:

| Method | Best for | Cost shape | Cardinality safety |
|--------|----------|------------|--------------------|
| **PutMetricData API** | Out-of-band scripts, one-off batch jobs, infrastructure-side metrics where you don't already have logs | $0.01/1000 calls + metric cost; latency on hot path | None |
| **EMF via stdout (Lambda Powertools)** | Lambda, ECS, containerized apps; the production default | Log ingestion + metric cost; near-zero latency | None (still up to you) |
| **CloudWatch Agent (custom config)** | EC2 fleets, on-host OS metrics + custom stats, StatsD/collectd ingest | Agent + ingestion + metric cost | None |
| **OTel Collector (`awsemf` exporter)** | Multi-backend estates where the same metric must reach both CloudWatch and Prometheus | Adds collector ops, otherwise same | None |

The decision tree:

```
Are you emitting from inside Lambda or a container with stdout?
  YES -> Use EMF via Powertools (Python/Node/Java/.NET).
  NO -> continue

Are you on EC2 or an on-prem host where the CloudWatch Agent is already installed?
  YES -> Use the agent. Configure your app to emit StatsD locally; agent forwards.
  NO -> continue

Do you need the same metric to land in CloudWatch AND Prometheus (e.g., hybrid backend)?
  YES -> Use OTel Collector with `awsemf` + `prometheusremotewrite` exporters.
  NO -> continue

Are you a one-off batch script or out-of-band utility?
  YES -> PutMetricData is fine.
```

**Never use raw PutMetricData from a Lambda hot path.** It is the wrong answer in 2026.

---

## Part 4: CloudWatch Logs -- The Logbook

CloudWatch Logs is the logbook side of the instrumentation room. It is also where EMF metrics live before extraction. The model is simple:

- A **log group** is the logbook for one product line -- one app, one Lambda function, one EKS cluster's container logs.
- A **log stream** is one continuous shift entry -- typically one container, one Lambda invocation cohort, one EC2 instance.
- Each log event is timestamped + a UTF-8 message (often JSON).
- Retention is set **per log group** -- default infinite, which is a quiet $/month per GB stored.

Two pricing dimensions matter:

| Dimension | Standard class | IA class |
|-----------|----------------|----------|
| Ingestion | $0.50/GB | $0.25/GB |
| Storage | ~$0.03/GB-month (gzip-compressed) | ~$0.03/GB-month |
| Insights scan | $0.005/GB scanned | $0.005/GB scanned |

Ingestion is by far the biggest line item; you pay for *every* byte that crosses into CloudWatch Logs, decompressed. Compression is server-side after ingestion, so a verbose log line is billed full-fat.

### Retention -- the biggest one-time-fix in most accounts

Default retention is **Never Expire**. This is the worst default in CloudWatch and the source of more shock-bills than any single other configuration. A reasonable starting heuristic:

| Log type | Retention | Why |
|----------|-----------|-----|
| Verbose debug logs | 7 days | You'll never look beyond a week |
| App / access logs | 30 days | Covers most incident postmortems |
| Audit / compliance logs | 90 days -- 7 years | Driven by your compliance regime, not engineering preference |
| Test/dev environments | 7 days | The shortest you can stomach |

Every log group should have a retention set explicitly. The one-line Terraform fix that quietly drops a lot of bills:

```hcl
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = 30
  log_group_class   = "STANDARD"  # or "INFREQUENT_ACCESS"
  kms_key_id        = aws_kms_key.logs.arn
}
```

Walk through every log group in your account once with `aws logs describe-log-groups --query 'logGroups[?!retentionInDays]'` to find the un-retentioned ones. Most accounts have dozens.

### Log classes (Nov 2023): Standard vs Infrequent Access

In November 2023, AWS introduced a second log-group **class** -- **Infrequent Access (IA)**. Half-price ingestion, identical storage cost, identical Logs Insights query support (with three exceptions). The feature matrix:

| Feature | Standard | Infrequent Access |
|---------|----------|-------------------|
| Ingestion price | $0.50/GB | $0.25/GB |
| Storage price | Same | Same |
| Logs Insights | Yes | Yes (no `pattern`, `diff`, `unmask`, `filterIndex`) |
| Live Tail | Yes | **No** |
| Metric filters | Yes | **No** |
| Subscription filters | Yes | **No** |
| Alarming from log filters | Yes | **No** |
| Data Protection (PII masking) | Yes | **No** |
| Container / Lambda Insights enrichment | Yes | **No** |
| Embedded Metric Format extraction | Yes | **No** -- EMF logs MUST be in Standard |

And the gotcha that bites everyone the first time: **the log class is set at log-group creation and cannot be changed.** If you put your EMF-emitting Lambda's log group into IA by accident, your custom metrics silently disappear. The only fix is to delete the log group and recreate it -- and you lose all historical log data in the process.

**Ideal IA targets** (low-query-frequency, no-EMF, no-metric-filter logs):

- **Vended logs** -- API Gateway access logs, VPC Flow Logs, ELB access logs. AWS writes these for you; you almost never query them in real time; you keep them for compliance and post-incident forensics.
- **Verbose debug logs** that are off in prod but on during incidents.
- **Compliance archive logs** with 90-day to 7-year retention.

**Never put in IA**:

- Lambda function log groups (EMF extraction breaks).
- Container application logs you alarm off via metric filters.
- Any log group whose group name you've configured as a Container/Lambda Insights enrichment source.

The IA price cut is real and ~50% on ingestion, but the irreversibility makes this a decision you must make at log-group creation time. Tag your IA log groups with `LogClass=IA` so future humans don't try to create a metric filter that silently never fires.

### Vended logs -- the canonical IA win

The biggest single bill-cutter in most AWS accounts: **move your VPC Flow Logs, API Gateway access logs, and ALB access logs to IA log groups**. These three log sources are typically your highest-ingestion sources, you almost never query them in real time, and they hit all the IA-friendly criteria (no EMF, no metric filter, no Live Tail need).

Terraform:

```hcl
resource "aws_cloudwatch_log_group" "vpc_flow" {
  name              = "/aws/vpc/flowlogs/main"
  retention_in_days = 90
  log_group_class   = "INFREQUENT_ACCESS"  # 50% savings on ingestion
}

resource "aws_flow_log" "vpc" {
  log_destination_type = "cloud-watch-logs"
  log_destination      = aws_cloudwatch_log_group.vpc_flow.arn
  traffic_type         = "ALL"
  vpc_id               = aws_vpc.main.id
}
```

### Metric filters -- the legacy log-to-metric path

Before EMF existed, the way you got "count of ERROR lines per minute" as a metric was a **metric filter**: a server-side pattern that watches a log group, matches log events with a filter expression, and increments a CloudWatch metric per match. It's the foreman walking the logbook with a tally counter -- one tally for every "FAILURE" he sees written down.

```hcl
resource "aws_cloudwatch_log_metric_filter" "error_count" {
  name           = "checkout-error-count"
  log_group_name = aws_cloudwatch_log_group.checkout.name
  pattern        = "[..., level=ERROR, ...]"  # space-separated terms; supports JSON paths

  metric_transformation {
    name          = "ErrorCount"
    namespace     = "MyApp/Checkout"
    value         = "1"
    default_value = "0"  # critical: emits 0 when no matches, so alarms don't break
  }
}
```

The filter pattern language has two flavours: **space-separated literal terms** (matching unstructured logs) and **JSON path syntax** (`{ $.level = "ERROR" && $.duration > 1000 }`) for structured logs. The metric is created lazily -- the first match writes the first datapoint, so you must set `default_value = "0"` to keep the metric publishing zeros when nothing matches (otherwise metric alarms fall into `INSUFFICIENT_DATA` during quiet periods).

When to still use metric filters in 2026:

- **You can't modify the emitting code.** Third-party software writing logs you don't control -- a database, an off-the-shelf agent, an open-source sidecar. Metric filters are how you get a metric out without instrumenting the source.
- **AWS-vended logs you need a metric off** -- counting 5xx responses in API Gateway access logs, or specific patterns in VPC Flow Logs.
- **Pre-EMF legacy code** you haven't migrated yet.

When **not** to use them:

- **Code you own and can emit EMF from.** EMF is cheaper, lower-latency, and more flexible -- a metric filter is one regex against every log line whereas EMF is a structured assertion at emit time.
- **Anything you put in an IA log group.** Metric filters silently don't fire on IA log groups; this is the most common "why isn't my alarm firing?" gotcha in the cost-optimized estate.
- **High-cardinality dimensions.** Metric filters can emit a metric per match but the dimension values come from the pattern -- you can't trivially add a dynamic dimension like `userId` (you'd need one filter per unique value, which is the wrong shape).

The mental model: **metric filters extract metrics from logs you can't change; EMF embeds metrics into logs you do control.** Both produce identical CloudWatch metrics on the read side.

### Subscription filters (mention-only depth)

A **subscription filter** is the auto-forward rule on the logbook -- "every log event matching this pattern, send it in real time to Lambda / Kinesis Data Stream / Firehose." Common uses:

- Stream logs to OpenSearch via Firehose for full-text search.
- Stream logs to a SIEM via Kinesis.
- Fan-out to Lambda for inline transformation.

The gotchas to remember without going deeper:

- **Each subscription filter is a real-time tail** -- it adds latency and cost compared to one-shot export.
- **Fan-out to Lambda counts as additional Lambda invocations.** A noisy log group can become a meaningful Lambda bill.
- **Two subscription filters per log group max.** Beyond that, you fan-out from the destination Lambda.
- **Subscription filters do not work on IA log groups.** Same as metric filters and alarming.

You won't reach for subscription filters often in a CloudWatch-centric estate, but they're how you bridge to OpenSearch or a SIEM.

---

## Part 5: Logs Insights -- The Highest-Leverage Day-to-Day Skill

Every production incident, every cost investigation, every "what just happened" question in AWS goes through **Logs Insights**. It is the librarian who can read your logbook and answer questions. Master it and you'll save hours per week; ignore it and you'll be reaching for `grep` against an export.

### The pipeline model

Logs Insights queries are a left-to-right pipeline of commands separated by `|`:

```
fields    | filter   | parse    | stats    | sort     | limit
```

The **canonical order**:

1. `fields` -- declare which fields to extract (the auto-discovered `@timestamp`, `@message`, etc., plus any JSON fields).
2. `filter` -- drop events you don't want. **Filter FIRST**. This is the most important cost rule (see below).
3. `parse` -- extract substrings from `@message` into named variables.
4. `stats` -- aggregate by field with `count()`, `avg()`, `pct()`, etc.
5. `sort` -- by any field or alias.
6. `limit` -- cap the result rows.

### Auto-discovered `@` fields

Every log event in CloudWatch Logs has a handful of auto-discovered fields:

| Field | What it is |
|-------|-----------|
| `@timestamp` | Event timestamp (millisecond resolution) |
| `@message` | Raw log line |
| `@logStream` | Which stream within the group |
| `@log` | Which log group (cross-group queries) |
| `@requestId` | Lambda invocation ID (Lambda log groups only) |
| `@duration`, `@billedDuration`, `@memorySize`, `@maxMemoryUsed` | Lambda runtime metrics (Lambda log groups only) |

If your log line is JSON, **every top-level JSON field is automatically a queryable field too**. So if your app logs `{"level":"ERROR","user_id":"u-123","path":"/checkout"}`, you can `filter level = "ERROR"` directly with no `parse` needed.

### The cost rule: filter FIRST

Logs Insights bills **$0.005 per GB scanned**. The query bills based on **how much log data CloudWatch had to read**, not how much it returned. Two seemingly equivalent queries can have wildly different costs:

```
# CHEAP -- filter prunes the scan window
fields @timestamp, @message
| filter level = "ERROR"
| stats count() by service

# EXPENSIVE -- stats runs on every line, then filter is meaningless
fields @timestamp, @message
| stats count() by service
| filter service = "checkout-service"
```

**Always put `filter` immediately after `fields`** to minimize the bytes that flow through the pipeline. Also: **narrow your time range**. A query across the last 24 hours of a 10-GB/day log group is 10 GB scanned ($0.05). The same query across 30 days is 300 GB ($1.50). Across multiple groups for 90 days, you can quickly land a $50+ query bill.

### Five realistic queries you'll actually run

**Lambda error rate over the last hour, grouped by function:**

```
fields @timestamp, @message, @log
| filter @message like /ERROR/
| stats count() as errors by @log
| sort errors desc
```

`@log` here is the log group identifier, so you can run this across multiple Lambda log groups in one query and see which function is the loudest.

**p95 latency from API Gateway access logs:**

```
fields @timestamp, integrationLatency, path
| filter @logStream like /api-gateway-access/
| stats pct(integrationLatency, 95) as p95 by path
| sort p95 desc
| limit 20
```

This relies on API Gateway access logs being structured (configured via CloudFormation/Terraform with `$context.integrationLatency` etc. in the access-log format string).

**Parse + extract (when your logs are not yet JSON):**

```
fields @timestamp, @message
| filter @message like /Order processed/
| parse @message "user=* order=* amount=*" as user, order, amount
| stats sum(amount) as total_volume, count() as order_count by bin(5m) as bucket
| sort bucket asc
```

After a `stats ... by bin(5m)`, the row shape is the aggregated grouping -- `@timestamp` is gone from the output. Sort by the bucket alias (or the bin expression directly), not by `@timestamp`. This trips everyone the first time.

`parse` with `*` wildcards is the cheap, fast extractor; it's also fairly forgiving and handles most structured-but-not-JSON logs (`user=alice path=/api/checkout status=200`). For regex you use `parse @message /(?<field_name>regex)/` with **named capture groups** -- bare `parse @message /pattern/` without named captures won't bind any fields downstream. Regex is slower than wildcard `*` parsing; prefer wildcards when your log format is consistent.

**Time-series count -- the bread-and-butter trend query:**

```
fields @timestamp, @message
| filter @message like /5\d\d/
| stats count() as five_hundreds by bin(5m)
| sort @timestamp asc
```

`bin(5m)` is the time-bucketing function -- you can pin to any duration (`1m`, `15m`, `1h`). This is the query that powers a Logs-Insights-based dashboard widget.

**Top-N pattern -- "which 10 things broke the most":**

```
fields @timestamp, @message
| filter level = "ERROR"
| parse @message /error_code=(?<error_code>\w+)/
| stats count() as occurrences by error_code
| sort occurrences desc
| limit 10
```

This is the post-incident query: "where's the noise coming from?" Use it during a postmortem to identify which downstream error code dominated the failure mode.

### IA-class restrictions

A handful of Logs Insights commands are **not supported on Infrequent Access log groups**:

- `pattern` -- automatic log pattern detection.
- `diff` -- comparison against historical baselines.
- `unmask` -- reveal data-protection-masked fields (Data Protection isn't on IA anyway).
- `filterIndex` -- index-accelerated filtering on Infrequent Access groups isn't supported.

If you write a query using these on an IA group, it errors out. Most production queries use only `fields | filter | parse | stats | sort | limit`, so this rarely bites in practice, but the `pattern` command is genuinely useful for ad-hoc incident triage and you lose it on IA groups.

### Save your queries

You can save queries per-account ("Saved queries" in the Logs Insights console). The pattern that works: every team maintains a "Top 5" -- error-rate-by-service, p95-by-endpoint, top-error-codes, slowest-DB-calls, cold-start-frequency -- as saved queries. When the on-call gets paged at 3 AM, they click the saved query, not retype the syntax from memory. Terraform supports `aws_cloudwatch_query_definition` for managing these as code.

### The investigation funnel -- the senior playbook for live triage

When you get paged at 2 AM with no pre-built dashboard, treat Logs Insights as a **funnel of four queries**, not a search. Each query narrows the next; skipping a step doesn't save time, it produces a Slack post nobody can act on.

**Step 1 -- Confirm: is this real?** (30-min window)

```
stats pct(totalLatencyMs, 99) as p99,
      pct(totalLatencyMs, 50) as p50,
      count(*) as reqs
   by bin(1m) as bucket
| sort bucket asc
```

What you're reading: p99 trajectory (is the spike sustained or one-off?), p50 divergence (is the median moving too -- a global problem -- or only the tail?), and `reqs` as a volume guard.

**Step 2 -- Diagnose: which dependency?** (window narrowed to last 10-15 min)

```
filter ispresent(downstreamService)
| stats pct(downstreamLatencyMs, 99) as p99,
        count(*) as calls
   by downstreamService
| sort p99 desc
```

What you're reading: a ranked list of downstreams by p99. The worst offender is usually unambiguous.

**Step 3 -- Scope: global or tenant-specific?**

```
filter downstreamService = "stripe"
| stats pct(downstreamLatencyMs, 99) as p99,
        count(*) as calls
   by merchantId
| filter calls > 20
| sort p99 desc
| limit 10
```

What you're reading: flat distribution = global infrastructure problem (call your runbook). Lopsided = one tenant's traffic pattern broke (call that customer's TAM). **Critical**: the `filter calls > 20` is the **volume guard** -- without it, a single 1-call merchant with one unlucky spike tops the list and wastes 20 minutes of triage.

**Step 4 -- Smoking gun: evidence for the channel**

```
fields @timestamp, @message
| filter downstreamService = "stripe" AND downstreamLatencyMs > 1000
| sort downstreamLatencyMs desc
| limit 5
```

What you're reading: 5 full log lines to paste in the incident channel. Every structured field rides along -- request ID, user ID, route -- so the on-call engineer arriving has everything they need without retyping the investigation.

**Three principles that distinguish a senior triage from a junior one:**

1. **`pct()` for latency, never `avg()`.** Averages lie about the experience tail latency creates. `pct(latencyField, 99)` is the workhorse function.
2. **Always include a volume guard.** `count(*) as calls` next to every percentile, plus `filter calls > N` before any `sort`. Percentiles on small denominators are noise.
3. **Time range is the cost lever, not query shape.** Start at 30 minutes. Narrow as you find the incident. Cancel queries you don't need. **Six 24-hour queries on a 5 TB log group is ~30 TB scanned at $0.005/GB = $150 in 20 minutes.** Never leave a Logs Insights tab on "last 24h" while running multiple queries.

**The channel post structure** -- treat it as a deliverable, not a transcript: TL;DR -> what changed -> scope -> evidence -> action taken -> what's needed. People join the bridge over time and scan; build for that audience.

---

## Part 6: Alarms -- The Red Lights

CloudWatch has three flavours of alarm. Most teams use only the first; the second and third are how senior engineers cut their pager noise in half.

### Metric alarm -- one gauge, one threshold

The simplest form: one metric crosses a threshold for N of M evaluation periods, alarm fires.

```hcl
resource "aws_cloudwatch_metric_alarm" "checkout_error_rate" {
  alarm_name          = "checkout-error-rate-high"
  alarm_description   = "Checkout service error rate > 5% for 3 of 5 minutes"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5     # M -- the window
  datapoints_to_alarm = 3     # N -- how many must breach
  threshold           = 0.05
  treat_missing_data  = "notBreaching"

  metric_query {
    id = "error_rate"
    expression = "errors / invocations"
    label      = "ErrorRate"
    return_data = true
  }
  metric_query {
    id = "errors"
    metric {
      namespace   = "AWS/Lambda"
      metric_name = "Errors"
      period      = 60
      stat        = "Sum"
      dimensions  = { FunctionName = "checkout-service" }
    }
  }
  metric_query {
    id = "invocations"
    metric {
      namespace   = "AWS/Lambda"
      metric_name = "Invocations"
      period      = 60
      stat        = "Sum"
      dimensions  = { FunctionName = "checkout-service" }
    }
  }

  alarm_actions = [aws_sns_topic.pagerduty.arn]
  ok_actions    = [aws_sns_topic.pagerduty.arn]
}
```

Three knobs people get wrong:

- **`evaluation_periods` (M)** -- how many consecutive periods are considered. Set to 5 if your period is 60s -> 5-minute window.
- **`datapoints_to_alarm` (N)** -- how many of those M must breach. **N-of-M defeats single-spike false positives.** "Alarm if 3 of the last 5 one-minute averages exceeded 80%" is far more robust than "alarm if the last one did."
- **`treat_missing_data`** -- one of `missing` (treat the gap as a missing datapoint; the alarm doesn't evaluate it at all, so state is effectively held until enough non-missing datapoints accumulate), `notBreaching` (count the gap as OK), `breaching` (count the gap as in-alarm), `ignore` (keep the current alarm state and skip evaluation entirely -- the "freeze the state" option). The default is `missing`, which is rarely what you want -- it interacts confusingly with M-of-N evaluation. **For most alarms, set `notBreaching`** so a temporary metric gap doesn't trip the alarm; use `ignore` only when you genuinely want the alarm to remain in its current state during data gaps (e.g., on/off binary signals).

Alarms cost $0.10 per alarm per month (standard resolution) or $0.30 per alarm per month (high-resolution). The cost is per **alarm**, not per evaluation -- so you pay even when the alarm sits idle in `OK` or `INSUFFICIENT_DATA`. **Orphan alarms** in `INSUFFICIENT_DATA` (their metric stopped publishing because the resource was deleted) silently bill forever. Audit periodically with `aws cloudwatch describe-alarms --state-value INSUFFICIENT_DATA` and delete the dead ones.

### Composite alarm -- the red light wired through a logic gate

A **composite alarm** is a boolean function (AND / OR / NOT) of other alarms. It does not watch a metric directly; it watches the *state* of other alarms via the `ALARM_RULE` DSL.

The killer use case: **suppress noisy alarms when something else is true.** Example -- you have an "error rate > 5%" alarm. But during a deploy that drops traffic to zero, error rate spikes are statistically meaningless (1 error out of 5 requests is 20% but it's noise). Wrap your error-rate alarm in a composite that only fires when traffic is also normal:

```hcl
resource "aws_cloudwatch_metric_alarm" "low_traffic" {
  alarm_name          = "checkout-low-traffic"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  threshold           = 100
  metric_name         = "Invocations"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  dimensions          = { FunctionName = "checkout-service" }
  treat_missing_data  = "breaching"
  # NOTE: this alarm has NO alarm_actions -- it's purely a signal for the composite
}

resource "aws_cloudwatch_composite_alarm" "checkout_pageable" {
  alarm_name        = "checkout-error-rate-pageable"
  alarm_description = "Pages only if error rate is high AND traffic is normal"

  alarm_rule = join(" AND ", [
    "ALARM(\"${aws_cloudwatch_metric_alarm.checkout_error_rate.alarm_name}\")",
    "NOT ALARM(\"${aws_cloudwatch_metric_alarm.low_traffic.alarm_name}\")",
  ])

  alarm_actions = [aws_sns_topic.pagerduty.arn]
  ok_actions    = [aws_sns_topic.pagerduty.arn]
}
```

What this does: page only when both "error rate is high" AND "traffic is NOT low." During a deploy with low traffic, the page is silenced. This pattern -- pair a noisy metric alarm with a silence-during-known-anomaly composite -- is the **single largest pager-noise reduction lever** in CloudWatch.

A few gotchas:

- **`ALARM_RULE` requires the literal `ALARM("name")` function with the alarm name in double quotes inside the parens.** Bare `ALARM(name)` is rejected by AWS at create time -- and Terraform string interpolation will hand you the unquoted form unless you escape the quotes (see the example above). Typos in the alarm name silently never fire -- the composite just sits in `OK` forever.
- **Composite alarms cost the same $0.10/month** as a metric alarm.
- **You can suppress an entire composite alarm** with `actions_suppressor` -- e.g., "don't page during the maintenance window when this other alarm is in ALARM."
- **Composite alarms can be nested** (one composite referencing other composites), but be careful with chains deeper than 2 -- debugging gets ugly.

### Anomaly detection alarm -- the gauge with a learned rhythm

For metrics with **strong daily or weekly seasonality** -- think traffic on a B2B SaaS app (low on weekends, high 9-5), login counts, queue depth -- a static threshold is the wrong tool. Set it loose enough to not page during the daily peak and you'll miss real incidents at night. Set it tight enough to catch night-time anomalies and you'll page every morning at 9 AM.

**Anomaly detection alarms** train a band around the metric using historical data and fire when the metric leaves the band:

```hcl
resource "aws_cloudwatch_metric_alarm" "login_count_anomaly" {
  alarm_name          = "login-count-anomalous"
  comparison_operator = "LessThanLowerOrGreaterThanUpperThreshold"
  evaluation_periods  = 2
  threshold_metric_id = "expected_range"

  metric_query {
    id          = "actual"
    return_data = true
    metric {
      namespace   = "MyApp/Auth"
      metric_name = "LoginsPerMinute"
      period      = 60
      stat        = "Sum"
    }
  }
  metric_query {
    id          = "expected_range"
    expression  = "ANOMALY_DETECTION_BAND(actual, 2)"  # band width = 2 std devs
    return_data = true
  }

  alarm_actions = [aws_sns_topic.pagerduty.arn]
}
```

Constraints:

- **2 weeks of training data minimum** -- a brand new metric won't get a useful band for two weeks.
- **3x the cost** of a standard alarm (~$0.30/month). The mechanism: an anomaly-detection alarm evaluates three "metrics" -- the actual metric, the upper band, and the lower band -- and you pay $0.10/month for each (3 x $0.10 = $0.30).
- **Deploys and incidents during training poison the model.** If you had a 12-hour outage during the first week of training, the model "learned" that as normal and won't fire on the next one.
- **Best paired with composite alarms.** Anomaly detection catches "weird"; composite filters down to "actually-paging-worthy." The pattern:

```
ALARM("login_count_anomalous") AND NOT ALARM("deploy_in_progress")
                                AND NOT ALARM("known_maintenance_window")
```

### The alarm decision framework

| Situation | Use |
|-----------|-----|
| Single metric, hard threshold (CPU > 80, latency p95 > 500ms, queue > 1000) | Metric alarm |
| Multiple conditions must all hold to page (high error AND normal traffic) | Composite alarm wrapping metric alarms |
| Metric has daily / weekly seasonality, no good fixed threshold | Anomaly detection alarm |
| You want to suppress paging during deploys / maintenance | Composite with `NOT ALARM("deploy_marker_alarm")` |
| Metric publishes intermittently | Metric alarm with `treat_missing_data = "notBreaching"` |
| You want to page on a derived quantity (error rate, p95 ratio) | Metric alarm with `metric_query` expression |

### The alarm-cost stack you need to memorize

Cost trips up senior interviews because the components stack non-obviously. Burn this in:

| Component | Monthly cost | Mechanism |
|-----------|--------------|-----------|
| Standard-resolution metric alarm | **$0.10** | One alarm = one billing unit. |
| High-resolution metric alarm | **$0.30** | 3x because it evaluates 3x more often (per 10s vs per 60s). |
| Composite alarm | **$0.10** | Same as a metric alarm. The boolean glue is free; you pay for the alarm itself. |
| Anomaly detection alarm | **$0.30** | 3x via mechanism: the alarm evaluates **3 sub-metrics** -- the actual metric + the upper band + the lower band, each billed at $0.10. |
| `INSUFFICIENT_DATA` orphan alarm | **$0.10/mo (or $0.30 for HR/anomaly)** | An alarm still bills in `INSUFFICIENT_DATA` -- they just silently accumulate as resources are deleted. |

So the senior-savvy pattern of "anomaly detection on rate, static threshold on volume floor, composite to AND them together" costs:

```
$0.30 (anomaly)  +  $0.10 (volume floor)  +  $0.10 (composite glue)  =  $0.50/month total
```

Five times the cost of a single metric alarm and still under a dollar. The highest-ROI move in CloudWatch by a wide margin -- because the composite is free glue, not a third $0.10 line. Junior engineers often double the composite cost in their heads; it isn't.

### The rate-based-alarm volume-guard principle

A senior pattern that bites every rate-based alarm: **error rate (or any ratio) is meaningless when the denominator is small.** 3 errors out of 8 requests is mathematically 37.5%, but it's noise. You cannot tune a single threshold around it without breaking the metric on real production traffic.

> **The principle: rate tells you whether something is bad; volume tells you whether the rate is trustworthy. Both questions must be answered before you page a human.**

The pattern: pair every rate-based metric alarm with a **volume-floor alarm** (e.g., `Invocations > 100 in 5 minutes`) and AND them together in a composite. The page only fires when the rate is anomalous *and* the volume is high enough for the math to mean anything. PagerDuty stays silent on quiet Saturday mornings; fires immediately on a Saturday-afternoon outage with real traffic.

This applies equally to:

- Error rate (errors / requests)
- Latency p99 (collapses on N=5 requests)
- Conversion rate (signups / visits)
- Cache hit ratio (hits / lookups)

Anything where the denominator can collapse to single digits needs a volume guard. **Without one, anomaly detection trained on those noisy periods will learn the noise as normal** -- a slightly quieter page Saturday morning at the cost of missing a real Saturday-night outage.

### The 2-week training-window gotcha for anomaly composites

A subtle but important production trap: if you set up `ALARM(anomaly_rate) AND ALARM(volume_floor)` on the day of deployment, **the anomaly alarm sits in `INSUFFICIENT_DATA` for two weeks** while it trains. During that window, the composite's `AND` evaluation can never resolve to `ALARM` -- you are silently flying blind on the very metric you just set up to monitor.

The fix: **keep your existing static-threshold metric alarm running in parallel for the entire training period**, then retire it once anomaly detection has stabilized and you've confirmed the band fits. The 2x alarm bill (~$0.20/mo extra) for those two weeks is cheap insurance against a "we set up an alarm and it never fired during the outage" postmortem.

Bonus: **never recreate the anomaly detection resource lightly.** Each `terraform destroy && terraform apply` resets the training band to zero. Use `terraform state rm` and import if you must rename it.

---

## Part 7: Synthetics and RUM -- The Showroom Walker and the Customer Survey

Two products that pair naturally and answer different halves of "is my user-facing surface healthy?"

### Synthetics canaries -- the dummy walker

A **canary** is a scheduled headless-browser script (Puppeteer for Node, Playwright as of 2024) that performs a critical user journey on a fixed cron, captures screenshots and HAR files, and emits success/failure metrics to CloudWatch. The factory analogy: a person who walks the showroom floor every 5 minutes pretending to be a customer, verifies the doors open, the lights work, and the demo product still rings up at the register.

Use cases:

- **Heartbeat probe** -- does `GET /healthz` return 200?
- **API canary** -- does a multi-step API workflow succeed end-to-end?
- **GUI workflow** -- can a user log in, add an item to cart, and reach the checkout page?
- **Endpoint SSL expiry** -- watch for certs about to expire.

Canary Terraform (simplified):

```hcl
resource "aws_synthetics_canary" "checkout_journey" {
  name                 = "checkout-journey"
  artifact_s3_location = "s3://${aws_s3_bucket.canary_artifacts.id}/canary/"
  execution_role_arn   = aws_iam_role.canary.arn
  handler              = "index.handler"
  runtime_version      = "syn-nodejs-playwright-7.0"
  zip_file             = "canary-source.zip"

  schedule {
    expression = "rate(5 minutes)"
  }
  run_config {
    timeout_in_seconds = 60
    memory_in_mb       = 1024
  }
  success_retention_period = 7
  failure_retention_period = 31
  tags = { Service = "checkout" }
}

resource "aws_cloudwatch_metric_alarm" "canary_failure" {
  alarm_name          = "checkout-canary-failure"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  datapoints_to_alarm = 2
  threshold           = 100   # SuccessPercent < 100 for 2 consecutive
  namespace           = "CloudWatchSynthetics"
  metric_name         = "SuccessPercent"
  dimensions          = { CanaryName = aws_synthetics_canary.checkout_journey.name }
  period              = 60
  statistic           = "Average"
  treat_missing_data  = "breaching"
  alarm_actions       = [aws_sns_topic.pagerduty.arn]
}
```

The under-rated gotcha: **canary failures do NOT auto-page**. The canary writes success/fail metrics to CloudWatch; you have to wire an alarm to those metrics, and the alarm has to fire to SNS / PagerDuty. The number of "we had a canary, we thought we were covered" stories that ended in surprise outages is unsettling. Always pair a canary with an alarm.

Other gotchas:

- **Runtime versions matter.** `syn-nodejs-puppeteer-*` is being deprecated in favour of `syn-nodejs-playwright-*`. Pin the version explicitly.
- **Each canary run is a billed Lambda-like execution.** Frequent canaries get expensive at scale -- a 1-minute canary is $42/month per canary just for executions.
- **Canary scripts can do `cw.put_metric_data`** to publish custom step-level metrics (e.g., "checkout-page-loaded-in-X-ms"). Use this to alert on degradation, not just outage.
- **Canaries run from AWS-owned IPs in specific regions.** They don't measure your real user geography. RUM does.

### RUM -- the customer survey

**RUM (Real User Monitoring)** is a JavaScript snippet you embed in your frontend. Real users' browsers report back page load times, JS errors, route changes, and Core Web Vitals (LCP, INP, CLS) to CloudWatch RUM. The factory analogy: a clipboard given to every customer leaving the showroom, asking "how was the experience?"

The complementary pattern is:

- **Synthetics tells you "is the site up?"** before users complain. Predictable, controlled, no user impact.
- **RUM tells you "for whom, and where, is it slow?"** after the fact. Real geography, real devices, real third-party-script noise.

RUM is the right answer when:

- You need to know which countries / ISPs / browsers experience your app slowly.
- You want JS error tracking (uncaught exceptions, unhandled rejections) tied to the page where they fired.
- You're measuring Core Web Vitals for SEO / UX SLOs.

Limitations:

- **RUM data is not in the Logs Insights query plane.** It has its own console + dashboard. You cannot join RUM events with your application logs in a single query. The cross-product correlation is via dashboards, not queries.
- **PII risk** -- RUM captures URLs, which can contain user IDs / tokens. Configure data-collection to strip query strings or sensitive paths.
- **Cost is per session + per event** -- a noisy public site with JS errors firing constantly can be expensive.

Pair them: a Synthetics canary for the heartbeat alarm, RUM for the post-deploy regression dashboard. Both write to CloudWatch, both can drive metric alarms.

---

## Part 8: The Insights Suite Map -- Five Products People Conflate

CloudWatch has **five** products with "Insights" in the name (or close to it), and people conflate them constantly. The instrumentation-room analogy: these are five specialist inspectors, each looking at a different machine family with different tools.

| Product | What it is | Best for | What it costs |
|---------|------------|----------|---------------|
| **Container Insights** | EKS / ECS infra metrics + (post-Nov-2023) control-plane + per-pod granularity | Cluster CPU/mem/disk/network, pod-level resource pressure, control-plane health | Unified per-cluster pricing including ingestion + storage |
| **Lambda Insights** | Lambda runtime telemetry via the Lambda Insights extension layer | Memory, CPU, network, file descriptors per invocation -- the stuff `Duration`/`Errors`/`Invocations` don't show | Extension layer adds ~$0.50/million invocations + standard CW costs |
| **Application Insights** | AWS-managed app-stack monitoring with auto-discovery and pattern recognition | Off-the-shelf monitoring for SQL Server, IIS, MySQL, SAP, etc. -- when you don't want to design dashboards yourself | Flat per-resource fee |
| **Application Signals** (GA 2024) | AWS's APM, auto-instrumented via ADOT | RED metrics + SLOs + service map for Java / Python / Node services | Per-trace, per-metric, per-signal cost; opt-in per service |
| **ServiceLens** | Older console feature that stitches X-Ray traces + CloudWatch metrics + logs into a service map | Pre-Application-Signals service maps; still useful for non-APM workloads | Per-trace and X-Ray cost |

### Container Insights -- the container-line inspector

Container Insights ships pre-built dashboards for ECS and EKS: cluster CPU/memory/disk/network, per-service pod counts, per-pod resource pressure. **The November 2023 "enhanced observability" launch** added control-plane component metrics (API server, etcd, scheduler) and per-pod/per-container/Kube-State granularity, bringing it close to feature-parity with kube-prometheus-stack for most operational use cases.

It's an EKS add-on. Enable it via the AWS console, IaC, or the `amazon-cloudwatch-observability` add-on:

```hcl
resource "aws_eks_addon" "cloudwatch_observability" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "amazon-cloudwatch-observability"
  service_account_role_arn = aws_iam_role.cw_observability.arn

  # Enable enhanced observability
  configuration_values = jsonencode({
    containerLogs = { enabled = true }
    agent = {
      config = {
        logs = {
          metrics_collected = {
            kubernetes = {
              enhanced_container_insights = true
              accelerated_compute_metrics = false  # set true for GPU nodes
            }
          }
        }
      }
    }
  })
}

resource "aws_iam_role" "cw_observability" {
  name = "${var.cluster_name}-cw-observability"
  assume_role_policy = data.aws_iam_policy_document.cw_observability_assume.json
}

resource "aws_iam_role_policy_attachment" "cw_observability_managed" {
  role       = aws_iam_role.cw_observability.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}
```

This installs the CloudWatch agent DaemonSet + Fluent Bit DaemonSet across the cluster. Two gotchas: **enhanced observability is opt-in** (the `enhanced_container_insights: true` flag); without it, you get the old-style cluster aggregates. And **the IAM role needs IRSA** -- in production, attach via an annotated service account, not via node-IAM.

**When to pick Container Insights vs kube-prometheus-stack:**

| Factor | Container Insights wins | kube-prometheus-stack wins |
|--------|------------------------|---------------------------|
| Operational overhead | Zero (managed add-on) | High (operator + storage + HA) |
| AWS-native integrations (X-Ray, Application Signals, Lambda) | Yes, deeply | No, bridge via OTel |
| PromQL expressivity | Not native -- use Metric Insights (SQL-like) or stream metrics into AWS Managed Prometheus for PromQL | Native |
| Cardinality tolerance | Disciplined; high-cardinality dimensions are expensive | High (this is what Prom is built for) |
| Cross-account observability | Native | Requires Thanos / Mimir |
| Cost at 50+ nodes | Often higher | Often lower |
| Per-pod custom metrics | Via EMF only | Via instrumented `/metrics` endpoint |

The honest answer: if you already run kube-prometheus-stack happily, you don't need Container Insights enhanced observability *as well*. But for AWS-native organizations with cross-account observability needs and no Prometheus team, Container Insights is the right answer.

### Lambda Insights -- the welding-station specialist

`Invocations`, `Errors`, `Duration`, `Throttles` ship for free with Lambda. **Lambda Insights** adds memory usage, CPU usage, network, file descriptors, and a per-invocation correlation -- the stuff that tells you *why* your function is slow, not just *that* it is.

It's an opt-in **extension layer**:

```hcl
resource "aws_lambda_function" "checkout" {
  function_name = "checkout-service"
  # ... your normal config ...

  layers = [
    # Lambda Insights extension -- region-specific ARN
    "arn:aws:lambda:us-east-1:580247275435:layer:LambdaInsightsExtension:53",
  ]
}

resource "aws_iam_role_policy_attachment" "checkout_lambda_insights" {
  role       = aws_iam_role.checkout.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchLambdaInsightsExecutionRolePolicy"
}
```

Gotchas:

- **Cross-region:** Lambda Insights in `us-east-1` only sees `us-east-1` functions. If you have a multi-region setup, you enable per region.
- **Extension layer adds cold-start overhead** -- typically 50-100ms. Not free.
- **Cost compounds with invocation count.** A high-volume function pays both the extension cost and the metric storage.
- **Best paired with Application Signals** if you want SLOs.

### Application Insights -- the auto-discovery generalist

A different beast: **Application Insights** is an AWS-managed monitoring product that auto-discovers your application components (SQL Server, IIS, Java, SAP, MySQL), recognizes common failure patterns, and sets up dashboards + alarms for you. It is the "I don't want to design monitoring" answer.

It is most useful for **lift-and-shift workloads on EC2** -- you import a Windows SQL Server app to AWS and let Application Insights wire it up. For cloud-native estates running EKS / Lambda, it's mostly redundant against Container Insights + Application Signals.

### Application Signals -- AWS's APM (GA 2024)

**Application Signals** is AWS's APM offering. Auto-instrumentation via the AWS Distro for OpenTelemetry (ADOT) -- you turn it on per service, and you get:

- **RED metrics per service operation** (Rate, Errors, Duration) automatically.
- **SLO definitions** with budget-burn tracking.
- **Service map** showing call patterns and latencies between services.

Critically: **Application Signals is AWS's opinionated take on OpenTelemetry semantic conventions for APM.** It uses standard OTel SDKs underneath -- you can write a custom OTel instrumentation and Application Signals will pick it up.

Constraints:

- **Auto-instrumentation requires the ADOT layer / sidecar to be enabled.** Not free; opt-in per service.
- **Java, Python, Node, and .NET are the well-supported runtimes.** Go is partial; other languages are manual-instrumentation only.
- **It is the natural answer to "I want APM on AWS without running Datadog or Honeycomb."**

### ServiceLens -- the older trace-stitcher

The console feature that predates Application Signals. ServiceLens stitches X-Ray traces + CloudWatch metrics + logs into a service map. It's still useful for **non-Application-Signals workloads** -- if you have X-Ray instrumentation but haven't migrated to Application Signals, ServiceLens is the unified view you'd reach for.

You'll see it referenced in older docs but most new builds in 2026 should jump straight to Application Signals.

---

## Part 9: CloudWatch vs Prometheus vs OpenTelemetry -- The Decision That Matters

You already know Prometheus deeply (April 20). You know OpenTelemetry deeply (April 27). The question that matters at $50K MRR is: **when do I use CloudWatch instead of Prometheus, when do I use both, and how does OTel bridge them?**

### The mental model

| Property | CloudWatch | Prometheus | OpenTelemetry |
|----------|------------|------------|---------------|
| Model | Managed AWS-integrated sink (push) | Self-managed pull-from-`/metrics` | Universal transport / collector |
| Cardinality philosophy | Disciplined, expensive at high cardinality | Tolerant, built for high cardinality | Neutral (it's a pipe) |
| Query language | CloudWatch Metric Insights (SQL-like) + Metric Math; native PromQL only via AWS Managed Prometheus | PromQL | LogQL / TraceQL / PromQL via backend |
| Multi-region native | Yes -- but bills per region | Self-built (Thanos / Mimir) | Yes via collector fanout |
| Cost shape | Per metric / per ingestion / per scan | Self-hosted infra + storage | Per-collector + downstream backend |
| AWS service integration | First-party for every AWS service | Bridge via exporters or AWS Managed Prometheus | Bridge via OTel receivers |
| Self-instrumented apps | EMF (via Powertools) | `/metrics` endpoint | OTel SDK |
| Ops burden | Zero (managed) | High (operator + HA + storage) | Medium (collector ops) |

### The cost inflection (InfraCloud benchmark)

The InfraCloud blog has a useful benchmark: at a 100-node EKS cluster scale:

- **CloudWatch** ingested ~19,000 metrics (because of dimension discipline -- AWS service metrics dominate, custom metrics with bounded dimensions).
- **Prometheus** stored ~1.5 million time series (kube-state-metrics + cAdvisor + node-exporter at default cardinality).
- **Self-managed Prometheus** cost ~$3,000/month (compute + EBS + ops overhead amortized).
- **Equivalent CloudWatch** cost ~$8,600/month (custom metric fees + log ingestion + Insights queries).

The rough inflection point:

- **Below ~20 nodes:** CloudWatch wins on operational overhead. You don't run Prometheus.
- **20-50 nodes:** It depends -- if your team has Prometheus operational expertise already, run it; if not, CloudWatch.
- **Above ~50 nodes:** Self-managed Prometheus is cheaper *and* gives you PromQL expressivity. Run it. Use AWS Managed Prometheus (AMP) if you don't want to self-host.
- **Always:** Use OTel as the transport so you're not locked in.

### The "both via OTel" pattern

The honest production answer for most 50-node-plus AWS estates: **use OTel collector to fan out**. The collector receives metrics once (from OTel SDK in your apps, or from Prometheus scrape) and exports to **both** CloudWatch (for AWS-native alarms + cross-account dashboards) and a Prometheus-compatible backend (for high-cardinality dashboards and PromQL alerting). Configuration:

```yaml
# otel-collector-config.yaml
receivers:
  prometheus:
    config:
      scrape_configs: [...]  # standard Prom scrape config
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }

processors:
  batch: {}
  resourcedetection:
    detectors: [eks, env, system]
  memory_limiter:
    limit_mib: 1024

exporters:
  awsemf:
    namespace: "MyApp"
    log_group_name: "/aws/otel/metrics"
    log_stream_name: "default"
    dimension_rollup_option: NoDimensionRollup
  prometheusremotewrite:
    endpoint: "https://prom.example.com/api/v1/write"

service:
  pipelines:
    metrics:
      receivers: [prometheus, otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters: [awsemf, prometheusremotewrite]
```

**The `awsemf` exporter is the magic** -- it converts OTel metrics into EMF JSON log lines and writes them to a CloudWatch Log group, where CloudWatch auto-extracts them as metrics. You get OTel semantic conventions on the source side and CloudWatch metric storage on the sink side, with `prometheusremotewrite` running in parallel for high-cardinality dashboards.

And: CloudWatch increasingly accepts OTLP-format ingestion via the OTel Collector's `awsemf` exporter, and **Metric Streams** can push raw CloudWatch metrics to a Prometheus-compatible backend via Firehose. The impedance mismatch is shrinking, but writing PromQL directly against CloudWatch's metric store is **not yet a thing in 2026** -- for PromQL on CloudWatch metrics, you stream them into AWS Managed Prometheus or use Metric Insights (CloudWatch's SQL-like query language) instead. Don't confuse "CloudWatch can ingest OTel" with "you can write PromQL against the CloudWatch console" -- they aren't the same thing.

### Log backend decision: CloudWatch Logs vs Loki vs OpenSearch vs S3+Athena

| Backend | Strengths | Cost shape | When |
|---------|-----------|------------|------|
| **CloudWatch Logs (Standard)** | AWS-native, Logs Insights, real-time tail, metric extraction, alarming | Per ingestion GB + storage + scanned GB | Default for AWS-native estates; ops logs |
| **CloudWatch Logs (IA)** | Same minus features; 50% cheaper ingest | Same shape | Vended/audit logs |
| **Loki** | Labels-as-index, cheap on S3, LogQL, lower cost at scale | Self-managed infra | Cloud-native estates with Grafana; >TB/day |
| **OpenSearch** | Full-text search, dashboards, complex aggregations | Managed cluster + storage | When grep-y full-text search is the main use case |
| **S3 + Athena** | Cheapest cold storage; SQL queries on-demand | $/GB storage + per-query | Compliance archives, post-hoc forensics |

The pragmatic default for AWS-native estates: **CloudWatch Logs Standard for hot ops logs (with 30-day retention), CloudWatch Logs IA for vended logs (90-day retention), S3 + Athena via Firehose for long-term archive (1-7 year retention)**. Loki and OpenSearch enter the picture when you outgrow Logs Insights' cardinality or query limits.

---

## Part 10: CloudWatch vs CloudTrail -- The Boundary

Tomorrow's topic is CloudTrail. Today's boundary statement, because people conflate them constantly:

| | CloudWatch Logs | CloudTrail |
|-|-----------------|------------|
| What it captures | Application output, OS logs, container stdout | AWS API calls (who called what, when, from where) |
| Who writes | Your application + OS + AWS services emitting operational logs | AWS itself, automatically, for every API call |
| Use case | "What did my app print?" | "Who called `iam:CreateUser` last Tuesday?" |
| Default state | You must opt in per log group | On by default for management events |
| Retention | Configurable per log group | Event history shows last 90 days of **management events only**; durable & complete via S3 trails |
| Query | Logs Insights | CloudTrail Lake (Athena-like SQL) |
| Where it lives | `cloudwatch.amazonaws.com` | `cloudtrail.amazonaws.com` |
| Can they cross? | Yes -- CloudTrail can write to a CloudWatch log group for real-time alerting | Yes (same direction only) |

The mental model: **CloudWatch Logs is the operational logbook -- what the factory floor printed during the shift. CloudTrail is the security guard's ledger -- who entered the building and which doors they opened.** They sometimes share infrastructure (CloudTrail writes to a CloudWatch log group when you want EventBridge alerting on API calls), but they answer fundamentally different questions and bill on fundamentally different models.

Tomorrow's doc is the full CloudTrail deep dive. Today, just internalize the boundary so you don't reach for the wrong tool.

---

## Part 11: Dashboards -- Stitching It Together

A CloudWatch **dashboard** is a JSON definition of widgets: metric graphs, text annotations, alarm-status grids, Logs Insights query results. Manage them in Terraform and version them in Git:

```hcl
resource "aws_cloudwatch_dashboard" "checkout_service" {
  dashboard_name = "checkout-service-overview"
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "Invocations + Errors (5m)"
          region = var.region
          metrics = [
            ["AWS/Lambda", "Invocations", "FunctionName", "checkout-service", { stat = "Sum" }],
            [".",          "Errors",     ".",            ".",                { stat = "Sum", color = "#d62728" }],
          ]
          period = 60
          view   = "timeSeries"
          yAxis  = { left = { min = 0 } }
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "p95 Duration"
          region = var.region
          metrics = [
            ["AWS/Lambda", "Duration", "FunctionName", "checkout-service", { stat = "p95" }],
          ]
          period = 60
          view   = "timeSeries"
        }
      },
      {
        type   = "text"
        x      = 0
        y      = 6
        width  = 24
        height = 2
        properties = {
          markdown = "## Runbook\n[Pager runbook](https://wiki.example.com/runbooks/checkout)\n[Owner: payments team](mailto:payments@example.com)"
        }
      },
      {
        type   = "log"
        x      = 0
        y      = 8
        width  = 24
        height = 8
        properties = {
          title  = "Top errors in last hour"
          region = var.region
          query  = "SOURCE '/aws/lambda/checkout-service' | fields @timestamp, @message | filter level = \"ERROR\" | stats count() as occurrences by error_code | sort occurrences desc | limit 10"
          view   = "table"
        }
      },
      {
        type   = "alarm"
        x      = 0
        y      = 16
        width  = 24
        height = 4
        properties = {
          title = "Alarms"
          alarms = [
            aws_cloudwatch_metric_alarm.checkout_error_rate.arn,
            aws_cloudwatch_composite_alarm.checkout_pageable.arn,
          ]
        }
      },
    ]
  })
}
```

Four widget types covered: metric chart, free-form text (markdown, links to runbooks), Logs Insights query result, alarm-status grid. Mixing all four into one dashboard creates an **operations homepage** -- the page the on-call opens at the start of every shift to verify nothing's on fire.

The under-rated dashboard pattern: **embed the runbook link as a text widget** at the top. When the page wakes you at 3 AM, the dashboard you click into already has the runbook one click away.

---

## Part 12: Cost Optimization -- Where Five-Figure Bills Come From

Every CloudWatch cost surprise traces back to one of seven causes. Memorize the seven, audit for them quarterly, and you'll stay within budget.

### 1. The cardinality trap (the #1 cause)

Already covered three times. Recap: every unique `namespace + name + dimension_combo` is $0.30/month (tiered down). One unbounded dimension (`userId`, `requestId`, `orderId`, `sessionId`, IP) on a moderate-traffic app = thousands to millions of metrics. Audit periodically: `aws cloudwatch list-metrics --namespace MyApp/Orders | jq '.Metrics | length'`. If the number is unexpectedly high, walk the dimensions.

### 2. High-resolution metrics where standard would do

4x the cost, rolls up to 1-minute after 3 hours anyway. Use high-resolution only for active short-term burst monitoring; otherwise standard.

### 3. Missing retention on log groups

Default = Never Expire. Verbose Lambda functions can ingest GB/day; over years that's TB stored. Set retention on **every log group**:

```bash
aws logs describe-log-groups --query 'logGroups[?!retentionInDays].[logGroupName]' --output text \
  | xargs -I{} aws logs put-retention-policy --log-group-name {} --retention-in-days 30
```

(Adjust 30 days per log group's actual need; this is a starter sweep.)

### 4. Standard class where IA would do

VPC Flow Logs, API Gateway access logs, ALB access logs -- 50% savings on ingestion by switching to IA. Remember the irreversibility -- delete + recreate, accept the historical-data loss.

### 5. Filter at the source

The cheapest log byte is the one you never ingest. Configure the CloudWatch agent / Fluent Bit / your application's logger to **drop noisy log lines before they hit ingestion**. Examples:

- Drop health-check log lines (`GET /healthz` flood).
- Drop INFO-level logs in production; keep WARN+.
- Drop verbose third-party library output.

```yaml
# Fluent Bit filter example -- drop healthz noise before ingestion
[FILTER]
    Name    grep
    Match   kube.*
    Exclude log  /healthz|/ready|/metrics
```

### 6. Anomaly detection where threshold would do

Anomaly detection alarms cost 3x. Only use them where a fixed threshold genuinely doesn't work (strong daily/weekly seasonality). Don't use them as a default "smart alarm" -- use a metric alarm with reasonable thresholds.

### 7. Orphan alarms

Alarms in `INSUFFICIENT_DATA` state still bill $0.10/month each (or $0.30 for high-res or anomaly). They accumulate as resources are deleted but alarms aren't. Audit:

```bash
aws cloudwatch describe-alarms --state-value INSUFFICIENT_DATA \
  --query 'MetricAlarms[].AlarmName' --output text
```

Delete the orphans. In an account with 5,000 alarms, the orphan tail can be 10-20% of the total alarm count.

### 8. Logs Insights queries on huge time ranges

Queries bill $0.005/GB scanned. A query across 30 days of a 10 GB/day log group is 300 GB ($1.50). Across 90 days and 5 groups, easily $50+. Always: narrow the time range first, filter early, save expensive recurring queries as scheduled reports instead of ad-hoc.

### Cost-first triage flow

When your CloudWatch bill spikes, look in this order:

1. **Custom metrics count** -- has it doubled since last month? (`list-metrics --namespace MyApp/*` and count.) If yes -> cardinality trap.
2. **Log ingestion GB** -- has it doubled? (Billing console / Cost Explorer by service.) If yes -> retention missing, a verbose new log group, or an unbounded log emitter.
3. **Insights scanned bytes** -- is someone running expensive queries? (Service quotas + CloudTrail `StartQuery` events.)
4. **Alarm count** -- count vs last month. Orphans?
5. **Container Insights / Lambda Insights enrollments** -- did someone enable enhanced observability across the whole fleet?

The order matters -- cardinality is usually the answer.

---

## Part 13: Production Gotchas -- 20 Items That Have Bitten Engineers

A non-exhaustive list of things that have ruined people's days:

1. **EMF cardinality blindness.** No warning when you accidentally mint 100K metrics; you find out on the bill. Audit `list-metrics` periodically.
2. **IA log class is irreversible.** Set at creation; cannot be changed. Plan the class deliberately or delete-and-recreate later.
3. **Metric filters silently don't fire on IA log groups.** Logs Insights still works (mostly), but metric extraction does not.
4. **Composite alarm `ALARM_RULE` typos silently never fire.** The composite sits in `OK` forever. Test by manually flipping the upstream alarms (`set-alarm-state`).
5. **Anomaly detection needs 2 weeks of clean training data.** Deploys / incidents during training poison the model -- retrain after major regime changes.
6. **High-resolution metrics roll up to 1-min after 3 hours.** Paying for sub-minute beyond 3 hours is wasted money.
7. **`PutMetricData` is synchronous and adds latency.** Switch to EMF for hot-path emissions.
8. **EMF custom metrics count against the same metric billing as PutMetricData.** EMF is cheaper on transport, identical on metric storage.
9. **Container Insights enhanced observability is opt-in.** Without the add-on flag, you get the old cluster-aggregate view.
10. **Application Signals requires ADOT auto-instrumentation per service.** Not free; opt-in.
11. **Subscription filter fan-out to Lambda is additional Lambda invocations.** Cost compounds. Throttle / batch downstream.
12. **Logs Insights queries on multi-month ranges can be very expensive.** Bound time and filter first.
13. **Orphan `INSUFFICIENT_DATA` alarms still bill.** $0.10/month each adds up.
14. **Cross-region observability is per-region.** Lambda Insights in `us-east-1` doesn't see `us-west-2`.
15. **Synthetics canary failures don't auto-page.** You must wire an alarm.
16. **RUM data is in a separate query plane.** No Logs Insights join with application logs.
17. **Lambda's built-in metrics (Invocations / Errors / Duration) ship for free** in the `AWS/Lambda` namespace. Don't re-emit them under your own custom namespace via EMF -- you'd be paying for a custom metric that duplicates a free AWS-namespace one. (Custom *business* metrics like `CheckoutLatency` are different -- those are what EMF is for.)
18. **`treat_missing_data` defaults to `missing`** -- meaning a metric gap is treated as a non-evaluated datapoint, interacting confusingly with M-of-N evaluation. Set explicitly to `notBreaching` or `breaching` per alarm semantics. Use `ignore` (not `missing`) if you actually want "freeze the current alarm state during gaps."
19. **CloudWatch Logs ingestion is billed pre-compression.** Verbose JSON costs full-fat at ingest even though stored compressed.
20. **The CloudWatch Agent's StatsD ingestion is local-only by default.** It listens on `127.0.0.1:8125`. Not directly addressable from other containers without sidecar.

---

## Part 14: Decision Frameworks

### When to use which custom-metric emission method

| Workload | Method |
|----------|--------|
| Lambda | EMF via Powertools |
| Container with stdout (ECS/EKS) | EMF via Powertools or `awsemf` OTel exporter |
| EC2 with CloudWatch Agent | StatsD -> Agent or direct PutMetricData via SDK |
| Multi-backend (CloudWatch + Prometheus) | OTel Collector with `awsemf` + `prometheusremotewrite` |
| One-off scripts | PutMetricData API directly |

### Log group class

| Log type | Class |
|----------|-------|
| Lambda function logs | Standard (EMF extraction) |
| Container app logs you alarm off | Standard (metric filters) |
| VPC Flow Logs | IA |
| API Gateway access logs | IA |
| ALB / NLB access logs | IA |
| CloudTrail trails | IA (rare query, durable archive) |
| Verbose debug logs | IA (with short retention) |

### Alarm type

| Scenario | Alarm type |
|----------|------------|
| Single metric, hard threshold | Metric alarm |
| Suppress noise during deploys / low traffic | Composite alarm with `NOT ALARM("...")` |
| Daily / weekly seasonality, no good fixed threshold | Anomaly detection alarm |
| Derived quantity (error rate, ratio) | Metric alarm with `metric_query` expression |

### Observability backend

| Scale | Choice |
|-------|--------|
| < 20 nodes / single-account / AWS-native | CloudWatch alone |
| 20-50 nodes | CloudWatch + AWS Managed Prometheus, or self-hosted Prom if you have the expertise |
| 50+ nodes | Self-hosted Prometheus + Grafana + Loki + Tempo; CloudWatch for AWS-native alarms only |
| Multi-cloud / hybrid | OTel collector fanning to both Prometheus and CloudWatch |

### Container observability

| Need | Pick |
|------|------|
| AWS-native dashboards, cross-account, no Prometheus team | Container Insights enhanced observability |
| Already running kube-prometheus-stack happily | Stay with it; don't bolt Container Insights on top |
| Mixed Lambda + EKS + ECS observability | Container Insights + Application Signals |

### Log storage at scale

| Scale | Pick |
|-------|------|
| < 100 GB / day | CloudWatch Logs Standard + IA |
| 100 GB - 1 TB / day | CloudWatch Logs IA for vended; consider Loki for app logs |
| > 1 TB / day | Loki or OpenSearch self-managed; archive cold to S3 + Athena |

### Cost-first triage when bill spikes

1. Custom metric count -- cardinality trap?
2. Log ingestion GB -- retention missing / new verbose source?
3. Insights scanned bytes -- expensive queries?
4. Alarm count -- orphans?
5. Insights extension enrollments -- accidental fleet-wide opt-in?

---

## Cheat Sheet -- Things to Memorize

### Log class feature matrix (recap)

| Feature | Standard | IA |
|---------|----------|-----|
| Ingestion price | $0.50/GB | $0.25/GB |
| Storage | Same | Same |
| Logs Insights | Yes | Yes (no `pattern`/`diff`/`unmask`) |
| Live Tail | Yes | No |
| Metric filters | Yes | No |
| Subscription filters | Yes | No |
| Alarming from filters | Yes | No |
| EMF extraction | Yes | **No** |
| Data Protection | Yes | No |
| Insights enrichment (Container/Lambda) | Yes | No |

### Alarm taxonomy

| Type | Cost | When |
|------|------|------|
| Metric alarm | $0.10/mo std, $0.30/mo high-res | Single metric + threshold |
| Composite alarm | $0.10/mo | Boolean of other alarms; killer use = noise suppression |
| Anomaly detection alarm | $0.30/mo (3x) | Seasonal metric, no good fixed threshold; needs 2 weeks training |

### Insights suite distinction

| Product | Use |
|---------|-----|
| Container Insights | EKS / ECS metrics + control plane (enhanced obs Nov 2023) |
| Lambda Insights | Lambda runtime telemetry via extension layer |
| Application Insights | Auto-discovery monitoring for lift-and-shift apps |
| Application Signals | AWS APM via ADOT; RED metrics + SLOs + service map (GA 2024) |
| ServiceLens | Older trace+metric+log stitcher; superseded by Application Signals |

### Resolution + retention

```
Custom metric resolution: standard (60s) or high-res (1/5/10/30s, 4x cost)
Auto rollup ladder:
   1s     -> 1m at 3h
   1m     -> 5m at 15d
   5m     -> 1h at 63d
   1h     -> gone at 15mo
```

### Logs Insights pipeline order

```
fields | filter | parse | stats | sort | limit
           ^
        FILTER FIRST -- $0.005/GB scanned
```

### Cardinality safe-list

| Dimension | Safe? | Why |
|-----------|-------|-----|
| Environment (prod/staging/dev) | Yes | ~3 values |
| Region | Yes | ~30 values |
| Service / FunctionName | Yes | typically <100 |
| Operation / Endpoint | Maybe | bounded if you whitelist |
| UserId | **No** | unbounded |
| RequestId | **No** | unbounded |
| OrderId | **No** | unbounded |
| SessionId | **No** | unbounded |
| IpAddress | **No** | unbounded |

---

## The 10 Commandments of CloudWatch in Production

1. **Emit metrics via EMF, not `PutMetricData`.** Use Lambda Powertools or the OTel `awsemf` exporter; reserve `PutMetricData` for out-of-band scripts.
2. **Treat dimension cardinality like radioactive material.** `userId`, `requestId`, `orderId`, IP, session never go in dimensions. They go in log context.
3. **Every log group has explicit retention.** Default `Never Expire` is the worst default in CloudWatch.
4. **IA log class for vended logs (VPC Flow, API Gateway, ALB); Standard for app logs that alarm or use EMF.** Remember IA is irreversible.
5. **Filter first in Logs Insights.** `$0.005/GB scanned`; the early `filter` clause is your biggest cost lever.
6. **Pair anomaly detection with composite alarms.** Detection catches "weird"; composite filters down to "actually-paging-worthy."
7. **Set `treat_missing_data` explicitly on every alarm.** Default `missing` rarely matches your intent.
8. **Synthetics canaries don't page themselves.** Always wire a metric alarm on `SuccessPercent`.
9. **Audit orphan alarms quarterly.** `describe-alarms --state-value INSUFFICIENT_DATA` and delete the dead ones.
10. **CloudWatch is one possible sink, not the only one.** Use OTel collector to fan out to CloudWatch + Prometheus at 50+ nodes; never lock yourself in.

---

## What's Next

Tomorrow (2026-05-13) is **CloudTrail Deep Dive** -- the audit/compliance counterpart that pairs with today's monitoring focus. The boundary statement to internalize before reading: CloudWatch Logs is "what your application printed during a shift"; CloudTrail is "who called what AWS API." The instrumentation room writes operational logbooks; the auditor's ledger records every door-swipe and key-issuance. Tomorrow covers Management vs Data vs Insights events, single-region vs multi-region vs Organization trails, CloudTrail Lake's federated SQL, advanced event selectors, and the cost trap nobody warns you about -- Data Events on S3 can balloon into thousands of dollars per day if you log every object-level operation across a bucket the size of your data lake.
