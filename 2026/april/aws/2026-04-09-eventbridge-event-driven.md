# Amazon EventBridge & Event-Driven Architecture -- The City Newspaper Analogy

> You built the on-demand kitchen three days ago (Lambda) -- a serverless compute engine that spins up cooking stations per order, charges by the millisecond, and scales to thousands of concurrent dishes. Two days ago, you installed the grand hotel concierge desk (API Gateway) -- the front-of-house that receives guests, verifies credentials, enforces rate limits, and routes requests. Yesterday, you deployed the air traffic control tower (Step Functions) -- the central coordinator that sequences takeoffs, routes planes to parallel runways, retries approaches, waits for ground crew signals, and orchestrates complex multi-step operations. But here is the question none of those systems answer: **how do independent parts of your city learn about things that happen elsewhere?** When a fire breaks out on Main Street, who notifies the fire department, the insurance company, the hospital, and the local newspaper -- all simultaneously, each receiving only the information relevant to them? When a new business opens, who tells the tax office, the utilities company, and the welcome committee? Who ensures that events from neighboring cities (partner SaaS providers) are routed to the right local departments? Who keeps a historical archive of every notable event so that a new department can catch up on what happened last month? And who runs the daily 6 AM alarm that wakes up the garbage collection crew on a schedule? None of your existing components handle this. Lambda executes a single function. API Gateway dispatches a single request. Step Functions coordinates a single workflow. For **loosely-coupled, event-driven communication between independent services that need to react to the same events differently**, you need a fundamentally different model. That model is **Amazon EventBridge** -- and the analogy that captures it is a **city newspaper system**. Think of EventBridge as the **central newsroom** for your entire city (AWS account). Events are **news stories** -- structured reports about something that happened. Event buses are the **newspaper editions** (the default city gazette publishes government news automatically; custom newspapers cover business stories you submit; partner newspapers carry stories from neighboring cities). Rules are **editorial desks** -- each desk reads every incoming story and decides "does this match my beat?" using sophisticated content filters (not just topic headers, but the actual story content -- "any crime story where the suspect is NOT from district 7 AND the damage exceeds $10,000"). Targets are the **subscribers** -- the fire department, the insurance office, the hospital -- each receiving a custom-formatted summary from the editorial desk, not the raw story. The Schema Registry is the **style guide** that documents the exact format of every type of story (so new reporters and subscribers know exactly what fields to expect). The Archive is the **newspaper morgue** (yes, that is the real journalism term for the clipping archive) -- a searchable repository of every published story that you can Replay to catch up new subscribers. EventBridge Scheduler is the **alarm clock system** -- scheduled alerts that go off at specific times or intervals, independent of any news event. EventBridge Pipes is the **dedicated courier** -- a direct, private connection from a single sender (a foreign correspondent) through a specialist (enrichment) to a single recipient, bypassing the public newspaper entirely. Understanding when to use the public newspaper (event bus rules) versus the private courier (Pipes) versus the alarm clock (Scheduler) -- and how they compose with the kitchen (Lambda), the concierge desk (API Gateway), and the control tower (Step Functions) -- is what separates someone who uses EventBridge from an architect who designs event-driven systems.

---

**Date**: 2026-04-09
**Topic Area**: aws
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Event | A news story -- a structured report describing something that happened ("fire on Main Street at 3 PM, damage $50K, 2 injuries") | JSON document with 9 top-level envelope fields (version, id, source, account, time, region, detail-type, resources, detail) up to 256 KB |
| Default Event Bus | The city gazette -- publishes government news (AWS service events) automatically at no cost, every resident receives it | Automatically receives events from 90+ AWS services (EC2, S3, GuardDuty, etc.); free for AWS-originated events; exists in every account |
| Custom Event Bus | A business newspaper you publish yourself -- you submit stories via PutEvents and define which editorial desks (rules) process them | Created via API; receives events from your applications; $1.00 per million custom events published; isolates workload event traffic |
| Partner Event Bus | A foreign newspaper -- stories from neighboring cities (SaaS partners like Datadog, PagerDuty, Auth0, Zendesk) are delivered directly | Receives events from SaaS partners via pre-built integrations; $1.00 per million partner events; requires partner setup |
| Rule | An editorial desk -- reads every incoming story and decides "does this match my beat?" based on content-based filtering criteria | Content-based event pattern matching with prefix/suffix/numeric/exists/anything-but/wildcard/$or operators; up to 300 rules per bus (adjustable) |
| Target | A subscriber -- the fire department, insurance company, or hospital that receives a custom-formatted summary from the editorial desk | 28+ target types (Lambda, Step Functions, SQS, SNS, Kinesis, ECS, API Destinations, other event buses); up to 5 per rule (adjustable); each can have DLQ + input transformer |
| Input Transformer | The editorial desk reformatting a story for each subscriber -- the fire department gets location and severity, the insurance company gets damage estimate and policy number | Extracts values via input paths (JSONPath) and constructs a custom payload via input template with `<variable>` placeholders; up to 100 variables |
| Event Pattern | The editorial desk's content filter -- "any crime story where suspect is NOT from district 7 AND damage > $10K" | JSON pattern matching on event fields; supports prefix, suffix, numeric, exists, anything-but, wildcard, equals-ignore-case, CIDR, $or |
| Schema Registry | The style guide -- documents the exact format of every story type so reporters and subscribers know what fields to expect | Repository of event schemas in OpenAPI 3.0 / JSON Schema Draft 4; AWS schemas pre-built; schema discovery auto-detects; code bindings for Java/Python/TypeScript/Go |
| Archive | The newspaper morgue -- a searchable repository of all published stories with configurable retention | Stores events matching an optional pattern; configurable retention (indefinite or N days); events stored internally (not directly accessible as S3 objects) |
| Replay | Republishing old stories from the morgue to catch up a new subscriber | Re-sends archived events by time range back to the source bus; consumers must be idempotent; max 10 concurrent replays per region; replayed events are new events |
| EventBridge Scheduler | The alarm clock system -- scheduled alerts at specific times or intervals, independent of any news event | Rate/cron/one-time schedules with timezone and DST handling; universal targets (270+ AWS services); flexible time windows; 14M/month free tier |
| EventBridge Pipes | A dedicated courier -- direct private pipeline from one sender through one specialist to one recipient | Source (SQS/DynamoDB/Kinesis/MSK/MQ/Kafka) -> Filter -> Enrich (Lambda/Step Functions/API Gateway/API Destinations) -> Target; point-to-point, not broadcast |
| API Destination | A foreign subscriber who receives stories via international mail (HTTP) -- with authentication and rate limiting | HTTP endpoints as targets; connection auth (Basic, OAuth, API Key); invocation rate limiting (1-300 per second per destination) |
| Cross-Account Bus | Sending your newspaper to the neighboring city's newsroom via a formal distribution agreement (resource-based policy) | Resource-based policy on target bus allows events:PutEvents from source account; 2025 feature enables cross-account targets directly |
| DLQ on Target | The "return to sender" box at each subscriber's mailbox -- if delivery fails after retries, the story is stored for investigation | SQS queue per target; EventBridge retries for up to 24 hours with exponential backoff before DLQ; critical for production reliability |
| Choreography | Each city department independently reacting to newspaper stories -- no central coordinator, departments publish their own follow-up stories | Decentralized event-driven pattern; services react independently to events; EventBridge routes; contrast with orchestration (Step Functions) |

---

## The City Newspaper Framework

Every EventBridge concept maps to a role in a city newspaper operation. Events are news stories. Buses are newspaper editions. Rules are editorial desks. Targets are subscribers. The entire system operates on a publish-subscribe model where producers (reporters) do not know or care about consumers (subscribers) -- the newsroom (EventBridge) handles all the routing.

The newspaper has six operational dimensions:

1. **The Newsroom (Event Buses)** -- Three newspaper editions serving different story sources: the city gazette (default), business newspapers (custom), and foreign newspapers (partner).

2. **The Editorial Desks (Rules & Event Patterns)** -- Content-based filtering that determines which stories reach which subscribers, using sophisticated pattern matching that goes far beyond simple topic headers.

3. **The Subscriber Network (Targets & Input Transformers)** -- Who receives stories, in what format, with what fallback if delivery fails.

4. **The Archive & Library (Schema Registry, Archive, Replay)** -- Documenting story formats, preserving history, and catching up late subscribers.

5. **The Alarm Clock System (Scheduler)** -- Time-based triggers independent of news events.

6. **The Dedicated Courier (Pipes)** -- Point-to-point pipelines with optional enrichment that bypass the public newsroom entirely.

```
THE CITY NEWSPAPER ARCHITECTURE -- EVENTBRIDGE OVERVIEW
══════════════════════════════════════════════════════════════════════════════

  NEWS SOURCES                      CENTRAL NEWSROOM (EventBridge)
  ────────────                      ──────────────────────────────

  ┌────────────┐                   ┌──────────────────────────────────────────────┐
  │ AWS Services│ ──(auto)──────▶ │  DEFAULT BUS (city gazette)                  │
  │ (EC2, S3,  │                  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
  │  GuardDuty)│                  │  │ Rule 1   │  │ Rule 2   │  │ Rule 3   │   │
  └────────────┘                  │  │(EC2 stop)│  │(S3 upload)│  │(GuardDuty│   │
                                  │  └────┬─────┘  └────┬─────┘  │ finding) │   │
  ┌────────────┐                  │       │             │        └────┬─────┘   │
  │ Your Apps  │ ──PutEvents───▶ │       ▼             ▼             ▼         │
  │ (custom    │                  │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
  │  events)   │                  │  │ Lambda  │  │ StepFn  │  │ SNS     │     │
  └────────────┘                  │  │ + DLQ   │  │ + DLQ   │  │ + DLQ   │     │
                                  │  └─────────┘  └─────────┘  └─────────┘     │
  ┌────────────┐                  │                                             │
  │ SaaS       │ ──(partner)───▶ │  CUSTOM BUS (business newspaper)            │
  │ (Datadog,  │                  │  ┌──────────┐  ┌──────────┐                │
  │  Auth0)    │                  │  │ Rule 4   │  │ Rule 5   │                │
  └────────────┘                  │  │(order.*)│  │(payment) │                │
                                  │  └────┬─────┘  └────┬─────┘                │
                                  │       │             │                       │
                                  │       ▼             ▼                       │
                                  │  ┌─────────┐  ┌─────────┐                  │
                                  │  │ SQS     │  │ Lambda  │                  │
                                  │  │ + DLQ   │  │ + DLQ   │                  │
                                  │  └─────────┘  └─────────┘                  │
                                  └──────────────────────────────────────────────┘
                                          │
                                          │  Archive / Replay
                                          ▼
                                  ┌──────────────────┐     ┌──────────────────┐
                                  │ Schema Registry  │     │ Event Archive    │
                                  │ (style guide)    │     │ (morgue)         │
                                  └──────────────────┘     └──────────────────┘

  SEPARATE SERVICES (same EventBridge family, different mechanisms)
  ─────────────────────────────────────────────────────────────────

  ┌──────────────────────────────────────────────────────────────────────┐
  │  SCHEDULER (alarm clock)              PIPES (dedicated courier)          │
  │  ┌──────────┐  ┌──────────┐          ┌──────┐ ┌──────┐ ┌───────┐  │
  │  │ Cron/Rate│  │ One-time │          │Source│→│Filter│→│Enrich │  │
  │  │ schedule │  │ schedule │          │(SQS) │ │      │ │(Lambda│  │
  │  └────┬─────┘  └────┬─────┘          └──────┘ └──────┘ │/ SFN) │  │
  │       │             │                                   └───┬───┘  │
  │       ▼             ▼                                       ▼      │
  │  ┌──────────────────────┐             ┌──────────────────────────┐ │
  │  │ Universal Targets    │             │ Target (Step Functions)  │ │
  │  │ (270+ AWS services)  │             │ (point-to-point)        │ │
  │  └──────────────────────┘             └──────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Event Structure -- The Anatomy of a News Story

### The News Story Format

**The analogy**: Every story in the city newspaper follows a strict format. The header contains mandatory fields: the story ID, the reporter's name (source), the story category (detail-type), the publication time, and the edition it appeared in. The body (detail) contains the actual story content in any structure the reporter chooses.

**The technical reality**: Every EventBridge event is a JSON document with a fixed envelope structure:

```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "source": "com.mycompany.orders",
  "account": "123456789012",
  "time": "2026-04-09T14:30:00Z",
  "region": "us-east-1",
  "detail-type": "Order Placed",
  "detail": {
    "orderId": "ORD-9876",
    "customerId": "CUST-1234",
    "total": 149.99,
    "items": ["widget-A", "widget-B"],
    "shippingMethod": "express",
    "warehouse": "us-east"
  },
  "resources": []
}
// NOTE: `replay-name` is NOT part of the normal envelope. It only appears on
// events generated by a Replay operation -- use it in event patterns to
// distinguish replayed events from live events.

