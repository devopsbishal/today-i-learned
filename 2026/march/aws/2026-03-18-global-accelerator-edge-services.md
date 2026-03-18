# AWS Global Accelerator & Edge Services -- The Global Express Lane Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront), and installed the regional load balancers that seat guests at the right table (ALB/NLB/GLB). But there is still a gap between Route53 answering "which region should I go to?" and CloudFront answering "how do I cache this HTTP content at the edge?" -- **what happens when you need instant, health-aware failover across regions for TCP and UDP traffic that cannot be cached?** When your gaming platform needs sub-30-second failover without waiting for DNS TTLs to expire, when your VoIP system needs two static IP addresses that never change regardless of which region is healthy, when your IoT fleet of 500,000 devices cannot be reconfigured to point at a new endpoint -- that is the domain of AWS Global Accelerator. It is the express lane onto the AWS backbone network: traffic enters at the nearest edge location and travels on AWS's private fiber instead of bouncing across the unpredictable public internet.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Global Accelerator | A global network of express highway on-ramps that route cars onto a private toll road instead of congested city streets | Provides two static Anycast IPv4 addresses that route TCP/UDP traffic onto the AWS backbone at the nearest edge location, delivering lower latency and instant failover without DNS propagation delays |
| Anycast IP addresses | A single phone number that connects you to the nearest switchboard operator, who transfers you on a private internal line to the right office | Two static IPs announced from all AWS edge locations simultaneously; BGP routing directs each client to the nearest edge, where GA proxies the connection over the AWS backbone to the endpoint region |
| Accelerator | The toll road authority that owns the on-ramps and decides the rules of the road | The top-level Global Accelerator resource that owns the two static Anycast IPs (or four for dual-stack) and a DNS name; all configuration hangs beneath it |
| Listener | The toll booth attendant who checks whether you are a car (TCP) or a truck (UDP) and which highway entrance (port) you are using | Processes inbound connections on a specific protocol (TCP, UDP, or both) and port range, then passes traffic to endpoint groups |
| Endpoint group | A regional highway hub where cars exit the toll road and enter local streets toward their final destination | Associated with exactly one AWS region; contains endpoints, a traffic dial (0-100%), and health check configuration; Global Accelerator routes to the nearest healthy endpoint group |
| Endpoint | The actual building (restaurant, office, warehouse) at the end of the local street | The resource receiving traffic: ALB, NLB, EC2 instance, or Elastic IP for standard accelerators; VPC subnets for custom routing accelerators |
| Traffic dial | A valve on the highway off-ramp that controls what percentage of traffic can exit at this hub | A 0-100% setting per endpoint group that controls what proportion of traffic routes to that region; set to 0% to drain a region during maintenance, use intermediate values for blue-green deployments |
| Endpoint weight | A numbered ticket system at the destination building that determines how many visitors each entrance handles | A 0-255 value per endpoint (default 128) that controls proportional distribution within an endpoint group; weight 0 stops all traffic to that endpoint |
| Client affinity | A VIP pass that ensures a repeat visitor always goes to the same building entrance | NONE (default): 5-tuple hash distributes connections across endpoints for maximum distribution. SOURCE_IP: 2-tuple hash (source IP + destination IP) ensures all connections from one client reach the same endpoint |
| Standard accelerator | The normal express highway that uses smart signs to route cars to the least congested exit | Routes and load-balances traffic across endpoints based on health, proximity, and weights; supports ALB, NLB, EC2, and EIP endpoints |
| Custom routing accelerator | A private highway where your dispatch center tells each car exactly which exit and parking spot to use | Creates a deterministic, static mapping between GA listener ports and specific EC2 private IPs + ports; your application logic decides the routing, not GA's algorithms; ideal for gaming matchmaking and VoIP |
| Network zones | Two separate highway systems (North Highway and South Highway) that both reach the same destinations, so if one highway is blocked, cars take the other | Each edge location has two isolated network zones, each announcing one of the two static IPs; if one zone experiences infrastructure failure, clients retry on the second IP via the independent zone |
| DT-Premium (Data Transfer Premium) | The extra toll for the express lane on top of what you already pay for the regular highway -- the base road fee (EC2 data transfer) stays the same, but you pay a premium for the faster, private lane | Per-GB surcharge on top of standard EC2 data transfer, charged only on the dominant direction per hour (inbound or outbound, not both); varies by source-destination pair |

---

## The Big Picture: Why the Public Internet Is Not Good Enough

Think of the public internet as a network of **city streets, county roads, and interstate highways** managed by hundreds of different entities. When a user in Singapore sends a packet to your ALB in us-east-1, that packet hops across 15-25 autonomous systems -- each with its own congestion, routing policies, and failure modes. The path changes unpredictably. Some hops are well-maintained interstates; others are potholed back roads. Latency varies. Packet loss spikes. And when a network in between fails, BGP reconvergence can take minutes while your users stare at loading spinners.

AWS Global Accelerator solves this by building **express highway on-ramps** at 130+ edge locations across 95 cities in 53 countries. Instead of your user's traffic bouncing across the public internet, it enters the nearest on-ramp and travels on the **AWS private global backbone** -- a dedicated fiber network that AWS controls end-to-end. The analogy:

1. **Without Global Accelerator**: Your user's car leaves Singapore, drives through city streets, merges onto various highways managed by different countries' road authorities, hits construction, takes detours, and eventually arrives in Virginia. The trip time is unpredictable, and if a highway segment closes, the car is stuck until alternative routing kicks in.

