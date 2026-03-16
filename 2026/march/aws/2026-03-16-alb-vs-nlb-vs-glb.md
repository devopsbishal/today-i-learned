# AWS Load Balancers Deep Dive -- ALB vs NLB vs GLB -- The Restaurant, Highway, and Customs Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), and deployed the global newspaper delivery network (CloudFront). But once traffic arrives at your AWS region -- whether from CloudFront, the internet, or internal services -- **who decides which specific server handles each request?** When a user hits `api.example.com/v2/orders`, which of your 47 backend containers receives that request? When a financial trading system needs sub-millisecond routing decisions with a static IP that never changes, how do you avoid the overhead of HTTP parsing? When you want to insert a Palo Alto firewall transparently into every packet's path without your applications knowing it exists? That is the domain of Elastic Load Balancing -- three distinct load balancer types (ALB, NLB, GLB), each built for a fundamentally different job, and understanding which to choose is one of the most consequential infrastructure decisions you will make.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| ALB (Application Load Balancer) | A restaurant host who reads your reservation, dietary preferences, and party size to seat you at the perfect table | Layer 7 load balancer that inspects HTTP/HTTPS/gRPC content -- headers, paths, query strings, host names -- to make intelligent routing decisions across multiple target groups |
| NLB (Network Load Balancer) | A highway toll booth that glances at your license plate and waves you through in microseconds without opening your trunk | Layer 4 load balancer that routes TCP/UDP/TLS connections using IP and port information without inspecting the payload, delivering ultra-low latency and static IP addresses per AZ |
| GLB (Gateway Load Balancer) | A customs checkpoint that photocopies every package, sends copies to inspectors, and forwards the originals only after clearance | Layer 3 transparent gateway that encapsulates all IP traffic via GENEVE and distributes it to virtual appliance fleets (firewalls, IDS/IPS) for inspection before returning it to the original path |
| Target group | The group of tables (servers) that a host can seat guests at, organized by section (API tables, web tables, VIP tables) | A logical grouping of targets (instances, IPs, Lambda functions, or ALBs) that receives traffic from a load balancer based on listener rules |
| Listener | The restaurant's front door with a posted menu of which host handles which reservation type | A process that checks for connection requests on a specific protocol and port, then forwards traffic to target groups based on rules |
| Listener rules | The host's seating chart: "parties of 8+ go to the private room, vegetarians go to the garden section" | Conditions (path, host header, HTTP method, source IP, query string) paired with actions (forward, redirect, fixed response, authenticate) that determine routing |
| Health checks | A manager who periodically visits each table section to confirm the kitchen can still serve that area | Periodic probes sent to targets to verify they can handle traffic; unhealthy targets stop receiving requests until they recover |
| Cross-zone load balancing | Whether a host can seat guests at tables in a different dining room on another floor | Controls whether LB nodes in one AZ can send traffic to targets in other AZs -- ALB enables it by default (free), NLB and GLB disable it by default (inter-AZ charges apply) |
| Connection draining (deregistration delay) | The host stops seating new guests at a closing section but lets current diners finish their meals | When a target is deregistered, in-flight requests complete within the delay window (default 300s) while new requests go elsewhere |
| Sticky sessions | The host remembers your face and always seats you at your favorite table on return visits | Session affinity via cookies (AWSALB for duration-based, AWSALBAPP for application-based) that routes a client to the same target for the cookie's lifetime |
| GENEVE encapsulation | A customs agent who wraps each package in a tamper-proof evidence bag so inspectors can examine it without altering the original | GLB's tunneling protocol (port 6081) that preserves original packet headers so appliances see real source/destination IPs and can inspect traffic transparently |
| Gateway Load Balancer Endpoint (GWLBe) | A pneumatic tube connecting a building's mailroom to the customs inspection facility next door | A VPC endpoint (powered by PrivateLink) that acts as a route table next-hop, sending traffic from the consumer VPC to the GLB in the appliance VPC and back |
| Slow start | Gradually increasing the number of guests seated at a new table section so the kitchen there can warm up | A ramp-up period (30-900 seconds) where a newly registered target receives a linearly increasing share of traffic instead of the full load immediately |
| NLB-to-ALB target | Putting a highway toll booth in front of the restaurant entrance so guests get both a fast lane with a fixed address and a smart host once inside | Registering an ALB as an NLB target to combine NLB's static IPs and PrivateLink support with ALB's Layer 7 routing |

---

## The Big Picture: Three Load Balancers for Three Different Jobs

The best way to understand AWS's three load balancer types is to recognize that each operates at a different layer of the OSI model and serves a fundamentally different purpose. They are not interchangeable alternatives -- they are specialized tools, and the right choice depends on **what** you need to inspect, **how fast** the decision must be, and **whether** you need transparency.

### The Three Analogies

**ALB is a restaurant host.** When a party arrives, the host does not just wave them at the first empty table. The host reads the reservation name (Host header), checks the party's dietary preferences (URL path, query strings, HTTP headers), looks at how many people are in the group (request content), and then makes a thoughtful decision about which section to seat them in. This takes a moment -- the host needs to read the reservation book, check the floor plan, maybe ask a question -- but the result is precise, intelligent routing. The host even has the ability to authenticate guests (OIDC/Cognito integration), redirect them to a different restaurant (3xx redirects), or hand them a pre-written note (fixed-response actions). But all of this requires the host to **read and understand the reservation** -- which means the host can only work with guests who follow the reservation protocol (HTTP/HTTPS/gRPC).

**NLB is a highway toll booth.** Cars (TCP/UDP packets) arrive at high speed. The toll booth glances at the license plate (source IP + port) and the destination sign (destination IP + port), collects the toll, and waves the car through in microseconds. The toll booth never opens the trunk, never asks where you are going for dinner, never checks your ID beyond the plate number. This is why it is blazingly fast -- it does less work per vehicle. And critically, the toll booth has a **fixed address** (static Elastic IP per AZ) that never changes, which matters when your GPS (DNS clients, firewall allowlists, partner integrations) needs a permanent address to program in. The car's driver also arrives at the destination with their **original license plate intact** -- the toll booth does not swap plates (source IP preservation).