```

**Key structural rules**:
- **`source`**: A free-form string identifying the event producer. AWS services use the pattern `aws.<service>` (e.g., `aws.ec2`, `aws.s3`). Your applications should use reverse-domain notation (`com.mycompany.orders`). This is **not** validated -- it is a convention. Do not use the `aws.` prefix for custom events.
- **`detail-type`**: A free-form string categorizing the event. AWS services use descriptive names like `EC2 Instance State-change Notification`. Your applications define their own.
- **`detail`**: The event body. Any valid JSON. This is where your business data lives and where content-based filtering operates.
- **`id`**: A UUID generated by EventBridge for events from AWS services, or by PutEvents for custom events. Not guaranteed globally unique across replays.
- **Maximum event size**: **256 KB** for a single event entry. This matches the Step Functions state payload limit (yesterday's deep dive) and the Lambda **asynchronous** invocation payload limit. (Lambda **synchronous** invocation allows 6 MB request/6 MB response -- do not confuse the two.)
- **Billing is metered in 64 KB chunks**: a 200 KB event counts as **4 event charges** on PutEvents and on archive storage. A 65 KB event counts as 2. Keep large payloads in S3 and publish references via EventBridge when cost matters.

### Publishing Custom Events -- PutEvents API

```python
import boto3
import json

client = boto3.client('events')

response = client.put_events(
    Entries=[
        {
            'Source': 'com.mycompany.orders',
            'DetailType': 'Order Placed',
            'Detail': json.dumps({
                'orderId': 'ORD-9876',
                'customerId': 'CUST-1234',
                'total': 149.99,
                'items': ['widget-A', 'widget-B'],
                'shippingMethod': 'express'
            }),
            'EventBusName': 'orders-bus'  # omit for default bus
        },
        {
            'Source': 'com.mycompany.orders',
            'DetailType': 'Order Cancelled',
            'Detail': json.dumps({
                'orderId': 'ORD-5555',
                'reason': 'customer-request'
            }),
            'EventBusName': 'orders-bus'
        }
    ]
)

# Check for partial failures -- PutEvents can PARTIALLY succeed
for entry in response.get('Entries', []):
    if 'ErrorCode' in entry:
        print(f"Failed: {entry['ErrorCode']} - {entry['ErrorMessage']}")
```

**Critical**: PutEvents accepts up to **10 entries per API call**. The call can partially succeed -- some entries may fail while others succeed. Always check the response `FailedEntryCount` and individual entry error codes. This is a common production oversight and a frequent interview question.

---

## Part 2: Event Buses -- The Newspaper Editions

### Default Event Bus -- The City Gazette

**The analogy**: The city gazette is published automatically by the government. Every government department (AWS service) submits stories whenever something notable happens -- a building changes status (EC2 state change), a package arrives at the post office (S3 object created), or the security team files a report (GuardDuty finding). You cannot stop the gazette from being published, but you can choose which stories your editorial desks (rules) pay attention to.

**The technical reality**: Every AWS account has one default event bus per region. It receives events from 90+ AWS services automatically. These AWS service events are **free** -- you pay nothing for their publication. The default bus is where you create rules to react to infrastructure events:
- EC2 instance state changes (running, stopped, terminated)
- S3 object notifications (since late 2021, S3 delivers to EventBridge natively -- the recommended path over S3 -> SNS/SQS/Lambda direct, as covered in the S3 Advanced deep dive, Mar 23)
- GuardDuty findings
- Health Dashboard alerts
- CodePipeline state changes
- ECS task state changes

### Custom Event Bus -- The Business Newspaper

**The analogy**: You start a business newspaper for your company's internal events. Reporters (microservices) submit stories via the editorial submission desk (PutEvents API). Each business newspaper is independent -- the orders newspaper does not mix with the payments newspaper, keeping concerns separated.

**Why create custom buses instead of using the default bus?**:
1. **Isolation**: Custom buses keep application events separate from AWS service events, preventing rule collisions and making permissions cleaner
2. **Cross-account access**: You can attach resource-based policies to custom buses (not the default bus in other accounts) to allow cross-account event delivery
3. **Archive granularity**: You can archive a custom bus independently with its own retention policy
4. **Schema discovery**: Enable discovery on specific buses to catalog only relevant event contracts
5. **Security boundary**: Restrict who can publish to and subscribe from each bus via IAM and resource policies

### Partner Event Bus -- The Foreign Newspaper

SaaS partners (Datadog, PagerDuty, Auth0, Zendesk, Shopify, and 40+ others) can deliver events directly to a partner event bus in your account. You create the partner event bus association in the EventBridge console, and the partner configures their end. Partner events cost $1.00 per million, same as custom events.

---

## Part 3: Rules and Event Patterns -- The Editorial Desks

### Content-Based Filtering -- The Beat System

**The analogy**: Each editorial desk has a beat -- a set of criteria that determine which stories it covers. The crime desk might cover "any story where the category is 'crime' AND the location starts with 'downtown' AND the damage exceeds $10,000 AND the suspect is NOT from district 7." This is content-based filtering -- the desk does not just look at the headline (topic), it reads the actual story content (detail fields) to decide.

**The technical reality**: EventBridge event patterns are the most powerful filtering mechanism in the AWS messaging ecosystem, surpassing both SNS subscription filter policies and SQS (which has no native filtering). Event patterns support these operators:

| Operator | Example | Matches |
|----------|---------|---------|
| **Exact match** | `"source": ["aws.ec2"]` | source equals "aws.ec2" |
| **Prefix** | `"region": [{"prefix": "us-"}]` | region starts with "us-" |
| **Suffix** | `"fileName": [{"suffix": ".csv"}]` | fileName ends with ".csv" |
| **Numeric** | `"price": [{"numeric": [">", 100, "<=", 500]}]` | 100 < price <= 500 |
| **Exists** | `"error": [{"exists": true}]` | field "error" is present |
| **Not exists** | `"error": [{"exists": false}]` | field "error" is absent |
| **Anything-but** | `"state": [{"anything-but": "initializing"}]` | state is NOT "initializing" |
| **Anything-but prefix** | `"region": [{"anything-but": {"prefix": "us-"}}]` | region does NOT start with "us-" |
| **Anything-but suffix** | `"file": [{"anything-but": {"suffix": ".tmp"}}]` | file does NOT end with ".tmp" |
| **Anything-but wildcard** | `"path": [{"anything-but": {"wildcard": "*/test/*"}}]` | path does NOT match glob |
| **Wildcard** | `"host": [{"wildcard": "*.example.com"}]` | host matches glob pattern |
| **Equals-ignore-case** | `"status": [{"equals-ignore-case": "success"}]` | case-insensitive match |
| **CIDR** | `"ip": [{"cidr": "10.0.0.0/8"}]` | IP in CIDR range |
| **$or** | `"$or": [{"source": ["A"]}, {"source": ["B"]}]` | source is A OR B |

**Complex pattern example** -- match order events where the total exceeds $100, the shipping method is NOT "pickup", and the warehouse starts with "us-":

```json
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["Order Placed"],
  "detail": {
    "total": [{ "numeric": [">", 100] }],
    "shippingMethod": [{ "anything-but": "pickup" }],
    "warehouse": [{ "prefix": "us-" }]
  }
}
```

**Pattern matching rules**:
- Multiple values in an array are OR within the same field: `"source": ["A", "B"]` means source is A **or** B
- Multiple fields are AND across fields: both `source` and `detail-type` must match
- Patterns only match fields that are present in the event -- if the pattern specifies a field that the event does not have, it does not match (unless using `"exists": false`)
- Patterns are **prefix-based for event structure** -- a pattern `{"source": ["X"]}` matches any event with source "X", regardless of other fields present in the event

### Testing Patterns Safely Before Deploying

The most common production failure with EventBridge rules is "works in dev, silently fails in prod" -- typically because the dev test events were hand-crafted to satisfy the pattern (right field shape, lowercase carrier values, all expected fields present), while real production events drift from that shape. Two specific traps:

1. **Case sensitivity**: pattern matching is exact-match by default. `["fedex"]` will NOT match `"FEDEX"`. Use `{"equals-ignore-case": "fedex"}` if the producer cannot guarantee a canonical form -- or fix the producer (preferred, since data quality belongs upstream).
2. **Missing-field semantics**: a pattern that references a field requires the field to be **present** in the event (unless you use `{"exists": false}`). A pattern like `"metadata": {"expedited": [false]}` will not match an event whose `detail` has no `metadata` key at all -- even though "logically" you might expect `false` to match "absent." This single gotcha is the source of an enormous number of "rule never fires" tickets.

The defense is a 3-layer test workflow before any pattern reaches production:

**Layer 1 -- Offline pattern validation via the `TestEventPattern` API.** Free, instant, no side effects, scriptable in CI:

```bash
aws events test-event-pattern \
  --event-pattern file://new-pattern.json \
  --event '{
    "source": "com.mycompany.orders",
    "detail-type": "Order Shipped",
    "detail": {
      "orderId": "ORD-9876",
      "shipment": { "carrier": "FEDEX", "trackingNumber": "1Z999", "weight": 5.2 }
    }
  }'