2. **With Global Accelerator**: Your user's car drives to the nearest express on-ramp (Singapore edge location), shows their pass (connects to the Anycast IP), and enters a private toll road (AWS backbone) that goes directly to Virginia. The toll road has no congestion, no construction surprises, and if the Virginia exit is closed, the toll road authority instantly reroutes the car to the Oregon exit -- without the car needing to re-enter the highway.

The two critical advantages are **performance** (the AWS backbone is faster and more consistent than the public internet, improving throughput and first-byte latency by up to 60% per AWS benchmarks (real-world results vary by geography and traffic pattern)) and **instant failover** (Global Accelerator detects endpoint health failures and reroutes within seconds, not minutes, because it does not depend on DNS TTL expiration like Route53 failover does).

```
TRAFFIC PATH: PUBLIC INTERNET vs GLOBAL ACCELERATOR
══════════════════════════════════════════════════════════════════════════════

WITHOUT GLOBAL ACCELERATOR (public internet path):
┌───────────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌───────────┐
│  User in  │───▶│ISP  │───▶│IX   │───▶│ISP  │───▶│ISP  │───▶│  ALB in   │
│ Singapore │    │(SG) │    │(HK) │    │(US) │    │(VA) │    │ us-east-1 │
└───────────┘    └─────┘    └─────┘    └─────┘    └─────┘    └───────────┘
                  15-25 hops across autonomous systems
                  Latency: 200-350ms (variable)
                  Failover: DNS TTL-dependent (60-300 seconds)


WITH GLOBAL ACCELERATOR (AWS backbone path):
┌───────────┐    ┌─────┐    ┌──────────────────────────────┐    ┌───────────┐
│  User in  │───▶│ISP  │───▶│  AWS Edge    AWS Private     │───▶│  ALB in   │
│ Singapore │    │(SG) │    │  Location    Backbone         │    │ us-east-1 │
└───────────┘    └─────┘    │  (SG)       (AWS fiber)      │    └───────────┘
                  1-3 hops  └──────────────────────────────┘
                  to edge    Controlled, consistent path
                             Latency: 140-250ms (stable)
                             Failover: <30 seconds (health-based)
```

### Connection Proxy at the Edge

A detail that matters for understanding GA's behavior: Global Accelerator **terminates TCP connections at the edge location** and establishes **new TCP connections** to your endpoints over the backbone. This **two-leg proxy model** (client→edge leg + edge→endpoint leg) is the same pattern ALB uses (as you learned on [Mar 16](2026-03-16-alb-vs-nlb-vs-glb.md)), but operating at the global edge level rather than at the regional level. Because GA owns both legs of the connection, it can reroute the backend leg to a different endpoint without the client knowing -- this is the technical foundation for instant failover, client IP preservation behavior, and idle timeout enforcement. The consequence is that your endpoint sees the edge location's IP as the source, not the original client IP -- but GA provides **client IP preservation** that varies by endpoint type:

| Endpoint Type | Client IP Preservation | Mechanism | Notes |
|--------------|----------------------|-----------|-------|
| **ALB** | Enabled by default | `X-Forwarded-For` HTTP header | Same header ALB already uses; your app reads it as normal |
| **EC2 instance** | Always on | Preserved in the IP packet header (source IP = original client) | No configuration needed; GA rewrites the source IP natively |
| **NLB (with security groups)** | Supported | Preserved in the IP packet header | NLB must have security groups enabled; without them, client IP is **not** preserved |
| **Elastic IP** | **Not supported** | N/A | The endpoint always sees the edge location's IP as the source |

This is different from NLB's Proxy Protocol mechanism -- GA has a **native client IP preservation feature** that works at the IP layer for EC2 and NLB endpoints, and via HTTP headers for ALB endpoints.

For UDP, there is no connection to terminate. GA simply forwards datagrams from the edge to the endpoint over the backbone.

**Idle timeouts**: TCP connections idle for 340 seconds are closed. UDP "connections" (tracked by 5-tuple) idle for 30 seconds are dropped. These are not configurable. **Important**: TCP keep-alive packets do **not** reset the 340-second idle timer -- only application-layer data does. If your application relies on TCP keep-alives to maintain long-lived connections, you must send actual data at least every 340 seconds or the connection will be dropped.

---

## Part 1: The Four-Level Hierarchy -- Accelerator, Listener, Endpoint Group, Endpoint

Global Accelerator's resource model has four levels, and understanding this hierarchy is essential for both configuration and interview questions. If you already know the ALB hierarchy (ALB -> Listener -> Listener Rules -> Target Group -> Targets), the GA hierarchy maps directly -- but at a global rather than regional scope.

