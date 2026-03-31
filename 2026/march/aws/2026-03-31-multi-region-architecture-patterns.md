# Multi-Region Architecture Patterns -- The Multinational Corporation Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock), equipped your instances with personal workbenches, shared libraries, and specialty studios (EBS, EFS, FSx), built the managed hotel chain with its revolutionary shared vault system (RDS and Aurora), organized the giant filing cabinet with drawer labels and folder tabs (DynamoDB with DAX and Global Tables), installed the deli counter for lightning-fast repeat orders (ElastiCache with Global Datastore), and yesterday mastered the hospital emergency preparedness playbook (DR Strategies from Backup & Restore to Multi-Site Active-Active). Each of those services is a powerful capability on its own. But individually, they are ingredients -- flour, eggs, butter, sugar sitting on a counter. Today you learn how to **bake the cake**: composing these services into a coherent multi-region architecture where every component has a role, every dependency is understood, and the system continues operating even when an entire AWS Region goes dark. This is the architectural capstone that ties every service you have studied into a unified, production-grade system. The jump from "I know how Aurora Global Database works" to "I know how to architect a multi-region application" is the jump from knowing how an engine works to designing the entire car.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Multi-Region Architecture | A multinational corporation with offices in multiple countries -- each office serves local customers independently, but they coordinate on inventory, pricing, and HR policies through shared systems | Running your application in two or more AWS Regions so users are served from the nearest location and the system survives a full Region failure |
| Data Plane vs Control Plane | The difference between **driving your car** (data plane: steering, accelerating, braking -- always works) and **calling the dealership to modify your car** (control plane: requesting changes, ordering parts -- may have a wait or be closed) | Data plane operations (serving traffic, answering DNS queries) have higher availability than control plane operations (creating resources, modifying configurations); failover must use data plane operations |
| Static Stability | A building with backup generators that are already running and connected, not generators sitting in a warehouse that need to be delivered and installed during a power outage | Pre-provisioning resources so your system continues operating without needing to make any changes during a failure; the opposite of "launch new resources when something breaks" |
| Active-Active vs Active-Passive | Two restaurant locations both serving diners every night (active-active) vs one restaurant with a fully equipped but closed second kitchen that opens only when the main kitchen catches fire (active-passive) | Active-active serves traffic from all Regions simultaneously; active-passive keeps secondary Regions idle or read-only until failover |
| Shared Fate | If all your restaurant's delivery drivers use the same bridge to cross the river, a bridge closure takes out your entire delivery operation -- the bridge is a shared fate dependency | Components that fail together because they share a common dependency; the goal is to eliminate cross-region shared fate |
| Cell-Based Architecture | A hotel chain where each hotel is a completely independent operation with its own reservations system, staff, and kitchen -- if one hotel has a fire, guests at other hotels are unaffected because nothing is shared; the booking website routes guests to specific hotels and never moves them mid-stay | Partitioning your application into independent, isolated stacks (cells) that each serve a subset of customers; a failure in one cell does not propagate to others because cells share nothing |
| CAP Theorem (applied to AWS) | You cannot have a restaurant chain where every location has identical menus updated instantly (consistency), every location is always open (availability), and the locations continue operating normally when the phone lines between them go down (partition tolerance) -- pick two | In distributed systems, network partitions are inevitable, so the real choice is between consistency (CP: reject requests during partitions) and availability (AP: serve possibly-stale data during partitions) |
| PACELC Extension | Even when the phone lines between restaurant locations are working fine (no partition), you still face a trade-off: do you wait for headquarters to confirm every menu change before serving customers (consistency, higher latency), or do you serve from the local menu and sync later (availability, lower latency)? | Extends CAP: when Partitioned, choose A or C; Else (normal operation), choose Latency or Consistency. Explains why DynamoDB Global Tables with MREC is low-latency eventual-consistent even without partitions |
| Sequential Deployment | A restaurant chain that rolls out a new menu to one city first, monitors customer reactions for a week, then rolls it out to the next city -- never to all cities at once | Deploying changes to one Region, monitoring for errors, then deploying to the next; prevents a bad deployment from taking down all Regions simultaneously |
| ARC Routing Controls | A set of physical master switches on a control panel that instantly redirect all customers from one restaurant location to another -- the switches work even if the phone system is down because they are hardwired | Data-plane-only routing controls that shift traffic between Regions without depending on control-plane APIs that may be degraded during the outage you are responding to |

---

## The Big Picture: From Individual Services to System Architecture

Yesterday's DR Strategies doc answered the question "what happens when a Region goes down?" Today's doc answers a harder question: **"how do I design a system that operates across multiple Regions every day, not just during disasters?"**

The distinction matters. DR is reactive -- you have a primary Region and a recovery plan. Multi-region architecture is proactive -- you design the system from day one to span Regions, serve users globally, and treat Region failure as a routine event rather than an emergency. Every DR strategy you studied maps to a multi-region pattern:

| DR Strategy (Yesterday) | Multi-Region Pattern (Today) | Key Difference |
|--------------------------|------------------------------|----------------|
| Backup & Restore | N/A (single-region with backups) | Not truly multi-region -- no live infrastructure in second Region |
| Pilot Light | Active-Passive (cold compute) | Data tier replicates; compute is off until needed |
| Warm Standby | Active-Passive (warm compute) | Full stack runs at reduced scale in secondary |
| Multi-Site Active-Active | Active-Active | Both Regions serve live traffic simultaneously |

But the architecture-level challenge goes far beyond choosing a DR strategy. You must compose dozens of services into a coherent system where:
- Every component knows its role in the multi-region topology
- No single-region dependency can take down the global system
- Data consistency is explicitly chosen per use case (not hoped for)
- Failover uses pre-provisioned resources and data-plane operations
- Deployments cannot break all Regions simultaneously
- Monitoring works even when the monitored Region is down

Think of it as building a multinational corporation. Having offices in London and New York is not enough. You need to decide: Does each office operate independently (active-active), or does London only open when New York closes (active-passive)? How do you keep inventory synchronized? What happens when the transatlantic cable goes down? Can the London office hire employees if the HR system in New York is unreachable? These are the architectural questions that determine whether your "multi-region" label is genuine or just marketing.

---

## Part 1: Data Plane vs Control Plane -- The #1 Multi-Region Principle

### The Analogy

Imagine the difference between **driving your car** and **calling the dealership to modify your car**. When you are driving, everything works through direct physical mechanisms -- you turn the steering wheel, the wheels turn; you press the accelerator, the engine responds; you press the brake, the car stops. These are data plane operations. They are simple, reliable, and always available because they use pre-provisioned, dedicated hardware already in your car.

Now imagine you want to add heated seats. You call the dealership (control plane). The request goes through a customer service system, gets routed to a technician, parts are ordered from a warehouse, and eventually someone schedules an installation. This is complex orchestration involving multiple systems. If the dealership's phone system is down, you cannot request the modification. But critically, **you can still drive your car**. The data plane (driving) operates independently of the control plane (modifications).

### The Technical Reality

Every AWS service has two planes:

```
DATA PLANE vs CONTROL PLANE -- THE CRITICAL DISTINCTION
════════════════════════════════════════════════════════════════════════════

  CONTROL PLANE                          DATA PLANE
  ─────────────                          ──────────
  "CRUDL" operations:                    "Serve traffic" operations:
  Create, Read, Update, Delete, List     Reading data, answering queries,
                                         routing packets, serving pages

  ┌────────────────────────────────┐     ┌────────────────────────────────┐
  │  Examples:                     │     │  Examples:                     │
  │                                │     │                                │
  │  ─ CreateDBCluster             │     │  ─ SQL queries to Aurora       │
  │  ─ ModifyDBInstance            │     │  ─ GetItem / PutItem in DDB    │
  │  ─ RunInstances (launch EC2)   │     │  ─ GET/PUT to S3 objects       │
  │  ─ ChangeResourceRecordSets   │     │  ─ Route53 DNS query answering │
  │    (editing Route53 records)   │     │  ─ ALB routing requests        │
  │  ─ CreateAutoScalingGroup      │     │  ─ Global Accelerator routing  │
  │  ─ UpdateService (ECS)         │     │  ─ CloudFront serving content  │
  │  ─ CreateFunction (Lambda)     │     │  ─ Lambda function invocation  │
  │                                │     │  ─ ARC routing controls        │
  │  Higher complexity.            │     │                                │
  │  More dependencies.            │     │  Simpler, fewer dependencies.  │
  │  Lower availability SLA.       │     │  Higher availability SLA.      │
  │  May be unavailable during     │     │  Designed to remain available  │
  │  the very outage that triggers │     │  even during major outages.    │
  │  your failover.                │     │                                │
  └────────────────────────────────┘     └────────────────────────────────┘

  THE GOLDEN RULE:
  ════════════════
  Your failover mechanism must NOT depend on the thing that is failing.
  If a Region is down, any control-plane API call that depends on that
  Region's control plane may also be down. Use data-plane operations
  for failover.
```

### The us-east-1 Control Plane Problem

This is the single most critical exam gotcha for multi-region architecture. Several global AWS services have their **control plane exclusively in us-east-1**:

| Service | Control Plane (us-east-1) | Data Plane (Global) |
|---------|---------------------------|---------------------|
| **Route53** | Creating/modifying hosted zones and records (`ChangeResourceRecordSets`) | Answering DNS queries (global anycast network) |
| **IAM** | Creating/modifying users, roles, policies | Authenticating and authorizing API calls (cached globally) |
| **CloudFront** | Creating/modifying distributions, cache policies | Serving cached content from 600+ edge POPs |
| **ACM** (for CloudFront) | Issuing/renewing certificates | TLS termination at edge |
| **WAF** (global scope) | Creating/modifying WebACLs, rules | Evaluating requests at edge |

**What this means in practice**: If us-east-1 experiences a significant outage, you **cannot edit Route53 DNS records**. You cannot create new IAM roles. You cannot modify CloudFront distributions. But -- and this is the key -- Route53 will continue **answering DNS queries** using the data plane. CloudFront will continue serving cached content. IAM will continue authenticating requests using globally cached credentials.

**The failover implication**: If your failover plan involves editing Route53 records to redirect traffic, that plan may fail precisely when you need it most -- during a major us-east-1 outage. Instead:

| Bad (Control Plane) | Good (Data Plane) |
|---------------------|--------------------|
| Manually editing Route53 records during an outage | Pre-configured Route53 health-check-based failover routing (data plane evaluates health and routes automatically) |
| Launching new EC2 instances in the DR Region during failover | Pre-provisioned instances already running (static stability) |
| Scaling up Auto Scaling groups during failover | Already-running capacity at target scale |
| Creating new load balancers during failover | Pre-provisioned ALBs already health-checked |
| Calling `UpdateService` to increase ECS task count | ARC routing controls shifting traffic to pre-provisioned capacity |

### Static Stability -- The Architectural Consequence

The data-plane-vs-control-plane distinction leads directly to the principle of **static stability**: design your system so it continues operating correctly without making any changes, even if the control plane is entirely unavailable. Your system should be stable in its current static configuration.

This means:
- Resources are **pre-provisioned**, not created on demand during failover
- Scaling is already at target capacity, not dependent on Auto Scaling API calls
- DNS records are already configured for failover, not edited manually
- Health checks are already evaluating and will automatically trigger routing changes

Static stability is the architectural foundation of Warm Standby and Active-Active patterns. It is why Pilot Light (which requires deploying compute via control-plane operations during failover) is less resilient than Warm Standby (where compute is already running).

---

## Part 2: Active-Active vs Active-Passive Decision Framework

Yesterday's DR doc covered **what** Active-Active and Active-Passive are. Today we focus on **when to use each** -- the decision factors that drive the choice.

### The Decision Matrix

```
ACTIVE-ACTIVE vs ACTIVE-PASSIVE -- WHEN TO USE WHICH
════════════════════════════════════════════════════════════════════════════

  Choose ACTIVE-PASSIVE when:
  ───────────────────────────
  ─ Your application has a single-writer database pattern
    (relational with transactions, strong consistency required)
  ─ Write conflicts are unacceptable (financial transactions,
    inventory management with exact counts)
  ─ Operational team is small (active-active is operationally complex)
  ─ Cost must be minimized (secondary Region runs at reduced scale)
  ─ Regulatory requirements mandate a "primary" processing location
  ─ RTO of minutes (not seconds) is acceptable

  Choose ACTIVE-ACTIVE when:
  ──────────────────────────
  ─ Users are geographically distributed and latency matters
    (serving EU users from eu-west-1, US users from us-east-1)
  ─ Write conflicts can be tolerated or resolved
    (last-writer-wins, application-level conflict resolution)
  ─ Data model supports partitioned or eventual-consistent writes
    (DynamoDB Global Tables, S3 CRR)
  ─ Zero RTO is required (no failover step for routing layer)
  ─ The cost of running full infrastructure in 2+ Regions is justified
    by the combination of latency improvement + availability guarantee
  ─ The team has the operational maturity to manage multi-region
    deployments, monitoring, and incident response
```

### The Five Decision Factors

**1. Write Conflict Complexity (Most Important)**

This is the primary driver. Your data model determines your multi-region pattern more than any other factor:

| Data Pattern | Multi-Region Model | Why |
|---|---|---|
| Relational with transactions, referential integrity | Active-Passive (Aurora Global DB, write forwarding) | Transactions cannot span Regions; single writer avoids conflicts |
| Key-value with independent items, eventual consistency acceptable | Active-Active (DynamoDB Global Tables MREC) | Last-writer-wins resolves conflicts; items are independent |
| Key-value with strong cross-region consistency required | Active-Active with constraints (DynamoDB MRSC, 3 Regions) | Quorum-based consistency; higher latency but correctness guaranteed |
| Object storage with region-owned prefixes | Active-Active (S3 bidirectional CRR) | No conflicts when each Region writes to its own prefix |
| Mixed (some relational, some NoSQL) | Hybrid: Active-Passive for relational + Active-Active for NoSQL | Most production systems use this pattern |

**2. Latency Requirements**

If your users span continents and need sub-100ms response times, Active-Active is the only option. A user in Frankfurt hitting a database in Virginia adds 80-120ms of network latency per round trip. Write forwarding (Aurora Global Database) mitigates this for reads but not for writes -- write forwarding still routes the write across the Atlantic, adding ~80-120ms of cross-region round-trip latency on top of the primary Region's normal write latency.

**3. Cost Tolerance**

Active-Active roughly doubles your infrastructure cost. But the framing matters: if you are serving traffic from both Regions (not just running an idle DR copy), the second Region is a **latency investment** as well as an **availability investment**. The cost is not purely DR overhead -- it also improves user experience for half your user base.

**4. Compliance Requirements**

GDPR, data residency laws, and industry regulations may dictate where data lives and where it is processed. Some regulations require that EU customer data be processed exclusively in EU Regions. This can force an Active-Active pattern where EU traffic stays in eu-west-1 and US traffic stays in us-east-1, with limited cross-region replication.

**5. Operational Maturity**

Active-Active is operationally harder than Active-Passive. You need:
- Sequential deployment pipelines (deploy to one Region, monitor, deploy to next)
- Per-region monitoring with cross-region dashboards
- Runbooks for partial failures (one Region degraded but not down)
- Game days and chaos engineering practice
- On-call teams that understand multi-region failure modes

If your team has never managed a multi-region system, start with Active-Passive. Graduate to Active-Active once the operational practices are mature.

---

## Part 3: Multi-Region Data Patterns -- Composing the Data Tier

You already know each data service's replication capabilities individually. This section shows how they compose into coherent multi-region data architectures. The key insight is that **different data in the same application can use different replication patterns**.

```
MULTI-REGION DATA PATTERN MATRIX
════════════════════════════════════════════════════════════════════════════

  Pattern           │ Service              │ Write Model         │ Consistency
  ──────────────────┼──────────────────────┼─────────────────────┼────────────
  Read Local,       │ Aurora Global DB     │ Single writer in    │ Eventual in
  Write Global      │ (write forwarding)   │ primary; forwarded  │ secondary
                    │                      │ writes from         │ (sub-1s lag);
                    │                      │ secondary at SQL    │ strong in
                    │                      │ layer               │ primary
  ──────────────────┼──────────────────────┼─────────────────────┼────────────
  Read Local,       │ DynamoDB Global      │ Any Region writes   │ Eventual
  Write Local       │ Tables (MREC)        │ locally; conflicts  │ (last-writer-
  (eventual)        │                      │ resolved by         │ wins by
                    │                      │ last-writer-wins    │ timestamp)
  ──────────────────┼──────────────────────┼─────────────────────┼────────────
  Read Local,       │ DynamoDB Global      │ Any Region writes;  │ Strong across
  Write Local       │ Tables (MRSC)        │ quorum across 3     │ 3 Regions
  (strong)          │                      │ Regions before      │ (higher write
                    │                      │ acknowledging       │ latency)
  ──────────────────┼──────────────────────┼─────────────────────┼────────────
  Read Local,       │ S3 Bidirectional     │ Each Region owns    │ Eventual
  Write Partitioned │ CRR                  │ certain prefixes;   │ (replication
                    │                      │ no conflicts by     │ in minutes)
                    │                      │ design              │
  ──────────────────┼──────────────────────┼─────────────────────┼────────────
  Read Local,       │ ElastiCache Global   │ Single primary      │ Eventual
  Write Global      │ Datastore            │ Region for writes;  │ (sub-1s lag
  (cache tier)      │                      │ read replicas in    │ to secondary)
                    │                      │ secondary Regions   │
  ──────────────────┼──────────────────────┼─────────────────────┼────────────
```

### How These Compose in a Real Application

A typical multi-region e-commerce platform uses **all of these patterns simultaneously** for different data types:

