# AWS Lambda Deep Dive -- The On-Demand Kitchen Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock), equipped your instances with personal workbenches, shared libraries, and specialty studios (EBS, EFS, FSx), built the managed hotel chain with its revolutionary shared vault system (RDS and Aurora), organized the giant filing cabinet with drawer labels and folder tabs (DynamoDB with DAX and Global Tables), installed the deli counter for lightning-fast repeat orders (ElastiCache with Global Datastore), mastered the hospital emergency preparedness playbook (DR Strategies), composed all those services into a coherent multinational corporation operating across countries (Multi-Region Architecture), deployed the centralized records management department (AWS Backup with Vault Lock), dissected database and storage DR patterns down to the replication byte level, and now -- after building a fortress of managed infrastructure across compute, storage, networking, security, and data layers -- you face a fundamentally different question: **what if you did not want to manage any of it?** What if, instead of provisioning servers, configuring auto-scaling groups, patching operating systems, and right-sizing instances, you just handed AWS a function and said "run this when something happens, and I will pay you for exactly the milliseconds it runs"? That is the domain of AWS Lambda -- serverless compute where the unit of deployment is a function, the unit of scaling is a single invocation, and the unit of billing is a millisecond. But "serverless" does not mean "simple." Under the surface, Lambda runs your code inside Firecracker microVMs that boot in under 125 milliseconds, manages an execution environment lifecycle with Init, Invoke, and Shutdown phases that determine whether your function starts cold or warm, applies concurrency controls that can either protect your downstream systems or throttle your users, and offers deployment strategies with versioned aliases that rival the blue-green patterns you built for ECS. This is the kitchen analogy: think of Lambda as an **on-demand kitchen** where you do not own the restaurant, you do not hire the chefs, and you do not pay rent. You write a recipe (your function code), hand it to a kitchen management company, and every time a customer places an order, they spin up a cooking station, prepare the dish, serve it, and charge you for exactly the time the stove was on. No orders? No charge. A thousand orders at once? A thousand cooking stations. The catch? The first dish at a cold station takes longer because the chef has to unpack equipment and preheat the oven. That is the cold start problem -- and understanding it, along with the full mechanics of how this kitchen operates, is what separates a developer who uses Lambda from an architect who designs with it.

---

**Date**: 2026-04-06
**Topic Area**: aws
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Firecracker microVM | A self-contained, single-use cooking station with its own burners, counter space, and ventilation -- built in seconds, torn down when idle | Lightweight VM technology that isolates each Lambda execution environment with hardware-level security in under 125ms boot time |
| Execution environment lifecycle (Init/Invoke/Shutdown) | Opening a new cooking station (unpack equipment, preheat oven, prep mise en place), cooking orders, then cleaning up at end of shift | Three phases: Init runs once (cold start), Invoke runs per request (warm), Shutdown gives 2s for cleanup; environment is frozen between invocations and reused |
| Cold start | The time it takes a chef to unpack equipment, preheat the oven, and do mise en place before cooking the first dish | One-time initialization latency (100ms-10s) when no warm environment exists; caused by new deployments, scaling events, config changes, or idle timeout |
| Reserved Concurrency | Reserving exactly 50 cooking stations for a specific restaurant -- guaranteed capacity, but those 50 stations are off-limits to other restaurants | Sets both the maximum AND guaranteed minimum concurrent executions for a function; free, but does NOT pre-warm stations (cold starts still happen) |
| Provisioned Concurrency | Pre-opening cooking stations with equipment unpacked, ovens preheated, and mise en place ready -- so the first order cooks instantly | Pre-initializes execution environments that are always warm; eliminates cold starts entirely; costs money even when idle |
| SnapStart | Photographing a fully prepped cooking station and making instant copies from the photo -- faster than setting up from scratch, but you need to refill perishables (refresh connections/credentials) | Snapshots the initialized microVM state at deployment time; restores from snapshot on cold start reducing init by up to 90%; Java/Python/.NET only |
| Lambda Layers | A shared pantry stocked with common ingredients that multiple kitchens can use instead of each bringing their own | Shared ZIP archives of libraries, custom runtimes, or data (up to 5 per function, 250 MB total unzipped) deployed independently from function code |
| Versions and Aliases | Version = a laminated copy of a specific recipe that can never be changed; Alias = a label like "Tonight's Special" that can be moved to point at any laminated recipe | Immutable published snapshots of code+config (versions) and mutable named pointers to versions (aliases) enabling deployment strategies without changing consumer ARNs |
| Weighted alias routing | Sending 10% of orders to the new recipe and 90% to the proven one -- if customers complain, pull the new recipe immediately | Traffic splitting between two versions via alias weights (0.0-1.0) for canary deployments; integrates with CodeDeploy for automated rollback |
| Destinations (async) | After cooking, sending a success photo to the review board or a failure report to the quality team -- richer than just tossing failures in a lost-and-found box (DLQ) | Routes async invocation results (success OR failure) to SQS, SNS, Lambda, or EventBridge with full invocation metadata; supersedes DLQ which only handles failures |
| DLQ (Dead Letter Queue) | A lost-and-found box where failed orders end up -- you know something failed, but not much about why | Legacy failure-only routing for async invocations to SQS or SNS; less metadata than Destinations; still used for SQS event source mapping errors (configured on the SQS queue, not Lambda) |
| Event Source Mapping | A waiter who continuously checks the order queue, bundles orders into batches, and brings them to the kitchen | Lambda-managed poller that reads from SQS, Kinesis, or DynamoDB Streams, batches records, and invokes your function; handles retry/error differently per source type |
| Bisect-on-error | When a batch of 100 orders fails, splitting them into two batches of 50, then 25, then 12... until you isolate the one bad order | Stream error handling that recursively halves failed batches to find the poison record; prevents one bad record from blocking the entire shard |
| VPC integration (Hyperplane) | Running your kitchen inside a private building (VPC) -- since 2019, the kitchen management company pre-installs a shared service entrance (Hyperplane ENI) instead of building a new door for each station | Lambda attaches to VPC subnets via shared Hyperplane ENIs (not per-environment ENIs), eliminating the historic 10s+ cold start penalty for VPC functions |
| Lambda@Edge | A sous-chef stationed at regional distribution warehouses who can modify dishes in transit -- slower and more capable than the newsstand clerk (CloudFront Functions) | Node.js/Python functions at 15 regional edge caches with up to 5s/30s execution, network access, and four CloudFront trigger points; covered in detail in the CloudFront deep dive (Mar 13) |
| Lambda extensions | Kitchen inspectors who observe the cooking process, take temperature readings, and send reports to headquarters without touching the food | Internal (in-process) or external (sidecar) plugins for monitoring, security, and governance that hook into the execution environment lifecycle |
| Recursive loop detection | A circuit breaker that stops orders from bouncing endlessly between the kitchen and the waiter -- Lambda->SQS->Lambda cycles are automatically halted after ~16 invocations | Automatic detection and interruption of recursive invocation loops between Lambda and SQS/SNS/S3 to prevent runaway costs |
| Lambda Function URLs | A walk-up window on the side of the building -- customers order directly without going through the host stand (API Gateway) | Free built-in HTTPS endpoint for simple webhooks/APIs without API Gateway; supports IAM auth or public access; no throttling (use reserved concurrency) |
| Container image deployment | Driving up in your own food truck -- fully equipped with custom equipment, plugs into the management company's electricity | Deploy up to 10 GB images from ECR; full OS control; no Layers support; lazy-loaded image layers; use when dependencies exceed 250 MB ZIP limit |
| Idempotency | Checking the order logbook before cooking: "Order #4523? Already cooked." -- prevents duplicate dishes from at-least-once delivery | Core Lambda design principle; implement via DynamoDB conditional writes, Lambda Powertools idempotency decorator, or S3 existence checks |
| Lambda Powertools | The professional kitchen toolkit -- thermometers (structured logging), timers (tracing), order logbook (idempotency), and ticket parsers (event parsing) | Open-source toolkit (Python/Java/TS/.NET) providing structured logging, X-Ray tracing, idempotency, event parsing, batch processing as simple decorators |
| ARM64 (Graviton2) | The energy-efficient induction cooktop -- cooks the same dishes at 20% lower energy cost | 20% lower duration price; one-line architecture change; most Python/Node.js/Java code works unchanged; combine with Compute Savings Plans for up to 34% savings |

---

## The On-Demand Kitchen Framework

Every Lambda concept maps to a role in an on-demand kitchen operation. You do not own the restaurant. You do not manage the building. You write recipes (functions), and a **kitchen management company** (the Lambda service) handles everything else: finding space, setting up cooking stations, staffing chefs, billing you per dish, and tearing down stations when the dinner rush ends.

The kitchen has five operational dimensions:

1. **The Station (Execution Environment)** -- A self-contained cooking station with its own equipment, prepped ingredients, and counter space. Each station handles one order at a time. The lifecycle of opening, using, and closing a station determines cold vs warm performance.

2. **The Capacity Plan (Concurrency)** -- How many stations can operate simultaneously, whether some are reserved for VIP menus, and whether any are kept pre-heated for instant service.

3. **The Recipe Book (Versions & Aliases)** -- How recipes are published, versioned, and labeled so you can safely introduce new dishes without ruining tonight's service.

4. **The Order System (Invocation Models)** -- How orders arrive (walk-in, phone, or catering queue), how failures are handled, and where problem orders end up.

5. **The Shared Pantry (Layers & Extensions)** -- Common ingredients and kitchen inspectors shared across multiple stations.

```
THE ON-DEMAND KITCHEN ARCHITECTURE
════════════════════════════════════════════════════════════════════════════

  ORDER SOURCES                    KITCHEN MANAGEMENT COMPANY (Lambda Service)
  ─────────────                    ──────────────────────────────────────────
                                   
  ┌──────────┐  Synchronous       ┌──────────────────────────────────────────┐
  │API Gateway│──(walk-in)──────▶│                                          │
  │   / ALB   │                   │  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
  └──────────┘                    │  │Station 1│ │Station 2│ │Station 3│   │
                                  │  │(warm)   │ │(warm)   │ │(cold    │   │
  ┌──────────┐  Asynchronous      │  │         │ │         │ │ start)  │   │
  │S3 / SNS / │──(phone order)──▶│  │ ┌─────┐ │ │ ┌─────┐ │ │ ┌─────┐ │   │
  │EventBridge│                   │  │ │Code │ │ │ │Code │ │ │ │Code │ │   │
  └──────────┘                    │  │ └─────┘ │ │ └─────┘ │ │ └─────┘ │   │
                                  │  │ Layer A │ │ Layer A │ │ Layer A │   │
  ┌──────────┐  Event Source      │  │ Layer B │ │ Layer B │ │ Layer B │   │
  │SQS/Kinesis│──Mapping────────▶│  └─────────┘ └─────────┘ └─────────┘   │
  │DDB Streams│  (catering        │       │            │            │        │
  └──────────┘   queue waiter)   │       ▼            ▼            ▼        │
                                  │  ┌──────────────────────────────────┐   │
                                  │  │  Shared Pantry (Lambda Layers)   │   │
                                  │  │  Kitchen Inspectors (Extensions) │   │
                                  │  └──────────────────────────────────┘   │
                                  └──────────────────────────────────────────┘
                                           │              │
                                    Success│       Failure│
                                           ▼              ▼
                                  ┌──────────┐   ┌──────────┐
                                  │Destination│   │Destination│
                                  │(EventBridge│   │(SQS DLQ) │
                                  │ success)  │   │          │
                                  └──────────┘   └──────────┘
```