```
GLOBAL ACCELERATOR COMPONENT HIERARCHY
══════════════════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────────────────┐
  │  ACCELERATOR                                                        │
  │  "The Toll Road Authority"                                          │
  │                                                                      │
  │  Owns: 2 static Anycast IPv4 addresses (4 for dual-stack)          │
  │  DNS:  a1234567890abcdef.awsglobalaccelerator.com                   │
  │  Also: Enabled/Disabled toggle, IP address type (IPv4/dual-stack)   │
  ├──────────────────────────────────────────────────────────────────────┤
  │                                                                      │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │  LISTENER (one or more per accelerator)                       │  │
  │  │  "Toll Booth Attendant"                                       │  │
  │  │                                                                │  │
  │  │  Protocol: TCP, UDP, or TCP+UDP                               │  │
  │  │  Ports: single port or port range (1-65535)                   │  │
  │  │  Client affinity: NONE or SOURCE_IP                           │  │
  │  ├────────────────────────────────────────────────────────────────┤  │
  │  │                                                                │  │
  │  │  ┌──────────────────────────────────────────────────────────┐  │  │
  │  │  │  ENDPOINT GROUP (one per AWS region per listener)        │  │  │
  │  │  │  "Regional Highway Hub"                                  │  │  │
  │  │  │                                                          │  │  │
  │  │  │  Region: us-east-1 (exactly one region per group)        │  │  │
  │  │  │  Traffic dial: 0-100% (default 100%)                     │  │  │
  │  │  │  Health check port/path/protocol/interval/threshold      │  │  │
  │  │  ├──────────────────────────────────────────────────────────┤  │  │
  │  │  │                                                          │  │  │
  │  │  │  ┌────────────────────────────────────────────────────┐  │  │  │
  │  │  │  │  ENDPOINTS (one or more per group)                │  │  │  │
  │  │  │  │  "The Destination Buildings"                       │  │  │  │
  │  │  │  │                                                    │  │  │  │
  │  │  │  │  Types: ALB, NLB, EC2 instance, Elastic IP        │  │  │  │
  │  │  │  │  Weight: 0-255 (default 128)                      │  │  │  │
  │  │  │  │  Client IP preservation: varies by type (see below) │  │  │  │
  │  │  │  └────────────────────────────────────────────────────┘  │  │  │
  │  │  └──────────────────────────────────────────────────────────┘  │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────┘
```

### Accelerator -- The Toll Road Authority

The accelerator is the top-level resource. When you create one, AWS assigns:
- **Two static Anycast IPv4 addresses** from the Amazon IP address pool (or you can bring your own via BYOIP -- must be a /24 or shorter prefix from ARIN, RIPE, or APNIC with ROA records created; BYOIP addresses cannot be used for other AWS services simultaneously)
- **A DNS name** like `a1234567890abcdef.awsglobalaccelerator.com` -- you can create a Route53 alias record pointing your custom domain to this DNS name (free, works at zone apex, as you learned on [Mar 11](2026-03-11-route53-routing-policies.md))
- **Two network zones** -- each of the two IPs is announced from an independent infrastructure zone at every edge location. If one zone's infrastructure fails at a given edge, clients fail over to the second IP in the independent zone

You can enable or disable an accelerator without deleting it. Disabling stops traffic flow but preserves the configuration and IP addresses. **Note**: Disabled accelerators still incur the $0.025/hour fixed fee -- you pay for IP reservation even when no traffic flows.

**Cross-account endpoints**: GA supports endpoints (ALBs, NLBs) in different AWS accounts than the accelerator. This is useful in multi-account architectures (as you designed with [Organizations](2026-03-02-aws-organizations-scps.md)) where the accelerator lives in a networking account and endpoints live in workload accounts.

### Listener -- The Toll Booth Attendant

A listener accepts inbound connections on a protocol and port. The configuration is minimal:

- **Protocol**: TCP, UDP, or both
- **Port range**: A single port (80) or a range (1000-2000). You can specify multiple ranges per listener.
- **Client affinity**: Controls how connections from the same client are distributed

**Client affinity** has two modes:

| Mode | Hash Input | Behavior | Use Case |
|------|-----------|----------|----------|
| **NONE** (default) | 5-tuple (src IP, src port, dst IP, dst port, protocol) | Each connection is independently distributed across endpoints | Stateless services, maximum distribution |
| **SOURCE_IP** | 2-tuple (src IP, dst IP) | All connections from the same source IP go to the same endpoint | Stateful services, session affinity, gaming |

**Key nuance**: Client affinity is configured as a **listener-level setting**, but the consistent routing it provides happens at the endpoint level within a group. When using SOURCE_IP affinity, GA first selects the nearest healthy endpoint group (region), then consistently routes all connections from that client to the same endpoint within that group. If that endpoint fails, GA reroutes to another healthy endpoint in the group (unlike ALB sticky sessions, which can lose stickiness on target failure).

### Endpoint Group -- The Regional Highway Hub

Each endpoint group is associated with **exactly one AWS region**. A listener can have multiple endpoint groups (one per region), and GA routes traffic to the nearest healthy group. This is the level where you control:

- **Traffic dial (0-100%)**: The percentage of traffic routed to this endpoint group. Set to 100% (default) for normal operation. Reduce to 0% to drain a region entirely. Use intermediate values for blue-green deployments across regions -- this is like Route53 weighted routing ([Mar 11](2026-03-11-route53-routing-policies.md)) but without the DNS TTL delay.

- **Health check settings**: GA performs health checks on endpoints in the group. For ALB and NLB endpoints, GA uses the target's own health status by default. For EC2 and EIP endpoints, GA performs its own health checks (configurable protocol, port, path, interval, threshold).

**Failover behavior**: When all endpoints in an endpoint group become unhealthy, GA fails over to the **next closest healthy endpoint group** in another region. This is the critical advantage over Route53 failover -- there is no DNS TTL to wait for. GA detects the failure within seconds and reroutes at the network level.

**Fail-open**: If there is only one endpoint group and all its endpoints are unhealthy, GA routes traffic to all endpoints anyway (fail-open), same behavior as ALB/NLB fail-open ([Mar 16](2026-03-16-alb-vs-nlb-vs-glb.md)).

### Endpoint -- The Destination Building

The actual resource receiving traffic. For standard accelerators:

| Endpoint Type | Notes |
|--------------|-------|
| **ALB** | Can be internet-facing or internal. GA health-checks via the ALB's own target health. Most common pattern for HTTP workloads. |
| **NLB** | Internet-facing or internal. Provides Layer 4 regional load balancing behind GA's global routing. Common for non-HTTP protocols. |
| **EC2 instance** | Must be in a subnet in the endpoint group's region. GA health-checks the instance directly. |
| **Elastic IP** | Routes to whatever is associated with the EIP. Useful for legacy architectures or appliances with static IPs. |

