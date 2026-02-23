# Transit Gateway Deep Dive -- The International Airport Analogy

> Understanding Transit Gateway architecture, route table mechanics, centralized egress, centralized inspection, inter-region peering, and multi-account sharing through the analogy of a major international airport -- where terminals are route tables, gates are attachments, flight boards are route propagation, and the air traffic control tower enforces which planes can reach which destinations.

---

## TL;DR

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Transit Gateway | International airport | A regional hub where every airline (VPC, VPN, DX) connects via gates; passengers transfer through the terminal instead of building private roads between every pair of cities |
| TGW ENI | Airport gate jetway | A network interface placed in a dedicated /28 subnet in your VPC; it is the stationary connector where packets pass between your VPC and the airport terminal |
| TGW Attachment | An airline's gate assignment | Each VPC, VPN, or DX connection gets a gate at the airport; the gate is how traffic enters and leaves the hub |
| TGW Route Table | A terminal with its own flight board | Each terminal shows only certain destinations; production flights and development flights depart from separate terminals with different boards |
| Association | "Which terminal does this airline depart from?" | Each attachment is associated with exactly one route table; that table decides where outbound traffic can go |
| Propagation | "Which flight boards list this destination?" | An attachment can advertise its destination to multiple terminals; any terminal that lists it can send passengers there |
| Static Route | A manually posted destination on the flight board | You add the route by hand; it does not update automatically when attachments change |
| Blackhole Route | A destination permanently closed on the flight board | Traffic matching this route is silently dropped; used to explicitly block certain CIDRs |
| Centralized Egress VPC | The airport's single international departures hall | All outbound internet traffic funnels through one terminal with customs (NAT Gateway) instead of every airline building its own customs booth |
| Centralized Inspection VPC | Airport security screening checkpoint | All traffic passes through security (Network Firewall or appliance) before reaching the destination gate |
| Appliance Mode | Assigning each passenger a specific security agent who handles both their departure screening and arrival customs | Ensures both request and response traverse the same AZ's firewall, preventing stateful inspection from breaking |
| TGW Peering | A bilateral flight agreement between two airports | Two airports in different regions agree to route passengers between them; flight boards must be manually updated (static routes only) |
| RAM Sharing | Airport franchise agreement | The airport owner (network account) lets other airlines (spoke accounts) use the gates without giving them access to air traffic control |
| GWLB + TGW | Third-party security contractors at the airport | When the airport's built-in security (Network Firewall) is insufficient, you bring in specialized contractors (Palo Alto, Fortinet) routed through a transparent load balancer |

---

## The Big Picture

In the [VPC Peering vs Transit Gateway doc](./2026-02-20-vpc-peering-vs-transit-gateway.md), you learned **when** to use Transit Gateway: more than ~10 VPCs, need for transitive routing, centralized egress, or hybrid connectivity. You learned the association-vs-propagation mental model at a conceptual level and saw a Terraform example.

Today you go deep into **how Transit Gateway actually works** -- the internal mechanics that determine whether a packet reaches its destination or vanishes into a blackhole. You are no longer the transportation secretary choosing between roads and interchanges. You are now the **airport operations director** who must design terminal layouts, write the flight boards, staff security checkpoints, negotiate bilateral agreements with foreign airports, and ensure 5,000 gates can operate without a single misdirected passenger.

The airport analogy works because Transit Gateway is fundamentally a **regional routing hub**. Like an airport, it sits at the center of a spoke topology. Traffic does not flow directly between endpoints -- it flows into the hub, gets evaluated against a routing table (the flight board), and gets forwarded to the correct outbound gate. The power is in the routing rules.

---

## Part 1: Transit Gateway Architecture -- Inside the Airport

### How TGW Actually Works

A Transit Gateway is a **regional, highly available Layer 3 virtual router** managed by AWS. It is not a single box in a data center -- AWS deploys redundant infrastructure across all Availability Zones in the region. You do not manage the underlying infrastructure, failover, or scaling.

When you create a VPC attachment, TGW places an **Elastic Network Interface (ENI)** in each subnet you specify (one per AZ). This ENI is the physical entry point -- the jetway connecting your VPC to the airport terminal. Packets leaving your VPC hit the VPC route table, match a route pointing to the TGW, and are forwarded to the TGW ENI. From there, the TGW evaluates the packet against the associated TGW route table and forwards it to the correct outbound attachment.

```
TGW PACKET FLOW — STEP BY STEP
=============================================================================

  VPC-A instance (10.0.5.20) wants to reach VPC-B instance (10.1.3.50)

  STEP 1: VPC-A subnet route table lookup
  ┌──────────────────────────────────────────────────┐
  │  Destination       │ Target                      │
  ├──────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ local                       │ ← no match
  │  10.1.0.0/16       │ tgw-0abc123def456789        │ ← MATCH
  │  0.0.0.0/0         │ nat-regional-1              │
  └──────────────────────────────────────────────────┘
  Packet sent to TGW ENI in VPC-A's TGW attachment subnet

  STEP 2: TGW receives packet on VPC-A attachment
  ├── TGW identifies the source attachment: vpc-a-attachment
  ├── Looks up which route table vpc-a-attachment is ASSOCIATED with
  └── Answer: prod-rt

  STEP 3: TGW route table (prod-rt) lookup
  ┌──────────────────────────────────────────────────┐
  │  Destination       │ Target Attachment   │ Type  │
  ├──────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ vpc-a-attachment    │ prop  │
  │  10.1.0.0/16       │ vpc-b-attachment    │ prop  │ ← MATCH
  │  10.2.0.0/16       │ shared-attachment   │ prop  │
  │  0.0.0.0/0         │ egress-attachment   │ static│
  └──────────────────────────────────────────────────┘
  TGW forwards packet to vpc-b-attachment

  STEP 4: Packet arrives at VPC-B via TGW ENI
  VPC-B's local route (10.1.0.0/16 → local) delivers to 10.1.3.50

  TOTAL HOPS: instance → VPC route table → TGW ENI → TGW routing →
              TGW ENI in VPC-B → destination instance
  LATENCY OVERHEAD: typically < 1ms additional in same-region deployments
```

### The TGW ENI and Dedicated /28 Subnets

This is one of the most important operational details. When you create a VPC attachment, you specify one subnet per AZ. TGW places an ENI in each subnet. AWS best practice is to use **dedicated /28 subnets** exclusively for TGW attachments:

```
WHY DEDICATED TGW SUBNETS MATTER
=============================================================================

  DO THIS:
  ┌──────────────────────────────────────────────────────────────────────┐
  │  VPC: 10.0.0.0/16                                                   │
  │                                                                      │
  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
  │  │ tgw-az-a        │  │ tgw-az-b        │  │ tgw-az-c        │     │
  │  │ 10.0.240.0/28   │  │ 10.0.240.16/28  │  │ 10.0.240.32/28  │     │
  │  │ (11 usable IPs) │  │ (11 usable IPs) │  │ (11 usable IPs) │     │
  │  │ [TGW ENI only]  │  │ [TGW ENI only]  │  │ [TGW ENI only]  │     │
  │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
  │                                                                      │
  │  WHY /28? TGW needs exactly 1 IP per AZ. /28 gives 11 usable IPs, │
  │  which is more than enough. Using anything larger wastes address     │
  │  space. The dedicated subnets ensure that:                          │
  │  ├── NACLs on TGW subnets are left OPEN (AWS best practice)        │
  │  │   TGW ENIs need unrestricted traffic flow; apply NACLs on       │
  │  │   your workload subnets instead                                  │
  │  ├── No other resources compete for IPs in the TGW subnet          │
  │  └── Route tables for TGW subnets can be purpose-built             │
  │      (they need routes back to the VPC CIDR, NAT, etc.)            │
  └──────────────────────────────────────────────────────────────────────┘

  DO NOT DO THIS:
  ├── Place TGW ENIs in your application subnets
  │   (NACL conflicts — TGW needs open NACLs, your apps need restrictive ones)
  ├── Use a single AZ for TGW attachment
  │   (lose cross-AZ connectivity if that AZ goes down)
  └── Forget to add routes in the TGW subnet route table
      (return traffic from TGW to the internet needs NAT or egress routes)
```