# Returns: {"Result": true}  or  {"Result": false}
```

Build a pattern test suite alongside your unit tests -- commit pattern + sample-event pairs to git so future pattern changes do not silently regress.

**Layer 2 -- Live verification via a parallel rule with a CloudWatch Logs target.** Deploy a *test* rule with the new pattern alongside the production rule, but point its target at a CloudWatch Logs group instead of the real consumer. CloudWatch Logs is a safe target -- no business side effects. Watch the logs fill up over 15-30 minutes with real production events, verify the match count and shape match expectations, then swap the target to the production consumer (or delete the test rule and enable the real one).

**Layer 3 -- Historical regression via archive replay.** For changes to long-established rules where you also need to verify the new pattern does not accidentally MISS events the old pattern would have matched: create the test rule (Layer 2), then `StartReplay` scoped via `Destination.FilterArns` to ONLY the test rule, replaying yesterday's events. Count log entries vs expectations. This catches the "I fixed the obvious bug but introduced a more subtle one" class of regression.

**Order matters**: Layer 1 catches syntax errors instantly. Layer 2 catches real-world data drift. Layer 3 catches historical regressions. Skip none of them if the rule has business-critical consumers.

### Rule Limits and Evaluation

- **Up to 300 rules per event bus** (adjustable via Service Quotas; some regions default to 100)
- **Up to 5 targets per rule -- this is a HARD limit, NOT adjustable.** If you need more than 5 targets for the same event pattern, create multiple rules with identical patterns (each rule can have its own 5 targets). Every rule is evaluated independently, so duplicating the pattern is the supported workaround.
- **All rules are evaluated for every event** -- there is no short-circuit or priority ordering. If an event matches 10 rules, all 10 rules' targets are invoked. This is fundamentally different from Step Functions Choice state (which evaluates top-to-bottom, first match wins).
- **Rule evaluation is free** -- you pay only for events published and events delivered to targets

---

## Part 4: Targets and Input Transformers -- The Subscriber Network

### Target Types -- 28+ Subscribers

EventBridge can deliver matched events to a wide range of AWS services:

| Category | Targets |
|----------|---------|
| **Compute** | Lambda function, ECS task (RunTask), Batch job, EC2 (CreateSnapshot, RebootInstances, StopInstances, TerminateInstances) |
| **Orchestration** | Step Functions (StartExecution for Standard, StartSyncExecution for Express) |
| **Messaging** | SQS queue, SNS topic, Kinesis Data Stream, Kinesis Data Firehose |
| **Developer tools** | CodeBuild project, CodePipeline |
| **Management** | SSM Run Command, SSM Automation, SSM OpsItem, Inspector assessment |
| **Analytics** | Redshift cluster (data API), SageMaker pipeline |
| **External** | API Destination (HTTP endpoint), another event bus (cross-account/region) |
| **Logging** | CloudWatch Log group |
| **Containers** | ECS task (Fargate or EC2 launch type) |

### Input Transformers -- Custom Story Formatting

**The analogy**: The editorial desk reformats each story for its subscriber. The fire department does not need the full crime report -- they need the location, the fire severity, and the number of people involved. The insurance company needs the damage estimate and the policy number. Same story, different formats for different subscribers.

**The technical reality**: Input transformers let you reshape the event before delivery using two components:

1. **Input Paths**: Extract values from the event into named variables using JSONPath
2. **Input Template**: Construct the new payload using those variables as `<variable>` placeholders

```json
{
  "InputPathsMap": {
    "orderId": "$.detail.orderId",
    "total": "$.detail.total",
    "customer": "$.detail.customerId",
    "time": "$.time"
  },
  "InputTemplate": "{\"orderReference\": <orderId>, \"amount\": <total>, \"who\": <customer>, \"when\": <time>, \"action\": \"process-payment\"}"
}
```

The event `{"detail": {"orderId": "ORD-9876", "total": 149.99, "customerId": "CUST-1234"}, "time": "2026-04-09T14:30:00Z"}` becomes `{"orderReference": "ORD-9876", "amount": 149.99, "who": "CUST-1234", "when": "2026-04-09T14:30:00Z", "action": "process-payment"}`.

This is analogous to Step Functions' Parameters/ResultSelector for reshaping data -- the same principle of decoupling the event shape from what the consumer expects. The key difference: Step Functions transforms data between states in a workflow, while EventBridge transforms data at the point of delivery to external targets.

### Retry Policy -- How EventBridge Handles Transient Failures

When EventBridge fails to deliver an event to a target, it retries with **exponential backoff and jitter**. The default retry policy is:

- **Maximum retry attempts**: **185** (the effective ceiling, applied if you set 0-185)
- **Maximum event age**: **24 hours (86,400 seconds)** -- the event is discarded once it exceeds this age, even if retries remain

Both are **configurable per-target** via `MaximumRetryAttempts` (0-185) and `MaximumEventAgeInSeconds` (60-86,400). Shorten them when an event becomes meaningless quickly (a "current price" update is useless 10 minutes later) or lengthen them when downstream outages are expected.

```hcl
resource "aws_cloudwatch_event_target" "example" {
  # ... rule, arn, etc.

  retry_policy {
    maximum_event_age_in_seconds = 3600  # Give up after 1 hour
    maximum_retry_attempts       = 10    # At most 10 retries
  }
}
```

### Dead Letter Queues on Targets -- Return to Sender

**Each target** can have its own SQS dead-letter queue. When the retry policy is exhausted (max attempts reached or max event age exceeded), the event is sent to the DLQ (if configured) or **silently dropped** (if not).

**This is different from Lambda DLQs.** EventBridge DLQs are configured per-target, not per-function. A single rule with 5 targets can have 5 different DLQs, one for each delivery path. The DLQ message is delivered with the original event body plus error metadata as SQS message attributes (`ERROR_CODE`, `ERROR_MESSAGE`, `RULE_ARN`, `TARGET_ARN`), making it possible to diagnose and reprocess failures.

**Production rule**: Always configure a DLQ on every target. An EventBridge target without a DLQ is a silent failure point. This mirrors the "always configure DLQ on async Lambda invocations" principle from the Lambda deep dive (Apr 6).

---

## Part 5: Schema Registry -- The Style Guide

**The analogy**: The style guide documents the exact format of every type of story the newspaper publishes. When a new reporter joins (a new microservice is added), they consult the style guide to know the exact fields, types, and structure of "Order Placed" stories. When the story format evolves (a new field is added), the style guide is versioned -- version 1 had 5 fields, version 2 has 6 fields. The style guide can even auto-generate reporter templates (code bindings) in multiple languages.

**The technical reality**: The Schema Registry maintains event schemas in three registries:

1. **AWS Event Schema Registry**: Pre-built schemas for all AWS service events. Want to know the exact JSON structure of an EC2 State-change Notification? It is already here.
2. **Custom Schema Registry**: Upload your own schemas manually (OpenAPI 3.0 or JSON Schema Draft 4).
3. **Discovered Schemas**: Enable **schema discovery** on an event bus, and EventBridge automatically detects and catalogs the schema of every event flowing through, including version tracking when schemas evolve.

**Code bindings**: From any schema (discovered or manually uploaded), you can generate type-safe code bindings in Java, Python, TypeScript, and Go. These bindings provide classes/interfaces matching the event structure, enabling compile-time validation of event handling code.

**Pricing**: Schema discovery costs **$0.10 per million events ingested** after a **5 million events/month free tier**. Events are metered in **8 KB chunks** for discovery (different from the 64 KB chunk used for PutEvents billing), so large events accelerate the meter. Code binding downloads and schema registry API calls are free.

**When to enable discovery**: Development and staging environments, always -- the 5M free tier usually covers non-production traffic entirely. Production, selectively -- discovery adds latency and cost at volume. A common pattern is to discover schemas in staging, export them to a custom registry, and use the custom registry in production.

---

## Part 6: Archive and Replay -- The Newspaper Morgue

**The analogy**: The newspaper morgue (the real journalism term for a clipping archive) stores copies of every published story. When a new city department opens six months later, you can replay last quarter's stories so the new department processes the historical events it missed. When a bug is found in how the insurance office processed fire stories, you can replay the fire stories from last week to reprocess them with the fixed logic.

**The technical reality**:

**Archive**:
- Stores events from an event bus to internal storage (not directly accessible S3 buckets)
- Can filter which events to archive using an event pattern (archive only "Order Placed" events, not all events)
- Configurable retention: **indefinite** (default) or a specified number of days
- Cost: standard EventBridge event pricing ($1.00/million events archived)

**Replay**:
- Re-sends archived events back to the **source event bus** within a specified time window
- By default, events are delivered to the bus as new events -- **every** rule on the bus evaluates them (this is the danger)
- **`Destination.FilterArns` scopes the replay to a subset of rules**. This is the production-critical mechanism: when launching a new consumer ("loyalty service") that needs historical events without re-triggering the existing six consumers, pass the new rule's ARN in `FilterArns` and only that rule will see the replayed events. Without `FilterArns`, replay is a fan-out catastrophe.
- Replay states: `STARTING` -> `RUNNING` -> `COMPLETED` (or `CANCELLED` / `FAILED`)
- Maximum **10 concurrent replays** per account per region
- Replayed events include a `replay-name` field in the envelope -- use this in your live rules' event patterns (`{"replay-name": [{"exists": false}]}`) to opt rules *out* of replays even when `FilterArns` isn't an option
- **Wait at least 10 minutes** before replaying recent events -- there is a small ingestion delay between an event landing on the bus and arriving in the archive. AWS recommends a 10-minute buffer from "now"
- Replays are processed in 1-minute time-window batches based on event time. **Within a batch, ordering is not guaranteed.** If your consumer requires strict ordering, replay is the wrong tool
- **Replayed events cost the same as publishing custom events** ($1.00/million)

```bash
# Replay last 7 days of orders into ONLY the new loyalty rule -- existing consumers untouched
aws events start-replay \
  --replay-name loyalty-backfill-2026-04-09 \
  --event-source-arn arn:aws:events:us-east-1:123456789012:archive/orders-archive \
  --destination "{
    \"Arn\": \"arn:aws:events:us-east-1:123456789012:event-bus/orders-bus\",
    \"FilterArns\": [\"arn:aws:events:us-east-1:123456789012:rule/orders-bus/rule-loyalty\"]
  }" \
  --event-start-time 2026-04-02T00:00:00Z \
  --event-end-time   2026-04-09T00:00:00Z