**Endpoint weight (0-255)**: Controls proportional distribution within a single endpoint group. Default is 128. If you have two endpoints with weights 128 and 128, each gets 50%. Weights 192 and 64 give a 75/25 split. Weight 0 stops all traffic to that endpoint (useful for draining a specific endpoint while keeping the region active).

---

## Part 2: Traffic Dials vs Endpoint Weights -- Two Layers of Traffic Control

These two mechanisms operate at different levels and serve different purposes. Confusing them is a common interview mistake.

**The analogy**: Imagine a highway system where traffic from the Northeast can exit at either the Virginia hub or the Oregon hub.

- **Traffic dial** is the off-ramp capacity at each hub. If Virginia's traffic dial is set to 80% and Oregon's is 100%, then only 80% of traffic that would naturally route to Virginia actually exits there -- the other 20% continues on the backbone to the next closest hub (Oregon). The traffic dial controls **how much traffic enters a region**.

- **Endpoint weight** is the parking lot capacity at each building within a hub. If Virginia has two ALBs with weights 192 and 64, the Virginia hub sends 75% of its traffic to ALB-1 and 25% to ALB-2. Endpoint weight controls **how traffic is distributed within a region**.

```
TRAFFIC DIALS vs ENDPOINT WEIGHTS
══════════════════════════════════════════════════════════════════════════════

  Global traffic ──▶ GA Anycast IP ──▶ Nearest edge location

  ┌─────────────────────────────────────────────────────────────────────┐
  │                    ENDPOINT GROUP: us-east-1                        │
  │                    Traffic Dial: 80%                                │
  │                                                                     │
  │  ──▶ 80% of traffic that would naturally route here enters ──▶     │
  │       ┌─────────────┐  weight 192 (75%)  ┌─────────────────┐      │
  │       │   ALB-1     │◀───────────────────│                 │      │
  │       └─────────────┘                    │  Distribution   │      │
  │       ┌─────────────┐  weight 64  (25%)  │  within group   │      │
  │       │   ALB-2     │◀───────────────────│                 │      │
  │       └─────────────┘                    └─────────────────┘      │
  │                                                                     │
  │  ──▶ 20% of traffic spills to next closest healthy group ──▶      │
  └─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                    ENDPOINT GROUP: eu-west-1                        │
  │                    Traffic Dial: 100%                               │
  │                                                                     │
  │  ──▶ Receives its natural traffic + overflow from us-east-1 ──▶   │
  │       ┌─────────────┐  weight 128 (50%)  ┌─────────────────┐      │
  │       │   ALB-3     │◀───────────────────│                 │      │
  │       └─────────────┘                    │  Distribution   │      │
  │       ┌─────────────┐  weight 128 (50%)  │  within group   │      │
  │       │   ALB-4     │◀───────────────────│                 │      │
  │       └─────────────┘                    └─────────────────┘      │
  └─────────────────────────────────────────────────────────────────────┘
```

**Blue-green deployment across regions**: To migrate traffic from us-east-1 to eu-west-1, gradually reduce the us-east-1 traffic dial from 100% to 0% in steps (100 -> 80 -> 50 -> 20 -> 0). Traffic spills to the next closest healthy region. This achieves the same outcome as Route53 weighted routing but takes effect immediately -- no DNS TTL to wait for.

**Canary within a region**: To canary a new ALB within us-east-1, set the new ALB's endpoint weight to 10 and the existing ALB's weight to 245. The new ALB receives ~4% of regional traffic. Gradually increase the weight as confidence grows.

---

## Part 3: Standard vs Custom Routing Accelerators

### Standard Accelerators -- The Normal Express Highway

Standard accelerators are what most people mean when they say "Global Accelerator." They load-balance traffic across endpoints using health, proximity, traffic dials, and weights. GA makes the routing decision -- your application just registers endpoints and configures weights.

**When to use**: Multi-region HTTP APIs behind ALBs, multi-region TCP services behind NLBs, any workload where GA should decide which endpoint receives each connection based on health and proximity.

### Custom Routing Accelerators -- The Dispatch-Controlled Highway

Custom routing accelerators are a fundamentally different model. Instead of GA deciding where traffic goes, **your application decides**. GA creates a deterministic, static mapping between listener ports and specific EC2 private IP addresses + ports within VPC subnets. Your application retrieves this mapping via the GA API and tells clients exactly which GA port to connect to.

**The analogy**: Imagine a large apartment building where each unit has a unique buzzer code. Standard accelerators are like a concierge who looks at your name and sends you to the best available unit. Custom routing is like the building publishing a directory: "Buzzer 5001 = Unit 3A, Buzzer 5002 = Unit 3B, Buzzer 5003 = Unit 4A." Your dispatch service (matchmaking server, session border controller) tells each visitor which buzzer to press, and they go directly to the right unit.