---

## Part 2: TGW Attachments -- All Five Gate Types

Transit Gateway supports five attachment types, each connecting a different kind of network to the hub.

```
FIVE TGW ATTACHMENT TYPES
=============================================================================

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                        TRANSIT GATEWAY                                  │
  │                                                                         │
  │  1. VPC ATTACHMENT              2. VPN ATTACHMENT                       │
  │  ┌───────────────────┐          ┌───────────────────┐                   │
  │  │ Connects a VPC    │          │ Site-to-Site VPN   │                   │
  │  │ via ENIs in /28   │          │ tunnels terminate  │                   │
  │  │ subnets (1/AZ)    │          │ directly on TGW    │                   │
  │  │                   │          │                    │                   │
  │  │ Most common type  │          │ Supports ECMP:     │                   │
  │  │ Up to 50 Gbps     │          │ multiple VPNs      │                   │
  │  │ per attachment     │          │ aggregate bandwidth│                   │
  │  └───────────────────┘          │ (up to ~50 Gbps)   │                   │
  │                                 │                    │                   │
  │                                 │ 1.25 Gbps/tunnel   │                   │
  │                                 │ Use BGP for dynamic│                   │
  │                                 │ route propagation  │                   │
  │                                 └───────────────────┘                   │
  │                                                                         │
  │  3. DIRECT CONNECT GATEWAY      4. TGW PEERING                         │
  │  ┌───────────────────┐          ┌───────────────────┐                   │
  │  │ Connects DX       │          │ Cross-region or    │                   │
  │  │ circuits to TGW   │          │ intra-region       │                   │
  │  │ via DX Gateway    │          │ peering between    │                   │
  │  │                   │          │ two TGWs           │                   │
  │  │ DX GW aggregates  │          │                    │                   │
  │  │ multiple DX       │          │ STATIC routes only │                   │
  │  │ connections into   │          │ (no propagation    │                   │
  │  │ a single TGW      │          │  across peering)   │                   │
  │  │ attachment         │          │                    │                   │
  │  │                   │          │ Encrypted AES-256  │                   │
  │  │ Transit VIF       │          │ for inter-region   │                   │
  │  │ required (not     │          │                    │                   │
  │  │ private VIF)      │          │ 1 peering per pair │                   │
  │  └───────────────────┘          └───────────────────┘                   │
  │                                                                         │
  │  5. TGW CONNECT                                                         │
  │  ┌───────────────────┐                                                  │
  │  │ GRE tunnel over   │                                                  │
  │  │ a VPC or DX       │                                                  │
  │  │ attachment for     │                                                  │
  │  │ SD-WAN appliances │                                                  │
  │  │                   │                                                  │
  │  │ Higher bandwidth  │                                                  │
  │  │ than VPN (5 Gbps  │                                                  │
  │  │ per Connect peer) │                                                  │
  │  │                   │                                                  │
  │  │ Supports BGP for  │                                                  │
  │  │ dynamic routing   │                                                  │
  │  └───────────────────┘                                                  │
  └─────────────────────────────────────────────────────────────────────────┘
```

### VPC Attachment Details

The VPC attachment is the most common type. Key details that matter for implementation:

- **One subnet per AZ, one ENI per subnet.** You must specify subnets during attachment creation. If a VPC spans 3 AZs, specify one subnet in each AZ. Traffic from any subnet in that AZ reaches TGW via the ENI in the TGW attachment subnet.
- **No additional cross-AZ data transfer charges.** Unlike NAT Gateways, there are no extra cross-AZ data transfer charges for traffic flowing through TGW between attachments in different AZs. You pay only the standard $0.02/GB TGW data processing charge regardless of whether traffic stays within the same AZ or crosses AZs.
- **VPC attachment bandwidth: up to 50 Gbps per attachment.** AWS documentation states up to 50 Gbps per VPC, Direct Connect gateway, or peered TGW connection. This scales automatically -- you do not provision capacity.

---

## Part 3: TGW Route Tables -- The Terminal Flight Boards

Route tables are where the real power of Transit Gateway lives. The Feb 20 doc introduced association and propagation conceptually. Here is the mechanical depth.

### Default Route Table Behavior

When you create a TGW, AWS creates a **default route table**. If you leave `default_route_table_association` and `default_route_table_propagation` enabled:

- Every new attachment is **automatically associated** with the default route table
- Every new attachment **automatically propagates** its routes to the default route table
- Result: **every VPC can reach every other VPC** -- a flat network with no segmentation

This is why production TGW deployments should **always disable both defaults** and create explicit route tables. You do not want a newly attached dev VPC to suddenly be reachable from production.

### Association vs Propagation -- The Mechanical Details

```
ASSOCIATION AND PROPAGATION -- THE MECHANICS
=============================================================================

  ASSOCIATION = "Which route table evaluates this attachment's OUTBOUND traffic?"
  ────────────────────────────────────────────────────
  ├── Each attachment is associated with EXACTLY ONE route table
  ├── When a packet arrives at TGW from this attachment, TGW looks up
  │   the associated route table to decide where to forward
  ├── You CANNOT associate an attachment with multiple route tables
  │   (the attachment has one and only one "departure terminal")
  └── Changing the association moves the attachment to a different table

  PROPAGATION = "Which route tables learn about this attachment's CIDR?"
  ────────────────────────────────────────────────────
  ├── An attachment can propagate to MANY route tables simultaneously
  ├── When an attachment propagates to a route table, TGW adds a
  │   DYNAMIC route entry: the attachment's CIDR → the attachment
  ├── Dynamic routes are updated automatically when VPC CIDRs change
  ├── VPN and DX attachments propagate learned BGP routes
  └── TGW peering attachments do NOT support propagation (static only)


  THE SEGMENTATION PRINCIPLE:
  ════════════════════════════════════════════════════════

  Isolation is achieved through the ABSENCE of routes.

  If prod-vpc propagates to prod-rt but NOT to dev-rt,
  then dev VPCs (associated with dev-rt) have no route to prod-vpc.
  Traffic destined for prod from dev matches no route and is DROPPED.

  You do not need "deny rules." You build connectivity by adding
  propagations. Everything else is denied by default.


  ROUTE PRIORITY RULES:
  ────────────────────────────────────────────────────
  1. Static routes take priority over propagated routes
  2. For overlapping CIDRs: most specific prefix wins (longest match)
  3. For identical CIDRs from different sources:
     ├── Static > propagated
     ├── VPN/DX BGP routes evaluated by AS path length, MED, etc.
     └── If still tied: VPC > DX > VPN (for propagated routes)
```

### Static Routes and Blackhole Routes

Not all routes come from propagation. Two critical static route types:

```
STATIC AND BLACKHOLE ROUTES
=============================================================================

  STATIC ROUTE:
  ├── Manually added route in a TGW route table
  ├── Points a CIDR to a specific attachment
  ├── Takes priority over propagated routes for the same CIDR
  ├── Use cases:
  │   ├── Default route (0.0.0.0/0) to an egress VPC
  │   ├── Routes to TGW peering attachments (propagation not supported)
  │   ├── Override a propagated route for traffic engineering
  │   └── Route to an inspection VPC for centralized security

  BLACKHOLE ROUTE:
  ├── A route that drops all matching traffic silently
  ├── No target attachment — packets matching this route are discarded
  ├── Use cases:
  │   ├── Explicitly block specific CIDRs (defense in depth)
  │   ├── Prevent RFC 1918 traffic from leaking to the internet:
  │   │   10.0.0.0/8     → blackhole
  │   │   172.16.0.0/12  → blackhole
  │   │   192.168.0.0/16 → blackhole
  │   │   (add these to the egress route table so spoke-to-spoke
  │   │    traffic does not accidentally flow through the NAT Gateway)
  │   └── Temporarily isolate a compromised VPC during incident response
  │       (add a blackhole for its CIDR in all route tables)
```