```

**The idempotency requirement**: Because replayed events are treated as new events, every consumer must be idempotent. If your Lambda function processes an "Order Placed" event and charges the customer, replaying that event will charge the customer again unless your function checks for duplicate processing. This ties directly to the idempotency patterns from the Lambda deep dive (Apr 6) -- DynamoDB conditional writes, Lambda Powertools idempotency decorator, or S3 existence checks.

**Event design for idempotency**: The best defense against replay duplication is an **idempotency key** in the event `detail` -- a stable, producer-generated identifier (often a ULID or the business transaction ID like `orderId`). Consumers use this key to conditionally write to DynamoDB (`ConditionExpression: "attribute_not_exists(orderId)"`) or to query a deduplication cache before processing. Relying on EventBridge's internal `id` field is **not** safe because replays generate new IDs. Design your events so every consumer can deduplicate without coordination.

**Managed rules -- rules EventBridge creates for you**: Some EventBridge features create "managed rules" automatically in your account. You will see them in the console prefixed with names like `Schemas-`, `StepFunctions-`, or archive-related patterns. These are owned by the service feature that created them -- do not delete them manually. For example, when you enable schema discovery on a bus, EventBridge creates a managed rule that routes every event through the discovery engine. When you start a replay, EventBridge uses managed routing internally (not a user-visible rule, but important to know it exists). Third-party services (Step Functions, EventBridge Scheduler, Security Hub, and others) also create managed rules on your default bus to listen for the events they need.

---

## Part 7: EventBridge Scheduler -- The Alarm Clock System

**The analogy**: The alarm clock system is independent of the newspaper. It does not react to news stories -- it fires at specific times or intervals. The 6 AM garbage collection alarm. The monthly tax reminder. The one-time "remind me to renew the business license on April 15." Each alarm can trigger any city department directly -- it does not need to go through the newsroom.

**The technical reality**: EventBridge Scheduler is a separate, purpose-built scheduling service (launched Nov 2022) that supersedes legacy EventBridge scheduled rules.

### Three Schedule Types

| Type | Example | Use Case |
|------|---------|----------|
| **Rate-based** | `rate(5 minutes)`, `rate(1 hour)` | Recurring at fixed intervals -- polling, health checks, cache warming |
| **Cron-based** | `cron(0 8 ? * MON-FRI *)` | Fine-grained recurring -- 8 AM weekdays, first Monday of month |
| **One-time** | `at(2026-04-15T09:00:00)` | Single invocation -- deferred actions, reminders, scheduled deletions |

### Timezone and DST Handling

All schedule types support **IANA Time Zone Database** names (e.g., `America/New_York`, `Europe/London`). DST handling:
- **Spring forward**: If the scheduled time falls in the skipped hour (e.g., 2:30 AM on DST spring-forward day), the invocation is **skipped**
- **Fall back**: If the scheduled time falls in the repeated hour, the invocation runs **once only** (not twice)

### Universal Targets -- 270+ AWS Services Without Lambda

This is the transformative feature. Scheduler can invoke any AWS API operation directly using the universal target format:

```
arn:aws:scheduler:::aws-sdk:<service>:<apiAction>
```

Examples:
- `arn:aws:scheduler:::aws-sdk:lambda:invoke` -- invoke a Lambda function
- `arn:aws:scheduler:::aws-sdk:ecs:runTask` -- run an ECS task
- `arn:aws:scheduler:::aws-sdk:s3:deleteObject` -- delete an S3 object on a schedule
- `arn:aws:scheduler:::aws-sdk:dynamodb:deleteItem` -- TTL-like deletion on a schedule

This eliminates the "Lambda cron" anti-pattern where you create a Lambda function solely to call another AWS API on a schedule. Same cost optimization pattern as Step Functions direct SDK integrations (Apr 8) and API Gateway direct AWS service integrations (Apr 7).

### Flexible Time Windows

Configurable windows (OFF, or 1-1440 minutes) disperse invocations to reduce thundering herd effects. If your schedule runs at 8:00 AM with a 15-minute flexible window, the actual invocation occurs at a random time between 8:00 and 8:15.

### Scheduler vs Legacy Scheduled Rules

| Feature | EventBridge Scheduler | Legacy Scheduled Rules |
|---------|----------------------|----------------------|
| One-time schedules | Yes (`at(...)`) | No |
| Timezone support | Yes (IANA) | No (UTC only) |
| Flexible time windows | Yes (1-1440 min) | No |
| Universal targets | Yes (270+ services) | No (limited target types) |
| Throughput | Millions of schedules per account | Limited by rules-per-bus quota (300 default) |
| Schedule groups | Yes (organization + management) | No |
| DLQ support | Yes (per-target on the schedule) | Not directly |
| Targets per schedule | **Exactly 1** | Up to 5 |
| Service principal | `scheduler.amazonaws.com` | `events.amazonaws.com` |
| **Recommendation** | **Always use for new workloads** | Legacy only |

### `ActionAfterCompletion` -- The One-Time Schedule Lifecycle Footgun

One-time `at(...)` schedules persist in the account **even after they fire**, indefinitely, counting against your schedule quota. At consumer scale (per-customer reminders, deferred deletes), this accumulation will eventually exhaust your quota and start failing schedule creation calls.

**Always set `ActionAfterCompletion: DELETE` on every one-time schedule.** This tells Scheduler to delete the schedule automatically once its target has been invoked successfully. Without it, you are signing up for a cleanup job (and the bug where the cleanup job dies and nobody notices for six months).

```hcl
resource "aws_scheduler_schedule" "follow_up_email" {
  name                         = "follow-up-${var.customer_id}"
  schedule_expression          = "at(${var.send_at_iso8601})"
  schedule_expression_timezone = "UTC"
  group_name                   = aws_scheduler_schedule_group.follow_ups.name

  flexible_time_window { mode = "OFF" }

  action_after_completion = "DELETE"   # critical for one-time schedules

  target {
    arn      = aws_lambda_function.send_email.arn
    role_arn = aws_iam_role.scheduler.arn
    input    = jsonencode({ customerId = var.customer_id, templateId = "DAY7_FOLLOWUP" })
  }
}
```

The same applies to recurring `rate()` and `cron()` schedules that you create programmatically and never want to live forever -- you just have to know when "completion" means for your use case (typically you delete recurring schedules explicitly via `DeleteSchedule`).

### One Target per Schedule -- The Capability Gap

Scheduler schedules have **exactly one target each**, unlike scheduled rules which support up to 5 targets per rule. For pure cron-style workloads ("do one thing daily") this is a non-issue, but if you need scheduled fan-out, four workarounds exist:

| Workaround | When to use | Trade-off |
|------------|-------------|-----------|
| **Multiple schedules with the same expression** | 2-3 simple targets, each needing its own IAM/input/retry policy | N things to keep in sync if the cron expression ever changes |
| **Schedule -> SNS topic -> N subscribers** | 3-10 simple fan-out, lowest latency, lowest cost | Loses content-based routing, schema, archive, replay |
| **Schedule -> EventBridge bus (PutEvents) -> N rules** | 3+ targets needing different routing, input transformers, or independent retry policies/DLQs | Slightly higher latency and one extra hop |
| **Schedule -> Step Functions Parallel state** | Targets must be coordinated (e.g., compensation on partial failure) or aggregated | Most expensive per execution; but gives you full workflow semantics |

### Pricing

$1.00 per million invocations with a generous **14 million invocations/month free tier**. This means a schedule that runs every minute (43,200 invocations/month) costs nothing within the free tier.

---

## Part 8: EventBridge Pipes -- The Dedicated Courier

**The analogy**: While the newspaper broadcasts stories to many subscribers (event bus fan-out), a dedicated courier is a private, point-to-point service. One sender (the source) hands a package to the courier, the courier optionally stops by a specialist's office to have it inspected or repackaged (the enrichment step), and then delivers it to exactly one recipient (the target). It is faster and more direct than going through the newsroom -- but it reaches only one destination, not many. (A wire service is the wrong mental model here, because real wire services like AP or Reuters broadcast to many newspapers. Pipes are strictly point-to-point, so a private courier is the right analogy.)

**The technical reality**: EventBridge Pipes (launched Dec 2022) provide a **point-to-point** integration pipeline with four stages:

```
SOURCE ──▶ FILTER ──▶ ENRICH ──▶ TARGET
(poll)     (pattern)  (optional)  (deliver)
```

### Supported Sources

| Source | Notes |
|--------|-------|
| SQS (Standard and FIFO) | Similar to Lambda event source mapping |
| DynamoDB Streams | Change data capture |
| Kinesis Data Streams | Real-time streaming |
| Amazon MSK (Managed Kafka) | Kafka topics |
| Amazon MQ (ActiveMQ, RabbitMQ) | Message broker queues |
| Self-managed Apache Kafka | Any Kafka cluster |

### Filtering

Uses the same event pattern syntax as event bus rules -- this is a **cost optimization lever**. You pay the $0.40/million Pipes request fee only for events that **pass through** the filter and reach the target. Events **rejected** by the filter incur no Pipes charge at all. Note: source polling costs (e.g., SQS ReceiveMessage) are still billed by the source service for every message read, regardless of whether it passes the Pipes filter.

### Enrichment

Optional, synchronous step. EventBridge invokes one of four enrichment services, waits for the response, and passes the enriched data to the target:

1. **Lambda function** -- most flexible, any transformation logic
2. **Step Functions state machine** -- **Express Workflows only** for enrichment (Standard is not supported as a Pipes enrichment)
3. **API Gateway endpoint** -- HTTP-based enrichment
4. **API Destination** -- external HTTP endpoint enrichment

### Pipes vs Event Bus Rules -- The Critical Decision

| Dimension | Event Bus Rules | Pipes |
|-----------|----------------|-------|
| **Topology** | Many-to-many (fan-out) | One-to-one (point-to-point) |
| **Sources** | Any event published to the bus | Polling sources only (SQS, DDB Streams, Kinesis, MSK, MQ, Kafka) |
| **Enrichment** | No | Yes (Lambda, Step Functions Express, API Gateway, API Destinations) |
| **Targets per config** | Up to 5 per rule | Exactly 1 |
| **Fan-out** | Yes (multiple rules per event) | No |
| **Ordering** | No guarantees | Maintained within batches |
| **Use when** | Multiple consumers for same events | Dedicated source-to-target pipeline with optional enrichment |

**The mental model**: Pipes replace the pattern of "Lambda event source mapping -> Lambda function that just transforms data and forwards it" with a managed, codeless integration. If you find yourself writing a Lambda function whose only job is to read from SQS, add some context, and invoke Step Functions, that is a Pipe.

### Deduplication at the Pipes -> Step Functions Boundary

Pipes is **at-least-once** delivery (SQS Standard duplication + Pipes target retries + visibility timeout edge cases). When the target is Step Functions Standard, the canonical exactly-once dedup pattern is to pass a **deterministic execution name** built from a stable business identifier:

```hcl
target_parameters {
  step_function_state_machine_parameters {
    invocation_type = "FIRE_AND_FORGET"
  }
  input_template = jsonencode({
    name = "msg-<$.body.orderId>"   # deterministic name from the business ID
    input = {
      orderId    = "<$.body.orderId>"
      customerId = "<$.body.customerId>"
      profile    = "<$.enrichment.profile>"
    }
  })
}
```

Step Functions Standard rejects duplicate `StartExecution` calls with the same name **within a 90-day window**, returning `ExecutionAlreadyExists`. Pipes can fire `StartExecution` ten times for the same `orderId` and only the first one creates a workflow -- the rest are no-ops.

**Operational footprint of this technique**: the rejected duplicates surface as failures in your Pipes `ExecutionFailed` metric. This is *correct behavior*, not an error -- the workflow already started. Filter `ExecutionAlreadyExists` out of your alerting (or treat it as success in your interpretation), otherwise you will page yourself on every dedup hit. The same applies to EventBridge rule -> Step Functions targets and to direct `StartExecution` calls from any caller.

This is the workflow-boundary half of idempotency. The other half is *inside* the workflow: human-approval double-callbacks via `.waitForTaskToken`, downstream API calls inside the workflow (each needing its own idempotency key based on the business ID, not the execution ARN), and the enrichment call itself if it is anything other than a GET.

---

## Part 9: Cross-Account and Cross-Region

### Cross-Account Event Delivery

**The analogy**: Sending your newspaper to the neighboring city's newsroom requires a formal distribution agreement (resource-based policy). The neighboring city's editor-in-chief signs a permission slip saying "yes, City A may deliver stories to our newsroom."

**Pattern 1: Intermediate event bus (classic)**

```
Account A (source)              Account B (target)
┌──────────────────┐           ┌──────────────────┐
│ Custom Bus       │           │ Custom Bus       │
│ Rule: match X ───┼──────────▶│ Rule: match X    │
│ Target: Bus B ARN│           │ Target: Lambda   │
└──────────────────┘           └──────────────────┘
                                Resource policy:
                                Allow Account A
                                events:PutEvents
```

Account B's event bus must have a resource-based policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAccountAPutEvents",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111111111111:root" },
    "Action": "events:PutEvents",
    "Resource": "arn:aws:events:us-east-1:222222222222:event-bus/orders-bus"
  }]
}
```

For Organization-wide access, use the `aws:PrincipalOrgID` condition key:

```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalOrgID": "o-1234567890"
  }
}
```

**Pattern 2: Cross-account targets (2025 feature)**

As of early 2025, EventBridge supports targeting resources in another account directly from a rule -- **without** needing an intermediate event bus in the target account. Supported cross-account target types are a subset of all targets: **SQS, Lambda, Kinesis Data Streams, SNS, and API Gateway**. Targets not in this list (Step Functions, ECS, Batch, etc.) still require the intermediate bus pattern.

Configuration requires:
1. The target resource (e.g., the SQS queue in Account B) must have a resource-based policy allowing `events.amazonaws.com` with a `SourceArn` condition matching the rule in Account A.
2. The rule in Account A references the cross-account ARN directly as the target.
3. No IAM role is needed on the rule side -- EventBridge uses the target's resource-based policy for authorization.

This pattern halves the per-event cost compared to the intermediate-bus approach (one publish instead of two).

### Cross-Region Event Delivery

EventBridge can route events from any commercial region to any other commercial region. For direct rule->target cross-region delivery, the target must be **another event bus** -- you cannot directly target a Lambda function or SQS queue in another region from a rule. For cross-region + cross-account, chain the patterns: source bus (us-east-1, Account A) -> target bus (eu-west-1, Account B) -> local rule -> local target.

### Global Endpoints -- Multi-Region Event Ingestion with Automatic Failover

For DR-grade resilience, EventBridge Global Endpoints provide a single managed endpoint that spans two regions (a primary and a secondary) with automatic failover driven by Route 53 health checks.

```
PutEvents to global-endpoint-abc.endpoint.events.amazonaws.com
                            │
               ┌────────────┴────────────┐
               │ Route 53 health check   │
               └────────────┬────────────┘
          (healthy)         │           (unhealthy)
               ▼                           ▼
      Primary region                Secondary region
      (us-east-1)                   (us-west-2)
      ┌──────────────┐              ┌──────────────┐
      │ Custom Bus   │              │ Custom Bus   │
      └──────┬───────┘              └──────┬───────┘
             │ (event replication -- same bus name in both regions)
             └──────────────────────────────┘
```

Key properties:
- Publishers use a single global endpoint -- no application-level awareness of which region is active
- Failover triggered by Route 53 health check failure (typically 30-60 seconds)
- Requires identically-named custom event buses in both regions with **event replication** enabled (bidirectional by default)
- Protects against regional EventBridge control plane failures, not just application-level issues
- Complements the multi-region patterns from the `multi-region-architecture-patterns.md` deep dive (Mar 31) and the database DR patterns from Apr 2 -- EventBridge now has a first-class multi-region data plane to match.

---

## Part 10: API Destinations -- Foreign Subscribers via HTTP

**The analogy**: A foreign subscriber who receives stories via international mail (HTTP). Because international mail requires authentication and rate limiting, you set up a formal connection with credentials and a delivery rate cap.

**The technical reality**: API Destinations let EventBridge invoke any HTTP endpoint as a rule target.

**Two resources**:
1. **Connection**: Stores authentication credentials (Basic auth, OAuth client credentials, or API Key). Credentials are stored in Secrets Manager (managed automatically).
2. **API Destination**: References the connection, defines the HTTP endpoint URL, method, and invocation rate limit.