---

## Part 1: The Execution Model -- How the Kitchen Operates

### Firecracker microVMs -- The Self-Contained Cooking Station

**The analogy**: Each cooking station is not just a spot on a shared countertop. It is a fully isolated, self-contained unit with its own burners, ventilation, counter space, and fire suppression. Even if the station next to yours catches fire, yours is unaffected. The kitchen management company can spin up a new station in under 125 milliseconds -- faster than a chef can wash their hands.

**The technical reality**: Lambda does not run your code in containers on shared hosts the way ECS does. Each execution environment is a **Firecracker microVM** -- a lightweight virtual machine developed by AWS that provides hardware-level isolation (KVM-based) with a minimal Linux kernel. Firecracker strips everything unnecessary: no BIOS, no PCI bus, no USB, no GPU. This extreme minimalism enables boot times under 125ms and memory overhead under 5 MiB per VM, allowing thousands of microVMs to run on a single physical host without security trade-offs.

Key differences from ECS containers:
- **Isolation**: Firecracker provides VM-level isolation (hardware boundary) vs container-level isolation (kernel namespace boundary)
- **Startup**: Firecracker microVM boots in <125ms; the Lambda cold start you experience is dominated by your code initialization, not the VM boot
- **Density**: Thousands of microVMs per host vs hundreds of containers per host
- **Lifecycle**: One microVM per concurrent invocation vs one container serving multiple requests

### Execution Environment Lifecycle -- Opening, Operating, and Closing the Station

**The analogy**: When a new cooking station opens, the chef goes through three phases: (1) **Setup** -- unpack equipment, preheat the oven, prepare the mise en place (chop vegetables, measure spices, heat oil); (2) **Cooking** -- prepare dishes as orders come in; between orders, everything stays warm and ready; (3) **Closing** -- at the end of a quiet period, clean up and shut down. The critical insight: setup happens once, then the station stays ready for multiple orders. If orders keep coming, the chef never has to re-setup.

**The technical reality**: Every Lambda execution environment goes through three lifecycle phases:

```
EXECUTION ENVIRONMENT LIFECYCLE
════════════════════════════════════════════════════════════════════════════

  ┌──── INIT PHASE (Cold Start) ───────────────────────────────────────┐
  │                                                                     │
  │  1. Extension Init  ──▶  2. Runtime Init  ──▶  3. Function Init    │
  │  (load extensions)      (start Node/Python/     (YOUR code outside │
  │                          Java runtime)           the handler:       │
  │                                                  SDK clients,       │
  │                                                  DB connections,    │
  │                                                  config loading)    │
  │                                                                     │
  │  ⏱️  Time limit: 10 seconds (on-demand) / 15 minutes (provisioned)  │
  │  ⚠️  If Init exceeds 10s on-demand, it continues but excess time   │
  │     counts against the function's configured timeout               │
  │  💰  Billed as part of first invocation duration                    │
  └─────────────────────────────────────────────────────────────────────┘
          │
          │ Init complete -- environment is now WARM
          ▼
  ┌──── INVOKE PHASE (Warm Path) ──────────────────────────────────────┐
  │                                                                     │
  │  ┌─ Invocation 1 ─┐   ┌─ Invocation 2 ─┐   ┌─ Invocation N ─┐   │
  │  │ handler(event)  │   │ handler(event)  │   │ handler(event)  │   │
  │  │ runs YOUR code  │   │ runs YOUR code  │   │ runs YOUR code  │   │
  │  └────────┬────────┘   └────────┬────────┘   └────────┬────────┘   │
  │           │  FREEZE              │  FREEZE              │           │
  │           └──────────────────────┘──────────────────────┘           │
  │                                                                     │
  │  ⏱️  Time limit: configured timeout (up to 15 minutes per invoke)   │
  │  Between invocations: environment is FROZEN (CPU halted, memory     │
  │  preserved, /tmp persists, connections may go stale)                │
  └─────────────────────────────────────────────────────────────────────┘
          │
          │ No invocations for a while (AWS decides -- typically minutes)
          ▼
  ┌──── SHUTDOWN PHASE ────────────────────────────────────────────────┐
  │                                                                     │
  │  Extensions receive SHUTDOWN event                                  │
  │  ⏱️  Time limit: 2 seconds for cleanup                              │
  │  Environment is destroyed -- /tmp, connections, state all gone      │
  └─────────────────────────────────────────────────────────────────────┘
```

**The Init phase in detail**: This is where cold start latency lives. The three sub-phases run sequentially:

1. **Extension init**: Any Lambda extensions (monitoring agents, security tools) initialize first
2. **Runtime init**: The language runtime starts (Node.js V8 engine, Python interpreter, JVM)
3. **Function init**: YOUR code that runs outside the handler executes -- this is where you should place SDK client initialization, database connection setup, configuration loading, and static computation

**What persists between invocations (environment reuse)**:
- Global/static variables initialized during Init
- SDK clients and HTTP connections (may need reconnection if stale)
- `/tmp` directory contents (up to 10 GB)
- Extension state

**What does NOT persist**:
- Handler function local variables
- Event/context objects
- Any guarantee of which environment handles the next request

**Critical optimization principle**: Initialize expensive resources OUTSIDE the handler. The Init phase runs once per environment creation. The handler runs on every invocation. Moving SDK client creation from inside the handler to module-level scope means it happens once (during Init) instead of on every single request.

```python
# WRONG -- creates a new DynamoDB client on every invocation
def handler(event, context):
    import boto3
    dynamodb = boto3.resource('dynamodb')  # This runs every time
    table = dynamodb.Table('orders')
    return table.get_item(Key={'id': event['order_id']})

# RIGHT -- creates client once during Init, reuses on every invocation
import boto3

dynamodb = boto3.resource('dynamodb')      # Runs once during Init (cold start)
table = dynamodb.Table('orders')           # Runs once during Init

def handler(event, context):
    return table.get_item(Key={'id': event['order_id']})  # Reuses warm client
```

---

## Part 2: Cold Starts -- The Preheat Problem

### What Causes a Cold Start

A cold start happens whenever Lambda must create a new execution environment. Four scenarios trigger this:

| Trigger | Kitchen Analogy | Frequency |
|---------|----------------|-----------|
| First invocation after deployment | Brand new kitchen opening day | Once per deployment |
| Scaling up (traffic increase) | Dinner rush requires more stations | During traffic spikes |
| No warm environments available (idle timeout) | Station shut down after quiet period | After minutes of inactivity |
| Configuration change (memory, env vars, VPC, layers) | Kitchen renovation requires full re-setup | Each config update |

### Cold Start Duration Factors

Cold start duration is NOT a single fixed number. It is the sum of multiple factors:

```
COLD START DURATION BREAKDOWN
════════════════════════════════════════════════════════════════════════════

  Total Cold Start = microVM Boot + Runtime Init + Function Init
                     (~125ms)       (varies)        (YOUR code)

  FACTOR 1: Runtime Choice
  ─────────────────────────────────────────────────────────────────────
  Python / Node.js     ████░░░░░░░░░░░░░░░░     ~100-200ms runtime init
  .NET                 ████████░░░░░░░░░░░░     ~200-400ms runtime init
  Java (no SnapStart)  ████████████████████     ~500ms-3s+ runtime init
                                                 (JVM class loading)

  FACTOR 2: Deployment Package Size
  ─────────────────────────────────────────────────────────────────────
  Small (<5 MB)        ██░░░░░░░░░░░░░░░░░░     Minimal download time
  Medium (5-50 MB)     ██████░░░░░░░░░░░░░░     Noticeable download
  Large (50-250 MB)    ██████████████░░░░░░     Significant download
  Container (up to 10GB) ████████████████████   Longest (but lazy-loaded --
                                                 Lambda only loads referenced
                                                 image layers on demand)

  FACTOR 3: Memory Allocation (= proportional CPU)
  ─────────────────────────────────────────────────────────────────────
  128 MB  (min)        Slowest CPU  ──▶  Init code runs slowly
  1,769 MB             1 full vCPU  ──▶  Sweet spot for most functions
  10,240 MB (max)      6 vCPUs      ──▶  Init code runs fastest

  FACTOR 4: VPC Attachment (Post-Hyperplane, 2019+)
  ─────────────────────────────────────────────────────────────────────
  No VPC               No additional latency
  With VPC (Hyperplane) ~1ms additional (shared ENI, negligible)
  With VPC (pre-2019)  10-30 SECONDS (dedicated ENI creation -- GONE)
```

### Cold Start Mitigation Decision Tree

```
IS THIS A LATENCY-SENSITIVE, SYNCHRONOUS WORKLOAD?
(API Gateway, ALB, real-time user requests)
    │
    ├── NO (async processing, batch jobs, event-driven)
    │   └── Cold starts likely do not matter
    │       └── Optimize for COST: smaller memory, no provisioned concurrency
    │           └── Apply code-level optimizations anyway (free performance)
    │
    └── YES
        │
        ├── CAN YOU TOLERATE sub-second cold starts? (p99 < 1s)
        │   │
        │   ├── YES ──▶ Use SnapStart (Java 11+, Python 3.12+, .NET 8+)
        │   │           Reduces cold start by up to 90%
        │   │           Much cheaper than Provisioned Concurrency
        │   │           ⚠️  Cannot combine with Provisioned Concurrency
        │   │           ⚠️  Cannot use with container images or EFS
        │   │
        │   └── NO ──▶ MUST you guarantee double-digit-ms response?
        │       │
        │       └── YES ──▶ Use Provisioned Concurrency
        │                   Eliminates cold starts entirely
        │                   Costs money even when idle
        │                   Combine with Auto Scaling for variable load
        │
        └── ALWAYS APPLY (regardless of above choice):
            ├── Initialize SDK clients outside the handler
            ├── Minimize deployment package (exclude dev dependencies)
            ├── Use Lambda Layers for shared large libraries
            ├── Set memory to at least 1,769 MB for CPU-bound init
            └── Lazy-load heavy libraries used only in rare code paths
```

---

## Part 3: Concurrency -- Kitchen Capacity Planning

### The Concurrency Formula

```
Concurrent Executions = (Requests per Second) x (Average Duration in Seconds)
```

Example: 100 requests/second with 200ms average duration = 100 x 0.2 = 20 concurrent executions. Simple -- but the nuances around reserved vs provisioned concurrency, burst limits, and throttling behavior are where architects earn their pay.

### Three Tiers of Concurrency Control