```
CUSTOM ROUTING: DETERMINISTIC PORT-TO-INSTANCE MAPPING
══════════════════════════════════════════════════════════════════════════════

  MATCHMAKING SERVER (your application logic)
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. Calls GA API: ListCustomRoutingPortMappings                │
  │  2. Receives mapping table:                                     │
  │     GA Port 10001 ──▶ 10.0.1.5:8080 (game-server-A)           │
  │     GA Port 10002 ──▶ 10.0.1.6:8080 (game-server-B)           │
  │     GA Port 10003 ──▶ 10.0.2.5:8080 (game-server-C)           │
  │  3. Player requests match → matchmaker picks game-server-B     │
  │  4. Tells player: "Connect to GA-IP:10002"                     │
  └──────────────────────────────────┬──────────────────────────────┘
                                     │
  PLAYER CLIENT                      │
  ┌──────────────┐                   │
  │ Connects to  │                   │
  │ GA-IP:10002  │                   │
  └──────┬───────┘                   │
         │                           │
         ▼                           │
  ┌─────────────────────┐            │
  │ AWS Edge Location   │            │
  │ (Anycast routing)   │            │
  └──────┬──────────────┘            │
         │ AWS backbone              │
         ▼                           │
  ┌─────────────────────┐            │
  │ 10.0.1.6:8080       │  ◀─── deterministic mapping, not load-balanced
  │ (game-server-B)     │
  └─────────────────────┘
```

**Critical constraints of custom routing**:

| Feature | Standard Accelerator | Custom Routing Accelerator |
|---------|---------------------|---------------------------|
| Endpoints | ALB, NLB, EC2, EIP | **VPC subnets only** |
| Routing decision | GA (health + proximity + weights) | **Your application** |
| Health checks | Yes, automatic failover | **No** -- your app owns failover |
| Traffic dials | Yes | No |
| Endpoint weights | Yes | No |
| Default traffic permission | All traffic allowed | **All traffic DENIED** by default |
| Protocol | TCP, UDP | TCP, UDP |

**Deny-by-default**: This is the most important custom routing detail. When you create a custom routing accelerator, all destination mappings are **denied** by default. You must explicitly call `AllowCustomRoutingTraffic` to enable traffic to specific subnets or individual socket-level destinations. You can granularly allow/deny at the subnet level or the individual IP:port level. This gives you precise control -- for example, only allow traffic to a game server after a player has been matched to it.

**Use cases**: Multiplayer gaming (matchmaking assigns players to specific game servers), VoIP (route callers to specific media servers), real-time collaboration (direct users to specific session hosts), any application where a broker/matchmaker needs to route clients to a pre-determined instance.

---

## Part 4: Global Accelerator vs CloudFront vs Route53 -- The Three-Way Comparison

This is the most important section for both architecture decisions and interviews. After studying Route53 ([Mar 11](2026-03-11-route53-routing-policies.md)), CloudFront ([Mar 13](2026-03-13-cloudfront-deep-dive.md)), and now Global Accelerator, you need to understand that these three services solve **different problems at different layers** -- they are not interchangeable alternatives.

**The analogy**:
- **Route53** is the **airport control tower** -- it tells planes (users) which airport (region) to fly to by answering DNS queries. The control tower makes a recommendation, but the plane caches that recommendation (TTL) and flies on its own. If the airport closes, the plane does not know until it asks the tower again after the TTL expires.
- **CloudFront** is the **global newspaper delivery network** -- it places copies of your content at newsstands worldwide so users do not need to fly to the printing press. It only works for HTTP content that can be cached or processed at the edge.
- **Global Accelerator** is the **express highway system** -- it does not store anything and does not make DNS recommendations. It physically reroutes the car (packet) onto a private road at the nearest on-ramp and drives it to the right destination, switching exits instantly if one is closed.

| Dimension | Route53 Latency Routing | CloudFront | Global Accelerator |
|-----------|------------------------|------------|-------------------|
| **OSI layer** | DNS layer (resolution only) | Layer 7 (HTTP content inspection) | Layer 4 (TCP/UDP transport) |
| **What it does** | Returns different DNS answers based on user proximity | Caches and delivers HTTP content from edge locations | Routes TCP/UDP packets onto AWS backbone from nearest edge |
| **Protocols** | DNS (resolves to any protocol) | HTTP/HTTPS, WebSocket | TCP, UDP (any application protocol over these) |
| **Caching** | DNS resolver caches the answer (TTL-controlled) | **Full content caching at 750+ POPs** -- the key differentiator | **No caching whatsoever** -- every packet goes to the origin endpoint |
| **Static IPs** | No (DNS names, IPs can change) | No (uses d123.cloudfront.net DNS) | **Yes -- two fixed Anycast IPv4 addresses that never change** |
| **Failover speed** | DNS TTL-dependent: 60-300 seconds typical | Origin failover per request (no DNS delay) | **Sub-30 seconds** -- health-check-based reroute at the network level |
| **Health checking** | Route53 health checks (10s or 30s intervals) | Origin groups (failover on 5xx) | Built-in health checks with automatic cross-region failover |
| **Edge compute** | No | Lambda@Edge, CloudFront Functions | No |
| **WAF integration** | No (but endpoints behind it can have WAF) | **Yes** -- Web ACL attached to distribution | No (but endpoints behind it can have WAF, and Shield Standard protects GA IPs) |
| **DDoS protection** | Route53 is inherently DDoS-resistant (distributed) | Shield Standard automatic, Shield Advanced optional | **Shield Standard automatic**, Shield Advanced optional |
| **Pricing model** | $0.40-$0.70 per million queries | Per-request + per-GB (no fixed fee) | **$0.025/hour fixed** + DT-Premium per-GB |
| **Monthly baseline** | ~$12 for 20M queries | $0 (pay per use) | **~$18 minimum** (just for having the accelerator) |
| **Best for** | DNS-level routing decisions, cheapest option | HTTP content delivery with caching, edge compute | Non-HTTP protocols, static IPs, instant failover, TCP/UDP acceleration |

### When to Use Which