```hcl
resource "aws_cloudwatch_event_connection" "webhook" {
  name               = "webhook-connection"
  authorization_type = "API_KEY"

  auth_parameters {
    api_key {
      key   = "x-api-key"
      value = var.webhook_api_key
    }
  }
}

resource "aws_cloudwatch_event_api_destination" "webhook" {
  name                             = "order-webhook"
  invocation_endpoint              = "https://api.partner.com/webhooks/orders"
  http_method                      = "POST"
  invocation_rate_limit_per_second = 10  # 1-300 per destination
  connection_arn                   = aws_cloudwatch_event_connection.webhook.arn
}
```

**Rate limiting**: 1-300 invocations per second per API Destination. This protects external APIs from being overwhelmed -- EventBridge queues excess invocations and delivers them as capacity allows. If the destination returns 429 (Too Many Requests) or 5xx errors, EventBridge retries using the configured retry policy, then routes to the DLQ.

### The Private Endpoint Trap

**API Destinations cannot reach private hostnames by default.** EventBridge invokes API Destinations from AWS-managed public infrastructure, which has no path into your VPC. If you point an API Destination at `https://customer-profiles.internal/...` or any RFC 1918 / Route 53 private hosted zone hostname, the call will fail in production with DNS resolution errors or connection timeouts. This is the #1 trap when migrating internal-service integrations onto Pipes/Rules with API Destinations -- the docs do not shout about it.

Three workarounds, in order of operational simplicity:

| Option | When to use | Trade-off |
|--------|-------------|-----------|
| **VPC-attached Lambda enrichment/target** | Simplest, lowest infra footprint | Adds Lambda compute cost (small, since it only runs for filtered traffic); leverages Hyperplane shared ENIs (no 10-30s cold start penalty per the Lambda deep dive Apr 6) |
| **API Gateway + VPC Link** | You already operate an API Gateway layer for auth/throttling | Heavier infra; pays API Gateway and VPC Link hourly fees |
| **EventBridge Connections + VPC Lattice resource configs** | Cleanest "no glue compute" option | Newest path (2024-2025); requires VPC Lattice setup; verify regional availability |

If you need the elegance of API Destinations but the target lives in a private network, the modern best practice is the VPC Lattice integration on the Connection. If that is not available, fall back to a VPC-attached Lambda as the enrichment or target.

---

## Part 11: Observability -- Metrics, Tracing, and Logs

**The analogy**: A newsroom that cannot measure its own throughput is flying blind. You need to know how many stories came in, how many made it to subscribers, how many bounced at the loading dock, and how long delivery took from press to front door. EventBridge exposes that operational telemetry through CloudWatch metrics, X-Ray tracing, and CloudTrail logs.

### CloudWatch Metrics -- The Newsroom Dashboard

All metrics are in the **`AWS/Events`** namespace. The critical ones:

| Metric | Dimension | What It Tells You |
|--------|-----------|-------------------|
| `Invocations` | RuleName, EventBusName | Total target invocations (successful + failed). The denominator for your success rate. |
| `FailedInvocations` | RuleName, EventBusName | Target invocations that failed after all retries. Your primary alarm metric. |
| `TriggeredRules` | RuleName, EventBusName | How many times a rule matched an event (not the same as Invocations -- a rule with 5 targets produces 5 Invocations per TriggeredRule). |
| `MatchedEvents` | EventBusName | Total events that matched at least one rule on the bus. |
| `ThrottledRules` | RuleName | Rules throttled because of target or global throughput limits -- alarm on any non-zero value. |
| `DeadLetterInvocations` | RuleName | Events sent to a target DLQ after retry exhaustion. |
| `IngestionToInvocationSuccessLatency` | RuleName, EventBusName | **Launched Sep 2024.** End-to-end latency from PutEvents to successful target delivery. The authoritative metric for "is EventBridge fast enough for my use case?" |
| `IngestionToInvocationStartLatency` | RuleName, EventBusName | Latency from ingestion to first delivery attempt (excludes target processing time). |
| `InvocationsFailedToBeSentToDlq` | RuleName | DLQ delivery itself failed -- these events are lost. This alarm is existential. |

**Scheduler and Pipes have their own namespaces**: `AWS/Scheduler` and `AWS/Pipes`. Scheduler publishes `InvocationAttemptCount`, `InvocationsFailedToBeSentToDeadLetterCount`, and `TargetErrorCount`. Pipes publishes `Invocations`, `ExecutionFailed`, `ExecutionThrottled`, `ExecutionPartialFailure`, and stage-specific latencies.

### X-Ray Distributed Tracing

EventBridge supports X-Ray active tracing on event buses. When enabled, the service propagates the trace header across the asynchronous boundary between PutEvents and target invocation. This lets you see a single trace graph spanning:

```
Client --> PutEvents --> EventBridge bus --> Rule --> Target (Lambda/SFN/API Destination)
```

Without X-Ray, the producer and consumer traces are disconnected segments; with it, you can see end-to-end latency and identify whether slowness is in the bus, the network, or the target. Pipes also support X-Ray tracing end-to-end across source, filter, enrichment, and target stages.

### CloudTrail Management Events

EventBridge control-plane API calls (PutRule, PutTargets, DeleteRule, CreateArchive, StartReplay, etc.) are logged in CloudTrail as management events by default. Data events (PutEvents) are **not** logged by default and must be enabled explicitly on a CloudTrail data event selector -- useful for audit but can be expensive at high volume.

### Alarming Recipe for Production

Every production EventBridge deployment should have at minimum:
1. **`FailedInvocations > 0`** per rule, over a 5-minute window
2. **`ThrottledRules > 0`** per rule (any throttling is a red flag)
3. **`ApproximateNumberOfMessagesVisible > 0`** on every target DLQ (any DLQ message is a silent delivery failure)
4. **`InvocationsFailedToBeSentToDlq > 0`** -- events lost entirely
5. **`IngestionToInvocationSuccessLatency` p99** against an SLO (e.g., < 2 seconds)

The Terraform example below wires up items 1, 2, and 3.

---

## Part 12: EventBridge vs SNS vs SQS -- The Decision Framework

This is the most frequent architectural interview question in the event-driven space. Tomorrow's SQS/SNS deep dive will cover those services in detail, but here is the decision framework from the EventBridge perspective:

| Dimension | EventBridge | SNS | SQS |
|-----------|------------|-----|-----|
| **Communication model** | Event-driven (content-based routing) | Push-based pub/sub (topic-based) | Pull-based queue (consumer polls) |
| **Filtering** | Advanced content-based (prefix, suffix, numeric, wildcard, $or, etc.) | Subscription filter policies (attribute-based, less powerful) | None (consumer sees all messages) |
| **Fan-out** | Rules (multiple rules per event) | Subscriptions (multiple subscribers per topic) | Single consumer per message (unless SQS FIFO + message groups) |
| **Ordering** | No guarantees | FIFO topics available | FIFO queues available |
| **Replay** | Yes (archive + replay) | No | No (once consumed, gone) |
| **Schema management** | Schema Registry + discovery | No | No |
| **Cross-account** | Resource-based policies on event buses | Cross-account subscriptions | Cross-account queue policies |
| **SaaS integration** | Partner event buses (40+ partners) | No | No |
| **Throughput** | Soft limit ~10,000 PutEvents/sec/region (adjustable) | Nearly unlimited | Nearly unlimited (Standard) |
| **Latency** | ~100-500ms typical (P99 dropped from ~2.2s in early 2023 to ~129ms by Aug 2024 after AWS infra improvements; measurable via the `IngestionToInvocationSuccessLatency` metric launched Sep 2024) | ~30ms | Consumer-dependent (long polling) |
| **Pricing** | $1.00/million custom events | $0.50/million publishes | $0.40/million requests (after 1M free) |
| **Persistence** | No (real-time delivery, archive is optional) | No (real-time delivery) | Yes (up to 14 days retention) |

### The Decision Tree

```
Do you need content-based routing beyond simple topic/attribute filtering?
├── YES ──▶ EventBridge
│
Do you need SaaS partner event integration?
├── YES ──▶ EventBridge
│
Do you need event archive and replay?
├── YES ──▶ EventBridge
│
Do you need schema discovery and code generation?
├── YES ──▶ EventBridge
│
Do you need simple topic-based fan-out with minimal latency?
├── YES ──▶ SNS (cheaper and lower latency for simple fan-out)
│
Do you need message buffering, rate control, or guaranteed processing?
├── YES ──▶ SQS (consumer controls pace, retry via visibility timeout)
│
Do you need strict message ordering?
├── YES ──▶ SQS FIFO or SNS FIFO
│
Default: ──▶ Start with EventBridge for flexibility, add SQS for buffering
```

### Common Compositions

These services compose rather than compete:
- **EventBridge -> SQS**: EventBridge routes by content, SQS buffers for reliable processing (most common pattern)
- **EventBridge -> SNS**: EventBridge filters, SNS fans out to SMS/email/HTTP/mobile push (SNS has delivery channels EventBridge lacks)
- **EventBridge -> SNS -> SQS**: Full chain -- content-based routing -> fan-out -> buffered processing
- **S3 -> EventBridge -> multiple targets**: Recommended over S3 direct notification for complex routing

---

## Part 13: Event-Driven Architecture Patterns

### Choreography vs Orchestration -- The Core Decision

**The analogy**: **Choreography** is like a newspaper-based city where every department independently reads the paper and decides what to do. No central coordinator. The fire department reads about a fire and dispatches trucks. The insurance company reads the same story and opens a claim. The hospital reads it and prepares beds. Nobody told them to do these things -- they each react independently to the published event. **Orchestration** is like air traffic control (Step Functions, Apr 8) where a central coordinator explicitly directs each step: "first dispatch fire trucks, then when the fire is out, open the insurance claim, then schedule the hospital follow-up."

| Dimension | Choreography (EventBridge) | Orchestration (Step Functions) |
|-----------|---------------------------|-------------------------------|
| **Coordination** | Decentralized -- services react independently | Centralized -- coordinator directs each step |
| **Coupling** | Loose -- producers/consumers are unaware of each other | Tight -- coordinator knows every participant |
| **Visibility** | Distributed -- no single view of the "workflow" | Centralized -- execution history shows every step |
| **Error handling** | Each service handles its own errors | Coordinator handles retries, catch, compensation |
| **Best for** | Loosely coupled domain events (2-4 step reactions) | Complex multi-step transactions requiring rollback |
| **Debugging** | Harder -- must trace events across services | Easier -- single execution history with visual graph |

**The key insight from this week**: Choreography and orchestration are **complementary, not competing**. A common production pattern:

```
EventBridge (choreography)                 Step Functions (orchestration)
─────────────────────────                 ────────────────────────────────
                                          
"Order Placed" event                      ┌─── Validate Order
  ├── Rule 1 ──▶ Step Functions ─────────▶├─── Process Payment (saga)
  │              (order fulfillment        ├─── Reserve Inventory
  │               workflow)                ├─── Ship Order
  │                                        └─── Send Confirmation
  ├── Rule 2 ──▶ Lambda                    
  │              (analytics tracker)       
  │                                        
  └── Rule 3 ──▶ SQS                      
                 (audit log queue)         
```

EventBridge handles the fan-out (choreography -- one event triggers multiple independent reactions). Step Functions handles the complex multi-step order fulfillment (orchestration -- sequential payment, inventory, shipping with saga compensation on failure).

### Event Sourcing

Store every state change as an immutable event. EventBridge Archive + Replay provides the infrastructure foundation: archive all events, replay them to rebuild state. However, EventBridge is not a full event sourcing solution on its own -- you need a durable event store (DynamoDB, Kinesis, or a purpose-built event store) for the source of truth. EventBridge acts as the event router, not the event store.

### CQRS (Command Query Responsibility Segregation)

Separate write (command) and read (query) models. EventBridge routes write-side events to read-side projections:

```
Write API ──▶ DynamoDB (command store)
                  │
                  ├── DynamoDB Streams ──▶ EventBridge Pipe ──▶ Lambda ──▶ ElastiCache (query store)
                  └── DynamoDB Streams ──▶ EventBridge Pipe ──▶ Lambda ──▶ OpenSearch (search index)
```