```
MULTI-REGION DATA COMPOSITION -- E-COMMERCE EXAMPLE
════════════════════════════════════════════════════════════════════════════

  Data Type           │ Service                │ Pattern              │ Why
  ────────────────────┼────────────────────────┼──────────────────────┼─────────
  Product catalog     │ Aurora Global DB       │ Read Local,          │ Complex
  (relational,        │ (write forwarding)     │ Write Global         │ queries,
  complex queries)    │                        │                      │ JOINs,
                      │                        │                      │ transactions
  ────────────────────┼────────────────────────┼──────────────────────┼─────────
  Shopping cart,      │ DynamoDB Global        │ Read Local,          │ Key-value,
  user sessions       │ Tables (MREC)          │ Write Local          │ high write
                      │                        │                      │ rate, LWW
                      │                        │                      │ acceptable
  ────────────────────┼────────────────────────┼──────────────────────┼─────────
  Order processing    │ Aurora Global DB       │ Read Local,          │ ACID
  (financial,         │ (primary Region only)  │ Write Global         │ transactions,
  inventory)          │                        │                      │ exact counts
  ────────────────────┼────────────────────────┼──────────────────────┼─────────
  User-uploaded       │ S3 Bidirectional       │ Read Local,          │ No conflicts;
  images (by          │ CRR                    │ Write Partitioned    │ each Region
  user region)        │                        │                      │ owns its keys
  ────────────────────┼────────────────────────┼──────────────────────┼─────────
  Frequently          │ ElastiCache Global     │ Read Local,          │ Sub-ms reads
  accessed product    │ Datastore              │ Write Global         │ in every
  data (cache)        │                        │                      │ Region
  ────────────────────┼────────────────────────┼──────────────────────┼─────────
```

The critical insight: **you are not choosing one pattern for your entire application**. You are choosing the right pattern for each data type based on its consistency requirements, write patterns, and query complexity.

---

## Part 4: Multi-Region Routing Patterns -- Getting Users to the Right Region

You have studied Route53, Global Accelerator, and CloudFront individually. Here is how they compose for multi-region routing:

```
MULTI-REGION ROUTING STACK
════════════════════════════════════════════════════════════════════════════

  LAYER 1: CDN (CloudFront)
  ─────────────────────────
  Static assets, cacheable API responses.
  CloudFront origin failover groups: if primary origin
  (us-east-1 ALB) returns 5xx, automatically try secondary
  origin (us-west-2 ALB). No DNS change needed.

  LAYER 2: Network (Global Accelerator) — or — DNS (Route53)
  ──────────────────────────────────────────────────────────
  Dynamic traffic that cannot be cached.

  Option A: Global Accelerator
  ─ Two static Anycast IPs → nearest edge → AWS backbone
  ─ Endpoint groups: us-east-1 (weight 100), us-west-2 (weight 100)
  ─ Health checks: if us-east-1 ALB fails, traffic instantly
    reroutes to us-west-2 at the NETWORK layer
  ─ No DNS caching problem (Anycast IPs never change)
  ─ Failover: sub-30 seconds

  Option B: Route53
  ─ Failover routing: PRIMARY → us-east-1 ALB, SECONDARY → us-west-2 ALB
  ─ Health checks: if primary fails, DNS resolves to secondary
  ─ Latency routing: resolves to closest healthy Region
  ─ Failover: 60-180 seconds breakdown:
      Detection: health check interval (10s) × failure threshold (3) = 30s
      DNS propagation: TTL (60s for failover alias records) + client cache
      Total: ~90s typical, up to 180s with aggressive ISP caching
  ─ DNS caching problem: ISPs and browsers may cache old records

  Option C: Both (belt and suspenders)
  ─ Route53 latency routing → Global Accelerator Anycast IPs per Region
  ─ GA handles instant failover; Route53 handles geographic routing
  ─ Most resilient, most expensive

  LAYER 3: Cell Routing (ARC Routing Controls)
  ─────────────────────────────────────────────
  Manual or automated failover using data-plane-only API.
  Route53 health checks can be wired to ARC routing controls.
  Routing controls are data-plane operations -- they work even
  when the Route53 control plane (us-east-1) is unavailable.

  ┌──────────────────────────────────────────────────────────────────┐
  │                        Users                                     │
  │                          │                                       │
  │          ┌───────────────┼───────────────┐                       │
  │          ▼               ▼               ▼                       │
  │    ┌──────────┐   ┌───────────┐   ┌──────────────┐              │
  │    │CloudFront│   │  Route53  │   │   Global     │              │
  │    │  (CDN)   │   │  (DNS)   │   │ Accelerator  │              │
  │    │ static + │   │ failover │   │  (network)   │              │
  │    │ cacheable│   │ or       │   │  Anycast IPs │              │
  │    │ content  │   │ latency  │   │              │              │
  │    └──┬───┬───┘   └────┬─────┘   └──────┬───────┘              │
  │       │   │            │                │                       │
  │       │   │  origin    │                │                       │
  │       │   │  failover  │                │                       │
  │       ▼   ▼            ▼                ▼                       │
  │    ┌───────────────────────────────────────────┐                │
  │    │          ARC Routing Controls              │                │
  │    │  (data-plane switches per cell/region)     │                │
  │    └──────────┬─────────────────┬──────────────┘                │
  │               │                 │                                │
  │               ▼                 ▼                                │
  │    ┌────────────────┐  ┌────────────────┐                       │
  │    │  us-east-1     │  │  us-west-2     │                       │
  │    │  ALB → compute │  │  ALB → compute │                       │
  │    │  → data tier   │  │  → data tier   │                       │
  │    └────────────────┘  └────────────────┘                       │
  └──────────────────────────────────────────────────────────────────┘
```

### Route53 vs Global Accelerator for Failover -- The Critical Comparison

| Dimension | Route53 Failover | Global Accelerator |
|---|---|---|
| **Failover mechanism** | DNS record changes (health check triggers return of secondary IP) | Network-layer rerouting (Anycast IP stays the same, routing changes) |
| **Failover speed** | 60-300 seconds (health check interval + TTL expiration) | Sub-30 seconds (network-layer detection + reroute) |
| **DNS caching problem** | Yes -- ISPs, browsers, OS resolvers may cache old records past TTL | No -- Anycast IPs never change; routing happens at edge |
| **Data plane?** | Health-check-based failover: YES. Manual record edits: NO (control plane) | YES -- all routing decisions are data plane |
| **Static IPs** | No (DNS names resolve to different IPs) | Yes -- two permanent Anycast IPs that never change (useful for IP allowlisting) |
| **Cost** | Per-query pricing (alias queries free) + health check costs | $0.025/hour per accelerator + data transfer premium |
| **Best for** | Cost-sensitive, DNS-native architectures | Latency-sensitive, non-HTTP (TCP/UDP), IP-dependent clients |

---

## Part 5: Dependency and Shared Fate Analysis

### The Analogy

Imagine your multinational corporation has offices in London and New York. Each office has its own staff, its own computers, and its own building. But both offices use the same cloud accounting software hosted by a vendor in New York. When that vendor has an outage, **both offices lose their accounting capability** -- even though the London office is otherwise fully operational. The vendor is a **shared fate dependency**. Your multi-region architecture is only as resilient as its most concentrated dependency.

### Identifying and Eliminating Shared Fate

```
SHARED FATE ANALYSIS -- WHAT BREAKS YOUR MULTI-REGION PROMISE
════════════════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────────────┐
  │  DEPENDENCY TYPE          │  EXAMPLE                   │  MITIGATION
  ├───────────────────────────┼────────────────────────────┼────────────
  │                           │                            │
  │  AWS Control Plane        │  Route53 management API    │  Use data-plane
  │  (us-east-1)              │  lives in us-east-1.       │  operations only
  │                           │  Cannot edit records if    │  for failover.
  │                           │  us-east-1 is down.        │  Pre-configure
  │                           │                            │  failover routing.
  │                           │                            │
  │  Single-Region Write      │  Aurora Global DB has      │  Accept this for
  │  Master                   │  one writer. If that       │  relational data.
  │                           │  Region fails, you must    │  Use DynamoDB
  │                           │  promote secondary         │  Global Tables for
  │                           │  (RTO > 0).                │  data that needs
  │                           │                            │  instant write
  │                           │                            │  failover.
  │                           │                            │
  │  Third-Party Services     │  Payment gateway (Stripe)  │  Cache tokens,
  │                           │  in single region.         │  queue writes,
  │                           │  Auth provider (Auth0)     │  use multi-region
  │                           │  in single region.         │  providers, or
  │                           │                            │  build graceful
  │                           │                            │  degradation.
  │                           │                            │
  │  Cross-Region Sync        │  Microservice A in         │  Async messaging
  │  Calls                    │  us-east-1 calls           │  (SQS, EventBridge)
  │                           │  Microservice B in         │  instead of sync
  │                           │  us-west-2 synchronously.  │  HTTP. Or co-locate
  │                           │  If network latency        │  tightly-coupled
  │                           │  spikes or link fails,     │  services in the
  │                           │  both services fail.       │  same Region.
  │                           │                            │
  │  Shared Data Store        │  Both Regions read config  │  Replicate config
  │                           │  from a single S3 bucket   │  to both Regions.
  │                           │  in us-east-1.             │  Use S3 CRR or
  │                           │                            │  store in DynamoDB
  │                           │                            │  Global Tables.
  │                           │                            │
  │  Monitoring in            │  CloudWatch dashboards     │  Monitor FROM
  │  Failed Region            │  and alarms in us-east-1   │  OUTSIDE the
  │                           │  cannot alert you that     │  Region. Use
  │                           │  us-east-1 is down.        │  Route53 health
  │                           │                            │  checks (global),
  │                           │                            │  external synthetics
  │                           │                            │  (Datadog, etc.),
  │                           │                            │  cross-region
  │                           │                            │  CloudWatch.
  │                           │                            │
  └───────────────────────────┴────────────────────────────┴────────────
```