---

## Part 4: Network Segmentation Patterns

### Pattern 1: Shared Services VPC Model

The most common enterprise pattern. A shared-services VPC hosts DNS, monitoring, CI/CD, VPC endpoints, and other centralized resources. Spoke VPCs (prod, dev, staging) can reach shared services but are isolated from each other based on environment.

```
SHARED SERVICES PATTERN — THREE ROUTE TABLE DESIGN
=============================================================================

  ┌─────────────────────────────────────────────────────────────────────┐
  │                        TRANSIT GATEWAY                              │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────┐      │
  │  │  PROD-RT (associated with: prod-app, prod-data)           │      │
  │  │  ┌───────────────────────────────────────────────────┐    │      │
  │  │  │  10.0.0.0/16  → prod-app      (propagated)       │    │      │
  │  │  │  10.1.0.0/16  → prod-data     (propagated)       │    │      │
  │  │  │  10.2.0.0/16  → shared-svc    (propagated)       │    │      │
  │  │  │  172.16.0.0/12 → vpn-attach   (propagated, BGP)  │    │      │
  │  │  └───────────────────────────────────────────────────┘    │      │
  │  │  Prod VPCs can reach: each other, shared-services, on-prem│      │
  │  │  Prod VPCs CANNOT reach: dev, staging                     │      │
  │  └───────────────────────────────────────────────────────────┘      │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────┐      │
  │  │  DEV-RT (associated with: dev-app, dev-data)              │      │
  │  │  ┌───────────────────────────────────────────────────┐    │      │
  │  │  │  10.8.0.0/16  → dev-app       (propagated)       │    │      │
  │  │  │  10.9.0.0/16  → dev-data      (propagated)       │    │      │
  │  │  │  10.2.0.0/16  → shared-svc    (propagated)       │    │      │
  │  │  └───────────────────────────────────────────────────┘    │      │
  │  │  Dev VPCs can reach: each other, shared-services           │      │
  │  │  Dev VPCs CANNOT reach: prod, staging, on-prem             │      │
  │  └───────────────────────────────────────────────────────────┘      │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────┐      │
  │  │  SHARED-RT (associated with: shared-svc, vpn-attachment)  │      │
  │  │  ┌───────────────────────────────────────────────────┐    │      │
  │  │  │  10.0.0.0/16  → prod-app      (propagated)       │    │      │
  │  │  │  10.1.0.0/16  → prod-data     (propagated)       │    │      │
  │  │  │  10.2.0.0/16  → shared-svc    (propagated)       │    │      │
  │  │  │  10.8.0.0/16  → dev-app       (propagated)       │    │      │
  │  │  │  10.9.0.0/16  → dev-data      (propagated)       │    │      │
  │  │  │  172.16.0.0/12 → vpn-attach   (propagated, BGP)  │    │      │
  │  │  └───────────────────────────────────────────────────┘    │      │
  │  │  Shared-services can reach: ALL VPCs + on-prem             │      │
  │  └───────────────────────────────────────────────────────────┘      │
  └─────────────────────────────────────────────────────────────────────┘

  ISOLATION VERIFICATION:
  ├── dev-app (10.8.0.0/16) sends packet to prod-app (10.0.0.0/16)
  ├── TGW looks up dev-app's associated route table: DEV-RT
  ├── DEV-RT has no route for 10.0.0.0/16 (prod-app did NOT propagate to DEV-RT)
  ├── No matching route → packet is DROPPED
  └── Isolation confirmed without any explicit deny rules
```

### Pattern 2: Isolated Prod/Dev with Centralized Egress

Combines network segmentation with a single internet exit point.

```
ISOLATED ENVIRONMENTS + CENTRALIZED EGRESS
=============================================================================

  ┌──────────────────────────────────────────────────────────────────────┐
  │                        TRANSIT GATEWAY                               │
  │                                                                      │
  │  SPOKE-RT (associated with: prod-app, prod-data, dev-app)           │
  │  ┌────────────────────────────────────────────────────────────┐      │
  │  │  10.0.0.0/16  → prod-app        (propagated)              │      │
  │  │  10.1.0.0/16  → prod-data       (propagated)              │      │
  │  │  10.2.0.0/16  → shared-svc      (propagated)              │      │
  │  │  10.8.0.0/16  → dev-app         (propagated)              │      │
  │  │  0.0.0.0/0    → egress-attach   (STATIC)                  │      │
  │  │                                                            │      │
  │  │  NOTE: If you want to BLOCK spoke-to-spoke traffic while   │      │
  │  │  still allowing egress, add blackhole routes:              │      │
  │  │  10.0.0.0/8   → blackhole (blocks all internal traffic)   │      │
  │  │  This works because of longest-prefix-match: a /16        │      │
  │  │  propagated route is more specific than the /8 blackhole,  │      │
  │  │  so traffic to known VPCs uses the propagated route, while │      │
  │  │  traffic to unknown internal IPs hits the /8 blackhole     │      │
  │  │  and is dropped. 0.0.0.0/0 catches internet-bound traffic. │      │
  │  └────────────────────────────────────────────────────────────┘      │
  │                                                                      │
  │  EGRESS-RT (associated with: egress-vpc-attachment)                  │
  │  ┌────────────────────────────────────────────────────────────┐      │
  │  │  10.0.0.0/16  → prod-app        (propagated)              │      │
  │  │  10.1.0.0/16  → prod-data       (propagated)              │      │
  │  │  10.2.0.0/16  → shared-svc      (propagated)              │      │
  │  │  10.8.0.0/16  → dev-app         (propagated)              │      │
  │  │                                                            │      │
  │  │  NO default route here — the egress VPC uses its own       │      │
  │  │  subnet route tables to reach the internet via IGW/NAT     │      │
  │  └────────────────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Centralized Egress Pattern -- One Customs Hall for the Airport

### The Analogy

Instead of every airline building its own customs and immigration hall (NAT Gateway per VPC), the airport has **one centralized international departures terminal**. All outbound passengers (internet-bound traffic) are routed to this terminal, pass through customs (NAT Gateway), and exit to the world. Return traffic enters through the same terminal and is routed back to the correct airline gate.

The benefit: one customs operation to manage, one set of security rules, one logging point. The cost: every passenger transits through the hub ($0.02/GB processing fee), and there is a slight detour for cross-AZ transfers.

### The Architecture

```
CENTRALIZED EGRESS — FULL TRAFFIC FLOW
=============================================================================

  SPOKE VPC (10.0.0.0/16)         EGRESS VPC (10.100.0.0/16)
  ┌────────────────────────┐      ┌──────────────────────────────────┐
  │                        │      │                                  │
  │  Instance 10.0.5.20    │      │  TGW Subnet    Public Subnet    │
  │  wants to reach        │      │  10.100.0.0/28  10.100.1.0/24   │
  │  8.8.8.8 (internet)    │      │                                  │
  │                        │      │  ┌──────┐      ┌──────┐         │
  │  Subnet RT:            │      │  │TGW   │      │ NAT  │         │
  │  0.0.0.0/0 → tgw      │      │  │ENI   │─────▶│ GW   │────▶IGW │
  │                        │      │  └──────┘      └──────┘         │
  │  Packet → TGW ENI      │      │                                  │
  │            │            │      │  TGW Subnet RT:                 │
  └────────────┼───────────┘      │  0.0.0.0/0 → nat-gw             │
               │                  │                                  │
               ▼                  │  Public Subnet RT:               │
        ┌──────────────┐          │  0.0.0.0/0 → igw                │
        │ TGW          │          │  10.0.0.0/8 → tgw               │
        │              │          │  (return traffic to spokes)      │
        │ SPOKE-RT:    │          │                                  │
        │ 0.0.0.0/0 → │──────────│▶ egress VPC attachment           │
        │ egress-att   │          │                                  │
        │              │          └──────────────────────────────────┘
        │ EGRESS-RT:   │
        │ 10.0.0.0/16→ │
        │ spoke-att    │          RETURN PATH:
        │ (propagated) │          8.8.8.8 responds → IGW → NAT GW
        └──────────────┘          → TGW subnet RT matches 10.0.0.0/8 → tgw
                                  → TGW EGRESS-RT matches 10.0.0.0/16
                                  → spoke-att → instance 10.0.5.20

  CRITICAL ROUTE TABLE ENTRIES (4 route tables involved):
  ════════════════════════════════════════════════════════

  1. SPOKE VPC private subnet RT:
     0.0.0.0/0 → tgw-xxxxxxxxx

  2. TGW SPOKE-RT (associated with spoke attachments):
     0.0.0.0/0 → egress-vpc-attachment (STATIC)
     10.0.0.0/16 → spoke-attachment (propagated, for return traffic)

  3. EGRESS VPC TGW subnet RT:
     0.0.0.0/0 → nat-gateway

  4. EGRESS VPC public subnet RT:
     0.0.0.0/0 → igw
     10.0.0.0/8 → tgw-xxxxxxxxx (return traffic to ALL spokes)