---

## Part 14: Pricing

| Component | Price | Free Tier |
|-----------|-------|-----------|
| **Custom/partner events published** | $1.00 per million (metered in 64 KB chunks) | None |
| **AWS service events** | Free | All |
| **Cross-account events** | $1.00 per million (charged once per side) | None |
| **Schema discovery** | $0.10 per million events ingested (metered in 8 KB chunks) | **5M events/month** |
| **Replayed events** | $1.00 per million | None |
| **Scheduler invocations** | $1.00 per million | 14M/month |
| **Pipes requests (after filter)** | $0.40 per million (events that pass the filter) | None |
| **Pipes data processed** | $0.15 per GB (region-dependent) | None |
| **Rule evaluation / filtering** | Free | All |
| **Archive processing (ingest)** | $0.10 per GB ingested | None |
| **Archive storage** | $0.023 per GB-month | None |

**Cost comparison for simple fan-out**: If you need to broadcast an event to 3 Lambda functions:
- **EventBridge**: $1.00/million events -- note this is charged **once at bus ingestion**, not per target invocation. Adding a 4th or 5th target to the same rule costs nothing extra.
- **SNS**: $0.50/million publishes + $0.00/million Lambda deliveries = $0.50/million
- **Result**: SNS is 50% cheaper for simple fan-out. EventBridge justifies the 2x premium with content-based routing, schema discovery, archive/replay, and cross-account/cross-region capabilities. The per-publish (not per-delivery) model means EventBridge becomes relatively cheaper as the number of targets grows -- 10 SNS subscribers still cost $0.50/million, but 10 EventBridge targets on one rule still cost only $1.00/million.

---

## Part 15: Limits and Quotas

| Limit | Default | Adjustable? |
|-------|---------|-------------|
| PutEvents entries per request | 10 | No (hard limit) |
| Event size | 256 KB | No (hard limit) |
| Rules per event bus | 300 (100 in some regions) | Yes (Service Quotas) |
| **Targets per rule** | **5** | **No (hard limit)** |
| PutEvents invocations/sec (per region) | 10,000 (soft limit) | Yes |
| PutEvents throughput (per region) | 24 MB/sec | Yes |
| Event buses per account | 100 | Yes |
| Archives per account | 300 | Yes |
| Concurrent replays per region | 10 | No |
| Maximum retry attempts per target | 185 | Configurable (0-185) |
| Maximum event age | 86,400 sec (24 h) | Configurable (60-86400) |
| API Destinations per account | 3,000 | Yes |
| Connections per account | 3,000 | Yes |
| API Destination invocation rate | 300/sec per destination | Configurable (1-300) |
| Scheduler schedules per account | 1,000,000 | Yes |
| Scheduler schedule groups per account | 500 | Yes |
| Pipes per account | **1,000** | Yes |
| Input transformer variables | 100 | No |
| Input transformer template | 8,192 characters | No |

---

## Production Terraform Example

This Terraform configuration deploys a comprehensive EventBridge architecture with a custom event bus, multiple rules with content-based filtering, input transformers, multi-target delivery (Lambda + SQS + Step Functions), archive with retention, Scheduler (cron + one-time), Pipe (SQS -> Lambda enrichment -> Step Functions), cross-account event bus policy, API Destination, and CloudWatch alarms.