```
CONCURRENCY MODEL
════════════════════════════════════════════════════════════════════════════

  Account-Level Limit: 1,000 concurrent executions (default, per region)
  ┌──────────────────────────────────────────────────────────────────────┐
  │                                                                      │
  │  ┌─ Function A ──────────┐  ┌─ Function B ──────────────────────┐   │
  │  │ Reserved: 100         │  │ Reserved: 200                     │   │
  │  │ (guaranteed 100,      │  │ (guaranteed 200, cannot exceed)   │   │
  │  │  cannot exceed 100)   │  │                                   │   │
  │  │                       │  │  ┌─ Provisioned: 50 ──────────┐  │   │
  │  │  All 100 are cold     │  │  │ 50 always warm (no cold    │  │   │
  │  │  start eligible       │  │  │ start). Remaining 150      │  │   │
  │  │                       │  │  │ reserved slots still get    │  │   │
  │  │                       │  │  │ cold starts on scale-up     │  │   │
  │  │                       │  │  └────────────────────────────┘  │   │
  │  └───────────────────────┘  └───────────────────────────────────┘   │
  │                                                                      │
  │  ┌─ All Other Functions ──────────────────────────────────────────┐  │
  │  │ Unreserved Pool: 1,000 - 100 - 200 = 700                      │  │
  │  │ Shared by ALL functions without reserved concurrency           │  │
  │  │ ⚠️  AWS always keeps 100 unreserved for functions without     │  │
  │  │    reservations -- you cannot reserve more than (limit - 100) │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────┘
```

| Concurrency Type | Kitchen Analogy | Cold Starts? | Cost | Use Case |
|-----------------|-----------------|-------------|------|----------|
| **Unreserved** (default) | All kitchens share the building's stations -- first come, first served | Yes | Free (pay per invocation) | Most functions; low-stakes workloads |
| **Reserved** | Stations permanently assigned to one kitchen -- guaranteed but also capped | Yes | Free (no extra charge) | Protect critical functions from noisy neighbors; throttle non-critical functions |
| **Provisioned** | Stations pre-opened, preheated, and fully prepped before any orders arrive | **No** (for provisioned slots) | Provisioned Concurrency pricing (pay even when idle) | Latency-critical APIs, financial trading, real-time user experiences |

### Burst Concurrency and Scaling Rate

Lambda does not scale from 0 to 10,000 instantly. AWS overhauled the scaling model in 2023, replacing the old region-dependent burst tiers with a much faster, per-function scaling rate:

**Current scaling model (post-2023)**:
- **Per-function scaling rate**: 1,000 new execution environments every 10 seconds (equivalent to 6,000 per minute)
- This is a **per-function** rate, not account-level -- each function scales independently
- Account-level concurrency quota still applies (1,000 default, soft limit -- request increase)
- **Requests-per-second (RPS) limit**: 10x your concurrency quota (e.g., 10,000 RPS at default 1,000 concurrency)

**Old model (pre-2023, often still referenced in outdated docs)**:
- Region-dependent burst: 500-3,000 immediate, then +500/minute sustained
- This model is obsolete -- the current model is 12x faster

**Throttling behavior differs by invocation model**:

| Invocation Model | What Happens on Throttle | Retry Behavior |
|-----------------|-------------------------|----------------|
| **Synchronous** (API Gateway, ALB) | Returns 429 TooManyRequestsException to caller | No automatic retry -- client must implement retry logic |
| **Asynchronous** (S3, SNS, EventBridge) | Lambda retries automatically with backoff | Up to 2 retries (configurable 0-2), then sends to DLQ/Destination |
| **Event Source Mapping** (SQS, Kinesis, DDB Streams) | Depends on source type | SQS: message returns to queue; Streams: retries entire batch until success/expiry |

### Provisioned Concurrency with Auto Scaling

For workloads with predictable traffic patterns, you can combine Provisioned Concurrency with Application Auto Scaling to dynamically adjust the number of warm environments:

```
PROVISIONED CONCURRENCY AUTO SCALING
════════════════════════════════════════════════════════════════════════════

  Provisioned     ┌────────────────────────────────────────────┐
  Concurrency     │            ╭──────╮                         │
  Count           │           ╱        ╲        Target: 70%    │
                  │     ╭────╯          ╲       utilization     │
                  │    ╱                 ╲                      │
                  │───╯                   ╲────                │
                  │ Min: 10                     Max: 200       │
                  └────────────────────────────────────────────┘
                  6am    9am    12pm    3pm    6pm    9pm

  Schedule-based scaling:  "Scale to 200 at 8:55 AM before the 9 AM rush"
  Target tracking:         "Keep provisioned utilization at 70%"
  Combine both:            Schedule pre-scales, target tracking handles surprises
```

---

## Part 4: Lambda Layers -- The Shared Pantry