**GLB is a customs checkpoint.** Every package (IP packet) entering or leaving the country (VPC) must pass through customs. The customs agent does not care what is in the package -- they photocopy the shipping label (preserve original headers), wrap the package in a tamper-proof evidence bag (GENEVE encapsulation), and send it to one of many inspectors (third-party appliances like Palo Alto, Fortinet, Check Point). The inspector examines the package, stamps it as approved or rejected, re-seals it, and sends it back through customs. The sender and receiver never know customs was involved -- the entire process is **transparent**. This is fundamentally different from ALB and NLB: GLB does not route to your application, it routes through security appliances before traffic reaches your application.

```
THREE LOAD BALANCER TYPES -- DIFFERENT LAYERS, DIFFERENT JOBS
══════════════════════════════════════════════════════════════════════════════

  CLIENT REQUEST ARRIVES
         │
         ├──── "I need smart HTTP routing" ──────────────────────────────┐
         │     (path, host, headers, auth)                              │
         │                                                              ▼
         │                                            ┌────────────────────────────┐
         │                                            │  ALB (Layer 7)             │
         │                                            │  "Restaurant Host"         │
         │                                            │                            │
         │                                            │  Reads: HTTP headers,      │
         │                                            │    path, host, cookies,    │
         │                                            │    query strings, method   │
         │                                            │  Protocols: HTTP/HTTPS/    │
         │                                            │    gRPC/WebSocket          │
         │                                            │  Terminates: connections   │
         │                                            │  IPs: dynamic (change)     │
         │                                            └────────────────────────────┘
         │
         ├──── "I need raw speed + static IPs" ──────────────────────────┐
         │     (TCP/UDP/TLS, no content inspection)                      │
         │                                                               ▼
         │                                            ┌────────────────────────────┐
         │                                            │  NLB (Layer 4)             │
         │                                            │  "Highway Toll Booth"      │
         │                                            │                            │
         │                                            │  Reads: source/dest IP,    │
         │                                            │    source/dest port,       │
         │                                            │    protocol                │
         │                                            │  Protocols: TCP/UDP/TLS    │
         │                                            │  Passes through: connections│
         │                                            │  IPs: static (Elastic IP)  │
         │                                            └────────────────────────────┘
         │
         └──── "I need transparent appliance insertion" ─────────────────┐
               (firewalls, IDS/IPS, packet inspection)                   │
                                                                         ▼
                                                      ┌────────────────────────────┐
                                                      │  GLB (Layer 3)             │
                                                      │  "Customs Checkpoint"      │
                                                      │                            │
                                                      │  Reads: ALL IP packets     │
                                                      │  Protocol: GENEVE (6081)   │
                                                      │  Transparent: yes          │
                                                      │  Preserves: entire packet  │
                                                      │  Purpose: distribute to    │
                                                      │    security appliances     │
                                                      └────────────────────────────┘
```

---

## Part 1: ALB -- The Restaurant Host (Layer 7)

### How ALB Works

When a client sends an HTTP request, it hits the ALB listener. The ALB **terminates** the TCP connection (this is critical -- the client talks to the ALB, and the ALB opens a **new** connection to the target). During this termination, the ALB has full visibility into the HTTP request: the URL path, the Host header, cookies, query strings, HTTP methods, and custom headers. This is what makes content-based routing possible.

The ALB evaluates listener rules **in priority order** (lowest number first) and forwards the request to the first matching target group. If no rules match, the **default action** fires (typically a forward to a default target group, or a fixed 404 response).

### Listener Rules -- The Seating Chart

ALB rules can match on six conditions, and you can combine up to five conditions in a single rule:

| Condition Type | What It Matches | Example |
|---------------|----------------|---------|
| **host-header** | The `Host` HTTP header | `api.example.com`, `*.staging.example.com` |
| **path-pattern** | The URL path | `/api/v2/*`, `/images/*.jpg` |
| **http-header** | Any HTTP header name + value | `X-Custom-Channel: mobile` |
| **http-request-method** | The HTTP method | `GET`, `POST`, `PATCH` |
| **query-string** | Key-value pairs in the query string | `version=v2&format=json` |
| **source-ip** | The client's source IP or CIDR | `10.0.0.0/8`, `203.0.113.0/24` |

Rules can trigger five actions:

| Action | What It Does | Analogy |
|--------|-------------|---------|
| **forward** | Send to one or more target groups (with optional weights) | Seat the guest at their assigned table |
| **redirect** | Return 301/302 to the client | "That restaurant moved -- here's the new address" |
| **fixed-response** | Return a static HTTP response | Hand the guest a pre-printed note without entering the restaurant |
| **authenticate-oidc** | Redirect to an OIDC provider (e.g., Google, Okta) before forwarding | "Show your membership card before I seat you" |
| **authenticate-cognito** | Redirect to Amazon Cognito for authentication | Same as above, but using the house membership system |

### Weighted Target Groups -- Canary Deployments

The forward action can distribute traffic across **multiple target groups** with weights, enabling canary and blue-green deployments without any external tooling:

```
ALB WEIGHTED TARGET GROUP ROUTING -- CANARY DEPLOYMENT
═══════════════════════════════════════════════════════

  Listener rule: path-pattern = /api/*
         │
         ▼
  ┌─────────────────────────────────┐
  │  Forward action (weighted)      │
  │                                 │
  │  Target Group: api-v1  ──▶ 90% │ ──▶ 4x EC2 instances (current stable)
  │  Target Group: api-v2  ──▶ 10% │ ──▶ 2x EC2 instances (canary release)
  │                                 │
  │  Group-level stickiness: ON     │
  │  Duration: 1 hour               │
  │  (once a user hits v2, they     │
  │   stay on v2 for the session)   │
  └─────────────────────────────────┘
```

