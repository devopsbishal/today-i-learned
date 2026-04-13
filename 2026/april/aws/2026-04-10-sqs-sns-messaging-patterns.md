# Amazon SQS, SNS & Messaging Patterns -- The Airport Baggage System Analogy

> You built the on-demand kitchen four days ago (Lambda) -- a serverless compute engine that spins up cooking stations per order, charges by the millisecond, and scales to thousands of concurrent dishes. Three days ago, you installed the grand hotel concierge desk (API Gateway) -- the front-of-house that receives guests, verifies credentials, enforces rate limits, and routes requests. Two days ago, you deployed the air traffic control tower (Step Functions) -- the central coordinator that sequences takeoffs, routes planes to parallel runways, retries approaches, and orchestrates complex multi-step operations. Yesterday, you launched the city newspaper system (EventBridge) -- the content-based event router that publishes stories to editorial desks, fans out to subscribers, archives history, and schedules alarms. But there are two critical problems the newspaper cannot solve alone. **First: what happens when a subscriber is overwhelmed?** The newspaper pushes stories immediately, but what if the fire department is already fighting three fires and cannot handle a fourth dispatch right now? The story is lost -- the newspaper does not hold stories for later. **Second: what happens when the same event needs to reach dozens of independent processors at different speeds, with guaranteed delivery, and each processor wants to work at its own pace?** The newspaper's 5-targets-per-rule hard limit, lack of message persistence, and push-only model all work against this scenario. What you need are the two foundational messaging primitives that sit underneath almost every decoupled AWS architecture: **Amazon SQS** (Simple Queue Service) and **Amazon SNS** (Simple Notification Service). The analogy that captures both services -- and how they compose together -- is an **international airport baggage handling system**. Think of **SQS as the baggage carousel** at arrivals. Bags (messages) arrive from the airplane (producer), enter a holding area (the queue), and sit on the carousel rotating until a passenger (consumer) picks them up. If nobody grabs a bag within a timeout, it becomes visible again for the next person. Bags that circle too many times without being claimed get sent to the lost-and-found office (dead letter queue). The carousel can be a **standard carousel** (bags arrive in roughly the order they were loaded, but wind and conveyor physics might shuffle a few -- best-effort ordering, nearly unlimited throughput) or a **FIFO carousel** (bags arrive in exact loading order per flight -- strict per-group ordering, but the carousel moves slower because it must maintain sequence). Think of **SNS as the airport PA system** (public announcement). When Flight 747 lands, the PA broadcasts the announcement to every listener simultaneously -- Gate 12 ground crew, baggage handlers at Carousel 5, the customs desk, the VIP lounge, and the taxi rank outside. Each listener hears the same announcement but takes different action. The PA system can also apply **frequency filters** -- the ground crew tunes into maintenance announcements only, customs listens for international arrivals only, and the VIP lounge filters for first-class passengers only. The broadcast is instant and push-based: listeners do not poll the PA system. But the PA system does not store announcements -- if a listener is offline when the broadcast happens and there is no queue catching the message, it is gone. The architectural power comes from **combining the PA system with carousels**: SNS broadcasts a flight arrival announcement, and each downstream service has its own SQS carousel catching the relevant bags. The ground crew carousel, the customs carousel, and the VIP lounge carousel each receive a copy of the bags, apply their own processing speed, and handle failures independently. This is the **fan-out pattern** -- the single most important messaging architecture in AWS. When you add EventBridge (yesterday's city newspaper) into the picture, you get the complete messaging story: **EventBridge for intelligent content-based routing, SNS for push-based fan-out, and SQS for buffered pull-based processing**. These services compose rather than compete, and understanding when to use each -- and how to wire them together -- is what separates someone who uses messaging from an architect who designs resilient distributed systems.

---

**Date**: 2026-04-10
**Topic Area**: aws
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| SQS Standard Queue | Standard baggage carousel -- bags arrive in roughly loading order, massive throughput, rare duplicates possible | Nearly unlimited TPS, best-effort ordering, at-least-once delivery; consumer polls and deletes after processing |
| SQS FIFO Queue | FIFO carousel -- bags arrive in exact loading order per flight (MessageGroupId), slower belt speed | 300 msg/s per API (3K batched, 70K+ high-throughput mode), strict per-group ordering, exactly-once dedup within 5-min window |
| Visibility Timeout | Claim tag -- when you grab a bag, it becomes invisible to others for N seconds; if you do not confirm pickup (delete), it reappears | Default 30s, max 12h; for Lambda set to 6x function timeout + MaximumBatchingWindowInSeconds; extendable via ChangeMessageVisibility |
| Dead Letter Queue | Lost-and-found office -- bags circled the carousel too many times without being claimed (maxReceiveCount exceeded) | Regular SQS queue referenced by redrive policy; must match source type (Standard->Standard, FIFO->FIFO); alarm on ApproximateNumberOfMessagesVisible |
| Long Polling | Patient waiting at carousel -- stand for up to 20s for bags instead of glancing and leaving immediately | ReceiveMessageWaitTimeSeconds 1-20s; samples all servers; reduces empty responses and cost by ~97% vs short polling |
| Message Deduplication | Duplicate bag tag scanner -- SHA-256 content hash or explicit ID prevents the same bag from entering the carousel twice within 5 minutes | Content-based (SHA-256 of body) or explicit MessageDeduplicationId; 5-minute dedup interval; FIFO only |
| MessageGroupId | Flight number on bag tag -- bags from Flight 747 maintain strict order among themselves, while Flight 123 bags are processed in parallel | Per-group ordering + parallel processing across groups; the partition key for FIFO queues; propagates from SNS FIFO topic |
| SNS Standard Topic | Airport PA system -- broadcasts to all listeners simultaneously with nearly unlimited throughput | Push-based pub/sub; subscribers: SQS, Lambda, HTTP/S, Email, SMS, Firehose, mobile push; $0.50/M publishes (delivery to SQS/Lambda free) |
| SNS FIFO Topic | PA system with guaranteed announcement order per channel -- only delivers to SQS FIFO (and SQS Standard) queues | 3,000/s per topic (300/s per group), strict ordering, exactly-once dedup; CANNOT deliver to Lambda, HTTP/S, Email, SMS; MessageGroupId propagates to SQS FIFO |
| Subscription Filter Policy | PA frequency filter -- ground crew tunes to maintenance-only, customs to international-only | Filter on MessageAttributes (default) or MessageBody (opt-in FilterPolicyScope); operators: exact, prefix, suffix, anything-but, numeric range, exists, IP range, wildcard, equals-ignore-case |
| SNS Subscription DLQ | Per-listener lost-and-found -- if the customs desk is closed when the PA broadcasts, the undelivered announcement goes to customs' dedicated box | Per-subscription (NOT per-topic); SQS queue with SNS SendMessage permission; filtered-out messages never reach DLQ (filtering is not failure) |
| Fan-Out Pattern | PA announcement -> multiple carousels -- one flight arrival triggers independent processing at ground crew, customs, VIP lounge, each at its own pace | SNS topic -> N SQS queues; the canonical decoupling pattern; each queue processes independently with its own DLQ and scaling |
| Buffering Pattern | Carousel as shock absorber -- bursty arrivals are absorbed by the carousel so customs can process at a steady pace | SQS between a bursty producer and rate-limited consumer; consumer controls pace via polling rate, batch size, Lambda concurrency |
| SQS Extended Client | Oversized luggage counter -- bag too large for carousel (>256 KB), so you store it in the oversized luggage area (S3) and put a claim ticket on the carousel | Java/Python library stores payload in S3, sends pointer via SQS; consumer must also use Extended Client to retrieve from S3 |
| Raw Message Delivery | Skip the PA envelope -- deliver the raw message body to SQS/HTTP/Firehose subscribers without wrapping in the SNS JSON metadata envelope | Enabled per-subscription; avoids double-parsing; recommended for SQS subscribers that do not need SNS envelope metadata |
| Delay Queue | Holding pen before carousel -- bags wait in a staging area for 0-15 minutes before appearing on the carousel | Queue-level DelaySeconds (0-900); per-message delay on Standard only (not FIFO); use cases: retry backoff, scheduled processing |

---

## The Airport Baggage System Framework

Every SQS and SNS concept maps to a role in an airport baggage handling operation. Producers are airplanes unloading bags. Queues are baggage carousels. Consumers are passengers picking up bags. Topics are the PA system broadcasting arrivals. Subscriptions are the listener tuning into specific frequencies. Dead letter queues are lost-and-found offices. The entire system operates on two complementary models: **pull-based** (passengers walk to the carousel and grab their bags when ready) and **push-based** (the PA system broadcasts to every listener simultaneously).

The baggage system has four operational dimensions:

1. **The Carousels (SQS Queues)** -- Two carousel types: Standard (high throughput, best-effort order) and FIFO (strict per-flight ordering, slower belt speed). Each carousel has a visibility timeout, a lost-and-found office (DLQ), and tunable polling behavior.

2. **The PA System (SNS Topics)** -- Push-based broadcast to every registered listener. Standard PA (nearly unlimited throughput) and FIFO PA (ordered announcements, SQS-only delivery). Frequency filters (subscription filter policies) route messages to relevant listeners.

3. **The Messaging Patterns** -- How carousels and PA systems compose: fan-out, buffering/throttling, priority queues, request-response, ordered processing, competing consumers, and poison pill handling.

4. **The Decision Framework** -- SQS vs SNS vs EventBridge: when to use each, when to combine them, and how they complement the services you studied this week.