```hcl
# ═══════════════════════════════════════════════════════════════════════
# EVENTBRIDGE DEEP DIVE -- PRODUCTION ARCHITECTURE
# ═══════════════════════════════════════════════════════════════════════
#
# Deploys:
# - Custom event bus with resource-based policy (cross-account via Org)
# - Rule 1: Order Placed -> Lambda (input transformer + retry policy + DLQ)
# - Rule 2: Order Placed high-value -> Step Functions (DLQ)
# - Rule 3: Order Placed -> SQS audit queue (DLQ)
# - API Destination rule for Order Shipped -> partner webhook (separate DLQ)
# - Archive with event pattern filter and 30-day retention
# - Scheduler: cron-based Lambda + one-time S3 DeleteObject (universal target)
# - Pipe: SQS source -> filter -> Lambda enrichment -> Step Functions target
# - CloudWatch alarms: FailedInvocations, ThrottledRules, DLQ depth (per DLQ),
#   InvocationsFailedToBeSentToDlq, IngestionToInvocationSuccessLatency p99 SLO
# - Least-privilege IAM roles for EventBridge, Scheduler, and Pipes
# ═══════════════════════════════════════════════════════════════════════

terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_region" "current" {}
data "aws_caller_identity" "current" {}
data "aws_organizations_organization" "current" {}

# ═══════════════════════════════════════════════════════════════════════
# CUSTOM EVENT BUS -- The business newspaper
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_event_bus" "orders" {
  name = "orders-bus"

  tags = {
    Environment = "production"
    Service     = "order-system"
  }
}

# Cross-account resource policy -- allow entire Organization to publish
resource "aws_cloudwatch_event_bus_policy" "orders_cross_account" {
  event_bus_name = aws_cloudwatch_event_bus.orders.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowOrganizationPutEvents"
        Effect    = "Allow"
        Principal = "*"
        Action    = "events:PutEvents"
        Resource  = aws_cloudwatch_event_bus.orders.arn
        Condition = {
          StringEquals = {
            "aws:PrincipalOrgID" = data.aws_organizations_organization.current.id
          }
        }
      }
    ]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# DEAD LETTER QUEUES -- One per target (never silently drop events)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sqs_queue" "rule1_dlq" {
  name                      = "orders-rule1-lambda-dlq"
  message_retention_seconds = 1209600  # 14 days
}

resource "aws_sqs_queue" "rule2_dlq" {
  name                      = "orders-rule2-sfn-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sqs_queue" "rule3_dlq" {
  name                      = "orders-rule3-audit-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sqs_queue" "webhook_dlq" {
  name                      = "orders-webhook-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sqs_queue" "scheduler_dlq" {
  name                      = "orders-scheduler-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sqs_queue" "audit_queue" {
  name                       = "order-audit-queue"
  visibility_timeout_seconds = 300
  message_retention_seconds  = 1209600
}

# ═══════════════════════════════════════════════════════════════════════
# RULE 1: Order Placed -> Lambda (with input transformer)
# Editorial desk: "All order events, reformat for the payment processor"
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_event_rule" "order_placed_lambda" {
  name           = "order-placed-to-payment-processor"
  description    = "Routes all Order Placed events to payment processing Lambda"
  event_bus_name = aws_cloudwatch_event_bus.orders.name

  event_pattern = jsonencode({
    source      = ["com.mycompany.orders"]
    detail-type = ["Order Placed"]
    detail = {
      shippingMethod = [{ "anything-but" : "pickup" }]
    }
  })

  tags = {
    Environment = "production"
    Rule        = "order-placed-lambda"
  }
}

resource "aws_cloudwatch_event_target" "order_placed_lambda" {
  rule           = aws_cloudwatch_event_rule.order_placed_lambda.name
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  target_id      = "payment-processor"
  arn            = var.payment_processor_lambda_arn

  # Input transformer -- reshape event for payment processor
  input_transformer {
    input_paths = {
      orderId    = "$.detail.orderId"
      customerId = "$.detail.customerId"
      total      = "$.detail.total"
      eventTime  = "$.time"
    }

    input_template = <<-TEMPLATE
      {
        "orderReference": <orderId>,
        "amount": <total>,
        "customer": <customerId>,
        "receivedAt": <eventTime>,
        "action": "process-payment"
      }
    TEMPLATE
  }

  # Retry policy -- give up faster for this target since stale payment
  # requests become meaningless after ~1 hour. Default is 24h / 185 attempts.
  retry_policy {
    maximum_event_age_in_seconds = 3600   # 1 hour
    maximum_retry_attempts       = 20
  }

  # Dead letter queue -- never silently drop events
  dead_letter_config {
    arn = aws_sqs_queue.rule1_dlq.arn
  }
}

# Lambda resource-based policy allowing EventBridge to invoke
resource "aws_lambda_permission" "allow_eventbridge_payment" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = var.payment_processor_lambda_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.order_placed_lambda.arn
}

# ═══════════════════════════════════════════════════════════════════════
# RULE 2: High-Value Order Placed -> Step Functions
# Editorial desk: "Orders over $500, send to fulfillment workflow"
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_event_rule" "high_value_order" {
  name           = "high-value-order-to-fulfillment"
  description    = "Routes high-value orders to Step Functions fulfillment workflow"
  event_bus_name = aws_cloudwatch_event_bus.orders.name

  event_pattern = jsonencode({
    source      = ["com.mycompany.orders"]
    detail-type = ["Order Placed"]
    detail = {
      total = [{ "numeric" : [">", 500] }]
    }
  })
}

resource "aws_cloudwatch_event_target" "high_value_sfn" {
  rule           = aws_cloudwatch_event_rule.high_value_order.name
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  target_id      = "fulfillment-workflow"
  arn            = var.fulfillment_state_machine_arn
  role_arn       = aws_iam_role.eventbridge_sfn.arn

  dead_letter_config {
    arn = aws_sqs_queue.rule2_dlq.arn
  }
}

# IAM role for EventBridge to start Step Functions executions
resource "aws_iam_role" "eventbridge_sfn" {
  name = "eventbridge-start-sfn-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "events.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "eventbridge_sfn" {
  name = "eventbridge-start-sfn"
  role = aws_iam_role.eventbridge_sfn.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid      = "StartExecution"
      Effect   = "Allow"
      Action   = "states:StartExecution"
      Resource = var.fulfillment_state_machine_arn
    }]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# RULE 3: All Order Events -> SQS Audit Queue
# Editorial desk: "All stories go to the archives department"
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_event_rule" "order_audit" {
  name           = "all-orders-to-audit"
  description    = "Routes all order events to SQS audit queue for compliance logging"
  event_bus_name = aws_cloudwatch_event_bus.orders.name

  event_pattern = jsonencode({
    source = ["com.mycompany.orders"]
  })
}

resource "aws_cloudwatch_event_target" "order_audit_sqs" {
  rule           = aws_cloudwatch_event_rule.order_audit.name
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  target_id      = "audit-queue"
  arn            = aws_sqs_queue.audit_queue.arn

  dead_letter_config {
    arn = aws_sqs_queue.rule3_dlq.arn
  }
}

# SQS resource-based policy allowing EventBridge to send messages
resource "aws_sqs_queue_policy" "audit_queue" {
  queue_url = aws_sqs_queue.audit_queue.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowEventBridgeSend"
      Effect    = "Allow"
      Principal = { Service = "events.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.audit_queue.arn
      Condition = {
        ArnEquals = {
          "aws:SourceArn" = aws_cloudwatch_event_rule.order_audit.arn
        }
      }
    }]
  })
}

# DLQ policies -- allow EventBridge to send to all DLQs
resource "aws_sqs_queue_policy" "dlq_policies" {
  for_each  = {
    rule1     = aws_sqs_queue.rule1_dlq
    rule2     = aws_sqs_queue.rule2_dlq
    rule3     = aws_sqs_queue.rule3_dlq
    webhook   = aws_sqs_queue.webhook_dlq
    scheduler = aws_sqs_queue.scheduler_dlq
  }

  queue_url = each.value.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowEventBridgeDLQ"
      Effect    = "Allow"
      Principal = { Service = "events.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = each.value.arn
    }]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# ARCHIVE -- The newspaper morgue (30-day retention)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_event_archive" "orders" {
  name             = "orders-archive"
  event_source_arn = aws_cloudwatch_event_bus.orders.arn
  retention_days   = 30

  # Archive only Order Placed events (not all bus traffic)
  event_pattern = jsonencode({
    source      = ["com.mycompany.orders"]
    detail-type = ["Order Placed", "Order Cancelled", "Order Shipped"]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# API DESTINATION -- External webhook with API Key auth
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_event_connection" "partner_webhook" {
  name               = "partner-webhook-connection"
  authorization_type = "API_KEY"

  auth_parameters {
    api_key {
      key   = "x-api-key"
      value = var.partner_webhook_api_key
    }
  }
}

resource "aws_cloudwatch_event_api_destination" "partner_webhook" {
  name                             = "partner-order-webhook"
  invocation_endpoint              = var.partner_webhook_url
  http_method                      = "POST"
  invocation_rate_limit_per_second = 50
  connection_arn                   = aws_cloudwatch_event_connection.partner_webhook.arn
}

# Rule to send Order Shipped events to external partner via API Destination
resource "aws_cloudwatch_event_rule" "order_shipped_webhook" {
  name           = "order-shipped-to-partner"
  event_bus_name = aws_cloudwatch_event_bus.orders.name

  event_pattern = jsonencode({
    source      = ["com.mycompany.orders"]
    detail-type = ["Order Shipped"]
  })
}

resource "aws_cloudwatch_event_target" "order_shipped_webhook" {
  rule           = aws_cloudwatch_event_rule.order_shipped_webhook.name
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  target_id      = "partner-webhook"
  arn            = aws_cloudwatch_event_api_destination.partner_webhook.arn
  role_arn       = aws_iam_role.eventbridge_api_dest.arn

  # Dedicated DLQ -- webhook failures should be isolated from internal targets
  # so partner outages do not pollute payment-processor reprocessing.
  dead_letter_config {
    arn = aws_sqs_queue.webhook_dlq.arn
  }
}

resource "aws_iam_role" "eventbridge_api_dest" {
  name = "eventbridge-api-destination-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "events.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "eventbridge_api_dest" {
  name = "eventbridge-invoke-api-dest"
  role = aws_iam_role.eventbridge_api_dest.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "events:InvokeApiDestination"
      Resource = aws_cloudwatch_event_api_destination.partner_webhook.arn
    }]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# EVENTBRIDGE SCHEDULER -- Alarm clock system
# ═══════════════════════════════════════════════════════════════════════

# Schedule group for organization
resource "aws_scheduler_schedule_group" "order_system" {
  name = "order-system-schedules"
}

# Cron-based schedule: daily report generation at 6 AM ET
resource "aws_scheduler_schedule" "daily_report" {
  name       = "daily-order-report"
  group_name = aws_scheduler_schedule_group.order_system.name

  schedule_expression          = "cron(0 6 * * ? *)"
  schedule_expression_timezone = "America/New_York"

  flexible_time_window {
    mode                      = "FLEXIBLE"
    maximum_window_in_minutes = 15
  }

  target {
    arn      = var.report_generator_lambda_arn
    role_arn = aws_iam_role.scheduler_role.arn

    input = jsonencode({
      reportType = "daily-orders"
      format     = "csv"
    })

    retry_policy {
      maximum_event_age_in_seconds = 3600   # Retry for up to 1 hour
      maximum_retry_attempts       = 3
    }

    dead_letter_config {
      arn = aws_sqs_queue.scheduler_dlq.arn
    }
  }
}

# One-time schedule: deferred cleanup (example -- would be created dynamically)
resource "aws_scheduler_schedule" "one_time_cleanup" {
  name       = "cleanup-temp-order-data"
  group_name = aws_scheduler_schedule_group.order_system.name

  schedule_expression          = "at(2026-04-15T09:00:00)"
  schedule_expression_timezone = "America/New_York"

  flexible_time_window {
    mode = "OFF"
  }

  target {
    # Universal target -- call S3 DeleteObject directly, no Lambda needed
    arn      = "arn:aws:scheduler:::aws-sdk:s3:deleteObject"
    role_arn = aws_iam_role.scheduler_role.arn

    input = jsonencode({
      Bucket = "temp-order-exports"
      Key    = "exports/2026-04/temp-batch.csv"
    })
  }
}

# IAM role for Scheduler
resource "aws_iam_role" "scheduler_role" {
  name = "eventbridge-scheduler-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "scheduler.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "scheduler_lambda" {
  name = "scheduler-invoke-lambda"
  role = aws_iam_role.scheduler_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "InvokeLambda"
        Effect   = "Allow"
        Action   = "lambda:InvokeFunction"
        Resource = var.report_generator_lambda_arn
      },
      {
        Sid      = "S3DeleteObject"
        Effect   = "Allow"
        Action   = "s3:DeleteObject"
        Resource = "arn:aws:s3:::temp-order-exports/*"
      },
      {
        Sid      = "SendToDLQ"
        Effect   = "Allow"
        Action   = "sqs:SendMessage"
        Resource = aws_sqs_queue.scheduler_dlq.arn
      }
    ]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# EVENTBRIDGE PIPE -- Dedicated courier: SQS -> Filter -> Lambda Enrich -> SFN
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sqs_queue" "pipe_source" {
  name                       = "order-enrichment-source"
  visibility_timeout_seconds = 300
}

resource "aws_pipes_pipe" "order_enrichment" {
  name     = "order-enrichment-pipe"
  role_arn = aws_iam_role.pipes_role.arn
  source   = aws_sqs_queue.pipe_source.arn
  target   = var.fulfillment_state_machine_arn

  source_parameters {
    sqs_queue_parameters {
      batch_size                         = 10
      maximum_batching_window_in_seconds = 30
    }

    filter_criteria {
      filter {
        pattern = jsonencode({
          body = {
            orderType = ["wholesale"]
            total     = [{ "numeric" : [">", 1000] }]
          }
        })
      }
    }
  }

  enrichment = var.order_enrichment_lambda_arn

  enrichment_parameters {
    input_template = jsonencode({
      orderId    = "<$.body.orderId>"
      customerId = "<$.body.customerId>"
    })
  }

  target_parameters {
    step_function_state_machine_parameters {
      invocation_type = "FIRE_AND_FORGET"
    }
  }

  tags = {
    Environment = "production"
    Service     = "order-enrichment"
  }
}

# IAM role for Pipes
resource "aws_iam_role" "pipes_role" {
  name = "eventbridge-pipes-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "pipes.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "pipes_policy" {
  name = "pipes-source-enrich-target"
  role = aws_iam_role.pipes_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadFromSQS"
        Effect = "Allow"
        Action = [
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes"
        ]
        Resource = aws_sqs_queue.pipe_source.arn
      },
      {
        Sid      = "InvokeEnrichmentLambda"
        Effect   = "Allow"
        Action   = "lambda:InvokeFunction"
        Resource = var.order_enrichment_lambda_arn
      },
      {
        Sid      = "StartStepFunctions"
        Effect   = "Allow"
        Action   = "states:StartExecution"
        Resource = var.fulfillment_state_machine_arn
      }
    ]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# CLOUDWATCH ALARMS -- Monitor failed invocations and DLQ depth
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sns_topic" "eventbridge_alarms" {
  name = "eventbridge-alarms"
}

# Alarm: Failed invocations on any rule
resource "aws_cloudwatch_metric_alarm" "failed_invocations" {
  alarm_name          = "eventbridge-failed-invocations"
  alarm_description   = "EventBridge target invocations are failing"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "FailedInvocations"
  namespace           = "AWS/Events"
  period              = 300
  statistic           = "Sum"
  threshold           = 5
  treat_missing_data  = "notBreaching"

  dimensions = {
    RuleName = aws_cloudwatch_event_rule.order_placed_lambda.name
  }

  alarm_actions = [aws_sns_topic.eventbridge_alarms.arn]
  ok_actions    = [aws_sns_topic.eventbridge_alarms.arn]
}

# Alarm: Throttled rules (hitting throughput limits)
resource "aws_cloudwatch_metric_alarm" "throttled_rules" {
  alarm_name          = "eventbridge-throttled-rules"
  alarm_description   = "EventBridge rules are being throttled"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ThrottledRules"
  namespace           = "AWS/Events"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  treat_missing_data  = "notBreaching"

  alarm_actions = [aws_sns_topic.eventbridge_alarms.arn]
}

# Alarm: Events lost entirely (DLQ delivery itself failed)
resource "aws_cloudwatch_metric_alarm" "dlq_delivery_failed" {
  alarm_name          = "eventbridge-dlq-delivery-failed"
  alarm_description   = "Events were lost -- DLQ delivery itself failed (existential alarm)"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "InvocationsFailedToBeSentToDlq"
  namespace           = "AWS/Events"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  treat_missing_data  = "notBreaching"

  alarm_actions = [aws_sns_topic.eventbridge_alarms.arn]
}

# Alarm: End-to-end latency SLO (P99 ingestion-to-delivery latency)
# Requires the IngestionToInvocationSuccessLatency metric (launched Sep 2024)
resource "aws_cloudwatch_metric_alarm" "p99_latency_slo" {
  alarm_name          = "eventbridge-p99-latency-slo"
  alarm_description   = "EventBridge end-to-end p99 latency exceeds 2s SLO"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "IngestionToInvocationSuccessLatency"
  namespace           = "AWS/Events"
  period              = 300
  extended_statistic  = "p99"
  threshold           = 2000  # milliseconds
  treat_missing_data  = "notBreaching"

  dimensions = {
    EventBusName = aws_cloudwatch_event_bus.orders.name
  }

  alarm_actions = [aws_sns_topic.eventbridge_alarms.arn]
}

# Alarm: DLQ depth -- messages accumulating means targets are failing
resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  for_each = {
    rule1     = aws_sqs_queue.rule1_dlq.name
    rule2     = aws_sqs_queue.rule2_dlq.name
    rule3     = aws_sqs_queue.rule3_dlq.name
    webhook   = aws_sqs_queue.webhook_dlq.name
    scheduler = aws_sqs_queue.scheduler_dlq.name
  }

  alarm_name          = "eventbridge-dlq-depth-${each.key}"
  alarm_description   = "DLQ ${each.value} has messages -- target delivery is failing"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  treat_missing_data  = "notBreaching"

  dimensions = {
    QueueName = each.value
  }

  alarm_actions = [aws_sns_topic.eventbridge_alarms.arn]
}

# ═══════════════════════════════════════════════════════════════════════
# VARIABLES
# ═══════════════════════════════════════════════════════════════════════

variable "payment_processor_lambda_arn" {
  description = "ARN of the payment processor Lambda function"
  type        = string
}

variable "payment_processor_lambda_name" {
  description = "Name of the payment processor Lambda function"
  type        = string
}

variable "fulfillment_state_machine_arn" {
  description = "ARN of the order fulfillment Step Functions state machine"
  type        = string
}

variable "report_generator_lambda_arn" {
  description = "ARN of the report generator Lambda function"
  type        = string
}

variable "order_enrichment_lambda_arn" {
  description = "ARN of the order enrichment Lambda function (Pipe enrichment)"
  type        = string
}

variable "partner_webhook_api_key" {
  description = "API key for partner webhook"
  type        = string
  sensitive   = true
}

variable "partner_webhook_url" {
  description = "HTTPS endpoint for partner webhook"
  type        = string
}

# ═══════════════════════════════════════════════════════════════════════
# OUTPUTS
# ═══════════════════════════════════════════════════════════════════════

output "event_bus_arn" {
  description = "ARN of the custom orders event bus"
  value       = aws_cloudwatch_event_bus.orders.arn
}

output "event_bus_name" {
  description = "Name of the custom orders event bus"
  value       = aws_cloudwatch_event_bus.orders.name
}

output "archive_name" {
  description = "Name of the orders event archive"
  value       = aws_cloudwatch_event_archive.orders.name
}

output "pipe_arn" {
  description = "ARN of the order enrichment pipe"
  value       = aws_pipes_pipe.order_enrichment.arn
}
```

**Key Terraform patterns**:

1. **`aws_cloudwatch_event_*` prefix for event bus resources**: EventBridge resources in the AWS Terraform provider still use the legacy CloudWatch Events naming. The `aws_pipes_pipe` and `aws_scheduler_schedule` resources use newer naming. Do not be confused by the `cloudwatch_event` prefix -- these are EventBridge resources.

2. **Separate DLQ per target**: Each target has its own DLQ with an SQS resource-based policy allowing `events.amazonaws.com`. This enables per-target failure isolation and independent reprocessing.

3. **Resource-based policies vs IAM roles**: Lambda targets use resource-based policies (`aws_lambda_permission`) for invocation. Step Functions, API Destinations, and Scheduler targets use IAM roles. SQS and SNS targets use queue/topic policies. This asymmetry is a common source of errors.

4. **Scheduler service principal is `scheduler.amazonaws.com`**, not `events.amazonaws.com`. Pipes service principal is `pipes.amazonaws.com`. Each EventBridge sub-service has its own principal.