### ALB + WAF Integration

ALB is one of the four resources that can have a WAF Web ACL attached (alongside CloudFront, API Gateway, and AppSync -- as you learned on [Mar 10](2026-03-10-aws-network-application-protection.md)). When a WAF Web ACL is attached to an ALB, every request is inspected against the rule set **before** the ALB evaluates its listener rules. This means WAF blocks malicious requests before they ever reach your application. Unlike CloudFront's WAF (which must be in us-east-1), an ALB's WAF Web ACL must be in the **same region** as the ALB.

### ALB Idle Connection Timeout

ALB has an **idle timeout** (default 60 seconds) -- if no data is sent or received on a connection for this duration, ALB closes it. This is one of the most common sources of **504 Gateway Timeout** errors in production. If your backend takes longer than 60 seconds to respond (e.g., a large report generation, file processing), ALB closes the connection before the response arrives.

The 504 and 502 errors have **distinct mechanisms**: a 504 means ALB gave up waiting (its idle timeout expired), while a 502 means the target dropped the connection before sending a valid response (either the application timed out internally, or the target's keep-alive expired and ALB tried to reuse a dead connection). When you see both intermittently, it usually means the target's timeout is shorter than ALB's -- medium requests hit the target timeout (502), large requests hit the ALB timeout (504).

**The golden rule**: `target application timeout > target keep-alive timeout > ALB idle timeout > expected max response time`. For example, if your longest request takes 3 minutes: set ALB idle timeout to 180s, target keep-alive to 200s, and target application timeout to 240s. This ensures ALB always times out first, producing a single consistent error code (504) that is easier to monitor and debug. For truly long-running operations, consider an async pattern instead: return 202 Accepted immediately, process in the background (SQS/Step Functions), and let clients poll for status.

---

## Part 2: NLB -- The Highway Toll Booth (Layer 4)

### How NLB Works

NLB operates at the transport layer. When a TCP connection or UDP datagram arrives, NLB makes a routing decision based on the **flow hash**: a hash of the protocol, source IP, source port, destination IP, destination port, and TCP sequence number. This flow hash is deterministic -- the same connection always maps to the same target (providing implicit stickiness for TCP connections without cookies).

The critical architectural difference: NLB's default mode for TCP is **connection pass-through** -- it does not terminate the client connection. The exception is TLS listeners, where NLB terminates TLS (discussed below). For TCP pass-through, the target sees the original client IP as the source address. This has two major implications:

1. **Source IP preservation**: Your application sees the real client IP in the socket-level source address, not in an `X-Forwarded-For` header. This matters for security appliances, logging, and protocols that embed IP addresses in the payload (SIP, FTP active mode).

2. **Ultra-low latency**: Because NLB does not parse HTTP, buffer the request, or terminate TLS (unless you configure a TLS listener), routing decisions take microseconds instead of milliseconds. NLB can handle **millions of requests per second** with latencies consistently under 100 microseconds for the load balancing decision itself.

### Source IP Preservation -- It Depends on the Target Type

Source IP preservation is not universal across all NLB configurations. The behavior depends on the **target type** and **protocol**:

| Target Type | Protocol | Source IP Preserved by Default? | Can Change? |
|-------------|----------|-------------------------------|-------------|
| **Instance** | TCP/TLS | **Yes** -- client IP visible at socket level | Cannot be disabled |
| **Instance** | UDP/TCP_UDP | **Yes** | Cannot be disabled |
| **IP** | TCP/TLS | **No** -- target sees NLB node's private IP | Must explicitly enable `preserve_client_ip.enabled` |
| **IP** | UDP/TCP_UDP | **Yes** | Cannot be disabled |

This distinction is an **operational gotcha for ECS Fargate users**. Fargate tasks use `awsvpc` networking, which means NLB registers them as **IP targets**. If you set up an NLB with Fargate and check your application logs, you will see NLB node private IPs -- not client IPs -- unless you explicitly enable client IP preservation on the target group.

When source IP preservation is not possible or you need additional metadata (client port, destination IP, TLS negotiation details), enable **Proxy Protocol v2** on the NLB target group. Proxy Protocol v2 prepends a binary header to each connection containing the original client connection metadata. Your application must be configured to parse this header (Nginx: `proxy_protocol`; HAProxy: `accept-proxy`). This is especially important for **TLS listeners** where NLB terminates TLS, breaking the pass-through model.

### NLB Security Groups

NLB did not support security groups until 2023, and even now they are **optional** (disabled by default). This is a significant operational difference from ALB, where security groups are mandatory. Without security groups on NLB:

- With **instance targets**: Targets must accept traffic from any source IP since NLB preserves client IPs. Your target's security group must allow the entire client IP range.
- With **IP targets** (client IP preservation disabled): Targets see NLB node IPs, so you can restrict to NLB node IPs, but these change as NLB scales.

Enabling NLB security groups simplifies this: targets can reference the NLB's security group as a source, just like ALB.

### Static IPs -- The Permanent Address

Each NLB gets **one static IP address per AZ** (either auto-assigned or an Elastic IP you provide). These IPs never change for the lifetime of the NLB. This solves a class of problems that ALB cannot:

| Scenario | Why Static IPs Matter |
|----------|----------------------|
| Partner allowlists | Partners whitelist your IPs in their firewalls -- changing IPs breaks the integration |
| DNS-incapable clients | Some legacy or embedded clients hardcode IP addresses instead of resolving DNS |
| On-premises firewall rules | Corporate firewalls often use IP-based allow rules, not DNS-based |
| Compliance requirements | Some regulations require fixed, documented egress/ingress IP addresses |

### NLB Listeners and TLS

NLB supports three listener types:

- **TCP listener**: Pure pass-through. No TLS termination. Target handles everything.
- **UDP listener**: Same pass-through semantics but for UDP traffic (gaming, IoT, DNS, media streaming).
- **TLS listener**: NLB terminates TLS and re-encrypts (or sends plaintext) to the target. You get the performance benefit of NLB with TLS offloading. The certificate lives in ACM.

### NLB + PrivateLink -- The Only Combination That Works

PrivateLink (which you studied on [Feb 20](../../february/aws/2026-02-20-vpc-peering-vs-transit-gateway.md)) allows you to expose a service to other VPCs or accounts via a VPC endpoint, without peering, transit gateways, or public internet. PrivateLink **only works with NLB or GLB** as the service's front end -- ALB alone cannot serve as a PrivateLink service endpoint. This is one of the most important ALB-vs-NLB decision factors.

When you need both PrivateLink support **and** Layer 7 routing, you use the **NLB-to-ALB target** pattern:

```
NLB-TO-ALB PATTERN -- COMBINING STATIC IPs + LAYER 7 ROUTING
══════════════════════════════════════════════════════════════

  Consumer VPC                          Provider VPC
  ┌──────────────────┐                  ┌──────────────────────────────────────┐
  │                  │                  │                                      │
  │  Application ────┼──▶ VPC          │   NLB (static IPs)                   │
  │                  │    Endpoint      │     │                                │
  │                  │    (PrivateLink) │     │  TCP listener, port 443        │
  │                  │        │         │     │  Target: ALB (type: alb)       │
  └──────────────────┘        │         │     ▼                                │
                              │         │   ALB (Layer 7 routing)              │
                              └────────▶│     │                                │
                                        │     ├── /api/*   ──▶ API TG         │
                                        │     ├── /web/*   ──▶ Web TG         │
                                        │     └── default  ──▶ Default TG     │
                                        │                                      │
                                        └──────────────────────────────────────┘

  Constraints:
  - One ALB per NLB target group
  - Both must be in the same VPC and account
  - NLB target group protocol must be TCP
  - ALB can be a target of at most 2 NLBs
  - Each registered ALB reduces max targets per AZ by 50
```

---

## Part 3: GLB (GWLB) -- The Customs Checkpoint (Layer 3)

> **Naming note**: The official AWS abbreviation is **GWLB** (Gateway Load Balancer). "GLB" is common shorthand but does not appear in AWS documentation. Both are used interchangeably in practice.

### How GLB Works

GLB is architecturally different from ALB and NLB. It does not route to your application -- it routes **through** security appliances. Think of it as a transparent bump-in-the-wire that distributes traffic across a fleet of virtual appliances (firewalls, IDS/IPS, deep packet inspection tools) while preserving the original packet completely.

The three-step flow:

1. **Traffic enters**: A VPC route table directs traffic to a **Gateway Load Balancer Endpoint (GWLBe)** as the next-hop. The GWLBe is a VPC endpoint (powered by PrivateLink) in the consumer VPC.

2. **GENEVE encapsulation**: GLB wraps the original IP packet in a GENEVE tunnel (UDP port 6081), preserving all original headers (source IP, destination IP, protocol). It then distributes the encapsulated packet to a target appliance using a **5-tuple flow hash** (ensuring all packets in a flow go to the same appliance for stateful inspection).

3. **Inspection and return**: The appliance de-encapsulates the GENEVE wrapper, inspects the original packet, makes a permit/deny decision, re-encapsulates, and sends it back to GLB. GLB forwards the original packet to its destination.

```
GLB TRAFFIC FLOW -- TRANSPARENT APPLIANCE INSERTION
════════════════════════════════════════════════════════════════════════

  Application VPC                          Security/Appliance VPC
  ┌──────────────────────────────────┐     ┌───────────────────────────────┐
  │                                  │     │                               │
  │  ┌──────────┐                    │     │  ┌─────────────────────────┐  │
  │  │ EC2 /    │  Route table:      │     │  │  Gateway Load Balancer  │  │
  │  │ ECS App  │  0.0.0.0/0 ──▶    │     │  │  (Layer 3)              │  │
  │  │          │  GWLBe             │     │  │                         │  │
  │  └────┬─────┘                    │     │  │  Listens on ALL ports,  │  │
  │       │                          │     │  │  ALL protocols          │  │
  │       ▼                          │     │  │                         │  │
  │  ┌─────────┐    PrivateLink      │     │  │  5-tuple flow hash      │  │
  │  │ GWLBe   │ ──────────────────────────▶  │  routing to appliances  │  │
  │  │(VPC EP) │                     │     │  └───────┬─────────────────┘  │
  │  └─────────┘                     │     │          │                    │
  │       ▲                          │     │          ▼                    │
  │       │  return (inspected)      │     │  ┌───────────────────────┐   │
  │       └──────────────────────────────────  │ GENEVE encapsulated  │   │
  │                                  │     │  │ traffic to appliances │   │
  │  ┌──────────┐                    │     │  │                       │   │
  │  │ Internet │                    │     │  │  ┌───────┐ ┌───────┐  │   │
  │  │ Gateway  │                    │     │  │  │Palo   │ │Forti- │  │   │
  │  └──────────┘                    │     │  │  │Alto   │ │net    │  │   │
  │                                  │     │  │  │VM     │ │VM     │  │   │
  └──────────────────────────────────┘     │  │  └───────┘ └───────┘  │   │
                                           │  └───────────────────────┘   │
                                           └───────────────────────────────┘

  KEY DETAIL: GWLBe and application instances must be in DIFFERENT subnets.
  The route table on the application subnet points 0.0.0.0/0 at the GWLBe.
  The route table on the GWLBe subnet points 0.0.0.0/0 at the IGW.
  This asymmetric routing is what makes transparent insertion work.
```

### GLB vs Network Firewall -- The Decision

You studied AWS Network Firewall on [Mar 10](2026-03-10-aws-network-application-protection.md) -- it solves a similar problem (traffic inspection) but as a fully managed AWS service. Here is when to choose each:

| Factor | GLB + Third-Party Appliance | AWS Network Firewall |
|--------|---------------------------|---------------------|
| **Appliance choice** | Bring any vendor (Palo Alto, Fortinet, Check Point, custom) | AWS-managed Suricata engine only |
| **Management** | You manage appliance AMIs, patching, scaling, licensing | AWS manages compute, patching, scaling |
| **Inspection depth** | Vendor-specific capabilities (advanced threat intel, ML-based detection) | Suricata IPS/IDS rules, domain filtering, TLS SNI |
| **Existing investment** | Leverage existing on-prem appliance expertise and rule sets | Start fresh with Suricata or AWS managed rule groups |
| **Cost model** | GLB hourly + LCU + appliance EC2 instances + vendor licensing | Network Firewall hourly + GB processed |
| **Centralized architecture** | GLB + TGW + appliance mode | NFW + TGW + appliance mode (same TGW pattern) |

The key insight: **if you are already running Palo Alto or Fortinet on-premises and want consistent security policies in the cloud, GLB lets you deploy the same vendor's virtual appliances.** If you are cloud-native with no existing appliance investment, Network Firewall is simpler and cheaper to operate.

---

## Part 4: Target Groups -- The Table Sections

A target group is a **logical collection of targets** that receives traffic from a load balancer. Think of it as a section of tables in the restaurant: the API section, the web section, the admin section. Each section has its own health checks, routing algorithm, and draining behavior.

### Target Types

| Target Type | Available On | How It Works | Use Case |
|-------------|-------------|-------------|----------|
| **Instance** | ALB, NLB, GLB | Routes to the instance's primary private IP on the specified port | EC2 instances, ECS on EC2 |
| **IP** | ALB, NLB, GLB | Routes to any IP address (can be outside the VPC via peering or PrivateLink) | Containers with awsvpc networking, on-prem targets via DX/VPN, cross-VPC targets |
| **Lambda** | ALB only | Invokes a Lambda function, one per target group | Serverless backends behind ALB |
| **ALB** | NLB only | Routes to an ALB's listeners | Static IPs + Layer 7 routing, PrivateLink + ALB |

### Routing Algorithms

ALB target groups support three algorithms (NLB always uses flow hash, GLB always uses 5-tuple flow hash):

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| **Round robin** | Rotates through targets sequentially | Uniform request processing times |
| **Least outstanding requests** | Sends to the target with the fewest in-flight requests | Variable-latency requests (some take 50ms, some take 5s) |
| **Weighted random** (with Automatic Target Weights) | Randomly selects targets weighted by anomaly detection scores | Automatic mitigation of targets experiencing elevated error rates |

**Least outstanding requests** is typically the better default for API workloads. Consider: if you have 4 targets and one is slower (perhaps its CPU is higher, its cache is cold, or it is on degraded hardware), round robin will keep sending it 25% of traffic while requests pile up. Least outstanding requests will naturally shift traffic away because that target always has more in-flight requests.

### Slow Start Mode

When Auto Scaling launches a new instance and registers it in a target group (as you learned on [Feb 12](../../february/aws/2026-02-12-auto-scaling-deep-dive.md)), the instance may need time to warm up -- load classes, populate caches, establish database connection pools. Without slow start, the target immediately receives a full share of traffic and may buckle under the sudden load.

Slow start mode ramps traffic linearly from 0 to full share over a configurable duration (30-900 seconds). During this period, the target receives a progressively increasing number of requests. Once the window expires, the target participates normally in the routing algorithm.

**Constraints**: Slow start **only works with round robin**. It is incompatible with both least outstanding requests (which already naturally favors new targets since they have zero in-flight requests) and weighted random (which uses anomaly scores, not connection counts). It also disables if the target group has only one registered target.

---

## Part 5: Health Checks -- The Manager's Rounds

Health checks are how the load balancer discovers that a target cannot handle traffic. The load balancer sends periodic probes and evaluates the responses to determine target health.

### Configuration Parameters

| Parameter | ALB Default | NLB Default | Range | Notes |
|-----------|------------|------------|-------|-------|
| **Protocol** | HTTP | TCP | HTTP, HTTPS, TCP | NLB can use TCP (just checks port open) or HTTP |
| **Port** | Traffic port | Traffic port | 1-65535 | Can override to a dedicated health port |
| **Path** | `/` | `/` (if HTTP) | Any valid path | Use a dedicated `/health` endpoint |
| **Interval** | 30s | 30s | 5-300s | Time between probes |
| **Timeout** | 5s | 10s (TCP: 6s) | 2-120s | Must be less than interval |
| **Healthy threshold** | 5 | 3 | 2-10 | Consecutive successes to mark healthy |
| **Unhealthy threshold** | 2 | 3 (must equal healthy) | 2-10 | Consecutive failures to mark unhealthy |
| **Success codes** | 200 | 200 (if HTTP) | Any HTTP codes | Use `200-299` for flexibility |

### ALB's Asymmetric Thresholds Are Intentional

ALB's default healthy threshold (5) is higher than the unhealthy threshold (2). This is deliberate:

- **Fail fast** (2 checks): You want to stop sending traffic to a failing target as quickly as possible. Two consecutive failures is enough confidence.
- **Recover slowly** (5 checks): A target that briefly responds to a health check might still be unstable (partially initialized, flapping, overloaded). Requiring five consecutive successes builds confidence before production traffic returns.

With ALB's default 30-second interval: a target fails in **60 seconds** (2 x 30s) but recovers in **150 seconds** (5 x 30s).

> **NLB is different**: NLB requires the healthy and unhealthy thresholds to be the **same value** -- you cannot configure them independently. Both default to 3. This means NLB targets recover and fail at the same speed (no asymmetry). This is a common exam question and a real operational difference from ALB.

### Fail-Open Behavior

If **all targets** across **all enabled AZs** in a target group become unhealthy, the load balancer enters **fail-open** mode: it sends traffic to all targets regardless of health status. The reasoning is that a partially working target is better than no target at all. This is a last-resort availability mechanism -- if you see fail-open in production, something is seriously wrong and you should investigate immediately.

### Health Check Grace Period

When using ALB/NLB with an ECS service or Auto Scaling group, you can configure a **health check grace period** -- a window after target registration during which the load balancer ignores health check failures. This prevents targets from being killed before they finish booting (which would create a vicious cycle: launch, fail health check during boot, terminate, launch again).

---

## Part 6: Cross-Zone Load Balancing -- Dining Rooms on Different Floors

Cross-zone load balancing controls whether a load balancer node in one AZ can send traffic to targets in a different AZ. The defaults differ between load balancer types, and this difference comes up frequently in exam questions and architecture reviews.

### The Problem Cross-Zone Solves

DNS distributes clients roughly equally across AZs (Route53 returns all LB node IPs, clients typically use the first one). But if your targets are not evenly distributed across AZs, even distribution of traffic to AZs causes uneven distribution of traffic to targets:

```
WITHOUT CROSS-ZONE LOAD BALANCING
══════════════════════════════════

  DNS resolves to LB nodes in both AZs (50/50 split)

  AZ-A (50% of traffic)          AZ-B (50% of traffic)
  ┌─────────────────────┐        ┌─────────────────────┐
  │  LB Node A          │        │  LB Node B          │
  │     │               │        │     │               │
  │     ├──▶ Target 1 (25%)│     │     ├──▶ Target 3  (6.25%)│
  │     │               │        │     ├──▶ Target 4  (6.25%)│
  │     └──▶ Target 2 (25%)│     │     ├──▶ Target 5  (6.25%)│
  │                     │        │     ├──▶ Target 6  (6.25%)│
  │  2 targets get 50%  │        │     ├──▶ Target 7  (6.25%)│
  │  of ALL traffic     │        │     ├──▶ Target 8  (6.25%)│
  └─────────────────────┘        │     ├──▶ Target 9  (6.25%)│
                                 │     └──▶ Target 10 (6.25%)│
                                 │                     │
                                 │  8 targets get 50%  │
                                 │  of ALL traffic     │
                                 └─────────────────────┘

  Result: Targets 1-2 each handle 25% of total traffic
          Targets 3-10 each handle 6.25% of total traffic
          Targets 1-2 are 4x more loaded than Targets 3-10!

WITH CROSS-ZONE LOAD BALANCING
═══════════════════════════════

  LB nodes can send to targets in ANY AZ

  AZ-A (50% of traffic)          AZ-B (50% of traffic)
  ┌─────────────────────┐        ┌─────────────────────┐
  │  LB Node A ─────────┼──┐  ┌─┼── LB Node B         │
  │     │               │  │  │ │     │               │
  │     ├──▶ Target 1   │  │  │ │     ├──▶ Target 3   │
  │     │               │  │  │ │     ├──▶ Target 4   │
  │     └──▶ Target 2   │  │  │ │     ├──▶ ...        │
  │                     │  │  │ │     └──▶ Target 10  │
  └─────────────────────┘  │  │ └─────────────────────┘
                           │  │
                           └──┘ Both LB nodes distribute
                                across ALL 10 targets

  Result: Each of the 10 targets handles exactly 10% of traffic
```

### Default Behavior by LB Type

| Load Balancer | Cross-Zone Default | Can Be Changed | Cost of Cross-Zone Traffic |
|---------------|-------------------|----------------|---------------------------|
| **ALB** | **Enabled** at load balancer level | Can be disabled per target group | **No charge** (AWS absorbs inter-AZ cost) |
| **NLB** | **Disabled** | Can be enabled per target group or load balancer level | **Inter-AZ data transfer charges apply** |
| **GLB** | **Disabled** | Can be enabled per target group | **Inter-AZ data transfer charges apply** |

**Why the difference?** ALB is designed for HTTP workloads where even distribution matters most -- an overloaded server means slow page loads. AWS absorbs the inter-AZ cost to make even distribution the default. NLB and GLB are designed for scenarios where **zonal isolation** often matters more: if you are running stateful firewalls (GLB) or low-latency financial services (NLB), you may specifically want traffic to stay within the same AZ. Inter-AZ data transfer also costs money at scale, so NLB and GLB charge for it to ensure you make a conscious decision.

---

## Part 7: Connection Draining (Deregistration Delay) -- Letting Diners Finish Their Meal

When a target is removed from a target group -- whether by Auto Scaling scale-in, a manual deregistration, a rolling deployment, or a failing health check -- it enters the **draining** state. During draining:

- **No new requests** are sent to the target
- **In-flight requests** continue processing until they complete or the deregistration delay expires
- After the delay, remaining connections are **forcibly closed**

| Parameter | Default | Range | Recommendation |
|-----------|---------|-------|----------------|
| Deregistration delay | **300 seconds** (5 min) | 0-3600 seconds | Match your longest expected request duration. For APIs: 30-60s. For WebSockets/long-polling: 300-3600s. |

### The Auto Scaling Interaction

This connects directly to your Auto Scaling study on [Feb 12](../../february/aws/2026-02-12-auto-scaling-deep-dive.md). When Auto Scaling decides to scale in, it deregisters the target and starts the deregistration delay. But Auto Scaling also has its own **instance termination lifecycle**. If the deregistration delay (e.g., 300s) is longer than the time Auto Scaling waits before terminating the instance, the instance may be killed while requests are still draining.

The solution: use Auto Scaling **lifecycle hooks** to hold the instance in a `Terminating:Wait` state until draining completes. Set the lifecycle hook timeout to be greater than or equal to your deregistration delay.

---

## Part 8: Sticky Sessions -- Remembering Your Favorite Table

Sticky sessions (session affinity) route a returning client to the **same target** that handled their previous request. ALB implements this with cookies.

### Two Cookie Mechanisms

| Type | Cookie Name | Who Generates It | Expiration | Use Case |
|------|------------|------------------|------------|----------|
| **Duration-based** | `AWSALB` (+ `AWSALBCORS` for CORS) | ALB | Fixed timer (1s to 7 days) | Simple stickiness without application changes |
| **Application-based** | `AWSALBAPP` (companion to your app's cookie) | ALB + application | When the app cookie expires or is removed | Application controls session lifecycle (e.g., stickiness ends on logout) |

### When Stickiness Helps

- Application stores session state **in-memory** on the server (not externalized to Redis/DynamoDB)
- WebSocket connections that must remain on the same backend
- Shopping carts stored in server memory (legacy pattern)

### When Stickiness Hurts

- **Uneven load distribution**: A popular user's requests all hit one target while others sit idle. This defeats the purpose of load balancing.
- **Scaling inefficiency**: New targets receive zero traffic from sticky users until their cookies expire.
- **Failure impact**: If a sticky target dies, all users pinned to it lose their sessions.

**Best practice**: Externalize session state to ElastiCache (Redis) or DynamoDB. This eliminates the need for sticky sessions entirely, allows any target to serve any request, and makes your application resilient to target failures. Use sticky sessions only as a temporary measure while migrating to externalized state.

### Constraints

- Sticky sessions require **cross-zone load balancing** to be enabled (attempting to enable stickiness with cross-zone disabled will fail)
- Stickiness is configured at the **target group** level, not the listener level
- NLB does not use cookie-based stickiness -- TCP connections are inherently sticky via flow hash (same source IP + port always maps to the same target for the connection's lifetime)

---

## Part 9: Decision Framework -- Which Load Balancer?

### The Decision Table

| Factor | ALB | NLB | GLB |
|--------|-----|-----|-----|
| **OSI Layer** | 7 (Application) | 4 (Transport) | 3 (Network) |
| **Protocols** | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TLS | All IP traffic |
| **Routing logic** | Content-based (path, host, headers, query string, method, source IP) | Flow hash (protocol, IPs, ports, TCP seq#) | 5-tuple flow hash |
| **Connection behavior** | Terminates and creates new connection | Passes through (preserves source IP) | GENEVE encapsulation (preserves entire packet) |
| **Static IPs** | No (IPs change, use DNS) | Yes (one Elastic IP per AZ) | No (uses GWLBe, not direct IPs) |
| **Source IP preservation** | Via `X-Forwarded-For` header | Native (socket-level source IP) | Native (via GENEVE original headers) |
| **TLS termination** | Yes (HTTPS listener) | Yes (TLS listener) | No |
| **WAF integration** | Yes | No | No |
| **PrivateLink support** | No (but can be NLB target) | Yes (service provider) | Yes (GWLBe is a VPC endpoint) |
| **Target types** | Instance, IP, Lambda | Instance, IP, ALB | Instance, IP |
| **Cross-zone default** | Enabled (free) | Disabled (costs $) | Disabled (costs $) |
| **Sticky sessions** | Cookie-based (AWSALB, AWSALBAPP) | Flow-hash-based (implicit) | 5-tuple flow-based (implicit) |
| **Latency** | Milliseconds (HTTP parsing overhead) | Microseconds (pass-through) | Microseconds (encapsulation overhead) |
| **Scale** | Thousands of RPS | Millions of RPS | Millions of PPS |
| **Cost (hourly)** | $0.0225/hr + LCU | $0.0225/hr + NLCU | $0.014/hr **per AZ** + GLCU |

### The Decision Flowchart

```
START: What kind of traffic are you load balancing?
│
├── HTTP/HTTPS/gRPC traffic to your application?
│   │
│   ├── Need content-based routing (path, host, headers)?  ──▶ ALB
│   ├── Need WAF integration?  ──▶ ALB
│   ├── Need authentication (OIDC/Cognito)?  ──▶ ALB
│   ├── Need Lambda as a target?  ──▶ ALB
│   ├── Need weighted target groups for canary?  ──▶ ALB
│   └── Just need basic HTTP load balancing?  ──▶ ALB (still the right choice)
│
├── TCP/UDP/TLS traffic (non-HTTP or extreme performance)?
│   │
│   ├── Need static IPs?  ──▶ NLB
│   ├── Need to expose via PrivateLink?  ──▶ NLB
│   ├── Need source IP preservation at socket level?  ──▶ NLB
│   ├── Gaming, IoT, financial trading (ultra-low latency)?  ──▶ NLB
│   ├── Non-HTTP protocols (SMTP, custom TCP)?  ──▶ NLB
│   └── Need static IPs + Layer 7 routing?  ──▶ NLB with ALB as target
│
└── Need to insert third-party security appliances?
    │
    ├── Palo Alto, Fortinet, Check Point, custom IDS/IPS?  ──▶ GLB
    ├── Need same vendor as on-premises firewalls?  ──▶ GLB
    ├── Traffic must be transparently intercepted?  ──▶ GLB
    └── Multi-VPC centralized inspection (third-party)?  ──▶ GLB + TGW
```

---

## Part 10: Architectural Patterns -- Combining Load Balancers

### Pattern 1: CloudFront + ALB (Most Common Web Application)

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│CloudFront│────▶│   WAF    │────▶│   ALB    │────▶│ EC2/ECS  │
│  (CDN)   │     │(Web ACL) │     │(Layer 7) │     │ Targets  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘

- CloudFront caches static content, terminates TLS at the edge
- WAF on ALB filters malicious requests
- ALB routes /api/* to API targets, /* to web targets
- ALB configured as CloudFront origin (via VPC Origins for private ALB)
```

### Pattern 2: NLB + ALB (Static IPs + Layer 7 Routing)

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ Client / │────▶│   NLB    │────▶│   ALB    │────▶│ EC2/ECS  │
│PrivateLink│    │(Static IP)│    │(Layer 7) │     │ Targets  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘

- NLB provides static IPs for firewall allowlists or PrivateLink
- ALB provides path/host-based routing, WAF, authentication
- Used when PrivateLink consumers need Layer 7 features
```

### Pattern 3: GLB + TGW (Centralized Multi-VPC Inspection)

```
  Spoke VPC A          Spoke VPC B          Security VPC
  ┌──────────┐        ┌──────────┐        ┌───────────────────┐
  │  App     │        │  App     │        │  ┌─────┐  ┌─────┐ │
  │  Servers │        │  Servers │        │  │Palo │  │Forti│ │
  └────┬─────┘        └────┬─────┘        │  │Alto │  │net  │ │
       │                   │              │  └──┬──┘  └──┬──┘ │
       ▼                   ▼              │     │        │    │
  ┌─────────────────────────────────┐     │  ┌──▼────────▼──┐ │
  │     Transit Gateway             │     │  │     GLB      │ │
  │     (appliance mode enabled)    │─────▶  │   (Layer 3)  │ │
  │                                 │     │  └──────────────┘ │
  └─────────────────────────────────┘     │  ┌──────────────┐ │
                                          │  │    GWLBe     │ │
                                          │  └──────────────┘ │
                                          └───────────────────┘

- All inter-VPC and internet-bound traffic routes through TGW
- TGW forwards to Security VPC where GWLBe intercepts traffic
- GLB distributes to appliance fleet using GENEVE encapsulation
- TGW appliance mode ensures return traffic uses same AZ (stateful inspection)
- Same pattern as centralized Network Firewall but with third-party appliances
```

### Pattern 4: CloudFront + NLB + ALB (Full Stack)

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│CloudFront│────▶│   NLB    │────▶│   ALB    │────▶│ ECS      │────▶│   RDS    │
│(Global)  │     │(Static IP│     │(Routing) │     │ Fargate  │     │(Database)│
│          │     │ per AZ)  │     │          │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘     └──────────┘

- CloudFront for global caching and edge TLS termination
- NLB for static IPs (required by partner integrations or compliance)
- ALB for path-based routing across microservices
- Rarely needed -- adds cost and latency. Only when static IPs are mandatory.
```

---

## Key Takeaways

- **ALB is for HTTP intelligence**: Choose ALB whenever you need to make routing decisions based on request content (path, host, headers, query strings). It terminates connections, which enables TLS offloading, WAF integration, authentication, weighted target groups, and cookie-based sticky sessions. The tradeoff is higher latency than NLB (milliseconds, not microseconds).

- **NLB is for raw speed and static IPs**: Choose NLB when you need static Elastic IPs per AZ, PrivateLink integration, source IP preservation at the socket level, ultra-low latency, or non-HTTP protocols. NLB passes connections through without inspection, which makes it faster but unable to make content-based routing decisions.

- **GLB is for transparent appliance insertion**: Choose GLB when you need to distribute traffic to a fleet of third-party virtual appliances (firewalls, IDS/IPS) without applications knowing the appliances exist. GLB uses GENEVE encapsulation to preserve original packets and is powered by GWLBe (PrivateLink endpoints) as route table next-hops.

- **Cross-zone defaults differ and it matters**: ALB enables cross-zone by default (free). NLB and GLB disable it by default (inter-AZ charges apply). This means an NLB with unevenly distributed targets will create hot spots unless you explicitly enable cross-zone and accept the data transfer costs.

- **Connection draining (deregistration delay) defaults to 300 seconds**: This is generous for most workloads. Reduce to 30-60 seconds for API services to speed up deployments. Increase for WebSocket or long-polling workloads. Always coordinate with Auto Scaling lifecycle hooks to prevent instance termination before draining completes.

- **ALB health checks fail fast (2) and recover slowly (5); NLB uses symmetric thresholds (3/3)**: ALB's asymmetric thresholds are intentional -- fail quickly, recover cautiously. NLB forces both thresholds to be equal (you cannot set them independently). Use a dedicated `/health` endpoint that verifies downstream dependencies (database, cache), not just a `200 OK` from the web server framework.

- **Externalize session state instead of using sticky sessions**: Sticky sessions cause uneven load distribution, reduce scaling efficiency, and increase blast radius of target failures. Use ElastiCache or DynamoDB for session state and eliminate the need for stickiness entirely.

- **NLB-to-ALB is the pattern for PrivateLink + Layer 7 routing**: When consumers need to access your service via PrivateLink but your application requires path-based routing, put NLB in front of ALB. This also solves the "I need a static IP for my ALB" problem.

- **GLB + TGW with appliance mode is the centralized third-party inspection pattern**: Same architecture as centralized Network Firewall ([Mar 10](2026-03-10-aws-network-application-protection.md)) but with your choice of vendor appliance. TGW appliance mode ensures symmetric routing for stateful inspection.

- **Fail-open is a last resort, not a feature**: When all targets are unhealthy, the LB routes to all of them anyway because a partially working target beats no target. If you see fail-open in production, your health checks or your application have a systemic problem.

---

## Further Reading

- [How Elastic Load Balancing Works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html) -- Cross-zone load balancing, routing algorithms, and AZ distribution explained with concrete examples
- [ALB Target Groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) -- Target types, routing algorithms, slow start, deregistration delay
- [GLB Introduction](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html) -- GENEVE encapsulation, GWLBe architecture, appliance integration
- [Using NLB with ALB as Target](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/application-load-balancer-target.html) -- Constraints and configuration for the NLB-in-front-of-ALB pattern
- [ELB Best Practices Guide](https://aws.github.io/aws-elb-best-practices/) -- Workload architecture, security patterns, and failure management
- Related learning docs: [CloudFront (ALB as origin)](2026-03-13-cloudfront-deep-dive.md), [Network Firewall (GLB alternative)](2026-03-10-aws-network-application-protection.md), [Auto Scaling (target registration)](../../february/aws/2026-02-12-auto-scaling-deep-dive.md), [VPC Peering & PrivateLink](../../february/aws/2026-02-20-vpc-peering-vs-transit-gateway.md)