```

### Terraform: Centralized Egress

```hcl
# ──────────────────────────────────────────────────────────
# Egress VPC — centralized internet exit point
# ──────────────────────────────────────────────────────────

resource "aws_vpc" "egress" {
  cidr_block           = "10.100.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "egress-vpc" }
}

# Dedicated TGW subnets in the egress VPC
resource "aws_subnet" "egress_tgw" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.egress.id
  cidr_block        = local.egress_tgw_cidrs[count.index]  # /28 per AZ
  availability_zone = local.azs[count.index]

  tags = { Name = "egress-tgw-${local.azs[count.index]}" }
}

# Public subnets for NAT Gateways and IGW
resource "aws_subnet" "egress_public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.egress.id
  cidr_block        = local.egress_public_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = { Name = "egress-public-${local.azs[count.index]}" }
}

# TGW attachment for egress VPC
resource "aws_ec2_transit_gateway_vpc_attachment" "egress" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.egress.id
  subnet_ids         = aws_subnet.egress_tgw[*].id

  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  tags = { Name = "egress-vpc-attachment" }
}

# ──────────────────────────────────────────────────────────
# TGW Route Tables for Centralized Egress
# ──────────────────────────────────────────────────────────

# Spoke route table: default route to egress VPC
resource "aws_ec2_transit_gateway_route" "spoke_default" {
  destination_cidr_block         = "0.0.0.0/0"
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.egress.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.spoke.id
}

# Optional: blackhole RFC 1918 ranges in the egress route table
# to prevent spoke-to-spoke traffic from flowing through NAT
resource "aws_ec2_transit_gateway_route" "egress_blackhole_rfc1918_10" {
  destination_cidr_block         = "10.0.0.0/8"
  blackhole                      = true
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.egress.id
}

# Egress route table: propagate spoke CIDRs for return traffic
resource "aws_ec2_transit_gateway_route_table_association" "egress" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.egress.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.egress.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "spoke_to_egress" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.prod_app.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.egress.id
}

# ──────────────────────────────────────────────────────────
# Egress VPC Route Tables
# ──────────────────────────────────────────────────────────

# TGW subnet route table: forward internet traffic to NAT
resource "aws_route_table" "egress_tgw" {
  vpc_id = aws_vpc.egress.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.egress.id
  }

  tags = { Name = "egress-tgw-rt" }
}

# Public subnet route table: return traffic to spokes via TGW
resource "aws_route_table" "egress_public" {
  vpc_id = aws_vpc.egress.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.egress.id
  }

  # Return traffic for ALL spoke CIDRs goes back through TGW
  route {
    cidr_block         = "10.0.0.0/8"
    transit_gateway_id = aws_ec2_transit_gateway.main.id
  }

  tags = { Name = "egress-public-rt" }
}
```

### Centralized Egress Cost Trade-Off

```
COST ANALYSIS: CENTRALIZED vs DISTRIBUTED EGRESS (10 VPCs, 3 AZs each)
=============================================================================

  DISTRIBUTED (NAT per VPC):
  ├── 10 VPCs x 3 AZs x $0.045/hr x 730 hrs = $985.50/mo
  ├── Data processing: 1 TB total x $0.045/GB = $46.08/mo
  └── TOTAL: ~$1,031/mo (but each VPC pays only its own share)

  CENTRALIZED (single egress VPC):
  ├── 1 egress VPC x 3 AZs x $0.045/hr x 730 = $98.55/mo
  ├── NAT data processing: 1 TB x $0.045/GB = $46.08/mo
  ├── TGW data processing: 1 TB x $0.02/GB = $20.48/mo (ADDITIONAL COST)
  ├── TGW attachment hourly: 11 attachments x $0.05/hr x 730 = $401.50/mo
  └── TOTAL: ~$567/mo

  SAVINGS: ~$464/mo (45%) with centralized egress at 10 VPCs

  BUT: at 50+ VPCs with heavy egress, the TGW processing charge grows
  linearly. At ~10 TB/month egress, TGW processing alone = $204.80/mo,
  still cheaper than 50 NAT GWs ($4,927/mo). Centralized egress almost
  always wins on cost AND operational simplicity at scale.
```

---

## Part 6: Centralized Inspection Pattern -- The Security Checkpoint

### The Analogy

The airport has a **mandatory security checkpoint** that all passengers must pass through. Passengers arriving at one gate (spoke VPC) destined for another gate (destination VPC) must first go through security in a dedicated inspection terminal.

Here is the key requirement: each passenger is assigned a **specific security agent** who handles both their departure screening and their arrival customs. The same agent processes both directions of the journey. If a different agent handled the return trip, they would have no record of the original screening and would reject the passenger. This is **appliance mode** -- pinning both directions of a flow to the same AZ's firewall so stateful inspection works.

### Appliance Mode -- Why It Matters

```
APPLIANCE MODE — SYMMETRIC ROUTING
=============================================================================

  WITHOUT APPLIANCE MODE (BROKEN):
  ────────────────────────────────────────────────────

  VPC-A (AZ-1) → TGW → Inspection VPC Firewall (AZ-1) → TGW → VPC-B (AZ-2)
  VPC-B (AZ-2) → TGW → Inspection VPC Firewall (AZ-2) → TGW → VPC-A (AZ-1)
                                                   ↑
                                         DIFFERENT AZ's firewall!

  The return packet goes through a DIFFERENT firewall instance in AZ-2.
  That firewall has no record of the original connection from AZ-1.
  Stateful inspection FAILS — the return packet is DROPPED.


  WITH APPLIANCE MODE (CORRECT):
  ────────────────────────────────────────────────────

  VPC-A (AZ-1) → TGW → Inspection VPC Firewall (AZ-1) → TGW → VPC-B (AZ-2)
  VPC-B (AZ-2) → TGW → Inspection VPC Firewall (AZ-1) → TGW → VPC-A (AZ-1)
                                                   ↑
                                         SAME AZ's firewall!

  TGW uses a flow hash (source/dest IP) to pin the entire connection
  to a single AZ's network interface. Both directions of the flow
  traverse the same firewall instance. Stateful inspection works.


  WHEN TO ENABLE APPLIANCE MODE:
  ├── ALWAYS when routing traffic through a stateful firewall (Network
  │   Firewall, Palo Alto, Fortinet) via a TGW attachment
  ├── ALWAYS when using Gateway Load Balancer for inspection
  └── NEVER for normal VPC-to-VPC traffic without inspection
      (appliance mode prevents AZ-affinity optimization)