**Use Route53 latency routing when**: You need the cheapest global routing and can tolerate DNS TTL-based failover delays (60-300 seconds). Good for: web applications where seconds of failover time are acceptable, initial routing decision before CloudFront or GA.

**Use CloudFront when**: Your workload is HTTP/HTTPS and benefits from caching or edge compute. This is the right choice for: static websites, API responses that can be cached, media streaming, any workload where serving from a 750+ POP cache network is the primary value.

**Use Global Accelerator when**: You need any of these -- (1) static IP addresses that never change (IoT fleets, firewall allowlists, partner integrations), (2) instant failover without DNS TTL delays (financial services, gaming, healthcare), (3) non-HTTP protocols (gaming over UDP, VoIP, MQTT for IoT, custom TCP), (4) deterministic routing to specific instances (gaming matchmaking via custom routing).

**Can they work together?** Yes, but with constraints:
- **Route53 + CloudFront**: The most common pairing. Route53 alias record points to CloudFront distribution. Free alias queries, zone apex support.
- **Route53 + Global Accelerator**: Route53 alias record points to GA's DNS name. Provides a friendly domain name for GA's Anycast IPs.
- **CloudFront + Global Accelerator**: They solve different problems. Route different hostnames with Route53 -- HTTP traffic to CloudFront, TCP/UDP real-time traffic to GA. **You cannot put GA in front of CloudFront or vice versa** -- GA routes at Layer 4 (TCP/UDP) and would not know how to handle CloudFront's HTTP protocol requirements, and CloudFront cannot originate to an Anycast IP that is not an HTTP endpoint.

```
ARCHITECTURAL PATTERN: SPLITTING TRAFFIC BY PROTOCOL
══════════════════════════════════════════════════════════════════════════════

  Route53 Hosted Zone: game.example.com
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │  www.game.example.com ──▶ CloudFront Distribution (alias record)       │
  │  (static assets, web UI, HTTP APIs with caching)                       │
  │                                                                         │
  │  play.game.example.com ──▶ Global Accelerator DNS (alias record)       │
  │  (real-time UDP game traffic, matchmaking, voice chat)                 │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘
                │                                    │
                ▼                                    ▼
  ┌──────────────────────┐            ┌──────────────────────────────┐
  │    CloudFront        │            │    Global Accelerator        │
  │    750+ POPs         │            │    2 Anycast IPs             │
  │    Caches content    │            │    No caching                │
  │    Runs edge code    │            │    Instant failover          │
  │    HTTP/HTTPS only   │            │    TCP/UDP                   │
  └──────────┬───────────┘            └──────────────┬───────────────┘
             │                                       │
             ▼                                       ▼
  ┌──────────────────────┐            ┌──────────────────────────────┐
  │  ALB (us-east-1)     │            │  Endpoint Group: us-east-1   │
  │  Web servers / APIs  │            │    NLB ──▶ game servers      │
  └──────────────────────┘            │  Endpoint Group: eu-west-1   │
                                      │    NLB ──▶ game servers      │
                                      └──────────────────────────────┘
```

---

## Part 5: Global Accelerator vs Route53 -- The Failover Speed Deep Dive

Since you studied Route53 health checks and failover routing on [Mar 11](2026-03-11-route53-routing-policies.md), you know that Route53 failover works like this:

1. Health checker detects failure (10s or 30s intervals, configurable threshold -- typically 30-90 seconds to declare unhealthy)
2. Route53 updates DNS answers to exclude the unhealthy endpoint
3. **Clients must wait for their cached DNS TTL to expire** before they ask Route53 again and get the new answer
4. Total failover time: health check detection (30-90s) + DNS TTL (commonly 60s) = **90-150 seconds typical, up to 300+ seconds worst case**

Global Accelerator failover works differently:

1. GA health checker detects failure (configurable, can be as fast as 10 seconds)
2. GA immediately reroutes traffic at the network level to the next closest healthy endpoint group
3. **No DNS TTL is involved** -- the Anycast IPs do not change, so clients keep connecting to the same IPs and GA internally reroutes
4. Total failover time: **under 30 seconds typical**

**The analogy**: Route53 failover is like a GPS app that recalculates your route -- but only if you ask for new directions (DNS query), and you only ask after your cached directions expire (TTL). If you just got directions 59 seconds ago with a 60-second TTL, you drive the old route for another second before asking again. Global Accelerator failover is like a highway system with electronic variable message signs -- the moment an exit closes, the signs immediately redirect all cars to the next exit. No one needs to ask for new directions; the highway physically routes them differently.

This difference is why GA is the preferred failover mechanism for applications with strict RTO requirements -- gaming, financial trading, healthcare systems, real-time communications.

---

## Part 6: Cross-Region Failover Patterns

### Pattern 1: Active-Active Multi-Region with GA

```
  Global Accelerator (2 Anycast IPs)
  ┌─────────────────────────────────────────────────────────────────┐
  │  Listener: TCP 443                                              │
  │                                                                 │
  │  Endpoint Group: us-east-1          Endpoint Group: eu-west-1  │
  │  Traffic Dial: 100%                 Traffic Dial: 100%         │
  │  ┌───────────────────┐              ┌───────────────────┐      │
  │  │  ALB (primary)    │              │  ALB (primary)    │      │
  │  │  Weight: 128      │              │  Weight: 128      │      │
  │  └───────────────────┘              └───────────────────┘      │
  └─────────────────────────────────────────────────────────────────┘

  Behavior:
  - US users ──▶ us-east-1 (nearest healthy group)
  - EU users ──▶ eu-west-1 (nearest healthy group)
  - If us-east-1 ALB fails ──▶ US users reroute to eu-west-1 (<30s)
  - If eu-west-1 ALB fails ──▶ EU users reroute to us-east-1 (<30s)
```