```
THE AIRPORT BAGGAGE SYSTEM ARCHITECTURE -- SQS & SNS OVERVIEW
==============================================================================

  PRODUCERS                    MESSAGING LAYER                    CONSUMERS
  ---------                    ---------------                    ---------

  ┌────────────┐              ┌──────────────────────────────┐
  │ Airplane 1 │──SendMsg───▶│  SQS STANDARD QUEUE          │   ┌──────────┐
  │ (Producer) │              │  (standard carousel)          │──▶│ Passenger│
  └────────────┘              │  ┌──────┐ ┌──────┐ ┌──────┐ │   │ (Lambda) │
                              │  │ msg3 │ │ msg1 │ │ msg2 │ │   └──────────┘
                              │  └──────┘ └──────┘ └──────┘ │
                              │  ~unlimited TPS, best-effort │
                              │  ordering, at-least-once     │
                              └─────────────┬────────────────┘
                                            │ maxReceiveCount exceeded
                                            ▼
                              ┌──────────────────────────────┐
                              │  DLQ (lost-and-found)        │──▶ CloudWatch
                              │  Same type as source queue   │    Alarm!
                              └──────────────────────────────┘

  ┌────────────┐              ┌──────────────────────────────┐
  │ Airplane 2 │──SendMsg───▶│  SQS FIFO QUEUE (.fifo)      │   ┌──────────┐
  │ (Producer) │              │  (FIFO carousel)              │──▶│ Worker   │
  └────────────┘              │  ┌──────┐ ┌──────┐ ┌──────┐ │   │ (Lambda) │
                              │  │ msg1 │ │ msg2 │ │ msg3 │ │   └──────────┘
                              │  │ grpA │ │ grpB │ │ grpA │ │
                              │  └──────┘ └──────┘ └──────┘ │
                              │  300/s (3K batch, 70K+ HT)   │
                              │  strict per-group ordering   │
                              └──────────────────────────────┘


                   THE FAN-OUT PATTERN (PA System -> Carousels)
                   ────────────────────────────────────────────

  ┌────────────┐    Publish    ┌──────────────────────────────┐
  │ Airplane   │──────────────▶│  SNS TOPIC (PA system)       │
  │ (Producer) │               │  Standard: ~unlimited TPS    │
  └────────────┘               │  FIFO: 3K pub/s              │
                               └──────┬──────┬──────┬─────────┘
                                      │      │      │
                    ┌─────────────────┘      │      └──────────────────┐
                    │ filter: type=domestic   │ filter: type=intl      │ no filter
                    ▼                        ▼                        ▼
  ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
  │ SQS: Ground Crew     │ │ SQS: Customs Desk    │ │ SQS: Analytics       │
  │ (carousel 1)         │ │ (carousel 2)         │ │ (carousel 3)         │
  │ ┌──────┐ ┌──────┐   │ │ ┌──────┐ ┌──────┐   │ │ ┌──────┐ ┌──────┐   │
  │ │ msg1 │ │ msg3 │   │ │ │ msg2 │ │ msg4 │   │ │ │ msg1 │ │ msg2 │   │
  │ └──────┘ └──────┘   │ │ └──────┘ └──────┘   │ │ │ msg3 │ │ msg4 │   │
  │ ┌──────┐             │ │ ┌──────┐             │ │ └──────┘ └──────┘   │
  │ │ DLQ  │             │ │ │ DLQ  │             │ │ ┌──────┐             │
  │ └──────┘             │ │ └──────┘             │ │ │ DLQ  │             │
  └──────────┬───────────┘ └──────────┬───────────┘ │ └──────┘             │
             ▼                        ▼             └──────────┬───────────┘
        ┌─────────┐             ┌─────────┐                   ▼
        │ Lambda  │             │ Lambda  │             ┌─────────┐
        │ (crew)  │             │ (customs│             │ Lambda  │
        └─────────┘             └─────────┘             │(analytics│
                                                        └─────────┘


              COMPOSING ALL THREE (EventBridge + SNS + SQS)
              ─────────────────────────────────────────────

  ┌──────────┐    ┌────────────────┐    ┌──────────────┐    ┌─────────┐
  │ Producer │───▶│ EventBridge    │───▶│ SNS Topic    │───▶│ SQS     │
  │          │    │ (routing)      │    │ (fan-out)    │    │ (buffer)│
  └──────────┘    │ content-based  │    │ 1-to-many    │    │ pull    │
                  │ $1.00/M        │    │ $0.50/M      │    │ $0.40/M │
                  └────────────────┘    └──────────────┘    └────┬────┘
                                                                 │
                                                                 ▼
                                                           ┌─────────┐
                                                           │ Lambda  │
                                                           │ (worker)│
                                                           └─────────┘
```

---

## Part 1: SQS Standard Queues -- The Standard Carousel

### The Analogy