5. **`for_each` on DLQ alarms**: Creates one CloudWatch alarm per DLQ, ensuring every failure path is monitored. Any message in a DLQ means a target is failing -- alarm on threshold 0.

---

## Critical Gotchas and Interview Traps

**1. "PutEvents can PARTIALLY succeed -- a batch of 10 entries can have 3 failures and 7 successes."**
The PutEvents API returns a `FailedEntryCount` and individual entry error codes. A 200 HTTP response does NOT mean all events were published. You must check `FailedEntryCount` and retry failed entries individually. This is different from SQS SendMessageBatch, which also partially succeeds, and different from Step Functions StartExecution, which is all-or-nothing.

**2. "EventBridge event bus resources in Terraform use `aws_cloudwatch_event_*` -- not `aws_eventbridge_*`."**
This legacy naming from when EventBridge was CloudWatch Events trips up every new Terraform user. `aws_cloudwatch_event_bus`, `aws_cloudwatch_event_rule`, `aws_cloudwatch_event_target` are EventBridge resources despite the prefix. Only Pipes (`aws_pipes_pipe`) and Scheduler (`aws_scheduler_schedule`) use newer naming.

**3. "Event patterns match on field PRESENCE, not absence. If your pattern specifies a field the event does not have, it does not match."**
A pattern `{"detail": {"error": ["timeout"]}}` does NOT match events where the `error` field is missing. To match events where a field is absent, use `{"detail": {"error": [{"exists": false}]}}`. This asymmetry is the source of many "my rule is not firing" debugging sessions.

**4. "All rules on a bus evaluate every event -- there is no short-circuit or priority."**
Unlike Step Functions Choice state (first match wins), every rule evaluates every event independently. If 10 rules match, all 10 sets of targets fire. This means a single event can trigger 10 x 5 = 50 target invocations. This is by design (fan-out), but it means misconfigured rules can cause unexpected load spikes and costs.

**5. "EventBridge has no native ordering guarantee. Events can arrive at targets in any order."**
Unlike SQS FIFO or Kinesis (which preserves order per partition key), EventBridge event buses provide best-effort delivery with no ordering guarantees. If "Order Placed" must be processed before "Order Shipped," you need either SQS FIFO or application-level ordering logic. This is a fundamental architectural constraint.

**6. "Replayed events are treated as NEW events -- all rules evaluate them again, not just the 'new subscriber' you intended."**
When you replay archived events to catch up a new service, every existing rule also processes them. Unless existing consumers are idempotent, they will re-execute. Use the `replay-name` field in event patterns to create replay-aware rules: `{"replay-name": [{"exists": false}]}` for rules that should ignore replays, or `{"replay-name": ["my-replay"]}` for rules that should only process replays.

**7. "The 5-targets-per-rule limit is a HARD cap and is NOT adjustable via Service Quotas."**
If you need to send one event type to 15 targets, you need at least 3 rules (5 targets each) all matching the identical event pattern. Each rule is evaluated independently, so duplicating the pattern works correctly. Unlike most EventBridge quotas, the 5-targets-per-rule limit cannot be raised -- this is often mistaken in documentation and blog posts.

**8. "EventBridge Scheduler and EventBridge Scheduled Rules are DIFFERENT services."**
Legacy `aws_cloudwatch_event_rule` with `schedule_expression` creates a scheduled rule on the default event bus. `aws_scheduler_schedule` creates a Scheduler schedule. Scheduler is strictly superior (timezone, one-time, universal targets, flexible windows, higher throughput). Always use Scheduler for new workloads. But legacy scheduled rules still exist in documentation and exam questions.

**9. "Pipes enrichment only supports Express Workflows -- not Standard."**
If you configure a Step Functions state machine as a Pipes enrichment, it must be an Express Workflow. Standard Workflows are not supported for enrichment because enrichment is synchronous (Pipes waits for the response), and Standard Workflows can run for up to a year. Standard Workflows ARE supported as Pipe targets (fire-and-forget).

**10. "Cross-region event bus targets are limited to OTHER EVENT BUSES -- you cannot directly target a Lambda or SQS in another region."**
To invoke a Lambda function in eu-west-1 from a rule in us-east-1, you must route through an intermediate event bus: us-east-1 rule -> eu-west-1 event bus -> eu-west-1 rule -> Lambda. This two-hop pattern adds latency and requires managing rules in both regions.

**11. "DLQ on targets is configured per-target, not per-rule and not per-bus."**
A rule with 3 targets needs 3 DLQ configurations. If any target lacks a DLQ, events that fail delivery to that specific target are silently dropped after the 24-hour retry window. The other targets (with DLQs) are unaffected. This is different from Lambda async DLQ (per function) and SQS DLQ (per queue).

**12. "AWS service events on the default bus are free, but the SAME events sent to a custom bus cost $1/million."**
If you create a rule on the default bus that forwards EC2 state change events to a custom bus, those events are now custom events on the target bus and incur charges. The "free" designation applies only to the initial publication on the default bus by the AWS service.

**13. "Input transformer template values that are strings MUST be quoted. Numeric and boolean values must NOT be quoted."**
In the input template, `"orderId": <orderId>` works when `orderId` is a string (the variable includes quotes). But `"amount": <total>` works when `total` is a number. Getting the quoting wrong produces malformed JSON that silently fails. Test input transformers with the "Test event pattern" console feature before deployment.

**14. "EventBridge Pipes charges are per request AFTER filtering -- but source polling still costs money."**
Pipes filtering is a cost optimization because you pay $0.40/million only for events that pass the filter. However, the source polling itself (e.g., SQS ReceiveMessage calls) still incurs SQS costs. For a high-volume SQS queue where you filter 90% of messages, you pay full SQS costs for all messages but only Pipes costs for the 10% that pass.

**15. "The Schema Registry discovery lag means newly discovered schemas may not appear immediately."**
Schema discovery is eventually consistent. A new event type published to a bus may take several minutes before its schema appears in the registry. Do not build automation that publishes an event and immediately queries for its schema.

**16. "The PutEvents `Resources` field accepts ANY string -- EventBridge does not validate ARN syntax."**
The `Resources` array on a `PutEntry` is documented as "AWS resources that the event primarily concerns," but EventBridge will happily accept `["potato", "banana"]` and ship it without complaint. This means producers can invent ARN-shaped strings (e.g., `arn:aws:dynamodb:...:table/orders/item/{orderId}` -- there is no item-level DynamoDB ARN format) and ship them indefinitely. Downstream consumers that *trust* the field as a real ARN -- pattern-match rules using the `resources` filter, lookup tools that try to call `GetItem` from the parsed ARN, audit/compliance scanners building resource inventories -- silently break or produce nonsense. **Use only valid ARNs in `Resources`, and put business identifiers in the `Detail` payload where they belong.**

**17. "Scheduler one-time `at()` schedules persist forever after firing -- always set `ActionAfterCompletion: DELETE`."**
Without it, completed schedules accumulate indefinitely and eventually exhaust your account quota. This is the #1 footgun with one-time schedules and the reason the auto-delete option was added. Set it on every `at()` schedule, always.

**18. "Scheduler IAM failures are silent. The wrong service principal (`events.amazonaws.com` instead of `scheduler.amazonaws.com`) means the schedule deploys cleanly but never invokes anything."**
Scheduler does not raise an error when AssumeRole fails -- it just bumps an `InvocationAttemptCount` / DLQ-related metric and moves on. Unless you are alarming on Scheduler-specific failure metrics, you will not notice for days. Each EventBridge sub-service has its own service principal (`events`, `scheduler`, `pipes`); the trust policies are not interchangeable.

**19. "Scheduler DST: spring-forward schedules are SKIPPED, fall-back schedules fire ONCE only."**
A `cron(30 2 * * ? *)` in `America/New_York` will not fire at all on the spring-forward Sunday (the 2:30 AM wall-clock time does not exist that day). A `cron(30 1 * * ? *)` in the same timezone will fire only once on the fall-back Sunday even though 1:30 AM happens twice. Schedule batch jobs outside the DST-affected hours (4:30 AM is safe both directions; 2:30 AM is dangerous).

---

## Key Takeaways

- **EventBridge is a content-based event router, not a queue and not a simple pub/sub system.** It occupies a fundamentally different architectural niche from SQS (buffering) and SNS (topic-based fan-out). The premium you pay ($1/million vs $0.50/million SNS) buys you advanced filtering, schema discovery, archive/replay, and cross-account/region routing.

- **Three EventBridge sub-services serve three distinct purposes.** Event buses + rules = many-to-many content-based routing (the newspaper). Scheduler = time-based triggers (the alarm clock). Pipes = point-to-point source-to-target pipelines with enrichment (the dedicated courier). Conflating these three is the most common conceptual mistake.

- **Always configure a DLQ on every target.** An EventBridge target without a DLQ is a silent failure point. EventBridge retries with exponential backoff up to 185 attempts within the 24-hour event-age window (both knobs configurable per target), then drops the event forever. This mirrors the Lambda async DLQ principle -- never allow silent event loss.

- **Choreography (EventBridge) and orchestration (Step Functions) are complementary.** Use EventBridge for loosely coupled domain event fan-out. Use Step Functions for complex multi-step transactions with error handling and compensation. A production system typically uses both.

- **PutEvents partially succeeds -- always check FailedEntryCount.** A 200 HTTP response does not mean all events were published. This is the number one production bug with EventBridge custom events.

- **Replayed events trigger ALL rules, not just the one you intended.** Build idempotent consumers (Lambda Powertools, DynamoDB conditional writes) and use the `replay-name` field in event patterns to create replay-aware routing.

- **Scheduler supersedes scheduled rules.** One-time schedules, timezone support, flexible time windows, universal targets (270+ services), and higher throughput. Use Scheduler for all new time-based triggers. Legacy scheduled rules exist for backward compatibility only.

- **Pipes replace the "Lambda glue" pattern.** If you have a Lambda function whose only job is to read from SQS, add context, and invoke Step Functions, replace it with a Pipe. Same principle as Step Functions direct SDK integrations (Apr 8) and API Gateway direct service integrations (Apr 7) -- eliminate unnecessary compute.

- **Cross-region targets must be other event buses.** You cannot target a Lambda function or SQS queue in another region directly. Use the intermediate bus pattern for cross-region delivery.

- **Event pattern matching operates on field presence, not absence.** Use `{"exists": false}` to match missing fields. The default behavior (field specified in pattern but missing in event = no match) catches many developers by surprise.

- **EventBridge Terraform resources use the legacy `aws_cloudwatch_event_*` prefix.** Only Pipes and Scheduler resources use newer naming conventions. Each sub-service has its own IAM service principal (`events.amazonaws.com`, `scheduler.amazonaws.com`, `pipes.amazonaws.com`).

---

## Study Resources

Curated reading list for this topic: [`study-resources/aws/eventbridge-event-driven.md`](../../study-resources/aws/eventbridge-event-driven.md)

**Key references**:
- [EventBridge Event Patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html) -- Complete filtering operator reference
- [EventBridge Targets](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html) -- All 28+ target types
- [EventBridge Pipes](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes.html) -- Source-filter-enrich-target pipeline
- [EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html) -- Rate/cron/one-time with universal targets
- [SNS vs SQS vs EventBridge Decision Guide](https://docs.aws.amazon.com/decision-guides/latest/sns-or-sqs-or-eventbridge/sns-or-sqs-or-eventbridge.html) -- The official AWS decision framework
- [Choreography vs Orchestration](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/orchestration-choreography.html) -- AWS Prescriptive Guidance

**Cross-references in this repo**:
- [Lambda Deep Dive](./2026-04-06-lambda-deep-dive.md) -- Invocation models, async destinations (EventBridge as destination), idempotency patterns
- [API Gateway Deep Dive](./2026-04-07-api-gateway-deep-dive.md) -- Direct AWS service integrations, API Gateway as EventBridge Pipes enrichment
- [Step Functions Workflow Orchestration](./2026-04-08-step-functions-workflow-orchestration.md) -- Orchestration complement to choreography, EventBridge as trigger source
- [S3 Advanced Deep Dive](../../../2026/march/aws/2026-03-23-s3-advanced-deep-dive.md) -- S3 -> EventBridge notification delivery (recommended path)