```

### Inspection Architecture -- Two Subnets Per AZ

```
CENTRALIZED INSPECTION VPC — ARCHITECTURE
=============================================================================

  INSPECTION VPC (10.200.0.0/16)
  ┌──────────────────────────────────────────────────────────────────────┐
  │                                                                      │
  │  AZ-1                              AZ-2                             │
  │  ┌─────────────────────────┐       ┌─────────────────────────┐      │
  │  │ TGW Subnet              │       │ TGW Subnet              │      │
  │  │ 10.200.0.0/28           │       │ 10.200.0.16/28          │      │
  │  │ ┌─────────────┐         │       │ ┌─────────────┐         │      │
  │  │ │ TGW ENI     │         │       │ │ TGW ENI     │         │      │
  │  │ └──────┬──────┘         │       │ └──────┬──────┘         │      │
  │  └────────┼────────────────┘       └────────┼────────────────┘      │
  │           │ TGW subnet RT:                  │                       │
  │           │ 0.0.0.0/0 → firewall-endpoint   │                       │
  │           ▼                                  ▼                       │
  │  ┌─────────────────────────┐       ┌─────────────────────────┐      │
  │  │ Firewall Subnet         │       │ Firewall Subnet         │      │
  │  │ 10.200.1.0/24           │       │ 10.200.2.0/24           │      │
  │  │ ┌─────────────────────┐ │       │ ┌─────────────────────┐ │      │
  │  │ │ Network Firewall    │ │       │ │ Network Firewall    │ │      │
  │  │ │ Endpoint            │ │       │ │ Endpoint            │ │      │
  │  │ └─────────────────────┘ │       │ └─────────────────────┘ │      │
  │  └─────────────────────────┘       └─────────────────────────┘      │
  │                                                                      │
  │  WHY TWO SUBNETS PER AZ:                                            │
  │  ├── Network Firewall cannot inspect traffic in the same subnet      │
  │  │   as its endpoint — this is an AWS architectural constraint       │
  │  ├── TGW subnet: receives traffic from TGW, routes to firewall      │
  │  ├── Firewall subnet: hosts the firewall endpoint                    │
  │  └── After inspection, traffic returns to TGW through the TGW subnet│
  └──────────────────────────────────────────────────────────────────────┘
```

### Terraform: Appliance Mode on Inspection VPC Attachment

```hcl
# Inspection VPC attachment — appliance mode ENABLED
resource "aws_ec2_transit_gateway_vpc_attachment" "inspection" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.inspection.id
  subnet_ids         = aws_subnet.inspection_tgw[*].id

  # THIS IS THE CRITICAL SETTING for centralized inspection
  appliance_mode_support = "enable"

  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  tags = { Name = "inspection-vpc-attachment" }
}

# Spoke route table: route all inter-VPC traffic to inspection
resource "aws_ec2_transit_gateway_route" "spoke_to_inspection" {
  destination_cidr_block         = "10.0.0.0/8"
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.inspection.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.spoke.id
}

# Inspection route table: propagate spoke CIDRs for post-inspection forwarding
resource "aws_ec2_transit_gateway_route_table" "inspection" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "inspection-rt" }
}

resource "aws_ec2_transit_gateway_route_table_association" "inspection" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.inspection.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.inspection.id
}

# All spoke VPCs propagate to the inspection route table
# so inspected traffic can reach the final destination
resource "aws_ec2_transit_gateway_route_table_propagation" "prod_to_inspection" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.prod_app.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.inspection.id
}
```

---

## Part 7: GWLB + TGW -- Third-Party Appliance Inspection

When AWS Network Firewall is insufficient and you need third-party appliances (Palo Alto, Fortinet, Check Point), Gateway Load Balancer provides transparent Layer 3 load balancing with GENEVE encapsulation.

```
GWLB + TGW ARCHITECTURE
=============================================================================

  SPOKE VPC → TGW → SECURITY VPC → GWLB Endpoint → GWLB → Appliances
                                                              │
                                                              ▼
                                                     Inspect + Forward
                                                              │
                                                              ▼
  DEST VPC  ← TGW ← SECURITY VPC ← GWLB Endpoint ← GWLB ← ┘

  KEY DIFFERENCES FROM NETWORK FIREWALL:
  ┌──────────────────────────────────────────────────────────────────┐
  │                    Network Firewall       GWLB + Appliances     │
  ├──────────────────────────────────────────────────────────────────┤
  │  Management         AWS-managed           Self-managed           │
  │  Appliance type     AWS-only              Any vendor (GENEVE)    │
  │  Scaling            Automatic             You scale EC2 fleet    │
  │  Stateful rules     AWS rule language     Vendor-specific        │
  │  IDS/IPS            Yes (Suricata-based)  Vendor-dependent       │
  │  Cost               $0.395/endpoint/hr    $0.014/GWLB/AZ-hr +   │
  │                     + $0.065/GB           $0.011/GWLBE/AZ-hr +  │
  │                                           GLCU usage +           │
  │                                           EC2 instance costs     │
  │  Encapsulation      N/A                   GENEVE (transparent)   │
  │  TGW appliance mode Required              Required               │
  │  Max throughput      100 Gbps/endpoint    100 Gbps/GWLB         │
  └──────────────────────────────────────────────────────────────────┘

  GWLB FLOW STICKINESS:
  ├── 5-tuple hash (src IP, dst IP, src port, dst port, protocol)
  │   for TCP/UDP flows — same session always hits the same appliance
  ├── 3-tuple hash (src IP, dst IP, protocol) for non-TCP/UDP
  └── Combined with TGW appliance mode, ensures full symmetric routing
      through the same AZ and the same appliance instance
```

---

## Part 8: Inter-Region Peering -- Bilateral Flight Agreements

### The Analogy

Your airport in us-east-1 wants to route passengers to an airport in eu-west-1. The two airports negotiate a **bilateral flight agreement** (TGW peering attachment). Unlike domestic flights where destinations are automatically posted on the flight board (propagation), international agreements require **manually posting destinations** (static routes only). Each airport must explicitly add the other's destinations to its flight boards.

### The Technical Reality

```
TGW INTER-REGION PEERING
=============================================================================

  us-east-1 TGW                          eu-west-1 TGW
  ┌─────────────────────┐                ┌─────────────────────┐
  │                     │   PEERING      │                     │
  │  VPC-A 10.0.0.0/16 │   ATTACHMENT   │  VPC-X 10.64.0.0/16│
  │  VPC-B 10.1.0.0/16 │◄──────────────▶│  VPC-Y 10.65.0.0/16│
  │                     │   (encrypted)  │                     │
  │  Route table:       │                │  Route table:       │
  │  10.64.0.0/14 →     │                │  10.0.0.0/14 →      │
  │    peering-attach    │                │    peering-attach    │
  │  (STATIC — manual)  │                │  (STATIC — manual)  │
  └─────────────────────┘                └─────────────────────┘

  KEY CONSTRAINTS:
  ├── STATIC ROUTES ONLY — no dynamic propagation across peering
  │   You must manually add routes in BOTH TGWs' route tables
  │   pointing to the peering attachment for the remote CIDRs
  │
  ├── ONE PEERING PER PAIR — two TGWs can have at most one peering
  │   attachment between them (no multi-path)
  │
  ├── UNIQUE ASNs RECOMMENDED — each TGW should have a different
  │   amazon_side_asn to avoid BGP confusion (even though BGP is
  │   not used across peering, it matters for VPN/DX attachments)
  │
  ├── ENCRYPTION — inter-region peering traffic is encrypted with
  │   AES-256 at the virtual network layer; physically encrypted again
  │   on links outside AWS's physical control (double encryption)
  │
  ├── DNS — private hosted zone resolution does NOT work across
  │   inter-region TGW peering; use Route 53 Resolver endpoints
  │   with forwarding rules (covered in Week 3)
  │
  ├── BANDWIDTH — limited by the inter-region bandwidth available
  │   between the two regions (AWS backbone); no published hard limit
  │   per peering attachment, but aggregate throughput depends on
  │   the region pair
  │
  └── MTU — 8,500 bytes (same as VPC attachments)