**A standard baggage carousel at a busy international airport.** Bags arrive from the airplane hold in roughly the order they were loaded, but the conveyor belt physics -- multiple loading chutes, variable belt segments, parallel conveyor forks -- can shuffle a few bags. Most bags arrive in order, but you cannot guarantee it. The carousel has massive throughput: it can handle bags from dozens of flights simultaneously. Occasionally, a bag might fall off the belt and get re-loaded (duplicate delivery). Every passenger must check the bag tag before walking away -- you might grab a bag that looks like yours but is not (idempotent processing is the consumer's responsibility).

### The Technical Reality

SQS Standard queues are the default, general-purpose queue type:

| Property | Standard Queue |
|----------|---------------|
| **Throughput** | Nearly unlimited API calls per second (SendMessage, ReceiveMessage, DeleteMessage) |
| **Ordering** | Best-effort -- messages are generally delivered in the order sent, but not guaranteed |
| **Delivery** | At-least-once -- messages may be delivered more than once (rare, but possible) |
| **Message retention** | 1 minute to 14 days (default 4 days) |
| **Max message size** | 256 KB (use Extended Client for larger payloads via S3) |
| **In-flight messages** | 120,000 (messages received but not yet deleted) |
| **Pricing** | $0.40 per million requests (after 1M free/month); each 64 KB chunk = 1 request |

### The Message Lifecycle -- Five Steps on the Carousel

Every SQS message follows this lifecycle:

```
1. SEND                2. STORE              3. RECEIVE            4. INVISIBLE           5. DELETE
────────────           ─────────             ─────────             ──────────             ─────────
Producer calls         SQS stores            Consumer calls        Message hidden         Consumer calls
SendMessage ──────▶   across multiple  ──▶  ReceiveMessage  ──▶  for visibility   ──▶  DeleteMessage
                       AZs redundantly       (long poll 20s)       timeout period         (confirms
                                                                                          processing)

                                                                   If NOT deleted before timeout:
                                                                   message reappears on carousel
                                                                   ReceiveCount increments
                                                                   After maxReceiveCount: → DLQ
```

**How the analogy maps**: (1) The airplane unloads bags onto the conveyor. (2) Bags are distributed across multiple belt segments for durability. (3) The passenger walks to the carousel and waits for bags to appear. (4) When a passenger grabs a bag, it becomes invisible to other passengers -- a "claim tag" that expires. (5) The passenger confirms pickup at the claim counter (DeleteMessage); if they do not confirm before the tag expires, the bag reappears on the carousel for the next passenger.

---

## Part 2: Visibility Timeout -- The Claim Tag

### The Analogy

**When you grab a bag off the carousel, the airport puts a temporary claim tag on it.** For the next 30 seconds (default), no other passenger can see that bag. If you process it (check the tag, confirm it is yours) and delete the message within that window, the bag is permanently removed. If the claim tag expires before you finish, the bag reappears on the carousel and someone else picks it up. You can radio the baggage office (ChangeMessageVisibility) to extend your claim tag if you need more time.

### The Technical Reality

| Property | Value |
|----------|-------|
| Default | 30 seconds |
| Range | 0 seconds to 12 hours |
| Extension | ChangeMessageVisibility (can extend up to 12h cumulative) |
| Queue-level setting | VisibilityTimeout attribute on the queue |
| Per-receive override | VisibilityTimeout parameter on ReceiveMessage |

### The Critical Lambda Interaction

This is the most common SQS production bug. When Lambda processes SQS messages via an event source mapping (covered in the Lambda deep dive, Apr 6), the visibility timeout **must** account for the maximum time Lambda might hold the message:

```
CORRECT:   VisibilityTimeout >= 6 * Lambda Function Timeout + MaximumBatchingWindowInSeconds

Example:   Lambda timeout = 30s, MaximumBatchingWindowInSeconds = 5s
           Visibility timeout >= 6 * 30 + 5 = 185 seconds

WHY 6x?    Lambda may retry the batch internally up to 6 times before
           reporting failure back to SQS. If visibility timeout < total
           possible processing time, SQS re-delivers the message to a
           SECOND Lambda invocation while the first is still running.
```

**The failure scenario**:
```
Timeline (visibility timeout = 30s, Lambda timeout = 30s -- WRONG)
─────────────────────────────────────────────────────────────────────

0s        Lambda A receives message, starts processing
28s       Lambda A still processing (complex payload)
30s       ⚠️ Visibility timeout expires! Message reappears on carousel
30s       Lambda B receives the SAME message, starts processing
32s       Lambda A finishes, calls DeleteMessage -- succeeds
35s       Lambda B also finishes, processes same message = DUPLICATE

Result: Same order processed twice, double charge, double shipment
```

---

## Part 3: Long Polling vs Short Polling -- Patient Waiting vs Glance-and-Leave

### The Analogy

**Short polling** is like glancing at the carousel for one second and leaving if no bags are visible. You paid for a cab to the airport (API request cost), walked in, looked, saw nothing, and left. The carousel might have bags arriving from the other side that you did not see (SQS samples a subset of servers). **Long polling** is like standing at the carousel for up to 20 seconds, watching the entire belt go around. You are much more likely to see bags, and if there truly are none, you only paid for one cab ride instead of twenty quick glances.

### The Technical Reality

| Property | Short Polling | Long Polling |
|----------|---------------|-------------|
| Wait time | 0 (immediate return) | 1-20 seconds (ReceiveMessageWaitTimeSeconds) |
| Server sampling | Subset of SQS servers | ALL SQS servers |
| Empty responses | Frequent (you pay for each) | Rare (~97% reduction) |
| Cost impact | Higher (many empty ReceiveMessage calls) | Lower (fewer API calls) |
| Latency | Immediate response | Up to WaitTimeSeconds if queue empty |
| Configuration | Queue-level default OR per-request override via WaitTimeSeconds parameter |

**Best practice**: Always set `ReceiveMessageWaitTimeSeconds = 20` on every queue. Override to 0 only when you specifically need immediate response behavior.

**Critical gotcha**: Even with long polling at 20 seconds, ReceiveMessage can return zero messages when the queue is non-empty. SQS is a distributed system -- the long poll samples all servers, but a message may not have propagated to any sampled server yet. Consumers must loop and keep polling; never treat a single empty response as "queue is empty."

---

## Part 4: Dead Letter Queues -- The Lost-and-Found Office

### The Analogy

**A bag that circles the carousel too many times without being claimed gets sent to the lost-and-found office.** After a configurable number of failed pickups (maxReceiveCount), the airport redirects the bag to a separate holding room. A staff member investigates: was the bag tag wrong? Did the passenger miss their connection? Is the bag damaged? The lost-and-found office is just another room with its own carousel (a regular SQS queue) -- it is not a special room type. Eventually, after investigation, the staff can reroute the bag back to the original carousel (DLQ redrive via StartMessageMoveTask).

### The Technical Reality

| Property | Value |
|----------|-------|
| Configuration | RedrivePolicy on the source queue: `{"deadLetterTargetArn": "...", "maxReceiveCount": N}` |
| maxReceiveCount | Typically 3-5 for production; each ReceiveMessage increments the count whether or not processing failed |
| DLQ type constraint | **Must match source queue type** -- Standard -> Standard DLQ, FIFO -> FIFO DLQ |
| Redrive allow policy | On the DLQ: controls which source queues can target this DLQ (`byQueue`, `allowAll`, `denyAll`) |
| Redrive to source | `StartMessageMoveTask` API: moves messages from DLQ back to source queue for reprocessing |
| Monitoring | CloudWatch alarm on `ApproximateNumberOfMessagesVisible > 0` -- any DLQ message means something broke |
| Message attributes | Original message attributes preserved; SQS adds `ApproximateFirstReceiveTimestamp` and `ApproximateReceiveCount` |

**Key nuance**: maxReceiveCount counts per ReceiveMessage call, not per failed processing. If a consumer receives a message, processes it successfully, but forgets to call DeleteMessage, the message reappears after the visibility timeout and the receive count increments. This is why consumers **must** delete every successfully processed message -- a missing delete is indistinguishable from a processing failure from SQS's perspective.

### DLQ Redrive (StartMessageMoveTask)

Launched in 2023, this API moves messages from a DLQ back to their source queue (or any other queue) for reprocessing:

```
aws sqs start-message-move-task \
  --source-arn arn:aws:sqs:us-east-1:123456789012:orders-dlq \
  --destination-arn arn:aws:sqs:us-east-1:123456789012:orders-queue \
  --max-number-of-messages-per-second 50
```

The `max-number-of-messages-per-second` parameter throttles the redrive rate to avoid overwhelming the consumer with a sudden burst of reprocessed messages.

---

## Part 5: SQS FIFO Queues -- The Ordered Carousel

### The Analogy

**A FIFO carousel organizes bags by flight number (MessageGroupId).** All bags from Flight 747 arrive in exact loading order. All bags from Flight 123 also arrive in exact order. But Flight 747 bags and Flight 123 bags can be processed in parallel because they are on separate conveyor tracks within the same carousel. A **duplicate bag tag scanner** (MessageDeduplicationId) at the carousel entrance checks every bag against the last 5 minutes of scanned tags -- if the same bag was already loaded, it is silently rejected. The trade-off: the FIFO carousel moves slower (300 bags/second per loading station, 3,000 with batch loading) because it must maintain strict sequence.

### The Technical Reality

| Property | FIFO Queue |
|----------|-----------|
| **Throughput (default)** | 300 messages/s per API action, 3,000 with batching (10 messages per batch) |
| **Throughput (high-throughput mode)** | 70,000+ transactions/s (partitions dedup scope to message group level) |
| **Ordering** | Strict FIFO within each MessageGroupId |
| **Deduplication** | Exactly-once within 5-minute window (content-based SHA-256 or explicit MessageDeduplicationId) |
| **Queue name** | **Must end in `.fifo`** (e.g., `orders-queue.fifo`) |
| **In-flight messages** | 20,000 (vs 120,000 for Standard) |
| **Pricing** | $0.50 per million requests (vs $0.40 Standard) |

### MessageGroupId -- The Partition Key

MessageGroupId is the most powerful and most misunderstood FIFO concept:

```
SCENARIO: E-commerce order processing

MessageGroupId = "order-123"    MessageGroupId = "order-456"
 ┌──────────────────────┐        ┌──────────────────────┐
 │ msg1: order-created  │        │ msg1: order-created  │
 │ msg2: payment-ok     │        │ msg2: payment-ok     │
 │ msg3: shipped        │        │ msg3: shipped        │
 └──────────────────────┘        └──────────────────────┘
         │                                │
         ▼                                ▼
   Processed strictly            Processed strictly
   in order: 1→2→3              in order: 1→2→3
   
   BUT order-123 and order-456 are processed IN PARALLEL
   by different consumers (different conveyor tracks)
```

**Design guidance**: Use the entity ID as the MessageGroupId (customer ID, order ID, tenant ID). This gives you strict ordering where it matters (within each entity) and parallelism where it helps (across entities). If you use a single MessageGroupId for all messages, you get strict global ordering but throughput is limited to one consumer at a time per group.

### Deduplication -- Content-Based vs Explicit

Two dedup modes, both using a 5-minute dedup interval:

| Mode | How It Works | When to Use |
|------|-------------|-------------|
| **Content-based** | SQS computes SHA-256 hash of message body; duplicate body = duplicate message | Message body is deterministic and uniquely identifies the operation |
| **Explicit** | Producer provides `MessageDeduplicationId`; same ID within 5 minutes = duplicate | Message body may be identical for different logical operations (e.g., two "retry" messages for different contexts) |

**Important precision**: "Exactly-once delivery" means SQS will not enqueue a duplicate within the dedup window. It does NOT mean the consumer sees the message only once. If a consumer receives a message, fails to delete it, and the visibility timeout expires, the message is re-delivered. Consumer-side idempotency remains necessary even with FIFO.

> **Key takeaway**: Do not let "exactly-once" lull you into skipping idempotency. FIFO dedup prevents duplicate *enqueue* (producer side). It does NOT prevent duplicate *delivery* (consumer side). Every FIFO consumer must still be idempotent — use DynamoDB conditional writes, the Lambda Powertools idempotency decorator, or database unique constraints (see [Lambda Deep Dive](./2026-04-06-lambda-deep-dive.md) Part 12: Idempotency).

**FIFO message group blocking behavior**: When a message in a FIFO message group is in-flight (being processed by a consumer), subsequent messages in the **same message group** are blocked until the in-flight message is either deleted (successful processing) or the visibility timeout expires (failed processing). Messages in **other message groups** are unaffected and continue processing in parallel. This is why choosing the right MessageGroupId granularity is critical — a single MessageGroupId for all messages creates a strict global bottleneck, while per-entity IDs (order ID, customer ID) allow independent parallel processing.

### High-Throughput Mode

Enabled via two queue attributes:
- `DeduplicationScope = messageGroup` (instead of `queue`)
- `FifoThroughputLimit = perMessageGroupId` (instead of `perQueue`)

This partitions dedup to the message group level, enabling 70,000+ transactions/s across many message groups. **Trade-off**: deduplication is only enforced within each message group, not across the entire queue. Use when you have many independent groups and do not need cross-group dedup.

---

## Part 6: Additional SQS Features

### Delay Queues -- The Staging Area

**The analogy**: A holding pen before the carousel. Bags sit in a staging area for a configured period before appearing on the belt.

| Property | Value |
|----------|-------|
| Queue-level | `DelaySeconds` attribute (0-900 seconds / 0-15 minutes) |
| Per-message | `DelaySeconds` on SendMessage -- overrides queue default for **Standard only** (FIFO does not support per-message delay) |
| Use cases | Retry backoff, scheduled processing, rate smoothing |

### Batching -- Efficiency at Scale

SQS supports batching on all three core operations:

| Operation | Batch Size | Cost Impact |
|-----------|-----------|-------------|
| `SendMessageBatch` | Up to 10 messages per call | Each 64 KB chunk per message counts as 1 request; batching reduces API calls |
| `ReceiveMessage` | `MaxNumberOfMessages` 1-10 | Single API call retrieves up to 10 messages |
| `DeleteMessageBatch` | Up to 10 messages per call | One API call instead of 10 individual deletes |

At high volume, batching reduces per-message cost by up to 10x.

### SQS Extended Client Library -- Oversized Luggage

**The analogy**: The oversized luggage counter. When a bag exceeds the carousel's 256 KB weight limit, it goes to the oversized luggage area (S3). A claim ticket (S3 pointer) is placed on the carousel instead. The passenger picks up the claim ticket and walks to the oversized area to collect their bag.

- Java library: `amazon-sqs-extended-client-lib` (supports messages up to 2 GB via S3)
- Python library: `amazon-sqs-extended-client` (same capability)
- **Critical**: The consumer must also use the Extended Client library to read S3-backed messages. A consumer using the standard SDK will receive the raw S3 pointer JSON, not the actual payload.

### Encryption

| Type | Details |
|------|---------|
| **SSE-SQS** | AWS-managed keys; free; default since 2023; no configuration needed |
| **SSE-KMS** | Customer-managed key (CMK); requires key policy granting `kms:GenerateDataKey` and `kms:Decrypt` to `sqs.amazonaws.com`; all producers AND consumers need KMS permissions |

### Temporary Queues -- Request-Response Pattern

The `AmazonSQSTemporaryQueuesClient` (Java) creates virtual queues for the request-response pattern. Instead of creating and deleting real SQS queues per request (expensive and slow), it multiplexes virtual queues onto a small number of real host queues using message attributes for routing.

### Access Control

- **Resource-based policies**: SQS queue policies for cross-account access and service-to-service access (SNS -> SQS, EventBridge -> SQS)
- **Condition keys**: `aws:SourceArn` (restrict to specific SNS topic or EventBridge rule), `aws:SourceAccount`
- **VPC endpoints**: Interface endpoints for private SQS access without NAT Gateway

### Purge Queue

`PurgeQueue` API deletes all messages in a queue. Can only be called once per 60 seconds. Use for development/testing cleanup, not production workflows.

### Pricing Summary

| Item | Price |
|------|-------|
| Standard requests | $0.40 per million (after 1M free/month) |
| FIFO requests | $0.50 per million (after 1M free/month) |
| Request sizing | Each 64 KB chunk = 1 request (a 256 KB message = 4 requests) |
| Data transfer | Standard AWS data transfer rates for cross-region |

---

## Part 7: SNS -- The PA System

### SNS Standard Topics

**The analogy**: The airport PA system. When Flight 747 lands, the PA broadcasts to every listener simultaneously. Ground crew, customs, VIP lounge, taxi rank -- everyone hears the announcement at the same time. The PA system does not store announcements; it is fire-and-forget push delivery.

| Property | Standard Topic |
|----------|---------------|
| **Throughput** | Nearly unlimited publishes per second |
| **Ordering** | Best-effort (no guarantees) |
| **Delivery** | At-least-once (duplicates possible) |
| **Max message size** | 256 KB (Extended Client for up to 2 GB via S3) |
| **Subscribers per topic** | 12,500,000 (soft limit) |
| **Topics per account** | 100,000 (soft limit) |

### Subscription Protocols

SNS Standard topics deliver to a wide range of protocols:

| Protocol | Notes |
|----------|-------|
| **SQS** | Most common; enables fan-out pattern; free delivery |
| **Lambda** | Direct invocation; free delivery (same as SQS) |
| **HTTP/S** | Webhook endpoints; $0.60/M deliveries; delivery retry policy (default ~50 attempts over ~6 hours; customizable) |
| **Email / Email-JSON** | Human notifications; $2.00/100K deliveries |
| **SMS** | Text messages; pricing varies by country |
| **Kinesis Data Firehose** | Stream to S3/Redshift/OpenSearch |
| **Mobile push** | APNs (iOS), FCM (Android), ADM (Amazon) |

### SNS FIFO Topics -- Ordered PA Announcements

**The analogy**: A PA system with guaranteed announcement ordering per channel. Flight 747's announcements (boarding, last call, gate closed) are broadcast in exact order. But the critical limitation: FIFO PA can only send to FIFO carousels (SQS FIFO queues) and Standard carousels (SQS Standard queues). It cannot send to Lambda, HTTP endpoints, email, SMS, or mobile push because those protocols cannot preserve ordering semantics.

| Property | FIFO Topic |
|----------|-----------|
| **Throughput** | 3,000 publishes/s per topic (300/s per message group), or 20 MB/s; high-throughput mode scales to 30K+ TPS |
| **Ordering** | Strict per-MessageGroupId |
| **Deduplication** | Exactly-once within 5-min window |
| **Subscribers** | **SQS FIFO queues and SQS Standard queues ONLY** -- NOT Lambda, HTTP/S, Email, SMS, Firehose, mobile push |
| **MessageGroupId** | Propagates from topic to SQS FIFO subscribers, preserving per-group ordering end-to-end |
| **Topic name** | Must end in `.fifo` |

**When to use SNS FIFO vs direct SQS FIFO**: Use SNS FIFO when you need ordered fan-out (multiple SQS subscribers each receiving the same ordered stream). If you only have one consumer, skip the topic and use SQS FIFO directly.

### Subscription Filter Policies -- The Frequency Tuner

**The analogy**: Each PA listener tunes to a specific frequency. Ground crew listens for maintenance announcements only. Customs listens for international arrivals only. VIP lounge filters for first-class passengers only. Messages that do not match a subscription's filter are never delivered to that subscriber -- reducing cost and downstream processing.

**Two filtering scopes** (determined by `FilterPolicyScope` attribute):

| Scope | Filters On | When to Use |
|-------|-----------|-------------|
| **MessageAttributes** (default) | Structured attributes attached to Publish call | Legacy; requires publishers to duplicate routing info into attributes |
| **MessageBody** (opt-in) | JSON body content directly | Modern default; eliminates "must remember to populate attributes" footgun; same pattern syntax as EventBridge |

**Filter operators**:

```json
{
  "order_type": ["international"],                    // Exact match
  "customer_tier": [{"prefix": "premium"}],           // Prefix match
  "source": [{"anything-but": ["test", "staging"]}],  // Exclude values
  "amount": [{"numeric": [">=", 100, "<", 10000]}],   // Numeric range
  "priority": [{"exists": true}],                      // Field must exist
  "source_ip": [{"cidr": "10.0.0.0/8"}],              // IP range match
  "region": [{"suffix": "-east"}],                     // Suffix match (Nov 2023+)
  "status": [{"wildcard": "order-*-confirmed"}],       // Wildcard match (Nov 2023+)
  "category": [{"equals-ignore-case": "ELECTRONICS"}]  // Case-insensitive match
}
```

**Constraints**: Up to 5 attribute names per policy, 200 filter policies per topic (soft, 10K per account), total cross-product of values across all attributes must not exceed 150 (e.g., 3 values x 5 values x 10 values = 150), 5 levels of nested JSON depth for body filtering.

**Cost implications**: Every message filtered out is one less SQS SendMessage, one less Lambda invocation, and one less downstream processing cost. At high scale, tight filter policies can be the single largest cost lever in a fan-out architecture.

**Propagation gotcha**: Filter policy changes take up to 15 minutes to fully propagate. Do not expect instant delivery behavior changes in tests.

### SNS Subscription DLQs -- Per-Listener Lost-and-Found

**The analogy**: Each PA listener has its own personal lost-and-found box. If the customs desk is closed when the PA broadcasts an international arrival, the undelivered announcement goes to customs' dedicated box -- not a shared topic-level box. The ground crew's lost-and-found is completely separate.

**Key rules** (this is the most commonly misunderstood SNS feature):

1. DLQ is configured at the **subscription** level via a `RedrivePolicy` JSON on the subscription, NOT at the topic level
2. Each subscription has its own DLQ -- a topic with 5 subscribers can have 5 different DLQs
3. The DLQ **must be an SQS queue** (not another SNS topic, not EventBridge)
4. The SQS DLQ must have a resource-based policy allowing `sns.amazonaws.com` to `sqs:SendMessage`
5. Messages **filtered out** by subscription filter policies are NEVER sent to the DLQ -- filtering is not a delivery failure; it is intentional routing

**Contrast with SQS DLQ and EventBridge target DLQ**: Three distinct failure modes:
- **SNS subscription DLQ**: Delivery failure -- the subscriber never received the message (SNS could not push to the endpoint)
- **SQS main-queue DLQ**: Processing failure -- the consumer received the message but failed to process it (maxReceiveCount exceeded)
- **EventBridge target DLQ**: Delivery failure -- EventBridge could not invoke the target after retry exhaustion (24h default)

A production fan-out architecture (SNS -> SQS -> Lambda) often needs DLQs at **multiple layers** to distinguish delivery failures from processing failures.

### Additional SNS Features

**Raw Message Delivery**: For SQS, HTTP/S, and Firehose subscribers, bypasses the SNS JSON envelope wrapping. Instead of receiving `{"Type": "Notification", "MessageId": "...", "Message": "your-actual-payload", ...}`, the subscriber receives `your-actual-payload` directly. Recommended for SQS subscribers to avoid double-parsing.

**Message Data Protection**: Built-in PII detection and de-identification. Audit, de-identify, or deny operations on sensitive data within messages without application code.

**Cross-Account Access**: SNS topic resource-based policy grants `sns:Publish` and `sns:Subscribe` to other accounts. Cross-account subscriptions require both the topic policy AND subscriber confirmation.

**SNS Encryption**: SSE with KMS CMK. Publishers need `kms:GenerateDataKey*`, subscribers need `kms:Decrypt` on the key.

### SNS Pricing

| Item | Price |
|------|-------|
| Publishes | $0.50 per million |
| SQS deliveries | Free |
| Lambda deliveries | Free (same as SQS) |
| HTTP/S deliveries | $0.60 per million |
| Email deliveries | $2.00 per 100K |
| SMS deliveries | Varies by country |
| Mobile push | $0.50 per million |

---

## Part 8: Messaging Patterns -- Where It All Comes Together

### Pattern 1: Fan-Out (SNS -> N SQS Queues)

**The canonical pattern.** One event triggers independent processing at multiple consumers, each with its own queue, scaling, and failure handling.

```
                         ┌─────────────────────────────┐
                         │    SNS Topic                │
                         │    "order-events"           │
                         └──────┬──────┬──────┬────────┘
                                │      │      │
              ┌─────────────────┘      │      └──────────────────┐
              ▼                        ▼                         ▼
   ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
   │ SQS: Inventory    │  │ SQS: Billing      │  │ SQS: Notification │
   │ Reserve stock     │  │ Process payment   │  │ Send email/SMS    │
   │ Vis timeout: 60s  │  │ Vis timeout: 300s │  │ Vis timeout: 30s  │
   │ maxRecv: 3        │  │ maxRecv: 5        │  │ maxRecv: 3        │
   └─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘
             ▼                      ▼                       ▼
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │ Lambda (10 conc)│    │ Lambda (5 conc) │    │ Lambda (50 conc)│
   └─────────────────┘    └─────────────────┘    └─────────────────┘
```

Each queue has its own visibility timeout, DLQ, and consumer scaling -- completely independent. The billing system can take 300 seconds per message while notifications fire in 30 seconds.

### Pattern 2: Fan-Out with Filtering

Same as Pattern 1, but each subscription has a filter policy. Only relevant messages reach each queue:

```json
// Inventory subscription: only "OrderPlaced" events
{"event_type": ["OrderPlaced"]}

// Billing subscription: only orders above $100
{"event_type": ["OrderPlaced"], "amount": [{"numeric": [">=", 100]}]}

// Notification subscription: all order events
// (no filter policy = receives everything)
```

Cost savings: If 60% of messages are under $100, the billing queue receives 40% fewer messages, reducing SQS costs, Lambda invocations, and processing time.

### Pattern 3: Buffering / Throttling

**The analogy**: The carousel absorbs a rush of bags from multiple simultaneous landings so that customs can process at a steady 50 bags/minute pace instead of being overwhelmed by 500 bags in 2 minutes.

```
  Bursty Producer                  SQS Queue                    Rate-Limited Consumer
  (500 msgs/sec burst)             (shock absorber)             (50 msgs/sec steady)
  ─────────────────                ─────────────                ────────────────────
        │                               │                              │
        ├── 500 msgs ──▶ ┌──────────────┤                              │
        │                │ msgs buffer  │ ──── 50/sec ──────▶ Lambda (reserved
        │                │ in queue     │      (controlled     concurrency = 50)
        │                │ (14 day max) │       by Lambda
        │                └──────────────┤       concurrency)
        │                               │
```

This pattern is how most production systems handle asymmetric capacity. Connect to Lambda reserved concurrency (Lambda deep dive, Apr 6) to limit the consumer's pull rate. The queue absorbs spikes and the consumer drains at its own pace.

### Pattern 4: Priority Queues

SQS does not have native priority support. Implement with multiple queues:

```
  Producer                    Priority Queues                    Consumer Logic
  ────────                    ───────────────                    ──────────────
      │
      ├── priority=high ──▶  SQS: orders-high    ──┐
      │                                              │
      ├── priority=med  ──▶  SQS: orders-medium  ──┼──▶ Consumer polls high first,
      │                                              │    then medium, then low.
      └── priority=low  ──▶  SQS: orders-low     ──┘    (weighted polling or
                                                          separate consumer groups)
```

**Implementation options**: (a) Separate consumer groups per priority with different scaling policies, (b) single consumer that always drains the high queue before checking medium, (c) weighted polling ratio (poll high 6x for every 3x medium, 1x low).

### Pattern 5: Request-Response (Correlation ID)

**The analogy**: You drop a bag at the airline counter with a return address label. The airline processes it and sends the result to your designated pickup carousel (reply queue). The return address label is the correlation ID.

```
  Client                      Request Queue                  Server
  ──────                      ─────────────                  ──────
    │                               │                           │
    │── SendMessage ──────────────▶│                           │
    │   ReplyToQueueUrl: reply-q    │── ReceiveMessage ───────▶│
    │   CorrelationId: "abc-123"    │                           │
    │                               │                           │
    │                          Reply Queue                      │
    │                          ───────────                      │
    │◀── ReceiveMessage ──────── │◀── SendMessage ─────────────│
    │    filter: CorrelationId    │   CorrelationId: "abc-123"  │
    │    = "abc-123"              │                              │
```

Use `AmazonSQSTemporaryQueuesClient` (Java) for virtual reply queues to avoid creating/deleting real queues per request.

### Pattern 6: Ordered Processing (FIFO per Entity)

```
  Order Service                 SQS FIFO Queue                 Processor
  ─────────────                 ──────────────                 ─────────
    │
    ├── Order 123: created  ──▶ MessageGroupId = "order-123"  ──▶ processed in order
    ├── Order 123: paid     ──▶ MessageGroupId = "order-123"  ──▶ within group
    ├── Order 456: created  ──▶ MessageGroupId = "order-456"  ──▶ processed in order
    ├── Order 123: shipped  ──▶ MessageGroupId = "order-123"  ──▶ within group
    └── Order 456: paid     ──▶ MessageGroupId = "order-456"  ──▶ (parallel across groups)
```

### Pattern 7: Competing Consumers

Multiple consumers poll the same SQS queue for horizontal scaling:

```
  SQS Queue                     Consumers (competing)
  ─────────                     ─────────────────────
  ┌──────────────┐              ┌──────────┐
  │ msg1 msg2    │─────────────▶│ Lambda A │ (gets msg1)
  │ msg3 msg4    │─────────────▶│ Lambda B │ (gets msg2)
  │ msg5 msg6    │─────────────▶│ Lambda C │ (gets msg3)
  └──────────────┘              └──────────┘
```

Each message is delivered to exactly one consumer (Standard: at-least-once with rare duplicates; FIFO: exactly-once within dedup window). This is how SQS + Lambda event source mapping works -- Lambda manages multiple concurrent pollers automatically.

### Pattern 8: Poison Pill Handling

```
  Message Processing Flow
  ───────────────────────

  SQS Queue ──▶ Consumer ──▶ Process OK? ──YES──▶ DeleteMessage ✓
                    │
                    NO (exception/timeout)
                    │
                    ▼
              Message reappears after visibility timeout
              ReceiveCount: 1 → 2 → 3 → ... → maxReceiveCount
                    │
                    ▼ (maxReceiveCount exceeded)
              ┌─────────────────┐
              │ Dead Letter Queue│──▶ CloudWatch Alarm fires
              │ (investigate!)   │    ├── SNS notification to on-call
              │                  │    ├── Inspect message body/attributes
              │                  │    ├── Fix consumer bug
              └────────┬─────────┘    └── StartMessageMoveTask (redrive)
                       │
                       ▼ (after fix)
              Redrive messages back to source queue
```

### Pattern 9: Decoupling (Classic Producer-Consumer)

The simplest and most common pattern. Producer writes to SQS, consumer reads from SQS. Neither knows about the other. Producer can scale independently. Consumer can go offline without losing messages (retained up to 14 days). Different teams can own each side.

---

## Part 9: SQS vs SNS vs EventBridge -- The Decision Framework

You studied EventBridge yesterday. Here is how all three services compare and compose:

### The Three-Column Comparison

| Dimension | SQS | SNS | EventBridge |
|-----------|-----|-----|-------------|
| **Communication model** | Pull-based (consumer polls) | Push-based (topic broadcasts) | Rule-based routing (events matched by content patterns, then pushed to targets) |
| **Pattern** | 1-to-1 (one message, one consumer) | 1-to-many (one publish, many subscribers) | Many-to-many (many sources, many rules, many targets) |
| **Persistence** | Yes -- up to 14 days retention | No -- fire-and-forget push | No -- real-time delivery (archive optional) |
| **Ordering** | FIFO queues (strict per MessageGroupId) | FIFO topics (strict per MessageGroupId) | No ordering guarantees |
| **Filtering** | None (consumer sees all messages) | Subscription filter policies (attribute or body); supports prefix, suffix, numeric, wildcard, anything-but, equals-ignore-case (near-parity with EventBridge since Nov 2023) | Advanced content-based patterns (prefix, suffix, numeric, wildcard, $or) + schema validation |
| **Replay** | No (once deleted, gone) | No | Yes (archive + replay) |
| **Schema** | No | No | Schema Registry + discovery |
| **Throughput** | Nearly unlimited (Standard) | Nearly unlimited (Standard) | ~10,000 PutEvents/sec/region (soft, adjustable) |
| **Latency** | Consumer-dependent (long poll 0-20s) | ~30ms | ~100-500ms typical |
| **Pricing** | $0.40/M requests | $0.50/M publishes | $1.00/M custom events |
| **Max message** | 256 KB (2 GB via Extended Client) | 256 KB (2 GB via Extended Client) | 256 KB |
| **Cross-account** | Queue resource policies | Topic resource policies | Bus resource policies + cross-account targets |
| **SaaS integration** | No | No | Partner event buses (40+ partners) |

### The Decision Tree

```
START: You need to send a message from Service A to Service B

Is this point-to-point (one producer, one consumer)?
├── YES ──▶ SQS (simplest, cheapest, buffered, pull-based)
│
├── NO ──▶ One producer, many consumers (fan-out)?
│          ├── YES ──▶ Do you need content-based routing, replay, or schema discovery?
│          │          ├── YES ──▶ EventBridge ($1.00/M, richer filtering)
│          │          └── NO  ──▶ SNS fan-out to SQS ($0.50/M, simpler, cheaper)
│          │
│          └── NO ──▶ Many producers, many consumers (event mesh)?
│                     └── EventBridge (content-based routing across sources)

Do you need message buffering or rate control?
├── YES ──▶ SQS must be in the path (as the terminal consumer buffer)
│
Do you need strict ordering?
├── YES ──▶ SQS FIFO (single consumer) or SNS FIFO → SQS FIFO (fan-out)
│           EventBridge has NO ordering guarantees
│
Do you need the consumer to control the processing pace?
├── YES ──▶ SQS (pull-based, consumer decides when and how fast to poll)
│           SNS and EventBridge are push-based (they decide when to deliver)
│
DEFAULT: Start with the simplest service that meets your requirements.
         SQS for buffering, SNS for fan-out, EventBridge for routing.
         Combine them as the architecture demands.
```

### Common Compositions (Services Compose, Not Compete)

| Pattern | When to Use | Cost |
|---------|------------|------|
| **SNS -> SQS** (fan-out) | Broadcast to multiple independent consumers with buffered processing | $0.50/M + $0.40/M per queue |
| **EventBridge -> SQS** (buffer) | Content-based routing + buffered consumption | $1.00/M + $0.40/M |
| **EventBridge -> SNS -> SQS** (route + fan-out + buffer) | Full chain: intelligent routing, fan-out to N consumers, each buffered | $1.00/M + $0.50/M + $0.40/M per queue |
| **EventBridge -> SNS** (route + multi-protocol) | EventBridge filters + SNS delivers to SMS, email, HTTP, mobile push (channels EB lacks) | $1.00/M + $0.50/M |
| **API Gateway -> SQS** (direct integration) | Ingest HTTP requests directly into a queue without Lambda (VTL mapping template, Apr 7) | API GW cost + $0.40/M |
| **Step Functions -> SQS** (callback pattern) | Send task token in SQS message, pause workflow until external system calls SendTaskSuccess (Apr 8) | SFN cost + $0.40/M |

**The production composition pattern to internalize**: "EventBridge for routing -> SNS for fan-out -> SQS for buffering -> Lambda for compute" -- each service plays to its strengths.

---

## Part 10: Limits and Quotas Quick Reference

| Limit | SQS Standard | SQS FIFO |
|-------|-------------|----------|
| Throughput | Nearly unlimited | 300/s per API (3K batch, 70K+ high-throughput) |
| In-flight messages | 120,000 | 20,000 |
| Message size | 256 KB | 256 KB |
| Message retention | 1 min - 14 days (default 4 days) | 1 min - 14 days (default 4 days) |
| Visibility timeout | 0s - 12h (default 30s) | 0s - 12h (default 30s) |
| Delay | 0-15 min (queue + per-message) | 0-15 min (queue-level only) |
| Batch size | 10 messages per batch | 10 messages per batch |
| Long poll wait | 0-20 seconds | 0-20 seconds |
| Message attributes | 10 per message | 10 per message |
| Queues per account | No hard limit (request if needed) | No hard limit (request if needed) |

| Limit | SNS Standard | SNS FIFO |
|-------|-------------|----------|
| Publishes | Nearly unlimited | 3,000/s per topic (300/s per group); 30K+ TPS in high-throughput mode |
| Subscribers per topic | 12,500,000 | 100 |
| Topics per account | 100,000 | 1,000 |
| Filter policies per topic | 200 (soft) | 100 |
| Filter attributes per policy | 5 | 5 |
| Message size | 256 KB | 256 KB |
| Subscription protocols | SQS, Lambda, HTTP/S, Email, SMS, Firehose, mobile push | SQS FIFO and SQS Standard **only** |

---

## Production Terraform Example

This Terraform configuration deploys a comprehensive SQS and SNS architecture with fan-out, FIFO, EventBridge integration, filtering, DLQs, encryption, alarms, and a Lambda event source mapping.

```hcl
# ═══════════════════════════════════════════════════════════════════════
# SQS, SNS & MESSAGING PATTERNS -- PRODUCTION ARCHITECTURE
# ═══════════════════════════════════════════════════════════════════════
#
# Deploys:
# 1. SQS Standard queue ("orders") with DLQ (maxReceiveCount=3)
# 2. SQS FIFO queue ("order-processing.fifo") with content-based dedup + KMS
# 3. SNS Standard topic ("order-events") with two SQS subscriptions + filter policies
# 4. SNS FIFO topic ("order-events.fifo") -> SQS FIFO subscription
# 5. EventBridge rule -> SQS queue (buffer pattern from EventBridge deep dive)
# 6. SQS resource policies for SNS and EventBridge access
# 7. Lambda event source mapping with ReportBatchItemFailures
# 8. CloudWatch alarms: DLQ depth, queue age, queue depth
# ═══════════════════════════════════════════════════════════════════════

terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# KMS KEY -- Shared encryption key for FIFO queues and topics
# ═══════════════════════════════════════════════════════════════════════

resource "aws_kms_key" "messaging" {
  description             = "CMK for SQS FIFO and SNS FIFO encryption"
  deletion_window_in_days = 14
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "RootAccountFullAccess"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        # SNS needs GenerateDataKey* to encrypt messages on publish
        # SQS needs Decrypt to read messages + GenerateDataKey to encrypt
        Sid       = "AllowSNSAndSQSUsage"
        Effect    = "Allow"
        Principal = { Service = ["sns.amazonaws.com", "sqs.amazonaws.com"] }
        Action = [
          "kms:GenerateDataKey*",
          "kms:Decrypt"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_kms_alias" "messaging" {
  name          = "alias/messaging-key"
  target_key_id = aws_kms_key.messaging.key_id
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# ═══════════════════════════════════════════════════════════════════════
# 1. SQS STANDARD QUEUE WITH DLQ
#    The standard baggage carousel with a lost-and-found office
# ═══════════════════════════════════════════════════════════════════════

# Dead Letter Queue -- the lost-and-found office
# NOTE: redrive_allow_policy is attached via a separate resource below
# to avoid a Terraform circular dependency (DLQ references main queue ARN
# and main queue references DLQ ARN). See hashicorp/terraform-provider-aws#22577.
resource "aws_sqs_queue" "orders_dlq" {
  name                      = "orders-dlq"
  message_retention_seconds = 1209600 # 14 days -- max retention for investigation
  # SSE-SQS (AWS managed) is the default since 2023; no config needed
  # Use SSE-KMS only when cross-account consumers need key policy access
}

# Main queue -- the standard carousel
resource "aws_sqs_queue" "orders" {
  name = "orders-queue"

  # Visibility timeout: 6x Lambda timeout (30s) + batching window (5s) = 185s
  visibility_timeout_seconds = 185

  # Long polling: always 20 seconds -- reduces empty responses by ~97%
  receive_wait_time_seconds = 20

  # Message retention: 7 days (enough for weekend outages)
  message_retention_seconds = 604800

  # No delay -- messages available immediately
  delay_seconds = 0
}

# Separate resources to break the circular dependency:
# orders -> orders_dlq (redrive) and orders_dlq -> orders (allow)
resource "aws_sqs_queue_redrive_policy" "orders" {
  queue_url = aws_sqs_queue.orders.url
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3
  })
}

resource "aws_sqs_queue_redrive_allow_policy" "orders_dlq" {
  queue_url = aws_sqs_queue.orders_dlq.url
  redrive_allow_policy = jsonencode({
    redrivePermission = "byQueue"
    sourceQueueArns   = [aws_sqs_queue.orders.arn]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# 2. SQS FIFO QUEUE WITH CONTENT-BASED DEDUP + KMS ENCRYPTION
#    The FIFO carousel with duplicate bag tag scanner
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sqs_queue" "order_processing_fifo_dlq" {
  name                        = "order-processing-dlq.fifo" # MUST end in .fifo
  fifo_queue                  = true
  kms_master_key_id           = aws_kms_key.messaging.arn
  kms_data_key_reuse_period_seconds = 300
  message_retention_seconds   = 1209600
}

resource "aws_sqs_queue" "order_processing_fifo" {
  name                        = "order-processing.fifo" # MUST end in .fifo
  fifo_queue                  = true
  content_based_deduplication = true # SHA-256 of body for dedup (no explicit ID needed)

  # KMS encryption with customer-managed key
  kms_master_key_id                 = aws_kms_key.messaging.arn
  kms_data_key_reuse_period_seconds = 300 # Cache data key for 5 min (reduces KMS calls)

  visibility_timeout_seconds = 185
  receive_wait_time_seconds  = 20
  message_retention_seconds  = 604800

  # High-throughput mode: uncomment for 70K+ TPS (weakens cross-group dedup)
  # deduplication_scope   = "messageGroup"
  # fifo_throughput_limit = "perMessageGroupId"
}

# Separate resources to break the circular dependency (same pattern as Standard pair above)
resource "aws_sqs_queue_redrive_policy" "order_processing_fifo" {
  queue_url = aws_sqs_queue.order_processing_fifo.url
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_processing_fifo_dlq.arn
    maxReceiveCount     = 3
  })
}

resource "aws_sqs_queue_redrive_allow_policy" "order_processing_fifo_dlq" {
  queue_url = aws_sqs_queue.order_processing_fifo_dlq.url
  redrive_allow_policy = jsonencode({
    redrivePermission = "byQueue"
    sourceQueueArns   = [aws_sqs_queue.order_processing_fifo.arn]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# 3. SNS STANDARD TOPIC -- FAN-OUT WITH FILTER POLICIES
#    The PA system broadcasting to multiple carousels
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sns_topic" "order_events" {
  name = "order-events"
}

# --- Subscription 1: Inventory queue (only OrderPlaced events) ---

resource "aws_sqs_queue" "inventory_dlq" {
  name                      = "inventory-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sqs_queue" "inventory" {
  name                       = "inventory-queue"
  visibility_timeout_seconds = 185
  receive_wait_time_seconds  = 20
  message_retention_seconds  = 604800

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.inventory_dlq.arn
    maxReceiveCount     = 3
  })
}

# SNS subscription DLQ for delivery failures (separate from SQS processing DLQ)
resource "aws_sqs_queue" "inventory_sns_dlq" {
  name                      = "inventory-sns-delivery-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sns_topic_subscription" "inventory" {
  topic_arn            = aws_sns_topic.order_events.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.inventory.arn
  raw_message_delivery = true # Skip SNS JSON envelope for cleaner processing

  # Payload-based filter: only deliver OrderPlaced events
  filter_policy_scope = "MessageBody"
  filter_policy = jsonencode({
    event_type = ["OrderPlaced"]
  })

  # Per-subscription DLQ for SNS delivery failures
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.inventory_sns_dlq.arn
  })
}

# --- Subscription 2: Billing queue (high-value orders only) ---

resource "aws_sqs_queue" "billing_dlq" {
  name                      = "billing-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sqs_queue" "billing" {
  name                       = "billing-queue"
  visibility_timeout_seconds = 300 # Billing processing takes longer
  receive_wait_time_seconds  = 20
  message_retention_seconds  = 604800

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.billing_dlq.arn
    maxReceiveCount     = 5 # More retries for billing (critical path)
  })
}

resource "aws_sqs_queue" "billing_sns_dlq" {
  name                      = "billing-sns-delivery-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sns_topic_subscription" "billing" {
  topic_arn            = aws_sns_topic.order_events.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.billing.arn
  raw_message_delivery = true

  # Payload-based filter: only high-value orders (>= $100)
  filter_policy_scope = "MessageBody"
  filter_policy = jsonencode({
    event_type = ["OrderPlaced"]
    amount     = [{ "numeric" = [">=", 100] }]
  })

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.billing_sns_dlq.arn
  })
}

# SQS resource policies: allow SNS to send messages to subscriber queues
resource "aws_sqs_queue_policy" "inventory" {
  queue_url = aws_sqs_queue.inventory.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowSNSSend"
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.inventory.arn
      Condition = {
        ArnEquals = { "aws:SourceArn" = aws_sns_topic.order_events.arn }
      }
    }]
  })
}

resource "aws_sqs_queue_policy" "billing" {
  queue_url = aws_sqs_queue.billing.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowSNSSend"
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.billing.arn
      Condition = {
        ArnEquals = { "aws:SourceArn" = aws_sns_topic.order_events.arn }
      }
    }]
  })
}

# Allow SNS to send to subscription DLQs
resource "aws_sqs_queue_policy" "inventory_sns_dlq" {
  queue_url = aws_sqs_queue.inventory_sns_dlq.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowSNSDLQ"
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.inventory_sns_dlq.arn
      Condition = {
        ArnEquals = { "aws:SourceArn" = aws_sns_topic.order_events.arn }
      }
    }]
  })
}

resource "aws_sqs_queue_policy" "billing_sns_dlq" {
  queue_url = aws_sqs_queue.billing_sns_dlq.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowSNSDLQ"
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.billing_sns_dlq.arn
      Condition = {
        ArnEquals = { "aws:SourceArn" = aws_sns_topic.order_events.arn }
      }
    }]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# 4. SNS FIFO TOPIC -> SQS FIFO QUEUE
#    Ordered PA system delivering to ordered carousel
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sns_topic" "order_events_fifo" {
  name                        = "order-events.fifo" # MUST end in .fifo
  fifo_topic                  = true
  content_based_deduplication = true

  # KMS encryption
  kms_master_key_id = aws_kms_key.messaging.arn
}

resource "aws_sns_topic_subscription" "order_processing_fifo" {
  topic_arn            = aws_sns_topic.order_events_fifo.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.order_processing_fifo.arn
  raw_message_delivery = true
  # Note: SNS FIFO only delivers to SQS FIFO (and SQS Standard) -- no Lambda, HTTP, etc.
}

# Allow SNS FIFO to send to SQS FIFO
resource "aws_sqs_queue_policy" "order_processing_fifo" {
  queue_url = aws_sqs_queue.order_processing_fifo.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowSNSFifoSend"
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.order_processing_fifo.arn
      Condition = {
        ArnEquals = { "aws:SourceArn" = aws_sns_topic.order_events_fifo.arn }
      }
    }]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# 5. EVENTBRIDGE RULE -> SQS QUEUE (Buffer Pattern)
#    City newspaper editorial desk routes stories to a baggage carousel
#    Cross-reference: EventBridge deep dive (Apr 9) Part 12
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sqs_queue" "eventbridge_buffer_dlq" {
  name                      = "eventbridge-buffer-dlq"
  message_retention_seconds = 1209600
}

resource "aws_sqs_queue" "eventbridge_buffer" {
  name                       = "eventbridge-buffer-queue"
  visibility_timeout_seconds = 185
  receive_wait_time_seconds  = 20
  message_retention_seconds  = 604800

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.eventbridge_buffer_dlq.arn
    maxReceiveCount     = 3
  })
}

# EventBridge rule: route "OrderPlaced" events from custom bus to SQS buffer
# Uses the orders event bus from yesterday's EventBridge Terraform
resource "aws_cloudwatch_event_rule" "order_to_sqs" {
  name           = "order-placed-to-sqs-buffer"
  description    = "Route OrderPlaced events to SQS for buffered processing"
  event_bus_name = "orders" # Reference existing custom bus from EventBridge deep dive

  event_pattern = jsonencode({
    source      = ["com.mycompany.orders"]
    detail-type = ["Order Placed"]
  })
}

resource "aws_cloudwatch_event_target" "order_to_sqs" {
  rule           = aws_cloudwatch_event_rule.order_to_sqs.name
  event_bus_name = "orders"
  target_id      = "sqs-buffer"
  arn            = aws_sqs_queue.eventbridge_buffer.arn

  dead_letter_config {
    arn = aws_sqs_queue.eventbridge_buffer_dlq.arn
  }
}

# SQS policy allowing EventBridge to send messages
resource "aws_sqs_queue_policy" "eventbridge_buffer" {
  queue_url = aws_sqs_queue.eventbridge_buffer.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowEventBridgeSend"
        Effect    = "Allow"
        Principal = { Service = "events.amazonaws.com" }
        Action    = "sqs:SendMessage"
        Resource  = aws_sqs_queue.eventbridge_buffer.arn
        Condition = {
          ArnEquals = {
            "aws:SourceArn" = aws_cloudwatch_event_rule.order_to_sqs.arn
          }
        }
      }
    ]
  })
}

# Allow EventBridge to send to the DLQ
resource "aws_sqs_queue_policy" "eventbridge_buffer_dlq" {
  queue_url = aws_sqs_queue.eventbridge_buffer_dlq.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowEventBridgeDLQ"
      Effect    = "Allow"
      Principal = { Service = "events.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.eventbridge_buffer_dlq.arn
    }]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# 6. LAMBDA EVENT SOURCE MAPPING (SQS -> Lambda)
#    Passenger (Lambda) pulls bags from the carousel with
#    ReportBatchItemFailures for partial success handling
#    Cross-reference: Lambda deep dive (Apr 6) event source mapping
# ═══════════════════════════════════════════════════════════════════════

resource "aws_lambda_event_source_mapping" "orders_processor" {
  event_source_arn = aws_sqs_queue.orders.arn
  function_name    = var.order_processor_lambda_arn
  enabled          = true

  # Batch settings
  batch_size                         = 10  # Max messages per Lambda invocation
  maximum_batching_window_in_seconds = 5   # Wait up to 5s to fill a batch

  # CRITICAL: ReportBatchItemFailures allows Lambda to report which specific
  # messages in a batch failed, instead of failing the entire batch.
  # Without this, one bad message fails all 10 messages in the batch.
  # Lambda handler must return {"batchItemFailures": [{"itemIdentifier": "msgId"}]}
  function_response_types = ["ReportBatchItemFailures"]

  # Event filter: only process messages matching this pattern
  # (reduces Lambda invocations for irrelevant messages)
  filter_criteria {
    filter {
      pattern = jsonencode({
        body = {
          event_type = ["OrderPlaced"]
        }
      })
    }
  }

  # Scaling: max 5 concurrent Lambda batches pulling from this queue
  scaling_config {
    maximum_concurrency = 5
  }
}

# Lambda permission to be invoked by SQS
# Note: SQS event source mapping uses an IAM role (Lambda execution role
# must have sqs:ReceiveMessage, sqs:DeleteMessage, sqs:GetQueueAttributes),
# NOT a Lambda resource-based permission. This is different from SNS -> Lambda
# (which uses resource-based permission).

# ═══════════════════════════════════════════════════════════════════════
# 7. CLOUDWATCH ALARMS -- Monitor DLQ depth, queue age, queue depth
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sns_topic" "messaging_alarms" {
  name = "messaging-alarms"
}

# Alarm: DLQ depth -- any message in a DLQ means something broke
resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  for_each = {
    orders_dlq           = aws_sqs_queue.orders_dlq.name
    inventory_dlq        = aws_sqs_queue.inventory_dlq.name
    billing_dlq          = aws_sqs_queue.billing_dlq.name
    fifo_dlq             = aws_sqs_queue.order_processing_fifo_dlq.name
    eb_buffer_dlq        = aws_sqs_queue.eventbridge_buffer_dlq.name
    inventory_sns_dlq    = aws_sqs_queue.inventory_sns_dlq.name
    billing_sns_dlq      = aws_sqs_queue.billing_sns_dlq.name
  }

  alarm_name          = "sqs-dlq-depth-${each.key}"
  alarm_description   = "DLQ ${each.value} has messages -- investigate failures"
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

  alarm_actions = [aws_sns_topic.messaging_alarms.arn]
  ok_actions    = [aws_sns_topic.messaging_alarms.arn]
}

# Alarm: Queue depth growing -- consumer is not keeping up
resource "aws_cloudwatch_metric_alarm" "orders_queue_depth" {
  alarm_name          = "sqs-orders-queue-depth"
  alarm_description   = "Orders queue depth exceeding threshold -- consumer backlog growing"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Sum"
  threshold           = 1000 # Tune based on expected throughput
  treat_missing_data  = "notBreaching"

  dimensions = {
    QueueName = aws_sqs_queue.orders.name
  }

  alarm_actions = [aws_sns_topic.messaging_alarms.arn]
}

# Alarm: Oldest message age -- messages sitting too long (consumer stalled?)
resource "aws_cloudwatch_metric_alarm" "orders_message_age" {
  alarm_name          = "sqs-orders-message-age"
  alarm_description   = "Oldest message in orders queue is older than 1 hour -- consumer may be stalled"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ApproximateAgeOfOldestMessage"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Maximum"
  threshold           = 3600 # 1 hour in seconds
  treat_missing_data  = "notBreaching"

  dimensions = {
    QueueName = aws_sqs_queue.orders.name
  }

  alarm_actions = [aws_sns_topic.messaging_alarms.arn]
}

# ═══════════════════════════════════════════════════════════════════════
# VARIABLES
# ═══════════════════════════════════════════════════════════════════════

variable "order_processor_lambda_arn" {
  description = "ARN of the order processor Lambda function"
  type        = string
}

# ═══════════════════════════════════════════════════════════════════════
# OUTPUTS
# ═══════════════════════════════════════════════════════════════════════

output "orders_queue_url" {
  description = "URL of the orders SQS queue"
  value       = aws_sqs_queue.orders.url
}

output "orders_queue_arn" {
  description = "ARN of the orders SQS queue"
  value       = aws_sqs_queue.orders.arn
}

output "order_events_topic_arn" {
  description = "ARN of the order-events SNS topic (fan-out)"
  value       = aws_sns_topic.order_events.arn
}

output "order_events_fifo_topic_arn" {
  description = "ARN of the order-events FIFO SNS topic"
  value       = aws_sns_topic.order_events_fifo.arn
}

output "fifo_queue_url" {
  description = "URL of the FIFO order processing queue"
  value       = aws_sqs_queue.order_processing_fifo.url
}

output "eventbridge_buffer_queue_url" {
  description = "URL of the EventBridge buffer queue"
  value       = aws_sqs_queue.eventbridge_buffer.url
}
```

**Key Terraform patterns**:

1. **Separate DLQs at every layer**: SQS processing DLQs (maxReceiveCount failures) AND SNS subscription DLQs (delivery failures). These capture different failure modes and require separate investigation.

2. **Resource-based policies for cross-service access**: SNS -> SQS uses `sns.amazonaws.com` principal with `aws:SourceArn` condition. EventBridge -> SQS uses `events.amazonaws.com` principal. Lambda -> SQS uses IAM execution role (NOT resource-based policy).

3. **`raw_message_delivery = true`** on SNS -> SQS subscriptions: Avoids double-parsing the SNS JSON envelope. The SQS consumer receives the raw message body directly.

4. **`filter_policy_scope = "MessageBody"`**: Modern payload-based filtering. Eliminates the need for publishers to duplicate routing information into message attributes.

5. **`function_response_types = ["ReportBatchItemFailures"]`**: Enables partial batch failure reporting. Without this, one bad message in a batch of 10 causes all 10 to be retried (and eventually DLQ'd).

6. **KMS key policy grants both `sns.amazonaws.com` and `sqs.amazonaws.com`**: Both services need `kms:GenerateDataKey*` (to encrypt) and `kms:Decrypt` (to read). Missing key policy grants are a silent encryption failure.

7. **`redrive_allow_policy` on DLQs**: Controls which source queues can target this DLQ, preventing accidental cross-queue DLQ sharing. In Terraform, use separate `aws_sqs_queue_redrive_policy` and `aws_sqs_queue_redrive_allow_policy` resources instead of inline attributes to avoid circular dependencies (see [hashicorp/terraform-provider-aws#22577](https://github.com/hashicorp/terraform-provider-aws/issues/22577)).

---

## Critical Gotchas and Interview Traps

**1. "Visibility timeout < Lambda timeout = duplicate processing."**
The single most common SQS + Lambda production bug. If your Lambda function timeout is 30 seconds and visibility timeout is 30 seconds, SQS re-delivers the message to a second Lambda invocation while the first is still running. Set visibility timeout to at least 6x function timeout + MaximumBatchingWindowInSeconds. This was covered in the Lambda deep dive (Apr 6) from the Lambda side; today you see it from the SQS side.

**2. "DLQ type must match source queue type -- Standard to Standard, FIFO to FIFO."**
A FIFO queue with a Standard DLQ will fail to create. A Standard queue with a FIFO DLQ will also fail. The types must match. This is a Terraform plan-time error, so you will catch it early, but it is a frequent exam question.

**3. "Long polling can still return 0 messages when the queue is non-empty."**
SQS is a distributed system. Even with `ReceiveMessageWaitTimeSeconds = 20`, the long poll samples all servers but a message may not have propagated to any sampled server yet. Consumers must loop and keep polling. Never treat a single empty response as "queue is empty."

**4. "SNS subscription DLQ is per-subscription, not per-topic."**
A topic with 5 subscribers can (and should) have 5 separate DLQs. This is different from SQS DLQ (per-queue) and EventBridge target DLQ (per-target). The DLQ captures delivery failures for that specific subscriber, not all subscribers. Filtered-out messages never reach the DLQ because filtering is not a delivery failure.

**5. "FIFO queue name must end in `.fifo`."**
The queue name `orders-queue` creates a Standard queue. The queue name `orders-queue.fifo` creates a FIFO queue. There is no boolean flag that overrides this naming requirement. SNS FIFO topic names must also end in `.fifo`.

**6. "SQS is pull-only -- it cannot push to consumers."**
SQS has no push mechanism. Consumers must poll (ReceiveMessage). Lambda event source mapping hides the polling behind an AWS-managed poller, but it is still pull-based internally. This is the fundamental architectural difference from SNS and EventBridge (both push-based).

**7. "Deleting a message after the visibility timeout expired is a no-op (or worse)."**
If your consumer takes longer than the visibility timeout, the message reappears and may be received by a different consumer. When the first consumer eventually calls DeleteMessage with the original receipt handle, the call succeeds but deletes nothing (the receipt handle is stale). Meanwhile, the second consumer may have already processed and deleted the message -- or may still be processing it, creating a duplicate.

**8. "maxReceiveCount counts per ReceiveMessage, not per failed processing."**
If a consumer receives a message, processes it successfully, but forgets to call DeleteMessage, the message reappears after the visibility timeout. The receive count increments even though processing succeeded. After maxReceiveCount attempts, the message goes to the DLQ despite never having a processing failure. Consumers MUST delete every successfully processed message.

**9. "SNS FIFO topics only deliver to SQS FIFO queues (and SQS Standard queues) -- NOT Lambda, HTTP/S, Email, SMS, etc."**
This is the most commonly missed SNS FIFO constraint. The reason: Lambda scales invocations in parallel, HTTP endpoints process concurrently -- neither can preserve ordering semantics downstream. Only SQS queues can maintain the ordering guarantee.

**10. "SSE-KMS requires key policy grants for ALL producers AND consumers."**
A KMS-encrypted SQS queue requires that every entity sending messages (SNS, EventBridge, Lambda, other services) has `kms:GenerateDataKey*` permission, and every entity receiving messages has `kms:Decrypt` permission. A missing grant for any single producer or consumer causes silent message delivery failures or processing errors. The key policy must explicitly grant the SNS and SQS service principals.

**11. "SQS Extended Client -- consumer must also use Extended Client to read S3-backed messages."**
If a producer uses the Extended Client Library to send a >256 KB message via S3, the SQS message body contains an S3 pointer (JSON with bucket and key). A consumer using the standard SQS SDK will receive the raw pointer JSON, not the actual payload. Both sides must use the Extended Client.

**12. "FIFO high-throughput mode weakens the deduplication guarantee."**
High-throughput mode (`DeduplicationScope = messageGroup`, `FifoThroughputLimit = perMessageGroupId`) scales to 70,000+ TPS but deduplication is only enforced within each message group, not across the entire queue. Two messages in different groups with the same dedup ID can both be enqueued. Use only when you have many independent groups.

**13. "PurgeQueue can only be called once per 60 seconds."**
If you call PurgeQueue and then send new messages and call PurgeQueue again within 60 seconds, the second call fails. This is a development/testing gotcha, not typically a production issue, but it appears in exam questions.

**14. "SNS filter policy changes take up to 15 minutes to propagate."**
Do not expect instant behavior changes after updating a subscription filter policy. During the propagation window, messages may be delivered that should have been filtered, or vice versa. Plan deployments accordingly.

**15. "SQS event source mapping with filter criteria: non-matching SQS messages are permanently deleted."**
When you add a `filter_criteria` to a Lambda SQS event source mapping, messages that do not match the filter are silently deleted from the queue -- they are NOT left on the queue for other consumers. This is by design (the Lambda poller owns the messages it receives), but it means filtered-out messages are lost forever. This is different from SNS filter policies, where filtered-out messages are simply not delivered but the original message on the topic is unaffected.

---

## Key Takeaways

- **SQS is the buffer, SNS is the broadcaster, EventBridge is the router.** SQS absorbs spikes and lets consumers work at their own pace (pull-based). SNS pushes to multiple subscribers simultaneously (pub/sub). EventBridge routes events by content to multiple targets (content-based routing). They compose: EventBridge for routing -> SNS for fan-out -> SQS for buffering -> Lambda for compute.

- **Always set long polling (ReceiveMessageWaitTimeSeconds = 20) on every SQS queue.** Short polling generates empty responses you pay for and samples only a subset of servers. Long polling reduces costs by ~97% and improves message visibility.

- **Visibility timeout must be at least 6x Lambda function timeout + batching window.** This is the #1 SQS + Lambda production bug. If the visibility timeout is too short, SQS re-delivers messages while Lambda is still processing them, causing duplicates.

- **DLQs belong at every failure boundary.** SNS subscription DLQ for delivery failures. SQS main-queue DLQ for processing failures. EventBridge target DLQ for invocation failures. Each captures a different failure mode. Alarm on all of them.

- **MessageGroupId is the partition key for FIFO.** Use entity IDs (order ID, customer ID) as the group ID to get strict ordering where it matters and parallelism where it helps. A single group ID = global ordering but single-consumer throughput.

- **Consumer-side idempotency is always required, even with FIFO exactly-once dedup.** FIFO prevents duplicate enqueue within 5 minutes, but visibility timeout expiry still causes re-delivery. At-least-once delivery is a property of all SQS consumers.

- **SNS filter policies are the biggest cost lever in fan-out architectures.** Every filtered-out message saves an SQS SendMessage, a Lambda invocation, and downstream processing. Use `FilterPolicyScope = MessageBody` for modern workloads to filter on the JSON body directly.

- **SNS FIFO topics can only deliver to SQS queues.** Not Lambda, not HTTP, not email. If you need ordered fan-out to Lambda, route through SNS FIFO -> SQS FIFO -> Lambda event source mapping.

- **SQS + Lambda event source mapping filter criteria permanently deletes non-matching messages.** Unlike SNS filters (which simply skip delivery), SQS event source mapping filters consume and delete the message. Filtered messages are gone forever.

- **The three services together handle 90% of production messaging needs.** Point-to-point buffering (SQS), broadcast fan-out (SNS -> SQS), content-based routing (EventBridge -> SQS), ordered processing (FIFO), priority queues (multiple SQS queues), throttling (SQS + Lambda reserved concurrency), poison pill handling (DLQ -> alarm -> investigate -> redrive).

---

## Study Resources

Curated reading list for this topic: [`study-resources/aws/sqs-sns-messaging-patterns.md`](../../../study-resources/aws/sqs-sns-messaging-patterns.md)

**Key references**:
- [SQS Basic Architecture](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-basic-architecture.html) -- Producer/queue/consumer triangle, distributed storage, at-least-once delivery
- [SQS Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html) -- The critical Lambda interaction rule (6x timeout)
- [SQS Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) -- maxReceiveCount, redrive policy, DLQ type matching
- [SQS FIFO Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html) -- MessageGroupId, deduplication, high-throughput mode
- [SNS Subscription Filter Policies](https://docs.aws.amazon.com/sns/latest/dg/sns-subscription-filter-policies.html) -- Attribute-based and payload-based filtering
- [SNS FIFO Topics](https://docs.aws.amazon.com/sns/latest/dg/sns-fifo-topics.html) -- Ordered fan-out to SQS FIFO only
- [SNS Dead-Letter Queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html) -- Per-subscription DLQ for delivery failures
- [SQS vs SNS vs EventBridge Decision Guide](https://docs.aws.amazon.com/decision-guides/latest/sns-or-sqs-or-eventbridge/sns-or-sqs-or-eventbridge.html) -- The official AWS decision framework
- [Jeremy Daly: SNS+SQS Distribute and Throttle](https://www.jeremydaly.com/how-to-use-sns-and-sqs-to-distribute-and-throttle-events/) -- The canonical fan-out + throttle pattern

**Cross-references in this repo**:
- [Lambda Deep Dive](./2026-04-06-lambda-deep-dive.md) -- Event source mapping with SQS (ReportBatchItemFailures, batch processing, filter criteria), recursive loop detection for Lambda -> SQS/SNS patterns, idempotency as a core design principle
- [API Gateway Deep Dive](./2026-04-07-api-gateway-deep-dive.md) -- Direct AWS service integrations (API Gateway -> SQS SendMessage, API Gateway -> SNS Publish without Lambda via VTL)
- [Step Functions Workflow Orchestration](./2026-04-08-step-functions-workflow-orchestration.md) -- `.waitForTaskToken` callback pattern using SQS/SNS task tokens for human-in-the-loop and long-running external work
- [EventBridge & Event-Driven Architecture](./2026-04-09-eventbridge-event-driven.md) -- Content-based routing, EventBridge -> SQS buffer pattern, EventBridge vs SNS vs SQS decision framework, Pipes (SQS as source)