**The analogy**: Instead of every kitchen bringing its own flour, sugar, olive oil, and spices, the management company stocks a **shared pantry** with common ingredients. Each kitchen grabs what it needs from the pantry. When the pantry updates (new olive oil brand), every kitchen using that pantry slot automatically gets the update. But the pantry has limits: each kitchen can pull from at most 5 pantry sections, and the total ingredients (including the kitchen's own) cannot exceed 250 MB of counter space.

**The technical reality**: A Lambda Layer is a ZIP archive containing libraries, a custom runtime, data, or configuration files. Layers are extracted to `/opt` in the execution environment.

| Constraint | Value |
|-----------|-------|
| Maximum layers per function | 5 |
| Maximum unzipped deployment size (function + all layers) | 250 MB |
| Layer versions are immutable | Each update creates a new version |
| Layer sharing | Same account, cross-account via resource policy, or public |

**Common layer use cases**:
- Shared SDK libraries (e.g., `boto3` with a newer version than the Lambda runtime bundles)
- Heavy dependencies shared across multiple functions (e.g., `pandas`, `numpy`, `Pillow`)
- Custom runtimes (e.g., running Rust, PHP, or a specific Node.js version)
- Shared configuration files or certificates

**Layer vs container image decision**: If your total dependencies exceed 250 MB, or you need full control over the OS and system libraries, use a container image (up to 10 GB). If you want fast deployments and your dependencies fit within 250 MB, use layers.

---

## Part 5: Invocation Models and Error Handling -- How Orders Arrive

This is the section that trips up architects in interviews more than any other. Lambda has three invocation models, and each has fundamentally different retry and error handling behavior.

### The Three Invocation Models

```
LAMBDA INVOCATION MODELS
════════════════════════════════════════════════════════════════════════════

  MODEL 1: SYNCHRONOUS (Walk-In Customer)
  ────────────────────────────────────────
  Customer walks in ──▶ Places order ──▶ Waits at counter ──▶ Gets dish (or error)

  Callers: API Gateway, ALB, CloudFront (Lambda@Edge), SDK invoke()
  Retry:   NONE from Lambda -- caller is responsible for retry logic
  Error:   Error returned directly to caller (4xx/5xx)
  Flow:    Caller ──▶ Lambda ──▶ Response to Caller

  MODEL 2: ASYNCHRONOUS (Phone Order)
  ────────────────────────────────────────
  Customer calls ──▶ Order placed in queue ──▶ "We'll call you when it's ready"
  Kitchen cooks when available ──▶ If fails, retries ──▶ Still fails? DLQ/Destination

  Callers: S3 events, SNS, EventBridge, SES, CloudFormation, CodeCommit, IoT
  Retry:   Lambda retries up to 2 times (configurable 0-2) with backoff
  Error:   After retries exhausted, routes to on-failure Destination or DLQ
  Flow:    Caller ──▶ Internal Queue ──▶ Lambda ──▶ Destination (success/failure)
           Caller gets 202 Accepted immediately (does not wait for result)

  MODEL 3: EVENT SOURCE MAPPING (Catering Queue with Dedicated Waiter)
  ────────────────────────────────────────
  Orders accumulate in a queue/stream ──▶ Waiter polls continuously ──▶
  Bundles batch of orders ──▶ Brings to kitchen ──▶ Error handling varies by source

  Sources: SQS, Kinesis, DynamoDB Streams, Amazon MQ, MSK, DocumentDB
  Retry:   Source-dependent (see below)
  Error:   Source-dependent (see below)
  Flow:    Source ◀── Lambda Poller ──▶ Lambda Function
           (Lambda polls the source -- the source does NOT push to Lambda)
```

### Destinations vs Dead Letter Queues -- The Full Picture

**Destinations** (recommended for async invocations):

| Feature | Destinations | DLQ |
|---------|-------------|-----|
| Triggers on | Success AND/OR Failure | Failure only |
| Supported targets | SQS, SNS, Lambda, EventBridge | SQS, SNS only |
| Metadata included | Full invocation record (request payload, response payload, error details, stack trace, retries) | Event payload only |
| Configuration scope | Per function or per alias/version | Per function |
| When to use | New designs -- always prefer Destinations | Legacy compatibility; SQS event source mapping errors (DLQ on the queue, not Lambda) |

**Critical distinction for event source mappings**: When processing SQS messages, the DLQ is configured **on the SQS queue itself** (redrive policy), NOT on the Lambda function. If a message fails processing and exhausts the SQS queue's maxReceiveCount, SQS (not Lambda) sends it to the SQS dead-letter queue. Lambda's Destination/DLQ configuration is irrelevant for SQS event source mappings.

For stream sources (Kinesis, DynamoDB Streams), you configure an **on-failure destination** on the event source mapping itself (SQS, SNS, or S3) to capture failed batch metadata.

### Event Source Mapping Deep Dive

Event source mappings are Lambda resources that continuously poll a source service, batch records, and invoke your function. SQS and SNS are covered in a [separate deep dive](2026-04-10-sqs-sns-messaging-patterns.md); here is the foundational explanation.

**How it works**: Lambda runs a fleet of internal pollers (managed by the Lambda service, invisible to you) that read records from your queue or stream. The pollers batch records according to your configuration, then synchronously invoke your function with the batch. This is NOT a push model -- the source does not call Lambda. Lambda calls the source.

#### Stream Sources (Kinesis, DynamoDB Streams)

| Behavior | Detail |
|----------|--------|
| Processing order | In-order per shard (partition key for Kinesis, partition key for DynamoDB) |
| Batch size | Up to 10,000 records |
| Batch window | 0-300 seconds (accumulate before invoking) |
| Parallelization factor | 1-10 concurrent batches per shard |
| On error (default) | Retries entire batch until success, record expiry, or max age/retry |
| BisectBatchOnFunctionError | Splits failed batch in half recursively to isolate poison record |
| MaximumRetryAttempts | -1 (infinite) to 10,000 |
| MaximumRecordAgeInSeconds | -1 (infinite) to 604,800 (7 days) |
| On-failure destination | SQS, SNS, or S3 -- receives batch metadata (not the records themselves for SQS/SNS; S3 gets the full batch) |
| Checkpoint | `ReportBatchItemFailures` -- return partial success, retry only failed items |

**The BisectBatchOnFunctionError pattern**: This is the production-grade approach for stream processing. When a batch of 100 records fails, Lambda splits it into two batches of 50. If the first 50 succeed but the second 50 fail, it splits again into 25+25. This continues recursively until either a batch succeeds, or a single record is identified as the poison pill. Combined with `MaximumRetryAttempts` and an on-failure destination, this ensures: (1) good records are not blocked, (2) the poison record is identified, and (3) it lands in a DLQ/destination for human inspection.

```
BISECT-ON-ERROR: ISOLATING THE POISON RECORD
════════════════════════════════════════════════════════════════════════════

  Batch [1..100] ──▶ FAIL
       │
       ├──▶ Batch [1..50]  ──▶ SUCCESS (processed, move on)
       └──▶ Batch [51..100] ──▶ FAIL
                │
                ├──▶ Batch [51..75]  ──▶ SUCCESS
                └──▶ Batch [76..100] ──▶ FAIL
                         │
                         ├──▶ Batch [76..88]  ──▶ SUCCESS
                         └──▶ Batch [89..100] ──▶ FAIL
                                  │
                                  └──▶ ... eventually isolates record #93
                                       ──▶ On-failure destination (SQS/S3)
```

#### Queue Sources (SQS)

| Behavior | Detail |
|----------|--------|
| Processing order | Unordered (Standard queues) or ordered per message group (FIFO queues) |
| Batch size | 1-10,000 messages (batch sizes >10 require `MaximumBatchingWindowInSeconds` ≥ 1s) |
| Scaling | Automatic based on queue depth; up to 1,000 concurrent Lambda invocations (or 300 for FIFO) |
| On error | Failed messages return to the queue (visibility timeout expires) |
| Retry mechanism | SQS handles retries via visibility timeout -- not Lambda |
| Dead letter queue | Configured on the **SQS queue** (redrive policy), NOT on Lambda |
| ReportBatchItemFailures | Return partial success -- only failed messages go back to the queue |
| MaximumConcurrency | `scaling_config.maximum_concurrency` limits concurrent Lambda invocations (prevents overwhelming downstream) |

#### Event Filtering

All event source mappings support **filter criteria** that evaluate records before invoking your function. Only records matching the filter pattern reach your function -- non-matching records are discarded (for queues, they are deleted; for streams, the iterator advances past them). Filtering happens at the Lambda service level, so your function is never invoked for irrelevant records, saving cost.

```json
{
  "FilterCriteria": {
    "Filters": [
      {
        "Pattern": "{ \"body\": { \"eventType\": [\"ORDER_PLACED\"], \"amount\": [{\"numeric\": [\">\", 100]}] } }"
      }
    ]
  }
}
```

This filter means: only invoke my function for SQS messages where `body.eventType` is `ORDER_PLACED` AND `body.amount` is greater than 100.

#### Tumbling Windows (Stream Sources)

For aggregation use cases, tumbling windows collect records within a fixed time window (up to 15 minutes) and pass state between invocations. The function receives the accumulated state and the new batch, performs aggregation, and returns the updated state. At the window close, the final state is passed with `isFinalInvokeForWindow: true`.

Use case: "Calculate the average order value per minute from a DynamoDB Streams source."

---

## Part 6: VPC Integration -- The Private Kitchen

**The analogy**: Some kitchens operate inside private buildings (VPCs) because they need access to internal supply rooms (RDS databases, ElastiCache clusters, private APIs) that are not accessible from the public street. Before 2019, every new cooking station required building a brand-new door (ENI) into the private building -- a process that took 10-30 seconds. In 2019, the kitchen management company installed **Hyperplane** -- a shared service entrance that all stations in the building use. Now, the door already exists; a new station just walks through it.

**The technical reality**: When Lambda functions need to access VPC resources (RDS, ElastiCache, internal ALBs, etc.), they must be configured with VPC subnets and security groups. The Lambda service creates **Hyperplane ENIs** (Elastic Network Interfaces) in your subnets during the initial setup (when the function configuration is created or changed), not during each cold start.

| Aspect | Pre-2019 (Legacy) | Post-2019 (Hyperplane) |
|--------|-------------------|----------------------|
| ENI creation | Per execution environment | Per unique subnet+SG combination |
| Cold start penalty | 10-30 seconds | <1 second (negligible) |
| IP consumption | One IP per concurrent execution | Shared -- far fewer IPs consumed |
| Setup timing | During each cold start | During function config create/update |

**VPC Lambda requirements**:
- At least one subnet (recommended: two subnets in different AZs for HA)
- A security group (controls what the function can access)
- IAM execution role needs `ec2:CreateNetworkInterface`, `ec2:DescribeNetworkInterfaces`, `ec2:DeleteNetworkInterface` (use the `AWSLambdaVPCAccessExecutionRole` managed policy)

**Critical gotcha -- internet access**: A Lambda function in a VPC **loses internet access** by default. VPC subnets do not have a public internet gateway path for Lambda. To access the internet (AWS APIs, external services), you must:
1. Place the function in a **private subnet** with a route to a **NAT Gateway** in a public subnet, OR
2. Use **VPC endpoints** for AWS services (S3, DynamoDB, SQS, etc.) -- cheaper and faster than NAT

```
VPC LAMBDA NETWORKING
════════════════════════════════════════════════════════════════════════════

  ┌─── VPC ──────────────────────────────────────────────────────────┐
  │                                                                   │
  │  ┌── Private Subnet (AZ-a) ──────────────────────────────────┐   │
  │  │                                                            │   │
  │  │  Lambda ──▶ Hyperplane ENI ──▶ RDS (private)              │   │
  │  │         ──▶ Hyperplane ENI ──▶ ElastiCache (private)      │   │
  │  │         ──▶ VPC Endpoint   ──▶ S3 (no internet needed)    │   │
  │  │         ──▶ VPC Endpoint   ──▶ DynamoDB (no internet)     │   │
  │  │         ──▶ NAT Gateway    ──▶ Internet (external APIs)   │   │
  │  │                                                            │   │
  │  └────────────────────────────────────────────────────────────┘   │
  │                                                                   │
  │  ┌── Private Subnet (AZ-b) ──────────────────────────────────┐   │
  │  │  (Same setup for HA -- Lambda uses both subnets)          │   │
  │  └────────────────────────────────────────────────────────────┘   │
  │                                                                   │
  │  ┌── Public Subnet ─────────────────────────────────────────┐    │
  │  │  NAT Gateway (for Lambda internet access)                 │    │
  │  │  Internet Gateway                                         │    │
  │  └───────────────────────────────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Versions, Aliases, and Deployment Strategies -- The Recipe Book

### Versions -- Laminated Recipes

**The analogy**: Every time you perfect a recipe, you laminate it and file it with a sequential number. Version 1, version 2, version 3. Once laminated, a recipe can never be changed -- if you want to modify it, you create a new version. There is also a working draft (`$LATEST`) that you scribble on freely, but you never serve that directly to customers.

**The technical reality**: A Lambda **version** is an immutable snapshot of your function code AND configuration (memory, timeout, environment variables, layers, runtime). Each version gets a unique ARN: `arn:aws:lambda:region:account:function:name:version-number`. The `$LATEST` version is the only mutable version -- it always points to the most recent code. Published versions are numbered sequentially (1, 2, 3, ...) and can never be modified. Versions can be deleted, but if an alias references a deleted version, the alias breaks -- Lambda does not prevent this.

### Aliases -- The "Tonight's Special" Label

**The analogy**: The "Tonight's Special" sign at the restaurant can be moved to point at any laminated recipe. Last week it pointed at version 5 (grilled salmon). This week you moved the label to version 6 (pan-seared halibut). The sign (alias) never changes for the customer -- they always order "Tonight's Special." But what they get depends on which recipe the sign currently points to.

**The technical reality**: An alias is a **named pointer** to a specific version, with its own ARN: `arn:aws:lambda:region:account:function:name:alias-name`. API Gateway, EventBridge, and other triggers reference the alias ARN (which never changes), while you shift the underlying version as you deploy.

### Weighted Alias Routing -- Canary Deployments

**The analogy**: Before fully replacing the old recipe, you tell the kitchen: "serve 90% of customers the proven salmon (version 5) and 10% the new halibut (version 6). If anyone gets food poisoning from the halibut, pull it immediately."

**The technical reality**: An alias can split traffic between **exactly two versions** using a weight between 0.0 and 1.0 for the secondary version. This enables canary deployments:

```
WEIGHTED ALIAS DEPLOYMENT STRATEGIES
════════════════════════════════════════════════════════════════════════════

  CANARY (manual):
  ────────────────
  1. Publish version 6 from $LATEST
  2. Update alias "prod" to: primary=v5 (90%), secondary=v6 (10%)
  3. Monitor CloudWatch errors, latency, business metrics
  4. If good: shift alias "prod" to v6 (100%)
  5. If bad: shift alias "prod" back to v5 (100%)

  CODEDEPLOY INTEGRATION (automated):
  ────────────────────────────────────
  Strategy                        Behavior
  ─────────────────────────────   ──────────────────────────────────────
  Canary10Percent5Minutes         10% for 5 min, then 100% (or rollback)
  Canary10Percent10Minutes        10% for 10 min, then 100% (or rollback)
  Canary10Percent15Minutes        10% for 15 min, then 100% (or rollback)
  Canary10Percent30Minutes        10% for 30 min, then 100% (or rollback)
  Linear10PercentEvery1Minute     10% increments every minute
  Linear10PercentEvery2Minutes    10% increments every 2 minutes
  Linear10PercentEvery3Minutes    10% increments every 3 minutes
  Linear10PercentEvery10Minutes   10% increments every 10 minutes
  AllAtOnce                       Immediate 0% → 100% (blue-green)

  CodeDeploy adds:
  - Pre-traffic hook: validation Lambda that runs BEFORE any traffic shifts
  - Post-traffic hook: validation Lambda that runs AFTER shift completes
  - Automatic rollback: triggered by CloudWatch Alarms (error rate, latency)
  - This mirrors the ECS blue-green pattern from Feb 16-17, using aliases
    instead of target groups
```

---

## Part 8: Lambda SnapStart -- Instant Kitchen Setup from a Photograph

**The analogy**: Instead of having every new chef unpack equipment, preheat the oven, and prep mise en place from scratch, you photograph a fully prepped station after the first chef finishes setup. When a new station is needed, you print the photograph in 3D -- instantly materializing a ready-to-cook station. The only catch: perishable items in the photo (milk, fresh herbs) might be expired by the time the copy materializes, so the new chef must check and replace them.

**The technical reality**: SnapStart captures a Firecracker snapshot of the initialized execution environment's memory and disk state when you publish a new version. On cold start, Lambda restores from this cached snapshot instead of running the full Init phase, reducing cold start by up to 90%.

| Aspect | Detail |
|--------|--------|
| Supported runtimes | Java 11+, Python 3.12+, .NET 8+ |
| Pricing | Java: free; Python/.NET: caching + restoration charges |
| Cold start reduction | Up to 90% (Java: ~3s down to ~200-400ms) |
| Works with | Published versions and aliases only (NOT `$LATEST`) |
| Does NOT work with | Provisioned Concurrency, EFS, container images, arm64 (Python/.NET only -- arm64 IS supported for Java since July 2024) |
| Ephemeral storage | Limited to 512 MB (vs 10 GB normally) |

**Runtime hooks for handling "perishable items"**:

| Runtime | Before Snapshot Hook | After Restore Hook |
|---------|--------------------|--------------------|
| Java | `beforeCheckpoint()` (CRaC API) | `afterRestore()` |
| Python | `before_snapshot()` | `after_restore()` |
| .NET | `RegisterBeforeSnapshot()` | `RegisterAfterRestore()` |

**What must be refreshed in the after-restore hook**:
- Database connections (may be stale or closed)
- Random number generators (must re-seed for security -- otherwise multiple environments from the same snapshot share the same random sequence)
- Temporary credentials (may be expired)
- Cached timestamps (will reflect snapshot time, not restore time)

**SnapStart vs Provisioned Concurrency decision**:

| Factor | SnapStart | Provisioned Concurrency |
|--------|-----------|------------------------|
| Cold start | Reduced by ~90% (still exists) | Eliminated entirely |
| Cost | Low (free for Java) | High (pay for idle capacity) |
| Consistency | sub-second p99 | single-digit ms p99 |
| Scaling | Unlimited (each new env restores from snapshot) | Limited to provisioned count; beyond that, cold starts resume |
| Use case | Cost-sensitive APIs tolerating sub-second cold starts | Latency-critical APIs requiring guaranteed fast response |
| Combinable? | Cannot combine with each other | Cannot combine with each other |

---

## Part 9: Lambda@Edge and CloudFront Functions -- Brief Lambda-Side View

The [CloudFront deep dive (Mar 13)](../../march/aws/2026-03-13-cloudfront-deep-dive.md) covered the CDN-perspective comparison in detail. Here is the Lambda-service-specific view:

| Aspect | Lambda@Edge | CloudFront Functions |
|--------|------------|---------------------|
| Managed by | Lambda service (shows in Lambda console) | CloudFront service |
| Runtime | Node.js, Python | JavaScript (ECMAScript 5.1+) |
| Execution location | 15 regional edge caches | 750+ edge locations |
| Memory | 128 MB (viewer) / 10 GB (origin) | 2 MB |
| Timeout | 5s (viewer) / 30s (origin) | 1 ms |
| Network access | Yes | No |
| Body access | Yes (origin triggers) | No |
| Trigger points | Viewer request/response, origin request/response | Viewer request/response only |
| Pricing | Standard Lambda pricing (higher at edge) | ~1/6th the cost of Lambda@Edge |
| Deployment | Publish to us-east-1, replicated to edge regions | CloudFront distribution deploy |
| **Lambda-specific note** | Must be authored in **us-east-1** and associated with a CloudFront distribution; the Lambda function is replicated automatically to regional edge caches; versions are required (cannot use `$LATEST`) | N/A -- not a Lambda feature |

---

## Part 10: Lambda Extensions -- Kitchen Inspectors

**The analogy**: The health department does not cook food, but they observe kitchens. **Internal inspectors** (internal extensions) stand inside the kitchen and watch the cooking process in real-time, taking notes without a separate station. **External inspectors** (external extensions) operate from a separate observation room with their own entrance and exit, running alongside the kitchen but independently.

**The technical reality**:

| Type | Internal Extension | External Extension |
|------|-------------------|-------------------|
| Process | Runs in the same process as the function | Runs as a separate process (sidecar) |
| Lifecycle | Shares the function's Init, Invoke, Shutdown | Has its own Init, Invoke (receives events), Shutdown |
| Use case | In-process instrumentation, APM agents, security scanning | Log forwarding, metrics collection, secret caching |
| Examples | Datadog APM, New Relic agent | AWS Parameters and Secrets Layer, CloudWatch Lambda Insights |
| Impact | Adds to function's execution time | Runs in parallel; uses shared CPU/memory budget |

Extensions participate in the lifecycle:
- **Init**: Extensions initialize before the runtime
- **Invoke**: External extensions receive invoke events and can process telemetry in parallel
- **Shutdown**: Extensions get 2 seconds to flush buffers and send final data

---

## Part 11: Recursive Loop Detection -- The Circuit Breaker

**The analogy**: Imagine this disaster: you cook a dish, send it to the waiter (SQS), the waiter brings it back to the kitchen because it needs garnish (Lambda trigger), you add garnish and send it back to the waiter, the waiter brings it back again... endlessly. Your bill skyrockets, and no customer ever gets served. AWS installs a **circuit breaker** that detects this bouncing pattern and cuts it off.

**The technical reality**: Lambda automatically detects recursive invocation loops between:
- Lambda -> SQS -> Lambda
- Lambda -> SNS -> Lambda
- Lambda -> S3 -> Lambda (via S3 event notifications)

When detected (approximately 16 recursive invocations), Lambda stops the loop and sends the event to a configured DLQ or Destination if available. The function is NOT disabled -- new, non-recursive invocations continue normally.

**How to avoid recursive loops**:
1. Never write output to the same resource that triggers the function (e.g., do not write to the same S3 bucket that triggers the Lambda)
2. Use event filtering to distinguish original events from processed events
3. Use idempotency tokens to detect reprocessed messages
4. Design with distinct input and output resources

---

## Part 12: Lambda Function URLs -- The Walk-Up Window

**The analogy**: Instead of requiring every customer to enter through the restaurant's formal host stand (API Gateway), the kitchen installs a **walk-up window** directly on the side of the building. Customers can walk up, place an order, and get their food without going through the dining room. It is simpler, cheaper, and faster -- but there is no host to manage reservations, check dress codes, or throttle the line.

**The technical reality**: Lambda Function URLs provide a dedicated HTTPS endpoint for your function without requiring API Gateway, ALB, or any other service. Introduced in 2022, they are ideal for simple webhooks, microservices, and single-function APIs.

| Aspect | Function URL | API Gateway (REST/HTTP API) | ALB |
|--------|-------------|---------------------------|-----|
| Cost | Free (pay only for Lambda invocations) | $1-3.50 per million requests | ~$16/month + LCU charges |
| Setup complexity | One-line config | Stages, routes, deployments, integrations | Target groups, listeners, rules |
| Auth | IAM (`AWS_IAM`) or None (`NONE`) | IAM, Cognito, Lambda authorizer, API keys | Cognito, OIDC |
| Throttling | No built-in throttling (use reserved concurrency) | Per-route throttling, usage plans, API keys | No Lambda-specific throttling |
| Custom domain | No native support (use CloudFront) | Custom domain with ACM cert | Via Route53 alias |
| Response streaming | Supported (Node.js, custom runtimes) | Not supported (REST) / partial (HTTP API) | Not supported |
| WebSocket | Not supported | Supported (REST API) | Not supported |
| Use case | Webhooks, simple APIs, internal tools | Production APIs needing auth/throttling/caching | Lambda behind existing ALB |

**When to use Function URLs**:
- Webhook receivers (Stripe, GitHub, Slack) where the caller just needs an HTTPS endpoint. Third-party services cannot use `AWS_IAM` auth (no SigV4), so use `NONE` auth and validate in your function code (e.g., Stripe's HMAC webhook signature). Bogus requests invoke your function but fail validation at sub-ms cost -- the real security gate is the signature check, not the transport auth
- Internal microservices where IAM auth is sufficient
- Single-function APIs where API Gateway's features add complexity without value
- Response streaming use cases (LLM/AI token streaming, large file generation)

**When NOT to use Function URLs**:
- You need request throttling, caching, or usage plans → API Gateway
- You need custom authorizers or Cognito integration → API Gateway
- You need to aggregate multiple Lambda functions behind one domain → API Gateway or ALB
- You need WebSocket support → API Gateway

```hcl
# Function URL -- zero additional cost, zero additional infrastructure
resource "aws_lambda_function_url" "webhook" {
  function_name      = aws_lambda_function.order_processor.function_name
  qualifier          = aws_lambda_alias.prod.name  # Attach to alias, not $LATEST
  authorization_type = "AWS_IAM"                   # Or "NONE" for public webhooks

  cors {
    allow_origins = ["https://example.com"]
    allow_methods = ["POST"]
    max_age       = 86400
  }
}
# Output: https://<url-id>.lambda-url.<region>.on.aws/
```

---

## Part 13: Container Image Deployment -- The Food Truck Kitchen

**The analogy**: Instead of bringing a recipe (ZIP) and using the management company's standard cooking station, you drive up in your own **food truck** (container image) -- fully equipped with your custom oven, specialty pots, and obscure spices pre-installed. The truck plugs into the management company's electricity and plumbing (Firecracker), but everything inside is yours. The trade-off: the truck takes longer to park (larger cold start for initial pull), but you have complete control over the kitchen equipment.

**The technical reality**: Lambda supports deploying functions as container images up to 10 GB, stored in Amazon ECR. The container must implement the Lambda Runtime Interface Client (RIC) to communicate with the Lambda service.

| Aspect | ZIP Deployment | Container Image |
|--------|---------------|----------------|
| Max size | 50 MB compressed / 250 MB unzipped | 10 GB |
| Lambda Layers | Yes (up to 5) | **No** -- bake dependencies into the image |
| Custom OS/system libraries | No (use Amazon Linux 2 runtime) | Yes -- full control |
| Build tooling | `zip` + `aws lambda update-function-code` | `docker build` + `docker push` to ECR |
| Cold start | Download ZIP from S3 | Lazy-load image layers from ECR (only referenced layers) |
| Local testing | SAM CLI, Lambda RIE | Standard `docker run` with Lambda RIE |
| Use case | Most functions (<250 MB dependencies) | ML models, large SDKs, custom runtimes, existing container workflows |

**Key points for interviews**:
- Container images are pulled from **ECR only** (not Docker Hub or other registries)
- Lambda caches the image layers -- subsequent cold starts from the same image are faster
- You can use AWS base images (`public.ecr.aws/lambda/python:3.12`) or any image that implements the Lambda Runtime Interface Client
- SnapStart works with container images for **Java only** (not Python/.NET container images)
- **Layers are NOT supported** with container images -- all dependencies must be in the image

---

## Part 14: Idempotency -- The "Already Cooked" Check

**The analogy**: A waiter brings the same order ticket to the kitchen twice (the first one got lost in the shuffle). Without an idempotency check, the kitchen cooks two identical dishes, charges the customer twice, and wastes ingredients. With an idempotency system, the chef checks the order number against a logbook: "Order #4523? Already cooked and served. Returning the same receipt." The customer gets the correct result without duplicate work.

**The technical reality**: Lambda provides **at-least-once delivery** for asynchronous invocations and event source mappings. This means your function can receive the same event more than once. Designing for idempotency is not optional -- it is a core Lambda design principle.

**Why duplicates happen**:
- Async invocations: Lambda retries on failure (up to 2 times); the same event triggers multiple invocations
- SQS: visibility timeout expires while function is still processing; SQS delivers the message again
- Streams: Lambda retries the entire batch on error; already-processed records in the batch are re-processed
- At-least-once delivery: Even without errors, Lambda's internal queue can rarely deliver an event twice

**Idempotency strategies**:

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| DynamoDB conditional writes | `PutItem` with `attribute_not_exists(PK)` | Simple, atomic, uses existing DynamoDB table | Requires DynamoDB, additional read/write cost |
| Idempotency key in dedicated table | Store `{idempotency_key, result, ttl}` before processing; return cached result on duplicate | Full result caching, configurable TTL | Extra table, extra latency for lookup |
| Lambda Powertools Idempotency | Decorator/middleware handles key extraction, locking, caching, TTL automatically | Production-ready, minimal code | Python/Java/TypeScript only, Powertools dependency |
| S3 object existence check | Check if output object already exists in S3 | Simple for file-processing pipelines | Eventual consistency race window |

**Lambda Powertools idempotency example**:
```python
from aws_lambda_powertools.utilities.idempotency import (
    idempotent,
    DynamoDBPersistenceLayer,
    IdempotencyConfig,
)

persistence_layer = DynamoDBPersistenceLayer(table_name="idempotency-store")
config = IdempotencyConfig(expires_after_seconds=3600)  # 1-hour TTL

@idempotent(config=config, persistence_store=persistence_layer)
def handler(event, context):
    # First call: processes and stores result in DynamoDB
    # Duplicate call: returns cached result without re-processing
    order_id = event["order_id"]
    result = process_order(order_id)
    return {"statusCode": 200, "body": result}
```

---

## Part 15: Lambda Powertools -- The Professional Kitchen Toolkit

**The analogy**: A professional kitchen does not just have recipes and ingredients -- it has standardized tools: thermometers for food safety (structured logging), timers that track each dish through the entire cooking process (distributed tracing), the order logbook for preventing duplicates (idempotency), and standardized ticket parsing (event parsing). Lambda Powertools is the professional toolkit that turns a home kitchen into a commercial-grade operation.

**The technical reality**: AWS Lambda Powertools is an open-source developer toolkit (Python, Java, TypeScript, .NET) that implements production best practices as simple decorators and utilities. The Terraform example in this doc references Powertools via the `POWERTOOLS_SERVICE_NAME` environment variable and the shared layer -- here is what it provides:

| Feature | What It Does | Without Powertools |
|---------|-------------|-------------------|
| **Structured Logging** | JSON-formatted logs with correlation IDs, cold start flags, and request context automatically injected | Manual `json.dumps()` in every log statement |
| **Tracing** | X-Ray tracing with automatic subsegment creation for handlers, service calls, and custom annotations | Manual `aws-xray-sdk` instrumentation |
| **Idempotency** | Decorator that handles key extraction, DynamoDB locking, result caching, and TTL expiry | Build your own locking/caching logic |
| **Event Parsing** | Pydantic-based event validation and parsing for API Gateway, SQS, EventBridge, etc. | Manual `event["body"]["field"]` with error handling |
| **Batch Processing** | Handles `ReportBatchItemFailures` response format automatically for SQS and Kinesis | Build the partial failure response format manually |
| **Parameters** | Cached retrieval from SSM Parameter Store, Secrets Manager, AppConfig with TTL | Manual boto3 calls with your own caching |

---

## Part 16: ARM64 (Graviton2) -- The Energy-Efficient Kitchen

**The analogy**: The kitchen management company offers two types of cooking stations: the traditional gas range (x86_64) and a newer, energy-efficient induction cooktop (ARM64/Graviton2). The induction cooktop cooks just as well for most dishes, uses 20% less energy (costs 20% less), and is often faster for common recipes. The only catch: a few specialty tools designed for gas ranges do not work on induction (native compiled binaries, some C extensions).

**The technical reality**: Lambda supports ARM64 architecture (AWS Graviton2 processors) with a **20% lower price per GB-second** compared to x86_64. This is one of the easiest cost optimization levers for Lambda.

| Aspect | x86_64 | ARM64 (Graviton2) |
|--------|--------|-------------------|
| Price (duration) | $0.0000166667/GB-second | $0.0000133334/GB-second (**20% cheaper**) |
| Performance | Baseline | Comparable or better for most workloads |
| Compatibility | Universal | Most Python/Node.js/Java code works unchanged |
| What needs attention | N/A | Native compiled extensions (C/C++/Rust), some ML libraries |
| SnapStart support | All supported runtimes | Java: yes; Python/.NET: **no** (x86_64 only) |

**Migration is typically one line**:
```hcl
resource "aws_lambda_function" "example" {
  # ... existing config ...
  architectures = ["arm64"]  # Change from default ["x86_64"]
}
```

**When to use ARM64**: Almost always, unless you depend on x86-specific compiled binaries. Test your function on ARM64 first -- most interpreted languages (Python, Node.js, Ruby) and JVM languages work without changes. Combined with Compute Savings Plans (up to 17% additional discount on duration), ARM64 can reduce Lambda costs by up to 34%.

---

## Part 17: Lambda Limits -- The Kitchen Specifications

| Limit | Value | Notes |
|-------|-------|-------|
| Timeout | 15 minutes (900 seconds) | Maximum per invocation |
| Memory | 128 MB - 10,240 MB (10 GB) | 1 MB increments; CPU scales proportionally |
| Ephemeral storage (`/tmp`) | 512 MB - 10,240 MB (10 GB) | Configurable; persists between invocations in same environment |
| Deployment package (ZIP) | 50 MB compressed / 250 MB unzipped | Including all layers |
| Container image | 10 GB | Alternative to ZIP deployment |
| Environment variables | 4 KB total | All key-value pairs combined |
| Concurrent executions | 1,000 per region (default) | Soft limit -- request increase to 10,000+ |
| Scaling rate | 1,000 environments per 10 seconds per function | Post-2023 model (12x faster than old burst model) |
| Layers | 5 per function | Each layer is a versioned ZIP |
| Invocation payload (sync) | 6 MB request / 6 MB response | |
| Invocation payload (async) | 256 KB | |
| `/tmp` directory | Cleared on new environment; persists within same environment | |
| Init phase timeout | 10 seconds (on-demand) / 15 minutes (provisioned/SnapStart) | |
| Shutdown phase | 2 seconds | For extensions cleanup |

---

## Part 18: Pricing Model -- Pay Per Dish

**The analogy**: You pay **rent per cooking station per minute it is open** (duration in GB-seconds), a **flat fee per dish served** (request charges), and a **premium for pre-heated stations sitting idle** (Provisioned Concurrency). Choosing the energy-efficient induction cooktop (ARM64) gives you a 20% discount on rent. Signing a long-term lease (Compute Savings Plans) saves another 17%.

| Component | Price x86_64 (us-east-1) | Price ARM64 | Free Tier |
|-----------|--------------------------|-------------|-----------|
| **Requests** | $0.20 per 1 million | $0.20 per 1 million | 1 million requests/month |
| **Duration** | $0.0000166667 per GB-second | $0.0000133334 per GB-second (**20% cheaper**) | 400,000 GB-seconds/month |
| **Provisioned Concurrency** | $0.0000041667/GB-s (provisioned) + $0.0000097222/GB-s (invocation) | 20% lower on both components | None |
| **Ephemeral storage** | $0.0000000309 per GB-second (above 512 MB) | Same | 512 MB included |

**Duration calculation**: Duration (GB-seconds) = (Memory allocated in GB) x (Execution time in seconds). A function with 1 GB memory running for 200ms = 1 x 0.2 = 0.2 GB-seconds.

**Provisioned Concurrency cost math**: You pay for provisioned capacity whether it is used or not, PLUS a reduced per-invocation rate when those provisioned environments handle requests. Example: 100 provisioned concurrency at 1 GB memory = 100 x 1 x 3600 x $0.0000041667 = ~$1.50/hour just for the provisioned capacity (before any invocations).

**Cost optimization levers**:
1. **ARM64 (Graviton2)**: 20% lower duration price, one-line change for most functions
2. **Compute Savings Plans**: Up to 17% additional discount on duration costs (1-year or 3-year commitment; applies to Lambda, EC2, and Fargate -- covered in [EC2 Purchasing Options, Feb 8](../../february/aws/2026-02-08-ec2-purchasing-options.md))
3. **Combined**: ARM64 + Compute Savings Plans = up to **34% savings** on duration
4. **Right-size memory**: More memory = more CPU = faster execution = potentially lower cost despite higher per-second rate (benchmark with [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning))

**Free tier is perpetual** (not just 12 months): The 1M requests + 400K GB-seconds free tier applies every month indefinitely, not just during the first year. This is different from most AWS free tiers.

---

## Production Terraform Example

This Terraform configuration deploys a production-ready Lambda architecture with versioned aliases, event source mappings, layers, destinations, VPC integration, and concurrency controls.

```hcl
# ═══════════════════════════════════════════════════════════════════════
# LAMBDA DEEP DIVE -- PRODUCTION ARCHITECTURE
# ═══════════════════════════════════════════════════════════════════════
#
# Deploys:
# - Lambda function in VPC with layers
# - Published version with alias (prod) and weighted routing
# - Reserved concurrency + Provisioned concurrency with auto-scaling
# - Async invocation destinations (EventBridge success, SQS failure)
# - DynamoDB Streams event source mapping with bisect-on-error
# - SQS event source mapping with filtering and DLQ
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

# ═══════════════════════════════════════════════════════════════════════
# DATA SOURCES
# ═══════════════════════════════════════════════════════════════════════

data "aws_region" "current" {}
data "aws_caller_identity" "current" {}

# ═══════════════════════════════════════════════════════════════════════
# IAM ROLE -- Lambda execution role with VPC access
# ═══════════════════════════════════════════════════════════════════════

resource "aws_iam_role" "order_processor" {
  name = "order-processor-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

# VPC access (Hyperplane ENI management)
resource "aws_iam_role_policy_attachment" "vpc_access" {
  role       = aws_iam_role.order_processor.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# Basic execution (CloudWatch Logs)
resource "aws_iam_role_policy_attachment" "basic_execution" {
  role       = aws_iam_role.order_processor.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Custom policy for DynamoDB, SQS, EventBridge, S3
resource "aws_iam_role_policy" "order_processor_permissions" {
  name = "order-processor-permissions"
  role = aws_iam_role.order_processor.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DynamoDBStreamRead"
        Effect = "Allow"
        Action = [
          "dynamodb:DescribeStream",
          "dynamodb:GetRecords",
          "dynamodb:GetShardIterator",
          "dynamodb:ListStreams"
        ]
        Resource = "${aws_dynamodb_table.orders.arn}/stream/*"
      },
      {
        Sid    = "DynamoDBTableAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:Query"
        ]
        Resource = [
          aws_dynamodb_table.orders.arn,
          "${aws_dynamodb_table.orders.arn}/index/*"
        ]
      },
      {
        Sid    = "SQSAccess"
        Effect = "Allow"
        Action = [
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:SendMessage"
        ]
        Resource = [
          aws_sqs_queue.order_queue.arn,
          aws_sqs_queue.order_dlq.arn,
          aws_sqs_queue.stream_failures.arn
        ]
      },
      {
        Sid    = "EventBridgePutEvents"
        Effect = "Allow"
        Action = ["events:PutEvents"]
        Resource = aws_cloudwatch_event_bus.order_events.arn
      }
    ]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# LAMBDA LAYER -- Shared utilities
# ═══════════════════════════════════════════════════════════════════════

resource "aws_lambda_layer_version" "shared_utils" {
  layer_name          = "shared-utils"
  description         = "Shared utility libraries (boto3, aws-lambda-powertools)"
  filename            = "${path.module}/layers/shared-utils.zip"
  source_code_hash    = filebase64sha256("${path.module}/layers/shared-utils.zip")
  compatible_runtimes = ["python3.12", "python3.13"]

  # Layer versions are immutable -- each deploy creates a new version
  # Old versions are retained and can be referenced by other functions
}

# ═══════════════════════════════════════════════════════════════════════
# LAMBDA FUNCTION
# ═══════════════════════════════════════════════════════════════════════

resource "aws_lambda_function" "order_processor" {
  function_name = "order-processor"
  description   = "Processes incoming orders from SQS and DynamoDB Streams"
  role          = aws_iam_role.order_processor.arn
  handler       = "order_processor.handler"
  runtime       = "python3.12"
  timeout       = 30                          # 30 seconds (not 15 min default)
  memory_size   = 1769                        # 1 full vCPU -- sweet spot
  filename      = "${path.module}/functions/order-processor.zip"
  source_code_hash = filebase64sha256("${path.module}/functions/order-processor.zip")

  # Publish a version on every change (required for aliases)
  publish = true

  # Lambda Layers (up to 5)
  layers = [
    aws_lambda_layer_version.shared_utils.arn,
  ]

  # VPC configuration (Hyperplane ENIs -- no cold start penalty post-2019)
  vpc_config {
    subnet_ids         = var.private_subnet_ids        # At least 2 for HA
    security_group_ids = [aws_security_group.lambda.id]
  }

  # Environment variables (4 KB total limit)
  environment {
    variables = {
      TABLE_NAME     = aws_dynamodb_table.orders.name
      EVENT_BUS_NAME = aws_cloudwatch_event_bus.order_events.name
      LOG_LEVEL      = "INFO"
      POWERTOOLS_SERVICE_NAME = "order-processor"
    }
  }

  # Ephemeral storage (default 512 MB, max 10 GB)
  ephemeral_storage {
    size = 512  # MB -- keep default unless processing large files
  }

  # SnapStart example (uncomment for Python 3.12+ SnapStart):
  # snap_start {
  #   apply_on = "PublishedVersions"
  # }
  # Note: Cannot combine with Provisioned Concurrency

  # Reserved concurrency -- guarantees AND caps at 200 concurrent executions
  # This protects downstream DynamoDB from being overwhelmed
  reserved_concurrent_executions = 200

  # Tracing with X-Ray
  tracing_config {
    mode = "Active"
  }

  depends_on = [
    aws_iam_role_policy_attachment.vpc_access,
    aws_iam_role_policy_attachment.basic_execution,
    aws_iam_role_policy.order_processor_permissions,
    aws_cloudwatch_log_group.order_processor,
  ]
}

# CloudWatch Log Group (create explicitly to control retention)
resource "aws_cloudwatch_log_group" "order_processor" {
  name              = "/aws/lambda/order-processor"
  retention_in_days = 30
}

# ═══════════════════════════════════════════════════════════════════════
# VERSIONS AND ALIASES -- Deployment Strategy
# ═══════════════════════════════════════════════════════════════════════

# The "prod" alias points to a published version
# When publish = true above, aws_lambda_function.order_processor.version
# contains the latest published version number
resource "aws_lambda_alias" "prod" {
  name             = "prod"
  description      = "Production alias -- API Gateway and triggers reference this"
  function_name    = aws_lambda_function.order_processor.function_name
  function_version = aws_lambda_function.order_processor.version

  # Weighted routing for canary deployments:
  # Uncomment to send 10% of traffic to a different version
  # routing_config {
  #   additional_version_weights = {
  #     (var.canary_version) = 0.1  # 10% to canary version
  #   }
  # }
}

# ═══════════════════════════════════════════════════════════════════════
# PROVISIONED CONCURRENCY + AUTO SCALING
# ═══════════════════════════════════════════════════════════════════════

# Provisioned Concurrency on the alias (NOT on $LATEST)
resource "aws_lambda_provisioned_concurrency_config" "prod" {
  function_name                     = aws_lambda_function.order_processor.function_name
  qualifier                         = aws_lambda_alias.prod.name
  provisioned_concurrent_executions = 10  # 10 always-warm environments
}

# Auto Scaling target for provisioned concurrency
resource "aws_appautoscaling_target" "lambda_provisioned" {
  max_capacity       = 100
  min_capacity       = 10
  resource_id        = "function:${aws_lambda_function.order_processor.function_name}:${aws_lambda_alias.prod.name}"
  scalable_dimension = "lambda:function:ProvisionedConcurrency"
  service_namespace  = "lambda"
}

# Target tracking: keep provisioned utilization at 70%
resource "aws_appautoscaling_policy" "lambda_provisioned" {
  name               = "order-processor-provisioned-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.lambda_provisioned.resource_id
  scalable_dimension = aws_appautoscaling_target.lambda_provisioned.scalable_dimension
  service_namespace  = aws_appautoscaling_target.lambda_provisioned.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value = 0.7  # Scale when 70% of provisioned concurrency is in use

    predefined_metric_specification {
      predefined_metric_type = "LambdaProvisionedConcurrencyUtilization"
    }
  }
}

# Scheduled scaling: pre-warm before the morning rush
resource "aws_appautoscaling_scheduled_action" "morning_rush" {
  name               = "morning-rush-prewarm"
  service_namespace  = aws_appautoscaling_target.lambda_provisioned.service_namespace
  resource_id        = aws_appautoscaling_target.lambda_provisioned.resource_id
  scalable_dimension = aws_appautoscaling_target.lambda_provisioned.scalable_dimension
  schedule           = "cron(55 8 ? * MON-FRI *)"  # 8:55 AM UTC weekdays

  scalable_target_action {
    min_capacity = 50  # Pre-scale to 50 before 9 AM rush
    max_capacity = 100
  }
}

# ═══════════════════════════════════════════════════════════════════════
# ASYNC INVOCATION DESTINATIONS
# ═══════════════════════════════════════════════════════════════════════

# Custom EventBridge bus for order events
resource "aws_cloudwatch_event_bus" "order_events" {
  name = "order-events"
}

# Configure async invocation destinations on the prod alias
resource "aws_lambda_function_event_invoke_config" "prod" {
  function_name = aws_lambda_function.order_processor.function_name
  qualifier     = aws_lambda_alias.prod.name

  maximum_event_age_in_seconds = 3600  # Discard events older than 1 hour
  maximum_retry_attempts       = 2     # Lambda retries twice (default)

  destination_config {
    on_success {
      destination = aws_cloudwatch_event_bus.order_events.arn
    }
    on_failure {
      destination = aws_sqs_queue.order_dlq.arn
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# SQS QUEUES -- Order Queue with DLQ
# ═══════════════════════════════════════════════════════════════════════

resource "aws_sqs_queue" "order_dlq" {
  name                      = "order-processing-dlq"
  message_retention_seconds = 1209600  # 14 days (maximum)

  # DLQ for the DLQ -- if you cannot process DLQ messages, alarm
  # In production, monitor this queue depth with CloudWatch Alarms
}

resource "aws_sqs_queue" "order_queue" {
  name                       = "order-processing-queue"
  visibility_timeout_seconds = 180  # 6x Lambda timeout (30s x 6)
  # Rule of thumb: set visibility timeout to at least 6x the Lambda timeout
  # to account for retries and processing time

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_dlq.arn
    maxReceiveCount     = 3  # After 3 failed attempts, send to DLQ
  })
  # NOTE: For SQS event source mappings, the DLQ is on the SQS QUEUE
  # (via redrive_policy), NOT on the Lambda function
}

# ═══════════════════════════════════════════════════════════════════════
# SQS EVENT SOURCE MAPPING -- With Filtering
# ═══════════════════════════════════════════════════════════════════════

resource "aws_lambda_event_source_mapping" "sqs_orders" {
  event_source_arn = aws_sqs_queue.order_queue.arn
  function_name    = aws_lambda_alias.prod.arn  # Target the alias, not $LATEST
  batch_size       = 10
  enabled          = true

  # Maximum concurrency -- prevents overwhelming downstream services
  scaling_config {
    maximum_concurrency = 50  # Max 50 concurrent Lambda invocations from this source
  }

  # Event filtering -- only process ORDER_PLACED events with amount > 0
  # Non-matching messages are automatically deleted from the queue
  filter_criteria {
    filter {
      pattern = jsonencode({
        body = {
          eventType = ["ORDER_PLACED"]
          amount    = [{ numeric = [">", 0] }]
        }
      })
    }
  }

  # Report partial batch failures -- only failed messages return to queue
  function_response_types = ["ReportBatchItemFailures"]
}

# ═══════════════════════════════════════════════════════════════════════
# DYNAMODB TABLE + STREAM
# ═══════════════════════════════════════════════════════════════════════

resource "aws_dynamodb_table" "orders" {
  name         = "orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  # Enable streams for CDC (Change Data Capture)
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"  # Both before and after images

  point_in_time_recovery {
    enabled = true
  }
}

# SQS queue for stream processing failures
resource "aws_sqs_queue" "stream_failures" {
  name                      = "order-stream-failures"
  message_retention_seconds = 1209600
}

# ═══════════════════════════════════════════════════════════════════════
# DYNAMODB STREAMS EVENT SOURCE MAPPING -- With Bisect-on-Error
# ═══════════════════════════════════════════════════════════════════════

resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.orders.stream_arn
  function_name     = aws_lambda_alias.prod.arn
  starting_position = "LATEST"  # Process only new changes (TRIM_HORIZON for all)

  batch_size                         = 100
  maximum_batching_window_in_seconds = 5     # Wait up to 5s to accumulate records
  parallelization_factor             = 2     # 2 concurrent batches per shard

  # Production-grade error handling for streams
  bisect_batch_on_function_error = true      # Split failed batches to find poison record
  maximum_retry_attempts         = 3         # Retry up to 3 times before giving up
  maximum_record_age_in_seconds  = 3600      # Skip records older than 1 hour

  # On-failure destination -- captures metadata about the failed batch
  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.stream_failures.arn
    }
  }

  # Report partial batch failures
  function_response_types = ["ReportBatchItemFailures"]

  # Event filtering -- only process INSERT and MODIFY events, not REMOVE
  filter_criteria {
    filter {
      pattern = jsonencode({
        eventName = ["INSERT", "MODIFY"]
      })
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# SECURITY GROUP -- Lambda VPC Security Group
# ═══════════════════════════════════════════════════════════════════════

resource "aws_security_group" "lambda" {
  name_prefix = "lambda-order-processor-"
  description = "Security group for order processor Lambda in VPC"
  vpc_id      = var.vpc_id

  # Outbound: allow access to DynamoDB, SQS, etc. via VPC endpoints
  # and to RDS/ElastiCache within the VPC
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound (VPC endpoints + NAT Gateway)"
  }

  # No inbound rules needed -- Lambda initiates all connections
  # Lambda is never a server, always a client

  lifecycle {
    create_before_destroy = true
  }
}

# ═══════════════════════════════════════════════════════════════════════
# CLOUDWATCH ALARMS -- Operational Monitoring
# ═══════════════════════════════════════════════════════════════════════

# Alarm on high error rate
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "order-processor-high-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  threshold           = 5
  alarm_description   = "Lambda function error count exceeds 5 in 2 consecutive periods"

  metric_name = "Errors"
  namespace   = "AWS/Lambda"
  period      = 300
  statistic   = "Sum"

  dimensions = {
    FunctionName = aws_lambda_function.order_processor.function_name
    Resource     = "${aws_lambda_function.order_processor.function_name}:${aws_lambda_alias.prod.name}"
  }

  alarm_actions = [var.sns_alarm_topic_arn]
}