### The Cross-Region Synchronous Call Anti-Pattern

This deserves special emphasis because it is the most common mistake in multi-region design. If Microservice A in us-east-1 makes a synchronous HTTP call to Microservice B in us-west-2 as part of serving a user request, you have **coupled the availability of both Regions**. If the cross-region network link degrades (latency spikes from 70ms to 2000ms), Microservice A's response time degrades proportionally. You have not gained availability -- you have created a system that is **less available** than a single-region deployment because it now depends on cross-region network health.

**The rule**: Within a Region, synchronous calls between services are fine. **Across Regions, all communication should be asynchronous** -- replicated databases, SQS queues, EventBridge event buses, S3 CRR, or DynamoDB Global Tables. Each Region should be able to serve requests using only local resources.

---

## Part 6: CAP Theorem Applied to AWS Multi-Region Services

### CAP Refresher

The CAP theorem states that a distributed system can provide at most two of three guarantees during a network partition:
- **Consistency (C)**: Every read receives the most recent write
- **Availability (A)**: Every request receives a response (not an error)
- **Partition tolerance (P)**: The system continues operating despite network partitions between nodes

Since network partitions between AWS Regions are **inevitable** (P is not optional), the real choice is between **CP** (consistent but may reject requests during partitions) and **AP** (available but may serve stale data during partitions).

### PACELC -- The Practical Extension

CAP only describes behavior during partitions. But even when the network is healthy, there is a trade-off between **Latency** and **Consistency**. PACELC says:

- **P**artition? → Choose **A**vailability or **C**onsistency
- **E**lse (no partition)? → Choose **L**atency or **C**onsistency

This explains why DynamoDB Global Tables with MREC has sub-second replication lag even in normal operation -- it chooses latency over consistency (PA/EL). It does not wait for all Regions to confirm a write before acknowledging it to the client.

### AWS Services on the CAP/PACELC Spectrum

```
AWS MULTI-REGION SERVICES ON THE CAP/PACELC SPECTRUM
════════════════════════════════════════════════════════════════════════════

  CP ◀────────────────────────────────────────────────────────▶ AP
  (Consistent, may reject)                    (Available, may be stale)

  ┌──────────────────┐
  │  DynamoDB Global  │
  │  Tables (MRSC)    │
  │                   │
  │  CAP: CP          │  Quorum across 3 Regions. Writes are not
  │  PACELC: PC/EC    │  acknowledged until majority confirms.
  │                   │  Higher write latency. May reject writes
  │  Strong           │  if quorum is unreachable.
  │  consistency      │
  │  across Regions   │  *** 3 full replicas OR 2 replicas + 1
  │                   │  witness (non-readable quorum participant
  │                   │  managed by DynamoDB). ***
  │                   │  *** Region-set constraint: US (Virginia/
  │                   │  Ohio/Oregon), EU (Ireland/London/Paris/
  │                   │  Frankfurt), AP (Tokyo/Seoul/Osaka).
  │                   │  Cannot mix sets (no US+EU). ***
  └──────────────────┘
           │
           │
  ┌──────────────────┐
  │  Aurora Global    │
  │  Database         │
  │                   │
  │  CAP: CP (writes) │  Single writer in primary Region.
  │       AP (reads)  │  Reads in secondary are eventually
  │  PACELC: PC/EL   │  consistent (sub-1s lag).
  │                   │  Writes from secondary use write
  │  Strong writes,   │  forwarding (cross-region latency).
  │  eventual reads   │  If primary fails, secondary must be
  │                   │  promoted (RTO > 0).
  └──────────────────┘
           │
           │
  ┌──────────────────┐
  │  ElastiCache      │
  │  Global Datastore │
  │                   │
  │  CAP: CP (writer)  │  Single primary Region for writes.
  │       AP (reads)  │  Read replicas in secondary Regions
  │  PACELC: PC/EL   │  serve eventually consistent data.
  │                   │  Like Aurora Global DB, this is a
  │  Single-writer    │  single-writer system -- CAP/PACELC
  │  with read        │  maps imperfectly since secondaries
  │  replicas         │  cannot accept writes at all.
  └──────────────────┘
           │
           │
  ┌──────────────────┐
  │  DynamoDB Global  │
  │  Tables (MREC)    │
  │                   │
  │  CAP: AP          │  Every Region accepts reads and writes.
  │  PACELC: PA/EL    │  Last-writer-wins by timestamp.
  │                   │  Eventually consistent across Regions
  │  Full             │  (sub-second replication in normal
  │  availability,    │  operation). During partitions, both
  │  eventual         │  Regions continue serving traffic with
  │  consistency      │  local data.
  └──────────────────┘
           │
           │
  ┌──────────────────┐
  │  S3 CRR           │
  │                   │
  │  CAP: AP          │  Asynchronous replication. Objects are
  │  PACELC: PA/EL    │  eventually consistent across Regions
  │                   │  (replication typically within minutes,
  │  Eventually       │  15-min SLA with RTC). Each Region can
  │  consistent       │  read/write independently.
  │  object storage   │
  └──────────────────┘
```

**The practical takeaway**: When someone asks "can I have strong consistency across Regions on AWS?", the answer is DynamoDB Global Tables with MRSC -- but it requires 3 participants (3 full replicas or 2 replicas + 1 witness) within the same region set (US, EU, or AP -- you cannot mix), and it imposes quorum latency on every write. For everything else, cross-region consistency is eventual. This is a fundamental physics constraint, not an AWS limitation.

---

## Part 7: Region Selection Criteria

The priority order for selecting AWS Regions:

```
REGION SELECTION PRIORITY
════════════════════════════════════════════════════════════════════════════

  1. COMPLIANCE (non-negotiable)
  ─────────────────────────────
  ─ Data residency laws override everything
  ─ GDPR: EU customer data in EU Regions
  ─ HIPAA: BAA signed for specific Regions
  ─ Government: GovCloud or specific Regions only
  ─ Financial services: often jurisdiction-specific
  ─ If compliance says "eu-west-1 only", discussion is over

  2. LATENCY (user experience)
  ────────────────────────────
  ─ Place Regions near your user populations
  ─ US users: us-east-1, us-west-2
  ─ EU users: eu-west-1, eu-central-1
  ─ APAC users: ap-southeast-1, ap-northeast-1
  ─ Use CloudFront/Global Accelerator for edge caching, but
    database latency depends on Region proximity
  ─ Cross-region latency: ~70ms US East ↔ US West,
    ~80-120ms US ↔ EU, ~150-200ms US ↔ APAC

  3. COST (varies by Region)
  ──────────────────────────
  ─ Pricing differs significantly between Regions
  ─ US Regions (us-east-1, us-west-2): generally cheapest
  ─ South America (sa-east-1): up to 50% more expensive
  ─ Asia Pacific: varies (ap-south-1 Mumbai cheaper, ap-northeast-1 Tokyo pricier)
  ─ For multi-region, cost doubles (or more) -- choose Regions wisely
  ─ Data transfer between Regions: $0.02/GB (adds up fast with replication)

  4. SERVICE AVAILABILITY (not all services in all Regions)
  ──────────────────────────────────────────────────────────
  ─ New services launch in us-east-1 first, then expand
  ─ Some services (Bedrock, specific SageMaker features) may not
    be available in your compliance-required Region
  ─ Check: https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/
  ─ For multi-region: both Regions must support all services you need
```

### Common Region Pairs

| Primary | Secondary | Use Case |
|---------|-----------|----------|
| us-east-1 (N. Virginia) | us-west-2 (Oregon) | US-focused, geographically separated, both full-service |
| eu-west-1 (Ireland) | eu-central-1 (Frankfurt) | EU-focused, GDPR compliant, good latency between them |
| ap-northeast-1 (Tokyo) | ap-southeast-1 (Singapore) | APAC-focused, covers Japan and Southeast Asia |
| us-east-1 | eu-west-1 | Global coverage, US + EU, highest latency between pair (~80-120ms) |

---

## Part 8: Operational Patterns for Multi-Region

### Sequential Deployment -- Never Deploy to All Regions at Once

```
SEQUENTIAL DEPLOYMENT -- THE MOST IMPORTANT OPERATIONAL PATTERN
════════════════════════════════════════════════════════════════════════════

  The #1 cause of multi-region outages is not infrastructure failure --
  it is a bad deployment pushed to all Regions simultaneously.

  BAD:                                      GOOD:
  ─────                                     ─────
  Deploy v2.1 ──┬──▶ us-east-1             Deploy v2.1 ──▶ us-east-1
                ├──▶ us-west-2                              │
                └──▶ eu-west-1                    Monitor 30 min
                                                  (error rates,
  If v2.1 has a bug, ALL                          latency, 5xx)
  Regions are broken.                                │
  No healthy Region to                    If healthy ──▶ us-west-2
  fail over to.                                         │
                                                  Monitor 30 min
                                                        │
                                              If healthy ──▶ eu-west-1

                                            If v2.1 has a bug, it breaks
                                            only us-east-1. Traffic fails
                                            over to healthy us-west-2.
                                            Rollback us-east-1. 
                                            No customer-facing outage.
```