```

### Terraform: Inter-Region TGW Peering

```hcl
# ──────────────────────────────────────────────────────────
# In us-east-1: Create peering request to eu-west-1 TGW
# ──────────────────────────────────────────────────────────

resource "aws_ec2_transit_gateway_peering_attachment" "us_to_eu" {
  transit_gateway_id      = aws_ec2_transit_gateway.us_east.id
  peer_transit_gateway_id = var.eu_west_tgw_id
  peer_region             = "eu-west-1"
  peer_account_id         = var.eu_west_account_id  # Same or different account

  tags = { Name = "us-east-to-eu-west-peering" }
}

# ──────────────────────────────────────────────────────────
# In eu-west-1: Accept the peering request
# ──────────────────────────────────────────────────────────

resource "aws_ec2_transit_gateway_peering_attachment_accepter" "eu_accepts" {
  provider                      = aws.eu_west
  transit_gateway_attachment_id = aws_ec2_transit_gateway_peering_attachment.us_to_eu.id

  tags = { Name = "us-east-to-eu-west-peering-accepted" }
}

# ──────────────────────────────────────────────────────────
# STATIC ROUTES — must be added on BOTH sides
# ──────────────────────────────────────────────────────────

# In us-east-1: route eu-west CIDRs to peering attachment
resource "aws_ec2_transit_gateway_route" "us_to_eu_routes" {
  destination_cidr_block         = "10.64.0.0/14"  # All eu-west VPCs
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_peering_attachment.us_to_eu.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.us_shared.id
}

# In eu-west-1: route us-east CIDRs to peering attachment
resource "aws_ec2_transit_gateway_route" "eu_to_us_routes" {
  provider                       = aws.eu_west
  destination_cidr_block         = "10.0.0.0/14"   # All us-east VPCs
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_peering_attachment.us_to_eu.id
  transit_gateway_route_table_id = var.eu_west_shared_rt_id
}

# IMPORTANT: Add the static route to EVERY route table that needs
# cross-region access. If prod-rt needs to reach eu-west, add the
# route to prod-rt as well, not just shared-rt.
```

---

## Part 9: Multi-Account with RAM -- Franchising the Airport

### The Analogy

The airport is owned and operated by a central network team (the networking account). Other business units (spoke accounts) want to use the airport but should not have access to air traffic control. AWS Resource Access Manager (RAM) is the **franchise agreement**: it lets spoke accounts create their own gates (VPC attachments) at the airport, but only the airport operator can modify the terminals (route tables) and flight boards (routes).

### The Technical Reality

```
MULTI-ACCOUNT TGW WITH RAM
=============================================================================

  NETWORKING ACCOUNT (owns the TGW):
  ├── Creates the Transit Gateway
  ├── Creates TGW route tables
  ├── Manages all route table associations and propagations
  ├── Shares TGW via RAM to the AWS Organization (or specific OUs/accounts)
  └── Sets auto_accept_shared_attachments = "enable" for seamless onboarding

  SPOKE ACCOUNTS (use the TGW):
  ├── See the shared TGW in their account (via RAM)
  ├── Create VPC attachments to the shared TGW
  ├── CANNOT create, modify, or delete TGW route tables
  ├── CANNOT create, modify, or delete routes or associations
  └── Can only manage their own VPC attachment (create/delete)

  WORKFLOW:
  ┌──────────────────────────────────────────────────────────────────┐
  │  1. Network team creates TGW in networking account               │
  │  2. Network team shares TGW via RAM to the Organization          │
  │  3. Spoke account creates VPC attachment to the shared TGW       │
  │  4. If auto_accept enabled: attachment is immediately active     │
  │     If auto_accept disabled: network team must accept manually   │
  │  5. Network team associates the attachment with the correct      │
  │     route table and configures propagation rules                 │
  │  6. Spoke account adds routes in their VPC route tables          │
  │     pointing to the TGW for cross-VPC traffic                   │
  └──────────────────────────────────────────────────────────────────┘
```

### Terraform: RAM Sharing

```hcl
# ──────────────────────────────────────────────────────────
# Networking Account: Share TGW via RAM
# ──────────────────────────────────────────────────────────

resource "aws_ram_resource_share" "tgw_share" {
  name                      = "transit-gateway-org-share"
  allow_external_principals = false  # Organization members only

  tags = { Name = "tgw-organization-share" }
}

resource "aws_ram_resource_association" "tgw" {
  resource_arn       = aws_ec2_transit_gateway.main.arn
  resource_share_arn = aws_ram_resource_share.tgw_share.arn
}

# Share with the entire organization
resource "aws_ram_principal_association" "org" {
  principal          = var.organization_arn  # arn:aws:organizations::MASTER_ACCT:organization/o-xxxxx
  resource_share_arn = aws_ram_resource_share.tgw_share.arn
}

# OR share with a specific OU
resource "aws_ram_principal_association" "production_ou" {
  principal          = var.production_ou_arn  # arn:aws:organizations::MASTER_ACCT:ou/o-xxxxx/ou-xxxxx
  resource_share_arn = aws_ram_resource_share.tgw_share.arn
}

# ──────────────────────────────────────────────────────────
# Spoke Account: Attach VPC to the shared TGW
# ──────────────────────────────────────────────────────────

# The spoke account references the TGW by ID (visible via RAM sharing)
resource "aws_ec2_transit_gateway_vpc_attachment" "spoke" {
  provider           = aws.spoke_account
  transit_gateway_id = var.shared_tgw_id  # TGW ID from the networking account
  vpc_id             = aws_vpc.spoke.id
  subnet_ids         = aws_subnet.spoke_tgw[*].id

  # These must be false — the networking account manages associations
  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  tags = { Name = "spoke-vpc-attachment" }
}

# ──────────────────────────────────────────────────────────
# Back in Networking Account: Associate and propagate the spoke
# ──────────────────────────────────────────────────────────