# Alarm on throttling (indicates concurrency limits being hit)
resource "aws_cloudwatch_metric_alarm" "lambda_throttles" {
  alarm_name          = "order-processor-throttled"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  threshold           = 0
  alarm_description   = "Lambda function is being throttled -- increase reserved concurrency or account limit"

  metric_name = "Throttles"
  namespace   = "AWS/Lambda"
  period      = 60
  statistic   = "Sum"

  dimensions = {
    FunctionName = aws_lambda_function.order_processor.function_name
    Resource     = "${aws_lambda_function.order_processor.function_name}:${aws_lambda_alias.prod.name}"
  }

  alarm_actions = [var.sns_alarm_topic_arn]
}

# Alarm on DLQ depth (failed messages accumulating)
resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  alarm_name          = "order-dlq-messages-available"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  threshold           = 0
  alarm_description   = "Messages appearing in order processing DLQ -- investigate failed orders"

  metric_name = "ApproximateNumberOfMessagesVisible"
  namespace   = "AWS/SQS"
  period      = 300
  statistic   = "Sum"

  dimensions = {
    QueueName = aws_sqs_queue.order_dlq.name
  }

  alarm_actions = [var.sns_alarm_topic_arn]
}

# ═══════════════════════════════════════════════════════════════════════
# VARIABLES
# ═══════════════════════════════════════════════════════════════════════