This pattern -- called **canary Region deployment** or **regional wave deployment** -- is how Amazon itself deploys. (Amazon's internal "one-box" terminology refers to deploying to a single host within a Region first; the Region-level staging is the "regional wave.") The first Region acts as a canary. If errors spike, deployment halts. The remaining Regions continue serving traffic on the previous stable version.

### Regional Observability -- Monitor FROM Outside the Region

If your monitoring, alerting, and dashboards live in us-east-1, and us-east-1 goes down, you have no visibility into the outage. You learn about it from Twitter, not your dashboards.

**The pattern**: Monitor each Region from outside that Region.

| Monitoring Type | Where to Run | Why |
|---|---|---|
| **Route53 health checks** | Global (AWS-managed from multiple locations worldwide) | Detects regional failures from outside the Region |
| **Synthetic canaries** | In a different Region AND external (Datadog, Pingdom) | Tests user-facing endpoints from outside |
| **CloudWatch cross-region dashboards** | In a "monitoring" Region different from production | Aggregates metrics from all production Regions |
| **CloudWatch cross-account observability** | Central monitoring account | Metrics, logs, and traces from all Regions in one place |
| **ARC readiness checks** | Global (AWS-managed) | Continuously validates that each Region is ready for failover |

### Failover Decision: Automated vs Manual

| Approach | Mechanism | When to Use |
|---|---|---|
| **Fully automated** | Route53 health checks + failover routing; Global Accelerator health-based rerouting | Clear failure signals (health check fails), well-tested failover path |
| **Semi-automated** | ARC routing controls triggered by human decision, but execution is automated | Ambiguous failures (degraded but not dead), failover has business implications |
| **Manual** | Oncall engineer executes runbook step-by-step | First time implementing, untested failover path, or regulatory requirement for human approval |

**The maturity progression**: Manual runbooks → SSM Automation → Step Functions orchestration → Fully automated with ARC routing controls + FIS validation.

### Game Days -- Practicing Failure

A multi-region architecture that has never been tested under failure conditions is a hypothesis, not a design. Use AWS Fault Injection Service (FIS) to run controlled chaos experiments:

1. **AZ failure simulation**: Block traffic to one AZ, verify ALB routes around it
2. **Region connectivity disruption**: Inject latency on cross-region links, verify replication lag handling
3. **Full Region failover drill**: Use ARC Region Switch to fail over all traffic, validate application health, fail back
4. **Dependency failure**: Block access to a third-party service, verify graceful degradation
5. **Bad deployment simulation**: Deploy a broken version to one Region, verify sequential deployment catches it

Schedule these quarterly at minimum. Amazon runs them continuously.

### ARC Readiness Checks -- Continuous Validation

ARC routing controls handle failover (Part 4). But ARC also provides **readiness checks** that continuously validate your secondary Region is actually ready to receive traffic. Readiness checks monitor:

- Resource configuration parity (security groups, IAM roles, Auto Scaling settings match)
- Capacity sufficiency (service quotas, instance availability)
- DNS and routing configuration correctness

This directly addresses Anti-Pattern #2 ("Schrodinger's DR") -- instead of discovering during a failover that your secondary Region's security groups are wrong, ARC readiness checks alert you proactively. Pair readiness checks with **zonal shift** (AZ-level traffic evacuation for AZ impairments without full Region failover) for a layered resilience approach.

---

## Part 8.5: Often-Overlooked Multi-Region Dependencies

### Cell-Based Architecture -- Blast Radius Containment

The TL;DR table introduced the hotel chain analogy for cell-based architecture. Here is the architectural detail.

A **cell** is an independent, isolated copy of your application stack that serves a subset of customers. Unlike simply running in multiple Regions (where all users in a Region share one stack), cell-based architecture partitions users across multiple cells, potentially within the same Region or across Regions.

```
CELL-BASED ARCHITECTURE -- BLAST RADIUS CONTAINMENT
════════════════════════════════════════════════════════════════════════════

  Without cells:                    With cells:
  ──────────────                    ───────────
  Region 1: ALL users               Region 1, Cell A: Users A-M
  If bad deployment breaks           Region 1, Cell B: Users N-Z
  Region 1, ALL users affected.     Region 2, Cell C: Users A-M (DR)
                                    Region 2, Cell D: Users N-Z (DR)

                                    Bad deployment to Cell A:
                                    only users A-M in Region 1 affected.
                                    Users N-Z unaffected. DR cells unaffected.
                                    Sequential deployment catches it before
                                    Cell B gets the update.
```

**Key cell-based design decisions:**
- **Cell sizing**: Small enough that a cell failure affects a tolerable % of users, large enough to be operationally manageable (typically 5-10 cells)
- **Customer-to-cell assignment**: Hash-based (deterministic, sticky) or lookup-based (flexible, requires a routing table)
- **Cell independence**: Each cell has its own compute, data store, and cache. No shared state between cells. The only shared component is the routing layer (ARC routing controls)
- **Deployment**: Sequential across cells, not just across Regions. Deploy to one cell, monitor, then the next

Cell-based architecture is central to the [AWS Multi-Region Fundamentals guide](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-multi-region-fundamentals/introduction.html) and is a frequent SAP exam topic.

### Multi-Region KMS Keys

Encrypted resources (Aurora, EBS snapshots, S3 objects) in a secondary Region require KMS keys in that Region. If your Aurora Global Database primary in us-east-1 is encrypted with a single-region KMS key, the secondary cluster in eu-west-1 uses a **different KMS key** in eu-west-1. This works transparently -- but if you need the same key material in both Regions (for cross-region snapshot copies, for example), use **multi-region KMS keys** (`mrk-*` prefix):

- Same key material replicated across Regions by AWS
- Encrypt in one Region, decrypt in another without cross-region API calls
- Essential for Aurora Global Database, cross-region EBS snapshot copies, and S3 CRR with SSE-KMS
- Key policy and grants must be configured independently in each Region

**Gotcha**: If us-east-1 is down and your KMS key is single-region in us-east-1, you cannot decrypt resources encrypted with that key -- even in another Region. Multi-region KMS keys eliminate this dependency.

### Secrets Manager Multi-Region Replication

Your application in eu-west-1 needs database credentials, API keys, and feature flags. If those secrets live only in us-east-1's Secrets Manager, the secondary Region cannot start or rotate credentials during a failover.

**AWS Secrets Manager multi-region secrets** replicate a secret to designated Regions:
- Primary secret in us-east-1, replica in eu-west-1
- Replicas are read-only (rotation happens on the primary, replicates automatically)
- Applications in each Region reference the same secret name -- Secrets Manager returns the local replica
- If the primary Region fails, you can promote a replica to a standalone secret

Similarly, replicate **SSM Parameter Store** parameters using a custom solution (Lambda + EventBridge) or store shared configuration in DynamoDB Global Tables for automatic replication.

### EventBridge Global Endpoints

For event-driven architectures, **EventBridge global endpoints** enable active-active event processing across Regions:

- A global endpoint automatically routes events to a primary Region
- If the primary Region's event bus becomes unhealthy (monitored by Route53 health check), events automatically route to the secondary Region
- Managed replication keeps events flowing to both Regions during normal operation
- No application code changes -- the global endpoint DNS name abstracts the Region routing

This complements the async messaging pattern from Part 5: instead of sending events to a Region-specific event bus (shared fate), send them to a global endpoint that handles Region-level failover automatically.

---

## Part 9: Multi-Region Anti-Patterns

### 1. Relying on Control Plane for Failover

**The mistake**: "When us-east-1 fails, our runbook says to edit the Route53 record to point to us-west-2."

**Why it fails**: Route53 record editing is a control-plane operation. The Route53 control plane lives in us-east-1. If us-east-1 is the Region that failed, you may not be able to edit records.

**The fix**: Pre-configure Route53 failover routing with health checks (data plane). Or use ARC routing controls (data plane). Or use Global Accelerator (data plane).

### 2. Untested Failover ("Schrodinger's DR")

**The mistake**: "We have a secondary Region with Aurora Global Database and pre-provisioned instances. We are multi-region."

**Why it fails**: Have you ever failed over to it? Are the security groups correct? Do the IAM roles exist? Are the service quotas sufficient? Is the application configuration (environment variables, secrets, feature flags) correct in the secondary Region? Is the database schema up to date? If you have never tested, your failover plan exists in a superposition of working and broken until you observe it.

**The fix**: Quarterly game days with FIS. Automated readiness checks with ARC. Continuous validation that both Regions can serve traffic.

### 3. Partial Failover Creating Untested Configurations

**The mistake**: Failing over individual components independently. Microservice A runs in us-east-1, Microservice B fails over to us-west-2, but they communicate synchronously.

**Why it fails**: You have created a **split-brain configuration** where some services run in Region A and others in Region B. This configuration has never been tested and introduces cross-region latency into synchronous call paths. As discussed in yesterday's DR doc, this is why mature organizations evolve toward dependency-graph or portfolio-level failover -- fail over the minimum set of related applications together, or fail over everything.

**The fix**: Fail over at the application or portfolio level, not the component level. If Service A depends on Service B, they fail over together.

### 4. Cross-Region Synchronous Dependencies

Covered in Part 5. If you call across Regions synchronously, you have coupled their availability. All cross-region communication must be asynchronous.

### 5. Monitoring in the Failed Region

Covered in Part 8. Monitor from outside. If your dashboards are in the failed Region, you are blind.

### 6. Same Deployment to All Regions Simultaneously

Covered in Part 8. Sequential deployment with bake time between Regions. A bad deployment to all Regions simultaneously is the most common cause of multi-region outages.

---

## Part 10: Production Multi-Region Active-Active Architecture

```
PRODUCTION MULTI-REGION ACTIVE-ACTIVE -- E-COMMERCE PLATFORM
════════════════════════════════════════════════════════════════════════════

                              Users (Global)
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              ┌──────────┐  ┌──────────┐  ┌──────────────┐
              │CloudFront │  │ Route53  │  │   Global     │
              │ (static  │  │ latency  │  │ Accelerator  │
              │  assets) │  │ routing  │  │ (dynamic     │
              │          │  │          │  │  TCP/UDP)    │
              │ Origin   │  │    ┌─────┘  │              │
              │ failover │  │    │        └──────┬───────┘
              │ group    │  │    │               │
              └────┬─────┘  │    │               │
                   │        │    │               │
         ┌─────────┴────────┴────┴───────────────┴──────────┐
         │              ARC Routing Controls                  │
         │         (data-plane traffic switches)              │
         └─────────┬─────────────────────────┬──────────────┘
                   │                         │
    ┌──────────────▼─────────────┐ ┌─────────▼──────────────────┐
    │      us-east-1 (CELL A)    │ │      eu-west-1 (CELL B)    │
    │                            │ │                             │
    │  ALB (active, serving)     │ │  ALB (active, serving)     │
    │    │                       │ │    │                        │
    │    ▼                       │ │    ▼                        │
    │  ECS Fargate (10 tasks)    │ │  ECS Fargate (10 tasks)    │
    │    │                       │ │    │                        │
    │    ├──▶ Aurora Global DB   │ │    ├──▶ Aurora Global DB    │
    │    │    PRIMARY (writer)   │ │    │    SECONDARY (reader)  │
    │    │    + write forwarding │◀┼────┤    writes forwarded    │
    │    │                       │ │    │    to us-east-1        │
    │    │                       │ │    │                        │
    │    ├──▶ DynamoDB Global    │ │    ├──▶ DynamoDB Global     │
    │    │    Table (read+write) │◀┼───▶│    Table (read+write)  │
    │    │    MREC: LWW          │ │    │    MREC: LWW           │
    │    │                       │ │    │                        │
    │    ├──▶ ElastiCache Global │ │    ├──▶ ElastiCache Global  │
    │    │    Datastore PRIMARY  │─┼───▶│    Datastore SECONDARY │
    │    │    (read+write)       │ │    │    (read-only)         │
    │    │                       │ │    │                        │
    │    └──▶ S3 (user uploads)  │ │    └──▶ S3 (user uploads)  │
    │         Bidirectional CRR  │◀┼───▶│    Bidirectional CRR   │
    │         prefix: us/*       │ │    │    prefix: eu/*        │
    │                            │ │                             │
    │  Monitoring:               │ │  Monitoring:                │
    │  ─ Route53 health checks   │ │  ─ Route53 health checks   │
    │    (global, checks eu-west)│ │    (global, checks us-east) │
    │  ─ Synthetics in eu-west-1 │ │  ─ Synthetics in us-east-1 │
    │  ─ CloudWatch x-region     │ │  ─ CloudWatch x-region     │
    │                            │ │                             │
    └────────────────────────────┘ └─────────────────────────────┘

  DATA CONSISTENCY MODEL:
  ─ Product catalog (Aurora): Read local, write global. EU reads are
    sub-1s stale. EU writes are forwarded to us-east-1 (adds ~80ms).
  ─ Shopping cart, sessions (DynamoDB MREC): Read local, write local.
    Truly active-active. LWW resolves rare conflicts.
  ─ Cache (ElastiCache): Read local, write to us-east-1 primary only.
    EU cache reads are sub-1s stale.
  ─ User uploads (S3 CRR): Write partitioned by user region.
    EU users write to eu-west-1 bucket. US users write to us-east-1 bucket.
    Bidirectional CRR ensures both Regions can read all uploads.

  FAILOVER SCENARIO (us-east-1 goes down):
  1. Route53 health checks detect us-east-1 ALB failure (60-90s)
  2. Global Accelerator detects endpoint failure (sub-30s) -- faster
  3. All traffic routes to eu-west-1
  4. Aurora: promote eu-west-1 secondary to independent writer (~1-2 min)
  5. DynamoDB: no action needed (already read-write in eu-west-1)
  6. ElastiCache: promote eu-west-1 secondary to primary
  7. S3: US user uploads now go to eu-west-1 bucket (app routing change)
  8. eu-west-1 ECS absorbs full load. For true static stability,
     pre-provision 20 tasks (full capacity) in both Regions so no
     control-plane scaling call is needed during failover. If cost
     is a concern, accept the trade-off that Auto Scaling (a control-
     plane operation) must succeed to reach full capacity.
```

### Terraform: Multi-Region Active-Active Core Infrastructure

```hcl
# ═══════════════════════════════════════════════════════════════════════
# MULTI-REGION ACTIVE-ACTIVE -- CORE INFRASTRUCTURE
# ═══════════════════════════════════════════════════════════════════════
# This builds on the Warm Standby Terraform from the DR Strategies doc.
# The key difference: both Regions are full production scale and serve
# live traffic simultaneously.

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
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

# ─── GLOBAL ACCELERATOR (network-layer failover, static Anycast IPs) ────

resource "aws_globalaccelerator_accelerator" "main" {
  name            = "ecommerce-global"
  ip_address_type = "IPV4"
  enabled         = true

  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = aws_s3_bucket.ga_logs.id
    flow_logs_s3_prefix = "global-accelerator/"
  }
}

resource "aws_globalaccelerator_listener" "https" {
  accelerator_arn = aws_globalaccelerator_accelerator.main.id
  protocol        = "TCP"

  port_range {
    from_port = 443
    to_port   = 443
  }
}

# Endpoint group: us-east-1 (full production traffic)
resource "aws_globalaccelerator_endpoint_group" "us_east_1" {
  listener_arn          = aws_globalaccelerator_listener.https.id
  endpoint_group_region = "us-east-1"
  traffic_dial_percentage = 100  # Set to 0 during failover/maintenance

  health_check_port             = 443
  health_check_protocol         = "HTTPS"
  health_check_path             = "/health"
  health_check_interval_seconds = 10
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.us_east_1.arn
    weight                         = 100
    client_ip_preservation_enabled = true  # ALB sees real client IP
  }
}

# Endpoint group: eu-west-1 (full production traffic)
# NOTE: No provider override needed -- GA API lives exclusively in us-west-2.
# The endpoint_group_region attribute tells GA which Region the endpoints are in.
resource "aws_globalaccelerator_endpoint_group" "eu_west_1" {
  listener_arn          = aws_globalaccelerator_listener.https.id
  endpoint_group_region = "eu-west-1"
  traffic_dial_percentage = 100

  health_check_port             = 443
  health_check_protocol         = "HTTPS"
  health_check_path             = "/health"
  health_check_interval_seconds = 10
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.eu_west_1.arn
    weight                         = 100
    client_ip_preservation_enabled = true
  }
}

# ─── ROUTE53 LATENCY-BASED ROUTING (DNS-layer, complements GA) ──────────

resource "aws_route53_record" "app_us" {
  zone_id = var.hosted_zone_id
  name    = "app.example.com"
  type    = "A"

  latency_routing_policy {
    region = "us-east-1"
  }

  set_identifier  = "us-east-1"
  health_check_id = aws_route53_health_check.us_east_1.id

  alias {
    name                   = aws_lb.us_east_1.dns_name
    zone_id                = aws_lb.us_east_1.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_eu" {
  zone_id = var.hosted_zone_id
  name    = "app.example.com"
  type    = "A"

  latency_routing_policy {
    region = "eu-west-1"
  }

  set_identifier  = "eu-west-1"
  health_check_id = aws_route53_health_check.eu_west_1.id

  alias {
    name                   = aws_lb.eu_west_1.dns_name
    zone_id                = aws_lb.eu_west_1.zone_id
    evaluate_target_health = true
  }
}

# Health checks -- GLOBAL (not Region-specific, monitors from worldwide)
resource "aws_route53_health_check" "us_east_1" {
  fqdn              = aws_lb.us_east_1.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  tags = { Name = "us-east-1-health" }
}

resource "aws_route53_health_check" "eu_west_1" {
  fqdn              = aws_lb.eu_west_1.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  tags = { Name = "eu-west-1-health" }
}

# ─── AURORA GLOBAL DATABASE (Read Local, Write Global) ───────────────────

resource "aws_rds_global_cluster" "ecommerce" {
  global_cluster_identifier = "ecommerce-global"
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"
  database_name             = "ecommerce"
  storage_encrypted         = true
}

# Primary cluster: us-east-1 (WRITER)
resource "aws_rds_cluster" "primary" {
  provider = aws.us_east_1

  cluster_identifier        = "ecommerce-us-east-1"
  global_cluster_identifier = aws_rds_global_cluster.ecommerce.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"
  master_username           = "admin"
  master_password           = var.db_password
  db_subnet_group_name      = aws_db_subnet_group.us_east_1.name
  vpc_security_group_ids    = [aws_security_group.aurora_us.id]

  backup_retention_period = 35  # Maximum PITR
  deletion_protection     = true
}

resource "aws_rds_cluster_instance" "primary_writer" {
  provider           = aws.us_east_1
  identifier         = "ecommerce-us-writer"
  cluster_identifier = aws_rds_cluster.primary.id
  instance_class     = "db.r6g.2xlarge"
  engine             = "aurora-postgresql"
}

resource "aws_rds_cluster_instance" "primary_readers" {
  provider = aws.us_east_1
  count    = 2

  identifier         = "ecommerce-us-reader-${count.index}"
  cluster_identifier = aws_rds_cluster.primary.id
  instance_class     = "db.r6g.xlarge"
  engine             = "aurora-postgresql"
}

# Secondary cluster: eu-west-1 (READER + write forwarding)
resource "aws_rds_cluster" "secondary" {
  provider = aws.eu_west_1

  cluster_identifier        = "ecommerce-eu-west-1"
  global_cluster_identifier = aws_rds_global_cluster.ecommerce.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"
  db_subnet_group_name      = aws_db_subnet_group.eu_west_1.name
  vpc_security_group_ids    = [aws_security_group.aurora_eu.id]

  # Write forwarding goes on the SECONDARY -- it tells this cluster to
  # forward writes to the global cluster's primary (us-east-1)
  enable_global_write_forwarding = true

  backup_retention_period = 35  # Independent PITR in secondary Region
  deletion_protection     = true

  depends_on = [aws_rds_cluster.primary]
}

resource "aws_rds_cluster_instance" "secondary_readers" {
  provider = aws.eu_west_1
  count    = 2  # Full production scale (active-active, not warm standby)

  identifier         = "ecommerce-eu-reader-${count.index}"
  cluster_identifier = aws_rds_cluster.secondary.id
  instance_class     = "db.r6g.xlarge"  # Same size as primary readers
  engine             = "aurora-postgresql"
  promotion_tier     = count.index  # Reader 0 promoted first on failover
}

# ─── DYNAMODB GLOBAL TABLE (Read Local, Write Local, MREC) ──────────────

resource "aws_dynamodb_table" "sessions" {
  provider = aws.us_east_1

  name         = "user-sessions"
  billing_mode = "PAY_PER_REQUEST"  # On-demand for unpredictable session traffic
  hash_key     = "session_id"

  attribute {
    name = "session_id"
    type = "S"
  }

  # TTL for automatic session cleanup
  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }

  # Enable PITR (data corruption replicates -- backups are your safety net)
  point_in_time_recovery {
    enabled = true
  }

  # Global Table replica in eu-west-1
  replica {
    region_name = "eu-west-1"
    # PITR enabled on replica too
    point_in_time_recovery {
      enabled = true
    }
  }

  # Server-side encryption with AWS-owned key (simplest for Global Tables)
  server_side_encryption {
    enabled = true
  }
}

# ─── S3 BIDIRECTIONAL CRR (Write Partitioned) ───────────────────────────

resource "aws_s3_bucket" "uploads_us" {
  provider = aws.us_east_1
  bucket   = "ecommerce-uploads-us-east-1"
}

resource "aws_s3_bucket_versioning" "uploads_us" {
  provider = aws.us_east_1
  bucket   = aws_s3_bucket.uploads_us.id
  versioning_configuration {
    status = "Enabled"  # Required for CRR
  }
}

resource "aws_s3_bucket" "uploads_eu" {
  provider = aws.eu_west_1
  bucket   = "ecommerce-uploads-eu-west-1"
}

resource "aws_s3_bucket_versioning" "uploads_eu" {
  provider = aws.eu_west_1
  bucket   = aws_s3_bucket.uploads_eu.id
  versioning_configuration {
    status = "Enabled"
  }
}

# CRR: us-east-1 → eu-west-1
resource "aws_s3_bucket_replication_configuration" "us_to_eu" {
  provider = aws.us_east_1
  bucket   = aws_s3_bucket.uploads_us.id
  role     = aws_iam_role.replication_us.arn

  rule {
    id     = "replicate-to-eu"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.uploads_eu.arn
      storage_class = "STANDARD"
    }

    # Enable replica modification sync for bidirectional
    source_selection_criteria {
      replica_modifications {
        status = "Enabled"
      }
    }

    # Delete marker replication for true bidirectional
    delete_marker_replication {
      status = "Enabled"
    }
  }
}

# CRR: eu-west-1 → us-east-1 (bidirectional)
resource "aws_s3_bucket_replication_configuration" "eu_to_us" {
  provider = aws.eu_west_1
  bucket   = aws_s3_bucket.uploads_eu.id
  role     = aws_iam_role.replication_eu.arn

  rule {
    id     = "replicate-to-us"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.uploads_us.arn
      storage_class = "STANDARD"
    }

    source_selection_criteria {
      replica_modifications {
        status = "Enabled"
      }
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

---

## Part 10.5: Migration Strategy -- Getting to Multi-Region Without Breaking Everything

If you are starting from a single-region deployment, do **not** attempt to jump directly to Active-Active. A phased approach delivers incremental resilience while building operational muscle.

```
PHASED MULTI-REGION MIGRATION
════════════════════════════════════════════════════════════════════════════

  Phase 1: SINGLE-REGION HARDENING (Weeks 1-4)
  ─────────────────────────────────────────────
  "If a workload cannot survive the loss of a single AZ,
   replicating it to another Region just replicates its vulnerabilities."

  ─ Enable Multi-AZ for all data stores (RDS, ElastiCache)
  ─ Spread compute across 3 AZs
  ─ Convert ALL infrastructure to IaC (StackSets/Terraform)
    (You cannot deploy to a second Region if infra is manual)
  ─ Set up cross-Region monitoring in the future secondary Region
  ─ No app changes, all traffic stays in primary Region

  Phase 2: READ-ONLY SECONDARY (Weeks 5-8)
  ─────────────────────────────────────────
  Lowest-risk multi-Region step: read traffic only.

  ─ Migrate RDS → Aurora Global Database (read replicas in secondary)
  ─ ElastiCache Global Datastore (read replica in secondary)
  ─ Deploy read-only app servers in secondary Region
  ─ Route53 latency-based routing for reads
  ─ CloudFront origin failover (primary → secondary)
  ─ Replicate supporting services: Secrets Manager, ECR images

  Immediate value: users near secondary Region get lower latency.

  Phase 3: WRITE FAILOVER -- ACTIVE-PASSIVE (Weeks 9-14)
  ──────────────────────────────────────────────────────
  Enable the secondary Region to take over writes if primary fails.

  ─ Deploy full async pipeline (SQS + workers) in secondary
  ─ KMS multi-Region keys for encrypted resources
  ─ Pre-provision IAM roles, ALBs, full compute (static stability)
  ─ Route53 health checks + ARC routing controls (data plane only)
  ─ Aurora write forwarding for transparent write path
  ─ Sequential deployment (deploy one Region, monitor, then next)
  ─ Test full failover → verify → failback cycle

  Phase 4: VALIDATION (Weeks 15-18)
  ──────────────────────────────────
  An untested multi-Region architecture is a hypothesis.

  ─ FIS experiments simulating Region-level failures
  ─ ARC readiness checks for continuous validation
  ─ Unannounced GameDay drills
  ─ Documented runbooks; team can execute without the architect
  ─ Measure actual: Aurora promotion time, ARC shift time,
    replication lag under peak, DNS propagation

  Phase 5 (FUTURE): ACTIVE-ACTIVE (Months 6-9)
  ──────────────────────────────────────────────
  Only after Phase 4 is proven in production.

  ─ Add DynamoDB Global Tables for write-local data types
  ─ S3 bidirectional CRR for user-generated content
  ─ Full production compute in both Regions
  ─ Graduate from Active-Passive to Active-Active incrementally
```

**The key insight**: shipping a tested Active-Passive in 4.5 months beats shipping an untested Active-Active in 3 months. Phase 2 delivers visible customer value (latency improvement) quickly, and each subsequent phase adds resilience without risking what is already working.

---

## Part 11: Cost Framework -- Where the Money Goes

Multi-region architecture roughly doubles your infrastructure cost, but not all components cost equally. Understanding the cost breakdown helps optimize.

```
MULTI-REGION COST MULTIPLIERS
════════════════════════════════════════════════════════════════════════════

  Component              │ Active-Passive Cost │ Active-Active Cost  │ Notes
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  Compute (ECS/EKS/EC2)  │ 20-50% (scaled down)│ 100% (full scale)   │ Biggest
                         │                     │                     │ line item
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  Aurora Global DB        │ ~60% (readers only, │ ~80% (readers,      │ Storage
  (secondary cluster)     │  smaller instances) │  full-size)         │ is shared
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  DynamoDB Global Tables  │ Per-replicated write│ Per-replicated write│ On-demand
  (replica)              │ (1.5x base write)   │ (1.5x base write)   │ scales
                         │                     │                     │ with load
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  ElastiCache Global     │ Node cost in DR     │ Node cost in        │ Same
  Datastore              │ Region              │ secondary           │ node count
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  Load Balancers         │ Hourly + LCU (min)  │ Hourly + LCU (full) │ Fixed
  (ALB/NLB)              │                     │                     │ cost even
                         │                     │                     │ idle
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  NAT Gateways           │ $0.045/hr + DT      │ $0.045/hr + DT      │ Per AZ,
                         │                     │                     │ adds up
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  Data Transfer           │ Replication DT only │ Replication DT +    │ $0.02/GB
  (cross-region)         │                     │ user traffic DT     │ inter-region
  ───────────────────────┼─────────────────────┼─────────────────────┼──────
  Global Accelerator     │ $0.025/hr + DT      │ $0.025/hr + DT      │ Fixed cost
                         │ premium             │ premium (both dirs) │ per accel
  ───────────────────────┼─────────────────────┼─────────────────────┼──────

  ROUGH TOTALS:
  ─ Active-Passive (Warm Standby): +25-50% of primary Region cost
  ─ Active-Active: +80-120% of primary Region cost
  ─ But: Active-Active also serves traffic from both Regions,
    improving latency for users near the secondary Region.
    The cost is part DR investment, part UX investment.

  WHERE TO SAVE:
  ─ Use Graviton instances (20% cheaper, same performance)
  ─ Use Aurora Serverless v2 readers in secondary Region (scale to
    minimum 0.5 ACU when traffic is low; auto-pause to zero requires
    explicit configuration)
  ─ Use DynamoDB on-demand (pay only for replicated writes, no
    provisioned capacity waste)
  ─ Use Reserved Instances / Savings Plans for baseline capacity
  ─ Minimize cross-region data transfer (cache aggressively per Region)
  ─ Use S3 Intelligent-Tiering for replicated objects
```

---

## Critical Exam Gotchas and Interview Traps

**1. "Route53 control plane lives in us-east-1; data plane is global."**
You cannot edit DNS records if us-east-1 is down. But Route53 will continue answering DNS queries globally. Pre-configure health-check-based failover routing so the data plane handles failover automatically. This is the #1 multi-region exam question.

**2. "Failover must use data-plane operations, not control-plane operations."**
Launching EC2 instances, scaling ASGs, creating load balancers, editing DNS records -- all control plane. Pre-provisioned instances, ARC routing controls, health-check-triggered failover -- all data plane. Control-plane operations may be unavailable during the exact outage that triggers your failover.

**3. "DynamoDB Global Tables is AP (MREC) or CP (MRSC). Aurora Global Database is CP for writes, AP for reads."**
When asked "which service provides strong consistency across Regions?", the answer is DynamoDB Global Tables with MRSC -- but only across exactly 3 Regions with higher write latency. Aurora Global Database provides strong consistency only in the primary Region; secondary Regions have eventual consistency with sub-1s lag.

**4. "Sequential deployment is the most important operational pattern."**
A bad deployment pushed to all Regions simultaneously is the #1 cause of multi-region outages -- worse than infrastructure failures. Deploy to one Region, monitor, then deploy to the next.

**5. "Multi-region can decrease availability if done wrong."**
Adding a second Region adds complexity: more deployment targets, more monitoring, more dependencies, more failure modes. If you do not eliminate cross-region synchronous dependencies, do not test failover regularly, and do not deploy sequentially, the added complexity **reduces** availability rather than improving it.

**6. "Cross-region synchronous calls defeat the purpose of multi-region."**
If Service A in Region 1 synchronously calls Service B in Region 2, the system availability is bounded by cross-region network availability. Each Region must be able to serve requests using only local resources.

**7. "Even Active-Active needs backups and PITR."**
Data corruption replicates instantly across all Regions via Global Tables, Aurora Global Database, and S3 CRR. Replication is not backup. PITR is your safety net against corruption and accidental deletion.

**8. "Monitor FROM outside the Region, not inside it."**
If your dashboards, alarms, and synthetics live in the Region that failed, you have no visibility. Use Route53 health checks (global), external synthetic monitors, and cross-region CloudWatch.

**9. "Service quotas must be pre-increased in the secondary Region."**
If your primary Region has quota for 500 EC2 instances and your secondary Region has the default 20, failover will fail when Auto Scaling tries to launch instances beyond quota.

**10. "ARC routing controls are data-plane operations."**
This is the key differentiator. Route53 record editing is control plane. ARC routing controls are data plane. During a major outage, ARC routing controls remain available for manual or automated failover even if the Route53 management API is degraded.

---

## Key Takeaways

- **Data plane vs control plane is the #1 multi-region principle.** Every failover mechanism you design must be evaluated: "Does this depend on a control-plane operation?" If yes, redesign. Pre-configure health-check-based failover, use ARC routing controls, and pre-provision resources. The Route53 control plane lives in us-east-1 -- if that Region fails, you cannot edit DNS records, but Route53 will continue answering queries using the data plane.

- **Static stability means pre-provisioning everything.** Your system should continue operating without making any changes, even if every control-plane API is unavailable. Resources are already running, DNS records are already configured for failover, and health checks are already evaluating. This is the foundation of Warm Standby and Active-Active.

- **Different data types in the same application use different replication patterns.** Relational data with transactions goes through Aurora Global Database (write global). Session data and shopping carts go through DynamoDB Global Tables (write local, MREC). User uploads go through S3 bidirectional CRR (write partitioned). Cached data goes through ElastiCache Global Datastore (write global). Choose per data type, not per application.

- **Active-Active vs Active-Passive is primarily driven by write conflict tolerance.** If your data model cannot tolerate eventual consistency and last-writer-wins conflicts, use Active-Passive with Aurora Global Database. If it can, use Active-Active with DynamoDB Global Tables. Most production systems use both patterns simultaneously for different data.

- **Sequential deployment prevents the #1 cause of multi-region outages.** Never deploy to all Regions simultaneously. Deploy to one Region, monitor for errors, then deploy to the next. A bad deployment to all Regions is worse than a Region failure because there is no healthy Region to fail over to.

- **Eliminate cross-region synchronous dependencies.** Every synchronous call between Regions couples their availability. All cross-region communication must be asynchronous (replicated databases, message queues, event buses). Each Region must serve requests using only local resources.

- **Monitor from outside the Region being monitored.** Route53 health checks (global), external synthetic monitors, and cross-region CloudWatch dashboards. If your monitoring lives in the failed Region, you learn about outages from customers, not dashboards.

- **Test failover regularly with FIS and game days.** An untested multi-region architecture is a hypothesis. Quarterly game days with AWS Fault Injection Service validate that security groups, IAM roles, service quotas, KMS keys, and application configuration are all correct in the secondary Region.

- **Multi-region roughly doubles cost, but the framing matters.** Active-Active is not purely a DR cost -- it also serves traffic from both Regions, improving latency for users near the secondary. Optimize with Graviton instances, Aurora Serverless v2 readers, DynamoDB on-demand, and aggressive per-region caching.

- **On the CAP spectrum, DynamoDB MRSC is the only strongly consistent cross-region option on AWS.** It requires 3 participants (3 full replicas or 2 replicas + 1 witness) within the same region set (US, EU, or AP -- cannot mix), and imposes quorum latency. Everything else (Aurora Global Database secondary reads, ElastiCache Global Datastore, S3 CRR, DynamoDB MREC) is eventually consistent across Regions. This is a physics constraint, not an AWS limitation.

---

## Further Reading

- [AWS Prescriptive Guidance -- Multi-Region Fundamentals](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-multi-region-fundamentals/introduction.html) -- The single most comprehensive official guide to multi-region architecture on AWS
- [Well-Architected Reliability Pillar -- Avoid Control Plane During Recovery (REL11-BP04)](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_withstand_component_failures_avoid_control_plane.html) -- The foundational data-plane-vs-control-plane principle
- [Creating an Organizational Multi-Region Failover Strategy](https://aws.amazon.com/blogs/architecture/creating-an-organizational-multi-region-failover-strategy/) -- Component vs application vs portfolio-level failover
- [Creating a Multi-Region Application (3-Part Series)](https://aws.amazon.com/blogs/architecture/creating-a-multi-region-application-with-aws-services-part-1-compute-and-security/) -- Practical implementation walkthrough
- [AWS CAP Theorem Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/cap-theorem.html) -- AWS's own framing of CAP for distributed systems
- [Region Selection Criteria](https://aws.amazon.com/blogs/architecture/what-to-consider-when-selecting-a-region-for-your-workloads/) -- Compliance > Latency > Cost > Service Availability
- [Disaster Recovery Strategies (Mar 30 deep dive)](2026-03-30-disaster-recovery-strategies.md) -- The four DR strategies, RPO/RTO, Active-Active write strategies, failover automation
- [Aurora Global Database (Mar 25 deep dive)](2026-03-25-rds-aurora-deep-dive.md) -- Managed switchover, unplanned failover, write forwarding
- [DynamoDB Global Tables (Mar 26 deep dive)](2026-03-26-dynamodb-deep-dive.md) -- MREC last-writer-wins, MRSC strong consistency
- [ElastiCache Global Datastore (Mar 27 deep dive)](2026-03-27-elasticache-caching-strategies.md) -- Single-writer cross-region replication
- [Route53 Routing Policies (Mar 11 deep dive)](2026-03-11-route53-routing-policies.md) -- Failover, latency, geolocation routing, health checks
- [Global Accelerator (Mar 18 deep dive)](2026-03-18-global-accelerator-edge-services.md) -- Anycast IPs, endpoint groups, traffic dials
- [S3 CRR (Mar 23 deep dive)](2026-03-23-s3-advanced-deep-dive.md) -- Bidirectional replication, RTC