# Network team associates the spoke attachment with the correct RT
resource "aws_ec2_transit_gateway_route_table_association" "spoke" {
  transit_gateway_attachment_id  = var.spoke_attachment_id  # From the spoke account
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "spoke_to_prod" {
  transit_gateway_attachment_id  = var.spoke_attachment_id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "spoke_to_shared" {
  transit_gateway_attachment_id  = var.spoke_attachment_id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.shared.id
}
```

---

## Part 10: Cost Model

```
TGW COST BREAKDOWN (us-east-1, February 2026)
=============================================================================

  TWO COST COMPONENTS:
  ────────────────────────────────────────────────────

  1. ATTACHMENT HOURLY: $0.05 per attachment per hour
     ├── Applies to ALL attachment types (VPC, VPN, DX GW, peering, Connect)
     ├── 10 VPC attachments = $0.50/hr = $365/month
     ├── 50 VPC attachments = $2.50/hr = $1,825/month
     └── This is a fixed cost regardless of traffic volume

  2. DATA PROCESSING: $0.02 per GB processed
     ├── Charged for traffic flowing through the TGW
     ├── Charged ONCE per flow (not double-charged for ingress + egress)
     ├── EXCEPTION: data sent FROM a peering attachment TO a TGW is NOT
     │   charged data processing fees (important for inter-region costs)
     ├── 1 TB/month = $20.48
     ├── 10 TB/month = $204.80
     └── This scales linearly with traffic volume


  COST OPTIMIZATION STRATEGIES:
  ────────────────────────────────────────────────────

  1. USE INTRA-REGION VPC PEERING FOR HIGH-BANDWIDTH PAIRS
     If two VPCs exchange > 5 TB/month, the TGW processing cost
     ($0.02/GB x 5,120 GB = $102.40/mo) may justify a direct
     peering connection ($0/month) even if they also use TGW
     for other routes. Add the peering connection AND keep TGW
     as a backup; longest-prefix-match routing will prefer peering.

  2. CONSOLIDATE VPN TUNNELS AT TGW
     Instead of 10 VPCs each with their own VPN ($0.05/hr x 10 = $365/mo),
     one TGW VPN attachment ($0.05/hr = $36.50/mo) serves all 10 VPCs.
     Savings: $328.50/mo minus the TGW attachment costs for the VPCs.

  3. DO NOT CREATE UNUSED ATTACHMENTS
     TGW charges per attachment even if no traffic flows. Remove
     attachments for decommissioned VPCs immediately.

  4. USE TGW FLOW LOGS TO IDENTIFY TRAFFIC PATTERNS
     Enable TGW flow logs to see which attachment pairs generate
     the most traffic. Target those pairs for optimization.


  EXAMPLE: 20-VPC ENTERPRISE (MONTHLY COST)
  ────────────────────────────────────────────────────

  Attachments:
  ├── 20 VPC attachments x $0.05/hr x 730 hrs     = $730.00
  ├── 1 VPN attachment                              = $36.50
  ├── 1 DX Gateway attachment                       = $36.50
  ├── 1 peering attachment (cross-region)            = $36.50
  └── Subtotal (hourly):                            = $839.50

  Data processing:
  ├── East-west (VPC-to-VPC): 2 TB x $0.02/GB      = $40.96
  ├── Egress (VPC-to-internet via egress VPC): 5 TB  = $102.40
  ├── Hybrid (VPC-to-on-prem): 1 TB                 = $20.48
  └── Subtotal (processing):                        = $163.84

  TOTAL: ~$1,003/month
  Compared to distributed model (20 NAT GWs + 190 peering connections):
  operational complexity would be 10x higher even if AWS costs were similar.
```

---

## Part 11: Key Quotas

```
TGW QUOTAS — KNOW THESE FOR INTERVIEWS AND DESIGN
=============================================================================

  RESOURCE                                  LIMIT           ADJUSTABLE?
  ──────────────────────────────────────    ──────────      ───────────
  TGWs per account per region               5               Yes
  Attachments per TGW                       5,000           Yes
  Route tables per TGW                      20              Yes
  Routes per TGW (TOTAL, across ALL RTs)    10,000          Yes (Service Quotas)
  Static routes per route table             10,000          Yes (Service Quotas)
  Peering attachments per TGW               50              Yes
  Peering between any two TGWs              1               No
  Bandwidth per VPC attachment              50 Gbps         No (auto-scales)
  Packets per second per attachment         5 million       No
  MTU (VPC, DX, peering attachments)        8,500 bytes     No
  MTU (VPN attachments)                     1,500 bytes     No

  THE GOTCHA: 10,000 TOTAL ROUTES
  ────────────────────────────────────────────────────

  The 10,000 route limit is shared across ALL route tables on a TGW.
  If you have 20 route tables, that is 500 routes per table on average.

  Example that hits the limit:
  ├── 200 VPC attachments, each propagating to 5 route tables
  ├── 200 x 5 = 1,000 propagated route entries
  ├── Plus static routes for default routes, blackholes, peering
  ├── At 500 VPCs, you are well over 10,000
  └── Solution: use summary routes (/14 instead of /16) and reduce
      the number of route tables, or request an increase via the
      Service Quotas console

  BANDWIDTH DETAIL:
  ────────────────────────────────────────────────────

  50 Gbps is the per-attachment aggregate bandwidth limit. This means:
  ├── A VPC attachment has up to 50 Gbps total, regardless of AZ count
  ├── Individual flows are limited by instance-level networking
  ├── TGW itself does not become a bandwidth bottleneck for
  │   typical enterprise workloads
  └── For comparison: VPN attachments are 1.25 Gbps per tunnel;
      use ECMP with multiple VPN connections to aggregate (up to ~50 Gbps)
```

---

## Part 12: Operational Tools and Gotchas

### TGW Flow Logs

TGW Flow Logs capture metadata about traffic flowing through your Transit Gateway. Unlike VPC Flow Logs (which capture at the ENI level), TGW Flow Logs capture at the attachment level — showing which attachment sent traffic to which attachment.

```hcl
# Enable TGW Flow Logs to S3
resource "aws_ec2_transit_gateway_flow_log" "main" {
  transit_gateway_id             = aws_ec2_transit_gateway.main.id
  log_destination                = aws_s3_bucket.tgw_flow_logs.arn
  log_destination_type           = "s3"
  max_aggregation_interval       = 60  # seconds (60 or 600)

  tags = { Name = "tgw-flow-logs" }
}

# Or send to CloudWatch Logs for real-time analysis
resource "aws_ec2_transit_gateway_flow_log" "cloudwatch" {
  transit_gateway_id             = aws_ec2_transit_gateway.main.id
  log_destination                = aws_cloudwatch_log_group.tgw_flows.arn
  log_destination_type           = "cloud-watch-logs"
  iam_role_arn                   = aws_iam_role.tgw_flow_log_role.arn
  max_aggregation_interval       = 60

  tags = { Name = "tgw-flow-logs-cw" }
}
```

Key differences from VPC Flow Logs:
- **Scope**: TGW Flow Logs show inter-attachment traffic patterns; VPC Flow Logs show per-ENI traffic
- **Use case**: Identify which VPC pairs generate the most cross-VPC traffic (cost optimization) and verify that segmentation is working (no traffic between isolated environments)
- **Destination**: S3 or CloudWatch Logs (same as VPC Flow Logs)

### TGW Route Analyzer

The **Transit Gateway Route Analyzer** is a built-in troubleshooting tool for complex TGW deployments. It analyzes whether a specific source-destination pair has a valid route path through the TGW.

Use it when:
- Debugging why traffic between two VPCs is not flowing
- Verifying that segmentation is working (expecting a "no route" result)
- Validating route table changes before applying them in production

Access it via the VPC console → Transit Gateway → Route Analyzer, or via the `aws ec2 search-transit-gateway-routes` CLI command.

### DNS Across TGW-Connected VPCs (Common Gotcha)

Connecting two VPCs via TGW does **not** automatically enable DNS resolution between them. If VPC-A has a Route 53 Private Hosted Zone and VPC-B is connected to VPC-A via TGW, VPC-B **cannot** resolve VPC-A's private DNS names unless you explicitly set up one of:

1. **Associate the PHZ with VPC-B** — The simplest option if both VPCs are in the same account. The PHZ must be associated with every VPC that needs to resolve its records.
2. **Route 53 Resolver rules** — For cross-account or complex topologies, create Resolver forwarding rules in a shared-services VPC and share them via RAM. This scales better than associating PHZs with hundreds of VPCs.

This gotcha applies to intra-region TGW connections. For inter-region TGW peering, PHZ resolution does not work at all — you must use Route 53 Resolver endpoints with forwarding rules (covered in Week 3).

### Multicast Support

TGW is the **only AWS networking construct that supports multicast**. VPC peering, PrivateLink, and standard VPC routing do not support multicast traffic. If your workload requires multicast (financial market data feeds, media streaming, cluster heartbeats), TGW multicast is how you do it in AWS.

- Create a TGW multicast domain and associate VPC attachments
- Register multicast group members (ENIs) as sources and receivers
- Supports both IGMPv2 for dynamic group membership and static group membership

---

## Complete Architecture: Hub-and-Spoke with Centralized Egress and Inspection

```
FULL TGW ARCHITECTURE — THE COMPLETE PICTURE
=============================================================================

                    ┌──────────────────────────────────┐
                    │         ON-PREMISES DC            │
                    │         172.16.0.0/12             │
                    └──────────────┬───────────────────┘
                                  │ VPN or Direct Connect
                                  │
  ┌───────────────────────────────┼────────────────────────────────────────┐
  │                  TRANSIT GATEWAY                                       │
  │                               │                                        │
  │  ┌────────────────────────────┼──────────────────────────────────────┐ │
  │  │                                                                   │ │
  │  │  SHARED-RT                 PROD-RT               DEV-RT          │ │
  │  │  (shared-svc,             (prod VPCs)            (dev VPCs)      │ │
  │  │   vpn/dx attach)          + shared-svc           + shared-svc    │ │
  │  │  sees: ALL                + on-prem              (no on-prem)    │ │
  │  │                                                                   │ │
  │  │  EGRESS-RT                INSPECTION-RT                          │ │
  │  │  (egress VPC)             (inspection VPC)                       │ │
  │  │  sees: all spoke CIDRs   sees: all spoke CIDRs                  │ │
  │  │  for return traffic       for post-inspection                    │ │
  │  │                           forwarding                             │ │
  │  └──────────────────────────────────────────────────────────────────┘ │
  │                                                                        │
  └────┬──────────┬──────────┬──────────┬──────────┬──────────┬───────────┘
       │          │          │          │          │          │
  ┌────▼────┐┌────▼────┐┌────▼────┐┌────▼────┐┌────▼────┐┌────▼────┐
  │PROD-APP ││PROD-DATA││SHARED   ││DEV-APP  ││EGRESS   ││INSPECT  │
  │10.0/16  ││10.1/16  ││SERVICES ││10.8/16  ││VPC      ││VPC      │
  │         ││         ││10.2/16  ││         ││10.100/16││10.200/16│
  │         ││         ││         ││         ││         ││         │
  │ App     ││ RDS     ││ DNS     ││ App     ││ NAT GW  ││ Network │
  │ servers ││ ElastiC ││ CI/CD   ││ servers ││ IGW     ││ Firewall│
  └─────────┘└─────────┘│ VPC EP  │└─────────┘└─────────┘└─────────┘
                        └─────────┘

  TRAFFIC FLOWS:
  ├── prod-app → prod-data:     PROD-RT has both CIDRs (direct via TGW)
  ├── prod-app → shared-svc:    PROD-RT has shared-svc CIDR (via TGW)
  ├── prod-app → internet:      PROD-RT: 0.0.0.0/0 → egress-attach (via TGW → NAT)
  ├── prod-app → dev-app:       PROD-RT has NO dev CIDR → DROPPED
  ├── prod-app → on-prem:       PROD-RT has 172.16.0.0/12 → vpn-attach
  ├── dev-app  → prod-app:      DEV-RT has NO prod CIDR → DROPPED
  ├── dev-app  → shared-svc:    DEV-RT has shared-svc CIDR (via TGW)
  ├── Any VPC  → inspected VPC: Traffic routed through inspection VPC
  │                              (appliance mode ensures symmetric path)
  └── us-east-1 → eu-west-1:   Static route to peering attachment
```

---

## Key Takeaways

### Architecture Fundamentals
1. **Transit Gateway is a regional Layer 3 virtual router, not a single box** -- AWS deploys TGW infrastructure redundantly across all AZs; you do not manage failover, and you never need multiple TGWs in the same region for high availability
2. **TGW places an ENI in each AZ's designated subnet** -- Use dedicated /28 subnets for TGW attachments with open NACLs; apply security filtering on your workload subnets, not on the TGW subnets
3. **Disabling default route table association and propagation is mandatory for production** -- The default behavior creates a flat network where every VPC can reach every other VPC; explicit route tables with deliberate association and propagation are required for network segmentation

### Route Table Mechanics
4. **Isolation is achieved through the absence of routes, not through deny rules** -- If VPC-A's CIDR is not propagated to the route table that VPC-B is associated with, VPC-B cannot reach VPC-A; you build connectivity by adding propagations, and everything else is denied by default
5. **Static routes override propagated routes for the same CIDR** -- Use static routes for default routes to egress VPCs, routes to peering attachments (propagation is not supported across peering), and blackhole routes for defense-in-depth isolation
6. **The 10,000 total route limit is shared across ALL route tables** -- This is the most surprising quota; at scale with many VPCs propagating to multiple route tables, you can hit this limit; use summary routes (/14 instead of /16) to conserve route capacity

### Centralized Egress
7. **Centralized egress requires four route tables: spoke VPC RT, TGW spoke-RT, egress VPC TGW subnet RT, and egress VPC public subnet RT** -- Missing any one of these breaks the traffic flow; trace the packet path end-to-end before deploying
8. **Add RFC 1918 blackhole routes in the egress TGW route table** -- This prevents spoke-to-spoke traffic from accidentally flowing through the NAT Gateway in the egress VPC; the blackhole catches any internal traffic that should not leave the TGW

### Centralized Inspection
9. **Appliance mode must be enabled on the inspection VPC attachment for stateful firewall inspection** -- Without it, TGW may route the request through AZ-1's firewall and the response through AZ-2's firewall, breaking stateful inspection because the return path traverses a different firewall instance with no connection state
10. **The inspection VPC needs two subnets per AZ** -- One for the TGW ENI and one for the firewall endpoint; Network Firewall cannot inspect traffic in the same subnet as its endpoint

### Inter-Region and Multi-Account
11. **TGW peering uses static routes only** -- There is no dynamic route propagation across peered TGWs; you must manually add routes on both sides pointing to the peering attachment, and you must update them whenever CIDRs change in either region
12. **Share TGW via RAM for multi-account architectures** -- The networking account owns and operates the TGW (route tables, associations, propagations); spoke accounts can only create and delete their own VPC attachments; this enforces centralized network governance

### Cost and Quotas
13. **TGW costs $0.05/attachment/hour plus $0.02/GB processed** -- The hourly cost is fixed regardless of traffic; the processing charge scales linearly; for 20 VPCs at 8 TB/month total cross-VPC traffic, expect approximately $1,000/month
14. **Consider direct VPC peering for high-bandwidth pairs** -- If two VPCs exchange more than 5 TB/month, the TGW processing charge ($102/month) may justify adding a direct peering connection alongside TGW; longest-prefix-match routing will prefer the peering path for that specific VPC pair
15. **Key quotas to memorize: 5,000 attachments, 20 route tables, 10,000 total routes, 50 Gbps per VPC attachment, 1,500 MTU for VPN** -- The 10,000 total routes and 1,500 VPN MTU are the most commonly overlooked constraints in production designs

---

## Further Reading

- [Transit Gateway Design Best Practices -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html)
- [Building Scalable Multi-VPC Network Infrastructure -- AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html)
- [Centralized Egress with NAT Gateway -- AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/using-nat-gateway-for-centralized-egress.html)
- [Centralized Network Security -- AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-network-security-for-vpc-to-vpc-and-on-premises-to-vpc-traffic.html)
- [Transit Gateway Peering Attachments -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html)
- [Transit Gateway Quotas -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html)
- [Centralize Network Connectivity Using TGW -- AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/centralize-network-connectivity-using-aws-transit-gateway.html)
- [Using GWLB with TGW for Centralized Security -- AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/using-gwlb-with-tg-for-cns.html)
- [VPC Peering vs Transit Gateway doc (Feb 20)](./2026-02-20-vpc-peering-vs-transit-gateway.md)
- [VPC Advanced: CIDR and Subnet Strategy doc (Feb 19)](./2026-02-19-vpc-advanced-cidr-subnet-strategy.md)

---

*Written on February 23, 2026*