variable "vpc_id" {
  description = "VPC ID for Lambda VPC configuration"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs for Lambda (at least 2 for HA)"
  type        = list(string)
}

variable "sns_alarm_topic_arn" {
  description = "SNS topic ARN for CloudWatch alarm notifications"
  type        = string
}

# variable "canary_version" {
#   description = "Lambda version number for canary deployment (weighted alias)"
#   type        = string
#   default     = ""
# }

# ═══════════════════════════════════════════════════════════════════════
# OUTPUTS
# ═══════════════════════════════════════════════════════════════════════

output "lambda_function_name" {
  value = aws_lambda_function.order_processor.function_name
}

output "lambda_function_arn" {
  value = aws_lambda_function.order_processor.arn
}

output "lambda_alias_arn" {
  description = "Use this ARN for API Gateway, EventBridge, and other triggers"
  value       = aws_lambda_alias.prod.arn
}

output "lambda_current_version" {
  value = aws_lambda_function.order_processor.version
}

output "sqs_queue_url" {
  value = aws_sqs_queue.order_queue.url
}

output "dynamodb_stream_arn" {
  value = aws_dynamodb_table.orders.stream_arn
}
```

---

## Critical Gotchas and Interview Traps

**1. "Provisioned Concurrency does NOT set reserved concurrency. They are independent."**
Reserved concurrency sets the maximum AND guarantees a minimum allocation from the account pool. Provisioned concurrency pre-warms environments. You can (and should) use both together: reserved=200 ensures the function can reach 200 and cannot exceed it; provisioned=50 ensures 50 of those 200 are always warm. Without reserved concurrency, a function with provisioned concurrency could theoretically be limited by other functions consuming all unreserved capacity.

**2. "Cold starts happen even WITH reserved concurrency."**
Reserved concurrency guarantees capacity in the account pool but does NOT pre-initialize environments. When a burst arrives, the reserved slots are available, but each new environment still goes through the Init phase. Only Provisioned Concurrency eliminates cold starts.

**3. "For SQS event source mappings, the DLQ is on the SQS queue, not on Lambda."**
The `dead_letter_config` on the Lambda function only applies to asynchronous invocations (S3, SNS, EventBridge triggers). For SQS event source mappings, failed messages return to the queue when the visibility timeout expires, and the SQS queue's `RedrivePolicy` (maxReceiveCount) determines when messages move to the SQS dead-letter queue. Configuring a DLQ on the Lambda function has zero effect on SQS event source mapping failures.

**4. "SnapStart and Provisioned Concurrency are mutually exclusive."**
You cannot enable both on the same function. SnapStart reduces cold starts at low cost. Provisioned Concurrency eliminates cold starts at high cost. Choose based on your latency requirements and budget.

**5. "Lambda in a VPC loses internet access. Hyperplane fixed cold starts, not routing."**
The 2019 Hyperplane improvement eliminated the 10-30 second ENI creation cold start penalty. It did NOT give VPC Lambda functions internet access. You still need a NAT Gateway or VPC endpoints for AWS API calls and external services.

**6. "Weighted aliases split between exactly two versions. Not three."**
An alias has a primary version and optionally one additional version with a weight. You cannot do a three-way split. For more complex traffic management, use API Gateway stage canary settings.

**7. "Lambda@Edge functions MUST be created in us-east-1."**
Even though they execute at regional edge caches worldwide, the function itself must be authored and published in the US East (N. Virginia) region. This is a CloudFront constraint, not a Lambda constraint.

**8. "$LATEST does not work with aliases, SnapStart, or Lambda@Edge."**
Aliases must point to a published numbered version. SnapStart only applies to published versions. Lambda@Edge only works with published versions. Always deploy with `publish = true` in Terraform.

**9. "Visibility timeout for SQS should be at least 6x your Lambda timeout."**
If your Lambda function has a 30-second timeout, set the SQS visibility timeout to at least 180 seconds. If visibility timeout is too short, SQS makes the message visible again while Lambda is still processing it, causing duplicate processing.

**10. "Event source mapping filtering deletes non-matching messages from SQS queues."**
For queue sources, filtered-out messages are permanently deleted (counted as successfully processed). For stream sources, the iterator simply advances past them. This means a filtering mistake on SQS can silently discard messages with no recovery path. Test filters thoroughly.

**11. "Function URLs have no built-in throttling."**
Unlike API Gateway (which offers per-route throttling, usage plans, and burst limits), Function URLs have no throttling mechanism. The only way to limit concurrency is via reserved concurrency on the function itself. For public-facing APIs with unknown traffic patterns, API Gateway is safer.

**12. "Lambda's at-least-once delivery means your function MUST be idempotent."**
Async invocations, SQS retries, and stream batch retries can all deliver the same event multiple times. If your function creates a database record, charges a payment, or sends a notification, duplicate invocations will duplicate the side effect unless you implement idempotency. This is not optional -- it is a core design requirement.

**13. "ARM64 gives you a 20% discount for one line of config."**
Most Python, Node.js, and Java functions work on ARM64 with zero code changes. The `architectures = ["arm64"]` setting is the single easiest cost optimization for Lambda. Combined with Compute Savings Plans (17% off), total savings reach 34%.

---

## Key Takeaways

- **Initialize expensive resources outside the handler.** SDK clients, database connections, and configuration loading belong at module scope, not inside the handler function. This is the single highest-impact optimization for Lambda performance because it turns repeated Init work into a one-time cost per execution environment.

- **Cold starts are the sum of three factors: runtime init + package download + your initialization code.** Choosing Python over Java, minimizing package size, and increasing memory (which increases CPU) are all levers. SnapStart and Provisioned Concurrency are the nuclear options for when code-level optimizations are not enough.

- **Reserved concurrency is free capacity planning; Provisioned Concurrency is paid warmth.** Use reserved concurrency on every production function to prevent noisy-neighbor problems and to protect downstream services. Add Provisioned Concurrency only for latency-critical synchronous paths.

- **The three invocation models have completely different error handling contracts.** Synchronous: caller handles errors. Asynchronous: Lambda retries twice, then routes to Destination/DLQ. Event source mapping: depends on source type (SQS uses visibility timeout + queue DLQ; streams use bisect + retry + on-failure destination). Memorize this matrix.

- **Destinations supersede DLQs for asynchronous invocations.** Destinations support both success and failure routing, include richer metadata (full invocation record with response and stack trace), and support more target types (EventBridge). Use DLQs only for legacy compatibility or SQS queue redrive policies.

- **BisectBatchOnFunctionError + MaximumRetryAttempts + on-failure destination is the production error handling pattern for stream sources.** Without bisect, one poison record blocks the entire shard indefinitely. With bisect, Lambda recursively halves the batch to isolate the problem record while processing all good records normally.

- **VPC Lambda requires NAT Gateway or VPC endpoints for internet/AWS API access.** Hyperplane eliminated the cold start penalty for VPC attachment, but it did not change the fundamental networking requirement. Budget for NAT Gateway costs or deploy VPC endpoints for the AWS services your function calls.

- **Weighted alias routing with CodeDeploy is Lambda's equivalent of ECS blue-green deployments.** The alias ARN never changes for consumers. The underlying version shifts gradually with automated rollback on CloudWatch Alarm triggers. Always deploy behind aliases, never reference `$LATEST` in production.

- **Recursive loop detection is automatic but not instant.** Lambda detects Lambda->SQS->Lambda, Lambda->SNS->Lambda, and Lambda->S3->Lambda loops after approximately 16 recursive invocations. Design defensively: use distinct input/output resources, event filtering, and idempotency tokens.

- **Design every Lambda function for idempotency.** At-least-once delivery is the default for async and event source mapping invocations. Use DynamoDB conditional writes, Lambda Powertools idempotency decorator, or S3 existence checks to prevent duplicate side effects. This is not an edge case -- it is a fundamental Lambda design requirement.

- **Function URLs are free HTTPS endpoints -- use them for simple webhooks and internal APIs.** No API Gateway cost, no ALB cost, no setup complexity. But no throttling, no custom domains, and limited auth (IAM or none). For production APIs needing rate limiting, caching, or custom authorizers, use API Gateway.

- **ARM64 + Compute Savings Plans = up to 34% savings with minimal effort.** ARM64 is a one-line config change that saves 20% on duration for most functions. Compute Savings Plans add another 17%. This is the lowest-effort, highest-impact cost optimization for Lambda at scale.

- **The Lambda free tier is perpetual.** Unlike most AWS free tiers that expire after 12 months, the 1M requests + 400K GB-seconds monthly allowance is permanent. A low-traffic function can run indefinitely at zero cost.

---

## Study Resources

Curated reading list for this topic: [`study-resources/aws/lambda-deep-dive.md`](../../study-resources/aws/lambda-deep-dive.md)

**Key references**:
- [Lambda Execution Environment Lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html) -- The foundational mental model (Init/Invoke/Shutdown phases)
- [Understanding and Remediating Cold Starts](https://aws.amazon.com/blogs/compute/understanding-and-remediating-cold-starts-an-aws-lambda-perspective/) -- Cold start causes, measurement, and mitigation
- [Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html) -- Reserved vs Provisioned vs Unreserved with formulas
- [Event Source Mappings](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html) -- SQS, Kinesis, DynamoDB Streams batching and error handling
- [SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) -- Snapshot-based cold start optimization
- [Lambda Error Handling Patterns](https://aws.amazon.com/blogs/compute/implementing-aws-lambda-error-handling-patterns/) -- Destinations vs DLQ across all invocation models
- [Alias Routing for Canary Deployments](https://docs.aws.amazon.com/lambda/latest/dg/configuring-alias-routing.html) -- Weighted aliases and CodeDeploy integration

**Related docs in this repo**:
- [CloudFront Deep Dive (Mar 13)](../../march/aws/2026-03-13-cloudfront-deep-dive.md) -- Lambda@Edge vs CloudFront Functions comparison from the CDN perspective
- [DynamoDB Deep Dive (Mar 26)](../../march/aws/2026-03-26-dynamodb-deep-dive.md) -- DynamoDB Streams that Lambda event source mappings consume
- [ECS Fundamentals (Feb 16)](../../february/aws/2026-02-16-ecs-fundamentals.md) -- Container-based compute for comparison with serverless model
- [VPC Advanced (Feb 19)](../../february/networking/2026-02-19-vpc-advanced.md) -- VPC networking that Lambda VPC integration relies on