### Pattern 2: Blue-Green Regional Migration with Traffic Dials

```
  Phase 1: 100% us-east-1         Phase 2: 80/20 split
  ┌────────────────────────┐      ┌────────────────────────┐
  │ us-east-1: dial 100%  │      │ us-east-1: dial 80%   │
  │ eu-west-1: dial 100%  │      │ eu-west-1: dial 100%  │
  │                        │      │                        │
  │ All US traffic ──▶ USE │      │ 80% US ──▶ USE        │
  │ All EU traffic ──▶ EUW │      │ 20% US ──▶ EUW        │
  └────────────────────────┘      │ All EU ──▶ EUW         │
                                  └────────────────────────┘

  Phase 3: 50/50 split             Phase 4: Migration complete
  ┌────────────────────────┐      ┌────────────────────────┐
  │ us-east-1: dial 50%   │      │ us-east-1: dial 0%    │
  │ eu-west-1: dial 100%  │      │ eu-west-1: dial 100%  │
  │                        │      │                        │
  │ 50% US ──▶ USE         │      │ All traffic ──▶ EUW    │
  │ 50% US ──▶ EUW         │      └────────────────────────┘
  │ All EU ──▶ EUW         │
  └────────────────────────┘
```

### Pattern 3: GA + Route53 + CloudFront (Full Edge Stack)

For applications that have both HTTP content (cacheable) and real-time protocols (not cacheable):

```
  Route53: example.com
  ├── app.example.com   ──▶ CloudFront ──▶ ALB (HTTP, cached)
  ├── api.example.com   ──▶ Global Accelerator ──▶ ALB (HTTP, non-cacheable, needs static IPs)
  └── game.example.com  ──▶ Global Accelerator ──▶ NLB (UDP, real-time)
```

---

## Part 7: DDoS Protection and Shield Integration

Global Accelerator's Anycast IPs are automatically protected by **AWS Shield Standard** (as you learned on [Mar 10](2026-03-10-aws-network-application-protection.md)). Because traffic enters at AWS edge locations, volumetric DDoS attacks are absorbed at the edge rather than reaching your endpoints. This is the same benefit CloudFront provides, but for TCP/UDP traffic.

For **Shield Advanced**, Global Accelerator is a supported protected resource. With Shield Advanced:
- DRT (DDoS Response Team) access for incident support
- Cost protection credits for scaling charges during attacks
- Enhanced DDoS detection with traffic baselines specific to your accelerator
- CloudWatch metrics for DDoS event visibility

**The analogy**: Your Anycast IPs are like having every on-ramp to your express highway guarded by security. A DDoS attack is a flood of cars trying to overwhelm one on-ramp, but since the same highway entrance exists at 130+ edge locations, the flood is distributed across all of them. No single on-ramp is overwhelmed, and the highway authority (Shield) can block malicious license plates at every on-ramp simultaneously.

Additionally, because GA masks your actual endpoint IPs behind its two static Anycast IPs, attackers cannot easily discover and directly target your ALBs, NLBs, or EC2 instances. The Anycast IPs serve as an **origin protection layer**, same concept as CloudFront hiding your origin behind its distribution domain.

### Monitoring and Flow Logs

GA provides **flow logs** that capture per-connection details: source/destination IPs, ports, bytes transferred, and packet counts. Flow logs are published to Amazon S3 and are essential for debugging connectivity issues, traffic analysis, and compliance audits.

Key **CloudWatch metrics** for GA include:
- `ProcessedBytesIn` / `ProcessedBytesOut` -- traffic volume through the accelerator
- `NewFlowCount` -- new connections per period (useful for detecting traffic spikes or DDoS)
- `HealthyEndpointCount` -- tracks endpoint health across endpoint groups

### Dual-Stack (IPv6) Nuance

When you create a dual-stack accelerator (4 IPs instead of 2), **clients can reach GA via IPv6**, but GA still communicates with endpoints over **IPv4 only**. Your endpoints must support IPv4. This means GA acts as an IPv6-to-IPv4 translation layer at the edge -- useful for supporting IPv6 mobile clients while keeping your backend infrastructure on IPv4.

---

## Part 8: Pricing Model -- Is the Express Lane Worth the Toll?

Global Accelerator is a **premium service** -- it always costs more than routing traffic over the public internet. The question is whether the latency reduction and instant failover justify the premium.

### Two Pricing Components

**1. Fixed hourly fee**: $0.025 per accelerator per hour = **$18/month** running 24/7. This is charged whether or not any traffic flows. You pay for having the on-ramp exist.

**2. Data Transfer Premium (DT-Premium)**: A per-GB surcharge on top of standard EC2 data transfer. The rate depends on the source region (where the user enters the edge) and the destination AWS region (where the endpoint lives). This is charged only on the **dominant traffic direction per hour** (inbound or outbound, whichever is greater in each hour -- not both; the dominant direction can switch between hours).

| AWS Region (source) | Edge Location (destination) | DT-Premium per GB |
|---------------------|---------------------------|-------------------|
| North America | North America | $0.015 |
| North America | Europe | $0.015 |
| North America | Asia Pacific | $0.035 |
| Europe | North America | $0.015 |
| Asia Pacific | North America | $0.035 |
| Australia/NZ | Australia/NZ | $0.007 |
| Australia/NZ | US/Canada | $0.105 |

### Example Calculation

A multi-region API serving 10,000 GB/month, mostly from North American users to us-east-1:
- Fixed fee: $18/month
- DT-Premium: 10,000 GB x $0.015/GB = $150/month
- Standard EC2 data transfer: billed separately as normal
- **Total GA premium: ~$168/month** on top of what you would pay without GA

### Cost Comparison with Alternatives

| Service | Monthly Cost (10K GB, 20M requests) | What You Get |
|---------|--------------------------------------|-------------|
| Route53 latency routing | ~$12 (queries only) | DNS-level routing, TTL-dependent failover |
| CloudFront | ~$850-1,200 (requests + data) | HTTP caching + edge compute + WAF + static content delivery |
| Global Accelerator | ~$168 premium + EC2 DT | TCP/UDP acceleration + instant failover + static IPs |

The pricing makes it clear: GA is not a replacement for CloudFront (which provides caching that reduces origin load and data transfer) or Route53 (which is dramatically cheaper for simple routing). GA is justified when you specifically need instant failover, static IPs, or non-HTTP protocol acceleration -- and the application's business value (gaming revenue, financial transaction speed, healthcare reliability) warrants the premium.

---

## Key Takeaways

- **Global Accelerator provides two static Anycast IPv4 addresses that never change**: These IPs are announced from 130+ edge locations across 95 cities. Traffic enters the AWS backbone at the nearest edge, improving throughput and first-byte latency by up to 60% (per AWS benchmarks) and providing a fixed entry point for IoT devices, firewall allowlists, and partner integrations that cannot handle changing IPs.

- **Failover is sub-30-seconds, not TTL-dependent**: This is the single most important differentiator from Route53 failover. GA detects endpoint health failures and reroutes at the network level without any DNS propagation delay. For applications where 60-300 seconds of Route53 failover time is unacceptable (gaming, finance, healthcare), GA is the answer.

- **Traffic dials control regional traffic, endpoint weights control per-endpoint distribution**: Traffic dials (0-100%) operate at the endpoint group (region) level and determine what percentage of traffic enters a region. Endpoint weights (0-255) operate within a region and distribute traffic proportionally across endpoints. Use dials for blue-green regional migrations, weights for canary deployments within a region.

- **Standard accelerators load-balance; custom routing accelerators let your app decide**: Standard accelerators route traffic based on health, proximity, and weights -- like an intelligent GPS. Custom routing accelerators create a deterministic port-to-instance mapping and your application tells clients which port to use -- like a dispatch system. Custom routing has no health checks, no traffic dials, no weights, and denies all traffic by default.

- **CloudFront is for HTTP caching, GA is for TCP/UDP acceleration -- they are not interchangeable**: The key mental model is "CloudFront = backbone shortcut WITH caching" vs "GA = backbone shortcut WITHOUT caching." If your traffic is HTTP and benefits from caching, CloudFront wins. If your traffic is TCP/UDP, needs static IPs, or needs instant failover, GA wins. For mixed workloads, use both with Route53 splitting traffic by hostname.

- **GA terminates TCP at the edge (proxy model) with per-endpoint-type client IP preservation**: Like ALB, GA terminates the client's TCP connection at the edge location and opens a new connection to the endpoint. Client IP preservation varies: ALB endpoints get it via `X-Forwarded-For` (enabled by default), EC2 endpoints always see the real client IP in the packet header, NLB endpoints require security groups enabled, and EIP endpoints do **not** support client IP preservation at all. UDP is forwarded without termination.

- **Two network zones provide infrastructure-level fault tolerance**: Each of the two Anycast IPs is announced from an independent network zone at every edge location. If one zone's infrastructure fails, clients retry on the second IP. This is why GA assigns two IPs, not one.

- **GA is a premium service -- always costs more than public internet routing**: $18/month fixed fee per accelerator plus a per-GB Data Transfer Premium ($0.015-$0.101/GB depending on geography). Justified for applications where latency reduction and instant failover have measurable business value, not for general-purpose web hosting.

- **Shield Standard protects GA Anycast IPs automatically**: DDoS attacks are absorbed at the edge across 130+ locations. GA's static IPs also serve as origin protection, masking your actual endpoint IPs from attackers. For mission-critical applications, upgrade to Shield Advanced for DRT access, cost protection, and enhanced detection.

- **GA + Route53 is a common pairing**: Create a Route53 alias record (free, zone apex support) pointing your custom domain to GA's DNS name. This gives users a friendly domain while GA handles the actual traffic routing. This is not an either/or choice -- Route53 handles name resolution, GA handles traffic acceleration.

---

## Further Reading

- [How AWS Global Accelerator Works](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html) -- Anycast routing, TCP termination at edge, network zones, connection timeouts
- [Global Accelerator Components](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-components.html) -- Four-level hierarchy reference with configuration details
- [Custom Routing Accelerators](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-custom-routing-how-it-works.html) -- Deterministic port mapping, deny-by-default, AllowCustomRoutingTraffic API
- [Global Accelerator Pricing](https://aws.amazon.com/global-accelerator/pricing/) -- Fixed fee + DT-Premium calculations with regional rate tables
- [Global Accelerator FAQs](https://aws.amazon.com/global-accelerator/faqs/) -- Consolidation resource covering common architecture questions
- Related learning docs: [Route53 Routing Policies & Health Checks](2026-03-11-route53-routing-policies.md), [Route53 Advanced -- Hybrid DNS](2026-03-12-route53-advanced-hybrid-dns.md), [CloudFront Deep Dive](2026-03-13-cloudfront-deep-dive.md), [ALB vs NLB vs GLB](2026-03-16-alb-vs-nlb-vs-glb.md), [Network & Application Protection (Shield)](2026-03-10-aws-network-application-protection.md)
