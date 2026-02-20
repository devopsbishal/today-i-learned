# VPC Peering vs Transit Gateway vs PrivateLink - The Interstate Highway Analogy

> Understanding VPC connectivity options through a real-world analogy of connecting cities in a growing nation. From point-to-point country roads (VPC Peering) to a centralized highway interchange (Transit Gateway) to private delivery tunnels that bypass the road system entirely (PrivateLink) -- each connectivity model has trade-offs in cost, complexity, and scalability that become clear when you think about how cities actually connect.

---

## TL;DR

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| VPC Peering | Country road between two cities | A direct, private road connecting exactly two cities; fast and cheap, but you need a separate road for every pair of cities |
| Full Mesh Problem (n(n-1)/2) | Every city connected to every other city by road | 10 cities need 45 roads; 100 cities need 4,950 roads -- the roadwork never ends |
| No Transitive Routing | Country roads do not chain | City A has a road to City B, and City B has a road to City C, but you cannot drive A-to-B-to-C; there is no through-traffic |
| No Edge-to-Edge Routing | Cannot borrow a neighbor's highway on-ramp | If City B has a highway exit (IGW), City A cannot use it through the peering road |
| Transit Gateway (TGW) | Central highway interchange | A massive cloverleaf interchange in the middle of the nation; every city builds one road to the interchange and can reach any other city |
| TGW Attachment | On-ramp from a city to the interchange | Each city (VPC), highway extension (VPN), or expressway (Direct Connect) connects via a single on-ramp |
| TGW Route Table | Signage system at the interchange | Directional signs that control which on-ramps can reach which exits; different sign sets create isolated traffic zones |
| TGW Association | "Which signs does this driver read?" | Each on-ramp is associated with exactly one sign set; that set determines where the driver's outbound traffic can go |
| TGW Propagation | "Advertise this exit to specific sign sets" | An on-ramp can advertise its destination to multiple sign sets, making it reachable from different traffic zones |
| AWS PrivateLink | One-way mail slot in a wall | A mail slot between two buildings; the consumer drops requests through the slot, responses come back, but neither side can enter the other's building |
| Endpoint Service (Provider) | Mail slot with a dispatch manager | The provider places an NLB behind the slot that load-balances requests to whichever service worker is least busy |
| Interface Endpoint (Consumer) | Local mailbox for the slot | An ENI with a local address in the consumer's subnet; traffic enters the mail slot and arrives at the provider's NLB |
| Overlapping CIDRs with PrivateLink | Two cities with identical postal codes | PrivateLink works because the tunnel uses the consumer's local address, never the provider's; postal codes do not matter |

---

## The Big Picture

In [yesterday's VPC Advanced doc](./2026-02-19-vpc-advanced-cidr-subnet-strategy.md), you were a city planner designing individual cities: postal codes (CIDRs), neighborhoods (subnets), road maps (route tables), exit guards (NAT Gateways), and private couriers (VPC endpoints). That was the internal architecture of a single city.

Today you step up to **national transportation secretary**. You have 10, 50, maybe 100 cities across the nation, and they need to talk to each other. The fundamental question is: **how do you connect them?**

You have three options:

1. **Country roads** (VPC Peering) -- Build a private road directly between two cities. Simple, fast, no middleman. But you need a separate road for every pair, and roads do not chain together.

2. **Central highway interchange** (Transit Gateway) -- Build one massive interchange in the center of the nation. Every city builds a single on-ramp to the interchange. Once there, traffic can reach any other city's off-ramp. The interchange has multiple sign sets (route tables) so you can control which cities can reach which.

3. **Private delivery tunnels** (PrivateLink) -- Do not build a road at all. Instead, dig a sealed tunnel from a consumer city directly to a provider's loading dock. No shared address space, no routing, no road network. The consumer sends packages through the tunnel; the provider never enters the consumer's city.

```
THE THREE CONNECTIVITY MODELS — VISUAL OVERVIEW
=============================================================================

  MODEL 1: VPC PEERING (country roads)
  ────────────────────────────────────────────────────

  ┌──────┐         ┌──────┐         ┌──────┐
  │VPC A │─────────│VPC B │─────────│VPC C │
  └──┬───┘         └──────┘         └──────┘
     │                                  │
     └──────────────────────────────────┘

  3 VPCs = 3 peering connections (each is independent)
  A cannot reach C through B (no transitive routing)
  Each connection requires route table updates in BOTH VPCs


  MODEL 2: TRANSIT GATEWAY (highway interchange)
  ────────────────────────────────────────────────────

  ┌──────┐      ┌──────┐      ┌──────┐
  │VPC A │      │VPC B │      │VPC C │
  └──┬───┘      └──┬───┘      └──┬───┘
     │             │             │
     │    ┌────────┴────────┐    │
     └────┤ TRANSIT GATEWAY ├────┘
          │                 │
          │  Route Table 1  │ ← controls who can reach whom
          │  Route Table 2  │ ← enables network segmentation
          └────────┬────────┘
                   │
          ┌────────┴────────┐
          │  VPN / Direct   │ ← on-premises connectivity
          │  Connect        │   consolidated at TGW
          └─────────────────┘

  3 VPCs = 3 attachments (not 3 peering connections)
  Any VPC can reach any other through the TGW
  Add VPC D? Just 1 new attachment, not 3 new peerings


  MODEL 3: PRIVATELINK (delivery tunnel)
  ────────────────────────────────────────────────────

  ┌────────────────────┐         ┌────────────────────┐
  │ CONSUMER VPC       │         │ PROVIDER VPC       │
  │ 10.0.0.0/16        │         │ 10.0.0.0/16        │
  │                    │         │  (same CIDR — OK!)  │
  │ ┌────────────────┐ │  tunnel │ ┌────────────────┐ │
  │ │ Interface      │─┼────────┼─│ NLB → Service  │ │
  │ │ Endpoint (ENI) │ │         │ │                │ │
  │ │ 10.0.5.99      │ │         │ └────────────────┘ │
  │ └────────────────┘ │         │                    │
  └────────────────────┘         └────────────────────┘

  No routing between VPCs — traffic flows through the tunnel
  Consumer uses a local IP (10.0.5.99) in their own subnet
  Overlapping CIDRs are perfectly fine
  Traffic is one-way: consumer initiates → provider serves
```

---

## Part 1: VPC Peering — The Country Road

### The Analogy

A VPC peering connection is a **country road built between exactly two cities**. It is private (traffic never leaves the AWS backbone), direct (no middleman), and simple to construct. But country roads have strict rules:

- **No through-traffic**: If City A has a road to City B, and City B has a road to City C, a driver from City A **cannot** drive through City B to reach City C. The road between A and B is a dead-end for C-bound traffic. This is the no-transitive-routing rule.
- **No borrowing infrastructure**: If City B has a highway on-ramp (IGW), City A cannot use it. If City B has a guarded exit (NAT Gateway) or a private expressway to headquarters (VPN/Direct Connect), City A cannot use those either. This is the no-edge-to-edge routing rule.
- **Unique postal codes required**: Two cities with the same postal code range cannot build a road between them. The postal system cannot decide which city a letter is for. This is the no-overlapping-CIDRs rule -- and it applies to all CIDRs, including secondary ones you do not intend to route.
- **Roads do not scale**: Connecting 5 cities requires 10 roads. Connecting 10 requires 45. Connecting 100 requires 4,950. And every road requires route table updates in **both** cities.

### The Technical Reality

```
VPC PEERING — KEY CONSTRAINTS
=============================================================================

  CONSTRAINT 1: NO TRANSITIVE ROUTING
  ────────────────────────────────────────────────────

  VPC A (10.0.0.0/16) ←──peering──→ VPC B (10.1.0.0/16)
  VPC B (10.1.0.0/16) ←──peering──→ VPC C (10.2.0.0/16)

  Can A reach C through B?  NO.

  Even if you add a route in VPC A pointing 10.2.0.0/16 → peering-AB,
  VPC B will NOT forward that traffic to peering-BC.
  AWS drops packets that arrive on one peering connection and are
  destined for another. This is enforced at the infrastructure level.

  To connect A to C, you must create a THIRD peering connection: A ↔ C.


  CONSTRAINT 2: NO EDGE-TO-EDGE ROUTING
  ────────────────────────────────────────────────────

  VPC A cannot use VPC B's:
  ├── Internet Gateway (no internet access through a peer)
  ├── NAT Gateway (no outbound internet through a peer)
  ├── VPN connection (no on-premises access through a peer)
  ├── Direct Connect connection
  ├── Gateway endpoint (S3/DynamoDB endpoints are local only)
  └── ClassicLink connection

  Each VPC must provision its own gateways and connections.


  CONSTRAINT 3: NO OVERLAPPING CIDRs
  ────────────────────────────────────────────────────

  VPC A: primary 10.0.0.0/16, secondary 10.100.0.0/16
  VPC B: primary 10.1.0.0/16, secondary 10.100.0.0/16

  Can A and B peer?  NO.

  Even though the primary CIDRs do not overlap, the secondary
  CIDRs do. AWS checks all CIDRs (primary + secondary) at
  creation and acceptance time. Additionally, if a secondary CIDR
  is added to either VPC while a peering request is pending, and
  that new CIDR overlaps with the peer, the request fails on
  acceptance. Already-active peering connections are NOT broken
  by new secondary CIDRs, but you cannot route overlapping ranges.
  This is why the CIDR planning you learned yesterday matters:
  every CIDR block across every VPC must be globally unique.


  CONSTRAINT 4: THE FULL MESH SCALING PROBLEM
  ────────────────────────────────────────────────────

  Formula: n(n-1)/2 peering connections for full mesh

  VPCs    Peering Connections    Route Table Entries (2 per connection)
  ────    ───────────────────    ─────────────────────────────────────
    3              3                        6
    5             10                       20
   10             45                       90
   20            190                      380
   50          1,225                    2,450
  100          4,950                    9,900

  QUOTA: 50 active peering connections per VPC (default)
  Adjustable up to 125 via AWS support request.
  At 126 VPCs, full mesh becomes physically impossible.

  But the real limit hits much earlier — at ~20 VPCs, the
  operational burden of managing 190 connections and 380
  route table entries across 20 accounts becomes untenable.
```

### When VPC Peering Is the Right Choice

Despite its limitations, VPC peering is the best option in specific scenarios:

```
VPC PEERING — SWEET SPOT
=============================================================================

  USE VPC PEERING WHEN:
  ├── You have 2-5 VPCs that need full connectivity
  ├── You need the lowest possible latency (no middleman)
  ├── You want zero per-GB data processing charges
  │   (peering data transfer is charged at standard inter-AZ or
  │   inter-region rates, but there is NO additional processing fee)
  ├── You need cross-region VPC connectivity
  │   (inter-region peering is supported; TGW peering is also,
  │   but peering is simpler for 2-3 regions)
  └── High-throughput, low-connection-count architectures
      (Ably processes 500K+ packets/sec over peering connections, per their 2024 analysis)

  AVOID VPC PEERING WHEN:
  ├── You have more than ~10 VPCs (full mesh becomes painful)
  ├── You need transitive routing (shared services model)
  ├── You have overlapping CIDRs (peering will be rejected)
  ├── You need centralized egress, inspection, or hybrid connectivity
  └── You expect rapid VPC growth (each new VPC requires n-1 new connections)
```

### Terraform: VPC Peering Between Two VPCs

```hcl
# ──────────────────────────────────────────────────────────
# VPC Peering: app-prod (10.0.0.0/16) ↔ data-prod (10.1.0.0/16)
# Same account, same region
# ──────────────────────────────────────────────────────────

# Create the peering connection (requester side)
resource "aws_vpc_peering_connection" "app_to_data" {
  vpc_id        = aws_vpc.app_prod.id      # Requester VPC
  peer_vpc_id   = aws_vpc.data_prod.id     # Accepter VPC
  auto_accept   = true                      # Same account → auto-accept

  tags = {
    Name = "app-prod-to-data-prod"
    Side = "requester"
  }
}

# Route in app-prod: traffic for 10.1.0.0/16 → peering connection
resource "aws_route" "app_to_data_private" {
  route_table_id            = aws_route_table.app_prod_private.id
  destination_cidr_block    = aws_vpc.data_prod.cidr_block  # 10.1.0.0/16
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_data.id
}

# Route in data-prod: traffic for 10.0.0.0/16 → peering connection
resource "aws_route" "data_to_app_private" {
  route_table_id            = aws_route_table.data_prod_private.id
  destination_cidr_block    = aws_vpc.app_prod.cidr_block   # 10.0.0.0/16
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_data.id
}

# IMPORTANT: Add routes to ALL relevant route tables in BOTH VPCs.
# If you only add routes to private route tables but your ALB is in
# a public subnet, the ALB cannot reach targets in the peered VPC.

# Isolated subnet route (for RDS cross-VPC replication, etc.)
resource "aws_route" "app_to_data_isolated" {
  route_table_id            = aws_route_table.app_prod_isolated.id
  destination_cidr_block    = aws_vpc.data_prod.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_data.id
}

resource "aws_route" "data_to_app_isolated" {
  route_table_id            = aws_route_table.data_prod_isolated.id
  destination_cidr_block    = aws_vpc.app_prod.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_data.id
}

# ──────────────────────────────────────────────────────────
# Cross-account peering (different pattern)
# ──────────────────────────────────────────────────────────

# In the REQUESTER account:
resource "aws_vpc_peering_connection" "cross_account" {
  vpc_id        = aws_vpc.app_prod.id
  peer_vpc_id   = var.peer_vpc_id              # VPC ID in the other account
  peer_owner_id = var.peer_account_id          # Account ID of the peer
  peer_region   = "us-east-1"                  # Required for cross-region

  # Cannot auto-accept cross-account — the accepter must explicitly accept
  auto_accept = false

  tags = {
    Name = "cross-account-peering"
  }
}

# In the ACCEPTER account (separate Terraform workspace or provider):
resource "aws_vpc_peering_connection_accepter" "cross_account" {
  provider                  = aws.peer_account  # Provider alias for the peer account
  vpc_peering_connection_id = var.peering_connection_id
  auto_accept               = true

  tags = {
    Name = "cross-account-peering-accepted"
  }
}
```

### Peering Gotchas: Security Groups and DNS

Two operational details that catch engineers after establishing peering:

**Security group cross-referencing** — For same-region peering, security groups in VPC A can reference security groups in VPC B by ID. This means instead of allowing a CIDR range, you can allow traffic from a specific security group in the peer VPC. This is more precise and avoids breakage when peer subnets change. Many engineers forget to update security groups after establishing peering and wonder why traffic is not flowing despite correct routes.

```hcl
# In VPC A: allow traffic from VPC B's app security group
resource "aws_security_group_rule" "allow_peer_app" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  security_group_id        = aws_security_group.app_a.id
  source_security_group_id = var.peer_vpc_app_sg_id  # SG ID from VPC B
  description              = "HTTPS from peered VPC B app tier"
}

# NOTE: Cross-region peering does NOT support SG cross-referencing.
# You must use CIDR blocks for cross-region peering security group rules.
```

**DNS resolution across peering** — VPC peering supports private DNS resolution when you enable `Allow DNS resolution from remote VPC` on the peering connection options. Without this, instances in VPC A cannot resolve private hosted zone records from VPC B. This ties directly into the Route 53 Resolver concepts from [yesterday's doc](./2026-02-19-vpc-advanced-cidr-subnet-strategy.md).

```hcl
# Enable DNS resolution for the peering connection
resource "aws_vpc_peering_connection_options" "requester" {
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_data.id

  requester {
    allow_remote_vpc_dns_resolution = true
  }
}

resource "aws_vpc_peering_connection_options" "accepter" {
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_data.id

  accepter {
    allow_remote_vpc_dns_resolution = true
  }
}
```

---

## Part 2: Transit Gateway — The Central Highway Interchange

### The Analogy

Instead of building country roads between every pair of cities, you build one **massive highway interchange** in the center of the nation. Every city constructs a single on-ramp to the interchange. Once a driver reaches the interchange, they can follow the signs to any other city's off-ramp.

The interchange has several powerful features:

- **Multiple sign sets** (route tables): You can hang different signs for different groups of drivers. Production cities see signs pointing to the shared-services city and to each other, but they never see signs pointing to development cities. Development cities see their own set of signs. This is **network segmentation** without building separate interchanges.

- **Association vs propagation** — remember two sentences and the rest follows:

  > **Association controls "where can I go?" Propagation controls "who can reach me?"**

  Each on-ramp is **associated** with exactly one sign set -- this determines which signs that city's outbound drivers read. But each on-ramp can **propagate** its destination to multiple sign sets -- this determines which sign sets include that city as a reachable destination.

- **Consolidated hybrid connectivity**: Instead of every city building its own private expressway to headquarters (VPN/Direct Connect), the interchange itself connects to headquarters. All cities reach HQ through the interchange -- one connection instead of N.

### The Technical Reality

```
TRANSIT GATEWAY ARCHITECTURE — THE MENTAL MODEL
=============================================================================

  TRANSIT GATEWAY = Regional Layer 3 Virtual Router
  ├── Up to 5,000 attachments per TGW
  ├── Up to 50 Gbps bandwidth per VPC attachment (burst)
  ├── Supports: VPC, VPN, Direct Connect Gateway, TGW Peering,
  │   Connect (SD-WAN), Network Firewall
  └── Regional service — one TGW per region (peer across regions)


  ATTACHMENT TYPES:
  ────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────────────┐
  │                     TRANSIT GATEWAY                                  │
  │                                                                     │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
  │  │ VPC         │  │ VPN         │  │ Direct      │                │
  │  │ Attachment  │  │ Attachment  │  │ Connect GW  │                │
  │  │             │  │             │  │ Attachment  │                │
  │  │ Requires:   │  │ Connects    │  │             │                │
  │  │ 1 subnet    │  │ site-to-    │  │ Connects    │                │
  │  │ per AZ      │  │ site VPN    │  │ DX circuits │                │
  │  │ (TGW ENI)   │  │ tunnels     │  │ across      │                │
  │  │             │  │ to TGW      │  │ regions     │                │
  │  └─────────────┘  └─────────────┘  └─────────────┘                │
  │                                                                     │
  │  ┌─────────────┐  ┌─────────────┐                                  │
  │  │ TGW Peering │  │ TGW Connect │                                  │
  │  │ Attachment  │  │ Attachment  │                                  │
  │  │             │  │             │                                  │
  │  │ Cross-      │  │ GRE tunnel  │                                  │
  │  │ region TGW  │  │ for SD-WAN  │                                  │
  │  │ peering     │  │ appliances  │                                  │
  │  └─────────────┘  └─────────────┘                                  │
  └─────────────────────────────────────────────────────────────────────┘


  ROUTE TABLE MECHANICS — translating the sign-set analogy into AWS terminology:
  ────────────────────────────────────────────────────

  ASSOCIATION: "Which route table does this attachment's
               OUTBOUND traffic get evaluated against?"
  ├── Each attachment is associated with exactly ONE route table
  ├── When traffic LEAVES this attachment heading through the TGW,
  │   the associated route table determines where it can go
  └── Think: "which set of highway signs does this driver read?"

  PROPAGATION: "Which route tables learn about this attachment's
                CIDR, so traffic can be routed TOWARD it?"
  ├── An attachment can propagate its routes to MULTIPLE route tables
  ├── When an attachment propagates to a route table, that table
  │   gets a dynamic route entry for the attachment's CIDR
  └── Think: "which sign sets include this city as a destination?"


  EXAMPLE — PACKET FLOW:
  ────────────────────────────────────────────────────

  VPC-A (10.0.0.0/16) wants to reach VPC-B (10.1.0.0/16)

  1. VPC-A's route table has: 10.1.0.0/16 → tgw-xxxxxxxxx
  2. Packet arrives at TGW via VPC-A's attachment
  3. TGW looks up VPC-A's ASSOCIATED route table
  4. That route table has: 10.1.0.0/16 → VPC-B attachment
     (because VPC-B PROPAGATED its routes to this table)
  5. TGW forwards packet to VPC-B's attachment
  6. VPC-B's local route (10.1.0.0/16 → local) delivers the packet
```

### Transit Gateway Route Table Segmentation

The real power of Transit Gateway is not just connecting VPCs -- it is controlling **who can reach whom** through multiple route tables.

```
NETWORK SEGMENTATION — THE ENTERPRISE PATTERN
=============================================================================

  SCENARIO: 6 VPCs across 3 environments + 1 shared services VPC
  GOAL: Prod can reach shared-services but NOT dev or staging
        Dev can reach shared-services but NOT prod or staging
        Shared-services can reach everyone

  ┌─────────────────────────────────────────────────────────────────────┐
  │                     TRANSIT GATEWAY                                  │
  │                                                                     │
  │  ROUTE TABLE: prod-rt                                               │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │  10.0.0.0/16 → prod-app attachment     (propagated)          │  │
  │  │  10.1.0.0/16 → prod-data attachment    (propagated)          │  │
  │  │  10.2.0.0/16 → shared-svc attachment   (propagated)          │  │
  │  │                                                               │  │
  │  │  Associations: prod-app, prod-data                            │  │
  │  │  (prod VPCs read this route table for outbound traffic)       │  │
  │  │                                                               │  │
  │  │  Propagations: prod-app, prod-data, shared-services           │  │
  │  │  (prod can reach itself + shared-services, NOT dev)           │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ROUTE TABLE: dev-rt                                                │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │  10.8.0.0/16 → dev-app attachment      (propagated)          │  │
  │  │  10.9.0.0/16 → dev-data attachment     (propagated)          │  │
  │  │  10.2.0.0/16 → shared-svc attachment   (propagated)          │  │
  │  │                                                               │  │
  │  │  Associations: dev-app, dev-data                              │  │
  │  │  Propagations: dev-app, dev-data, shared-services             │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ROUTE TABLE: shared-rt                                             │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │  10.0.0.0/16 → prod-app attachment     (propagated)          │  │
  │  │  10.1.0.0/16 → prod-data attachment    (propagated)          │  │
  │  │  10.2.0.0/16 → shared-svc attachment   (propagated)          │  │
  │  │  10.8.0.0/16 → dev-app attachment      (propagated)          │  │
  │  │  10.9.0.0/16 → dev-data attachment     (propagated)          │  │
  │  │                                                               │  │
  │  │  Associations: shared-services                                │  │
  │  │  Propagations: ALL attachments                                │  │
  │  │  (shared-services can reach every VPC)                        │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  RESULT:                                                            │
  │  ├── prod-app → prod-data: ALLOWED (same route table)              │
  │  ├── prod-app → shared-svc: ALLOWED (propagated to prod-rt)       │
  │  ├── prod-app → dev-app: BLOCKED (dev not in prod-rt)             │
  │  ├── dev-app → shared-svc: ALLOWED (propagated to dev-rt)         │
  │  ├── shared-svc → anything: ALLOWED (all propagated to shared-rt) │
  │  └── dev-app → prod-app: BLOCKED (prod not in dev-rt)             │
  └─────────────────────────────────────────────────────────────────────┘
```

### Common Transit Gateway Architecture Patterns

```
SIX KEY TGW ARCHITECTURES (from AWS docs)
=============================================================================

  1. CENTRALIZED ROUTER
     All VPCs connect through one TGW route table.
     Every VPC can reach every other VPC.
     Simplest model — good for small organizations.

  2. ISOLATED ROUTERS (the segmentation pattern above)
     Multiple route tables isolate environments.
     Prod cannot reach dev. Both can reach shared-services.
     The most common enterprise pattern.

  3. SHARED SERVICES VPC
     One VPC hosts centralized services (DNS, monitoring,
     CI/CD, VPC endpoints). All spoke VPCs route to it.
     Reduces cost by sharing expensive resources.

  4. CENTRALIZED INTERNET EGRESS
     One VPC has the IGW and NAT Gateways.
     All spoke VPCs route 0.0.0.0/0 through TGW to the
     egress VPC. Centralizes firewall rules and logging.
     Trade-off: cross-AZ data transfer charges.

  5. TRANSIT GATEWAY PEERING (cross-region)
     TGW in us-east-1 peers with TGW in eu-west-1.
     Traffic stays on the AWS backbone (no internet).
     Static routes only (no propagation across peering).

  6. APPLIANCE/INSPECTION VPC
     Traffic routes through a firewall VPC (Palo Alto,
     Fortinet, or AWS Network Firewall) before reaching
     the destination. TGW appliance mode ensures symmetric
     routing (both directions through the same AZ's firewall).
     The mechanism for steering traffic to appliances is Gateway
     Load Balancer Endpoints (GWLBE) — a third PrivateLink endpoint
     type (alongside Interface and Gateway endpoints) designed
     specifically for transparent network inspection.

  OPERATIONAL VISIBILITY:
  ────────────────────────────────────────────────────

  ├── TGW Flow Logs: capture packet-level metadata for traffic
  │   crossing the TGW (source, dest, action, bytes). Enable for
  │   troubleshooting connectivity issues between VPCs.
  └── Network Manager: centralized dashboard for visualizing your
      TGW topology, monitoring connections, and tracking events
      across regions. Useful when managing multi-region TGW peering.
```

### Terraform: Transit Gateway with Network Segmentation

```hcl
# ──────────────────────────────────────────────────────────
# Transit Gateway
# ──────────────────────────────────────────────────────────

resource "aws_ec2_transit_gateway" "main" {
  description = "Organization transit gateway"

  # Disable default route table — we create explicit ones for segmentation
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"

  # Enable DNS support for private hosted zone resolution across VPCs
  dns_support = "enable"

  # Enable ECMP for VPN attachments (aggregate bandwidth across tunnels)
  vpn_ecmp_support = "enable"

  # ASN for BGP (if using VPN or Direct Connect)
  amazon_side_asn = 64512

  auto_accept_shared_attachments = "enable"  # For cross-account via RAM

  tags = {
    Name = "org-transit-gateway"
  }
}

# Share TGW across the organization via RAM
resource "aws_ram_resource_share" "tgw" {
  name                      = "transit-gateway-share"
  allow_external_principals = false  # Organization only
}

resource "aws_ram_resource_association" "tgw" {
  resource_arn       = aws_ec2_transit_gateway.main.arn
  resource_share_arn = aws_ram_resource_share.tgw.arn
}

resource "aws_ram_principal_association" "org" {
  principal          = var.organization_arn
  resource_share_arn = aws_ram_resource_share.tgw.arn
}

# ──────────────────────────────────────────────────────────
# TGW Route Tables for Network Segmentation
# ──────────────────────────────────────────────────────────

resource "aws_ec2_transit_gateway_route_table" "prod" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id

  tags = {
    Name        = "prod-route-table"
    Environment = "production"
  }
}

resource "aws_ec2_transit_gateway_route_table" "dev" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id

  tags = {
    Name        = "dev-route-table"
    Environment = "development"
  }
}

resource "aws_ec2_transit_gateway_route_table" "shared" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id

  tags = {
    Name        = "shared-services-route-table"
    Environment = "shared"
  }
}

# ──────────────────────────────────────────────────────────
# VPC Attachments
# ──────────────────────────────────────────────────────────

# TGW attachment subnets — dedicated /28 subnets for TGW ENIs
# (from yesterday's subnet sizing guide: /28 is the minimum,
# TGW needs only 1 IP per AZ)
resource "aws_subnet" "tgw_prod_app" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.prod_app.id
  cidr_block        = local.tgw_subnet_cidrs_prod_app[count.index]
  availability_zone = local.azs[count.index]

  tags = {
    Name = "tgw-prod-app-${local.azs[count.index]}"
    Tier = "transit-gateway"
  }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "prod_app" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.prod_app.id
  subnet_ids         = aws_subnet.tgw_prod_app[*].id

  # Disable default route table association — we do it explicitly
  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  # Enable DNS support for cross-VPC DNS resolution
  dns_support = "enable"

  # Enable appliance mode if traffic will be inspected by a
  # network appliance (ensures symmetric routing through same AZ)
  # appliance_mode_support = "enable"

  tags = {
    Name        = "prod-app-attachment"
    Environment = "production"
  }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "shared_services" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.shared_services.id
  subnet_ids         = aws_subnet.tgw_shared_services[*].id

  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  dns_support = "enable"

  tags = {
    Name        = "shared-services-attachment"
    Environment = "shared"
  }
}

# ──────────────────────────────────────────────────────────
# Associations (which route table does each attachment use for outbound?)
# ──────────────────────────────────────────────────────────

# prod-app reads the prod route table for outbound decisions
resource "aws_ec2_transit_gateway_route_table_association" "prod_app" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.prod_app.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}

# shared-services reads the shared route table
resource "aws_ec2_transit_gateway_route_table_association" "shared_services" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.shared_services.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.shared.id
}

# ──────────────────────────────────────────────────────────
# Propagations (which route tables learn about each attachment's CIDR?)
# ──────────────────────────────────────────────────────────

# prod-app propagates to prod-rt (so prod VPCs can reach each other)
resource "aws_ec2_transit_gateway_route_table_propagation" "prod_app_to_prod_rt" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.prod_app.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}

# prod-app propagates to shared-rt (so shared-services can reach prod)
resource "aws_ec2_transit_gateway_route_table_propagation" "prod_app_to_shared_rt" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.prod_app.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.shared.id
}

# shared-services propagates to ALL route tables (reachable from everywhere)
resource "aws_ec2_transit_gateway_route_table_propagation" "shared_to_prod_rt" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.shared_services.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "shared_to_dev_rt" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.shared_services.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.dev.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "shared_to_shared_rt" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.shared_services.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.shared.id
}

# ──────────────────────────────────────────────────────────
# VPC Route Table Updates
# (VPC route tables must point to TGW for cross-VPC traffic)
# ──────────────────────────────────────────────────────────

# In prod-app VPC: route shared-services CIDR to TGW
resource "aws_route" "prod_app_to_shared" {
  route_table_id         = aws_route_table.prod_app_private.id
  destination_cidr_block = "10.2.0.0/16"  # shared-services CIDR
  transit_gateway_id     = aws_ec2_transit_gateway.main.id
}

# In shared-services VPC: route all prod CIDRs to TGW
resource "aws_route" "shared_to_prod_app" {
  route_table_id         = aws_route_table.shared_private.id
  destination_cidr_block = "10.0.0.0/16"  # prod-app CIDR
  transit_gateway_id     = aws_ec2_transit_gateway.main.id
}

# For centralized egress: spoke VPCs route 0.0.0.0/0 to TGW
# resource "aws_route" "spoke_default_to_tgw" {
#   route_table_id         = aws_route_table.spoke_private.id
#   destination_cidr_block = "0.0.0.0/0"
#   transit_gateway_id     = aws_ec2_transit_gateway.main.id
# }
```

---

## Part 3: AWS PrivateLink — The Private Delivery Tunnel

### The Analogy

Sometimes you do not want a road between two cities at all. You do not want shared address spaces, routing updates, or network-level connectivity. You just want City A to be able to **send a package to a specific service** in City B.

PrivateLink is a **one-way mail slot in a wall between two buildings**. The provider city installs a mail slot (NLB-backed endpoint service) in its outer wall with a delivery coordinator behind it (the NLB that load-balances across targets). The consumer city installs a local mailbox (interface endpoint/ENI) in their own neighborhood, with a local address from their own postal code range. When the consumer drops a request envelope through the slot, the delivery coordinator routes it to whichever service worker is least busy. A response comes back through the slot.

The critical insight: **neither side has a key to the other's building, and you cannot walk through the mail slot**. The two cities never exchange routes. Their postal codes can be identical. The consumer does not know or care about the provider's internal address scheme. Traffic is strictly one-directional: the consumer pushes requests through the slot, the provider responds through it, but the provider **cannot** initiate contact into the consumer's building. This is not just a limitation -- it is a security feature that minimizes blast radius.

### The Technical Reality

```
PRIVATELINK ARCHITECTURE
=============================================================================

  PROVIDER VPC (10.0.0.0/16)                CONSUMER VPC (10.0.0.0/16)
  ┌────────────────────────────┐            ┌────────────────────────────┐
  │                            │            │                            │
  │  ┌──────────────────────┐  │  Private   │  ┌──────────────────────┐  │
  │  │  Application         │  │  Link      │  │  Consumer App        │  │
  │  │  (EC2, ECS, Lambda)  │  │            │  │                      │  │
  │  │  10.0.5.10:8080      │  │            │  │  Calls:              │  │
  │  └──────────┬───────────┘  │            │  │  vpce-svc.region.    │  │
  │             │              │            │  │  vpce.amazonaws.com  │  │
  │  ┌──────────▼───────────┐  │            │  └──────────┬───────────┘  │
  │  │  Network Load        │  │            │             │              │
  │  │  Balancer (NLB)      │  │            │  ┌──────────▼───────────┐  │
  │  │  Internal            │  │◀───────────│  │  Interface Endpoint  │  │
  │  └──────────────────────┘  │            │  │  (ENI)               │  │
  │             │              │            │  │  10.0.8.99           │  │
  │  ┌──────────▼───────────┐  │            │  │  (consumer's subnet) │  │
  │  │  VPC Endpoint        │  │            │  └──────────────────────┘  │
  │  │  Service             │  │            │                            │
  │  │  (registered NLB)    │  │            │  DNS: private hosted zone  │
  │  └──────────────────────┘  │            │  resolves service name to  │
  │                            │            │  10.0.8.99 (ENI IP)        │
  └────────────────────────────┘            └────────────────────────────┘

  BOTH VPCs use 10.0.0.0/16 — this is fine because:
  ├── No routes are exchanged between the VPCs
  ├── The ENI (10.0.8.99) lives in the CONSUMER's address space
  ├── Traffic is tunneled directly from ENI to NLB (AWS backbone)
  └── The provider never knows or cares about the consumer's CIDR


  PRIVATELINK KEY PROPERTIES:
  ────────────────────────────────────────────────────

  ├── Unidirectional: consumer initiates → provider responds
  │   (provider CANNOT initiate connections to the consumer)
  ├── Source IP visibility: provider sees PrivateLink internal IPs by default,
  │   NOT the consumer's real IP. Enable Proxy Protocol v2 on the NLB to pass
  │   the consumer's original IP in a header (needed for logging, rate-limiting)
  ├── Overlapping CIDRs: fully supported
  ├── Cross-account: consumer and provider can be in different accounts
  ├── Cross-region: PARTIALLY supported (since late 2024)
  │   ├── AWS-managed services (S3, ECR, KMS, Lambda, etc.): YES
  │   └── Custom endpoint services (NLB-backed): NO
  │       (for cross-region custom services, use TGW peering + PrivateLink)
  ├── Connection approval: provider can require explicit acceptance
  │   of each consumer connection (or enable auto-accept)
  ├── No VPC route changes: the consumer adds an endpoint, not a route
  └── Scalable: no documented hard limit on provider-side consumers
  │   (consumer-side quota: 50 interface endpoints per VPC, adjustable)


  PRIVATELINK USE CASES:
  ────────────────────────────────────────────────────

  1. SaaS PROVIDER MODEL
     ├── You build a service in your VPC
     ├── Customers create interface endpoints in their VPCs
     ├── They access your service privately, never over the internet
     └── Example: Datadog, MongoDB Atlas, Snowflake all use PrivateLink

  2. OVERLAPPING CIDR ESCAPE HATCH
     ├── Team A and Team B both used 10.0.0.0/16 (the spreadsheet problem)
     ├── They cannot peer or use TGW (overlapping CIDRs)
     ├── PrivateLink lets them expose specific services to each other
     └── This is the fallback when CIDR planning was not done properly

  3. SHARED SERVICES WITHOUT FULL NETWORK CONNECTIVITY
     ├── Expose a centralized logging API, metrics endpoint, or
     │   authentication service via PrivateLink
     ├── Consumer VPCs get access to the service without any route
     │   to the provider's network — minimizes blast radius
     └── The provider controls who can connect via endpoint policies

  4. AWS SERVICE ACCESS FROM ON-PREMISES
     ├── Create interface endpoints in a VPC connected to on-prem via DX
     ├── On-prem resolves AWS service DNS to the endpoint's private IP
     ├── Traffic flows: on-prem → DX → VPC → endpoint → AWS service
     └── Never touches the public internet

  5. OVERLAPPING CIDR FULL NETWORK ACCESS (via Private NAT Gateway)
     ├── When PrivateLink's service-level access is insufficient (need SSH,
     │   debugging, database replication, monitoring with real source IPs)
     ├── Assign unique secondary CIDRs from 100.64.0.0/10 to each VPC
     │   (e.g., Company A = 100.64.0.0/16, Company B = 100.64.1.0/16)
     ├── Private NAT Gateways translate from overlapping primary (10.0.0.0/16)
     │   to unique secondary addresses; TGW routes between secondaries
     ├── Enables full network-level access but adds NAT cost ($0.045/GB)
     └── This is a bridge — the long-term fix is re-IPing with IPAM
```

### Terraform: PrivateLink Provider and Consumer

```hcl
# ──────────────────────────────────────────────────────────
# PROVIDER SIDE: Expose a service via PrivateLink
# ──────────────────────────────────────────────────────────

# NLB fronting the service
resource "aws_lb" "provider" {
  name               = "provider-nlb"
  internal           = true               # Must be internal for PrivateLink
  load_balancer_type = "network"
  subnets            = var.provider_private_subnet_ids

  enable_cross_zone_load_balancing = true

  tags = {
    Name = "provider-nlb"
  }
}

resource "aws_lb_target_group" "provider" {
  name     = "provider-tg"
  port     = 8080
  protocol = "TCP"
  vpc_id   = var.provider_vpc_id

  health_check {
    protocol            = "TCP"
    port                = "traffic-port"
    healthy_threshold   = 3
    unhealthy_threshold = 3
    interval            = 10
  }
}

resource "aws_lb_listener" "provider" {
  load_balancer_arn = aws_lb.provider.arn
  port              = 443
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.provider.arn
  }
}

# Register the NLB as a VPC endpoint service
resource "aws_vpc_endpoint_service" "provider" {
  acceptance_required        = true  # Require explicit approval
  network_load_balancer_arns = [aws_lb.provider.arn]

  # Restrict which accounts can create endpoints to this service
  allowed_principals = [
    "arn:aws:iam::${var.consumer_account_id}:root"  # IAM ARNs have no region field, hence ::
  ]

  tags = {
    Name = "my-api-endpoint-service"
  }
}

# Output the service name — consumer needs this to create their endpoint
output "endpoint_service_name" {
  value = aws_vpc_endpoint_service.provider.service_name
  # Format: com.amazonaws.vpce.us-east-1.vpce-svc-xxxxxxxxxxxxxxxxx
}

# ──────────────────────────────────────────────────────────
# CONSUMER SIDE: Create an interface endpoint to the service
# ──────────────────────────────────────────────────────────

# Security group for the endpoint ENI
resource "aws_security_group" "endpoint" {
  name_prefix = "privatelink-endpoint-"
  vpc_id      = var.consumer_vpc_id
  description = "Allow HTTPS to PrivateLink endpoint"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.consumer_vpc_cidr]
    description = "HTTPS from consumer VPC"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "privatelink-endpoint-sg"
  }
}

# Interface endpoint in the consumer VPC
resource "aws_vpc_endpoint" "consumer" {
  vpc_id              = var.consumer_vpc_id
  service_name        = var.endpoint_service_name  # From provider output
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = false  # Auto-override of public DNS not available for custom services; use Route 53 alias instead

  subnet_ids         = var.consumer_private_subnet_ids
  security_group_ids = [aws_security_group.endpoint.id]

  tags = {
    Name = "my-api-endpoint"
  }
}

# Private hosted zone for friendly DNS name
resource "aws_route53_zone" "endpoint" {
  name = "api.internal"

  vpc {
    vpc_id = var.consumer_vpc_id
  }
}

resource "aws_route53_record" "endpoint" {
  zone_id = aws_route53_zone.endpoint.zone_id
  name    = "my-service.api.internal"
  type    = "A"

  alias {
    name                   = aws_vpc_endpoint.consumer.dns_entry[0]["dns_name"]
    zone_id                = aws_vpc_endpoint.consumer.dns_entry[0]["hosted_zone_id"]
    evaluate_target_health = true
  }
}

# Consumer applications now call: https://my-service.api.internal
# → resolves to the endpoint ENI IP in the consumer's subnet
# → tunneled through PrivateLink to the provider's NLB
```

---

## Part 4: The Decision Framework — When to Use What

```
CONNECTIVITY DECISION TREE
=============================================================================

  How many VPCs need connectivity?
  │
  ├── 2-5 VPCs, stable count
  │   │
  │   ├── Need full network connectivity? → VPC PEERING
  │   │   (lowest latency, no per-GB processing, simple)
  │   │
  │   └── Need only API/service-level access? → PRIVATELINK
  │       (tightest security, no route exposure)
  │
  ├── 5-20 VPCs, moderate growth
  │   │
  │   ├── Need transitive routing or centralized egress? → TRANSIT GATEWAY
  │   │
  │   ├── All VPCs have unique CIDRs? → TRANSIT GATEWAY
  │   │
  │   └── Some VPCs have overlapping CIDRs? → PRIVATELINK for those pairs
  │       + TRANSIT GATEWAY for the rest
  │
  └── 20+ VPCs, rapid growth
      │
      └── TRANSIT GATEWAY (mandatory)
          ├── Full mesh peering is operationally impossible at this scale
          ├── Add PrivateLink for specific service exposure patterns
          └── Use TGW route table segmentation for isolation


COMPARISON TABLE — VPC PEERING vs TRANSIT GATEWAY vs PRIVATELINK
=============================================================================

  Feature                    VPC Peering       Transit Gateway     PrivateLink
  ──────────────────────     ──────────────    ───────────────     ──────────────
  Connectivity model         Point-to-point    Hub-and-spoke       Service tunnel
  Transitive routing         No                Yes                 N/A (no routes)
  Overlapping CIDRs          Not allowed       Not allowed         Allowed
  Cross-region               Yes               Yes (TGW peering)   Partial*
  Cross-account              Yes               Yes (via RAM)       Yes
  Max connections            125 per VPC       5,000 attachments   No hard limit
  Bandwidth                  No limit**        50 Gbps/attachment  10 Gbps/AZ (→100)
  Latency                    Lowest            Slight overhead     Slight overhead
  On-prem access via peer    No (no edge)      Yes (VPN/DX at TGW) Yes (via DX/VPN)
  Network segmentation       No                Yes (route tables)  Yes (by design)
  Traffic direction           Bidirectional     Bidirectional       Unidirectional

  * PrivateLink cross-region: supported for AWS-managed services (S3, ECR,
    KMS, Lambda, etc.) since late 2024; NOT supported for custom NLB-backed
    endpoint services.
  ** VPC peering bandwidth is limited by instance-level networking
    (ENI limits, placement group bandwidth), not by the peering itself.
  *** PrivateLink: 10 Gbps/AZ default, auto-scales up to 100 Gbps/AZ.


  COST COMPARISON (us-east-1 pricing, February 2026)
  ────────────────────────────────────────────────────

  VPC PEERING:
  ├── Hourly cost:           $0.00 (no charge for the connection)
  ├── Data transfer:         $0.01/GB (same-region, cross-AZ)
  │                          $0.02/GB (inter-region)
  ├── Per-GB processing:     $0.00 (none)
  └── 10 VPCs, 1 TB/month:  $10.00 (same-region) + $0 connection cost
      Full mesh: 45 connections to manage, but zero connection charges

  TRANSIT GATEWAY:
  ├── Hourly cost:           $0.05/attachment/hour
  │                          (10 VPCs = $0.50/hr = $365/month)
  ├── Data transfer:         $0.01/GB (same-region, cross-AZ)
  ├── Per-GB processing:     $0.02/GB (TGW data processing charge)
  └── 10 VPCs, 1 TB/month:  $365 (attachments) + $20 (processing)
                             + $10 (data transfer) = $395/month

  PRIVATELINK:
  ├── Hourly cost:           $0.01/endpoint/AZ/hour
  │                          (1 endpoint x 3 AZs = $21.90/month)
  ├── Data transfer:         $0.01/GB processed
  ├── Per-GB processing:     Included in data transfer
  └── 1 service, 3 AZs,     $21.90 (hourly) + $10 (1 TB processing)
      1 TB/month:            = $31.90/month


  COST CROSSOVER ANALYSIS — PEERING vs TGW
  ────────────────────────────────────────────────────

  VPC peering is cheaper when:
  ├── You have few VPCs (< ~10)
  ├── High traffic volume (TGW's $0.02/GB adds up fast)
  └── You do not need transitive routing or centralized services

  Transit Gateway is cheaper when:
  ├── Full mesh peering would require 50+ connections
  │   (operational cost of managing connections exceeds TGW hourly)
  ├── You need centralized egress (one NAT Gateway set instead of N)
  ├── You need centralized inspection (one firewall instead of N)
  └── You consolidate VPN/DX connections at TGW (fewer tunnels)

  Example from Ably (real-world): with 500K+ packets/sec and petabytes
  of data, Transit Gateway would add ~$20,000 per PB in processing
  charges. They chose VPC peering because their traffic volume made
  TGW prohibitively expensive.

  QUICK COST COMPARISON TABLE (same-region, per month)
  ────────────────────────────────────────────────────

  10 VPCs       100 GB traffic    1 TB traffic      10 TB traffic
  ──────────    ──────────────    ──────────────    ──────────────
  Peering       $1 (transfer)     $10 (transfer)    $100 (transfer)
  (45 conns)    + $0 connection   + $0 connection   + $0 connection
                = $1/mo           = $10/mo          = $100/mo

  TGW           $1 + $2 proc      $10 + $20 proc    $100 + $200 proc
  (10 attach)   + $365 hourly     + $365 hourly     + $365 hourly
                = $368/mo         = $395/mo         = $665/mo

  Crossover: TGW is ~37x more expensive at low traffic, but the
  gap narrows at high volume since peering's operational cost
  (managing 45 connections) is not captured in the dollar figure.

  RULE OF THUMB: Calculate your monthly data transfer volume.
  If TGW processing charges ($0.02/GB x monthly volume) are small
  relative to your total AWS spend, choose TGW for simplicity.
  If processing charges are a significant line item, evaluate
  whether peering can meet your routing requirements.
```

---

## Part 5: How All Three Work Together — The Enterprise Pattern

Real-world architectures do not pick one technology exclusively. They compose all three based on each connection's requirements.

```
ENTERPRISE CONNECTIVITY — THE COMPLETE PICTURE
=============================================================================

                              ┌─────────────────────────────┐
                              │       ON-PREMISES DC        │
                              │       172.16.0.0/12         │
                              └────────────┬────────────────┘
                                           │ Direct Connect
                                           │
  ┌────────────────────────────────────────┼──────────────────────────────┐
  │                        TRANSIT GATEWAY │                              │
  │                                        │                              │
  │  ┌──────────────────────────────┐      │                              │
  │  │ SHARED-RT (association:      │      │                              │
  │  │  shared-svc, DX attachment)  │      │                              │
  │  │ Propagations: ALL            │      │                              │
  │  └──────────────────────────────┘      │                              │
  │                                        │                              │
  │  ┌──────────────┐  ┌──────────────┐    │                              │
  │  │ PROD-RT      │  │ DEV-RT       │    │                              │
  │  │ (prod VPCs)  │  │ (dev VPCs)   │    │                              │
  │  │ + shared-svc │  │ + shared-svc │    │                              │
  │  └──────┬───────┘  └──────┬───────┘    │                              │
  │         │                 │            │                              │
  └─────────┼─────────────────┼────────────┼──────────────────────────────┘
            │                 │            │
  ┌─────────▼────┐   ┌───────▼──────┐   ┌─▼──────────────┐
  │ PROD VPCs    │   │ DEV VPCs     │   │ SHARED-SVC VPC │
  │              │   │              │   │                │
  │ 10.0.0.0/16  │   │ 10.8.0.0/16  │   │ 10.2.0.0/16    │
  │ 10.1.0.0/16  │   │ 10.9.0.0/16  │   │                │
  │              │   │              │   │ Centralized:   │
  │ TGW: cross-  │   │ TGW: cross-  │   │ ├── VPC        │
  │ VPC routing  │   │ VPC routing  │   │ │   Endpoints  │
  │              │   │              │   │ ├── DNS (R53    │
  └──────────────┘   └──────────────┘   │ │   Resolver)  │
                                        │ └── Monitoring │
         PRIVATELINK                    └───────┬────────┘
  ┌──────────────────────────────────┐          │
  │                                  │          │
  │  SAAS PROVIDER VPC               │     PrivateLink
  │  (third-party, overlapping CIDR) │◀─────────┘
  │                                  │  Consumer: shared-svc
  │  NLB → Analytics Service         │  Provider: SaaS vendor
  │  Endpoint Service registered     │
  └──────────────────────────────────┘  Works despite CIDR overlap

  CONNECTIVITY MAP:
  ├── Prod VPCs ↔ Prod VPCs: via TGW (prod-rt)
  ├── Dev VPCs ↔ Dev VPCs: via TGW (dev-rt)
  ├── Any VPC ↔ Shared-services: via TGW (all route tables)
  ├── Any VPC ↔ On-premises: via TGW + Direct Connect
  ├── Shared-services → SaaS vendor: via PrivateLink (overlapping CIDR)
  ├── Prod VPC A ↔ Prod VPC B (high-throughput): could ADD direct peering
  │   for cost savings if TGW processing charges are significant
  └── Prod ↔ Dev: BLOCKED (route table segmentation)
```

---

## Quick Reference: Key Quotas and Numbers

```
SERVICE LIMITS AND COSTS — INTERVIEW QUICK REFERENCE
=============================================================================

  VPC Peering:
  ├── Active connections per VPC:    50 (default), 125 (max with support)
  ├── Bandwidth:                     No peering-level limit (instance-level)
  ├── Data processing cost:          $0.00/GB
  └── Data transfer (cross-AZ):     $0.01/GB

  Transit Gateway:
  ├── Attachments per TGW:           5,000
  ├── Bandwidth per VPC attachment:  50 Gbps (burst)
  ├── Hourly cost per attachment:    $0.05/hr (~$36.50/month)
  └── Data processing cost:          $0.02/GB

  PrivateLink:
  ├── Interface endpoints per VPC:   50 (default, adjustable)
  ├── Bandwidth per AZ:              10 Gbps default, auto-scales to 100 Gbps
  ├── Hourly cost per endpoint/AZ:   $0.01/hr (~$7.30/month per AZ)
  └── Data processing cost:          $0.01/GB
```

---

## Key Takeaways

### VPC Peering
1. **VPC peering is the simplest and cheapest option for 2-5 VPCs** -- Zero connection cost, zero per-GB processing, lowest latency; the only charges are standard data transfer rates; choose peering when you have a small, stable set of VPCs with non-overlapping CIDRs
2. **The full mesh formula n(n-1)/2 is the key scaling constraint** -- 10 VPCs need 45 connections, 20 need 190, and the maximum quota of 125 peering connections per VPC (default 50, adjustable to 125) makes full mesh physically impossible beyond 125 VPCs; the operational limit hits much earlier around 10-20 VPCs
3. **No transitive routing means every pair needs its own connection** -- If A peers with B and B peers with C, A cannot reach C through B; this makes shared-services architectures painful because the shared-services VPC needs a peering connection to every consumer
4. **No edge-to-edge routing prevents infrastructure sharing** -- You cannot use a peered VPC's IGW, NAT Gateway, VPN, or Direct Connect; each VPC must provision its own gateways, which increases cost and complexity

### Transit Gateway
5. **Transit Gateway is a regional virtual router that replaces full mesh with hub-and-spoke** -- Each VPC needs only one attachment regardless of how many other VPCs exist; adding a new VPC is one attachment, not n-1 peering connections
6. **The association-vs-propagation model is the key mental model for TGW routing** -- Association controls "which route table does this attachment read for outbound?" (exactly one); propagation controls "which route tables learn about this attachment's CIDR?" (one or many); segmentation is achieved through the **absence** of routes: you build connectivity by propagating, you get isolation by not propagating; the number of route tables you need equals the number of distinct "visibility zones" in your network
7. **Transit Gateway costs $0.05 per attachment per hour plus $0.02 per GB processed** -- For 10 VPCs, this is approximately $395/month at 1 TB of cross-VPC traffic; the per-GB processing charge is the dominant cost driver at high traffic volumes and is the primary reason some organizations choose peering for high-throughput connections
8. **TGW consolidates hybrid connectivity** -- Instead of every VPC maintaining its own VPN or Direct Connect attachment, one TGW connects to on-premises and all VPCs reach it through the hub; this reduces VPN tunnel count and DX circuit requirements

### PrivateLink
9. **PrivateLink is the only option that works with overlapping CIDRs** -- Because it uses ENIs in the consumer's address space rather than IP routing between VPCs, two VPCs with identical 10.0.0.0/16 ranges can communicate; this is the escape hatch when CIDR planning was not done properly
10. **PrivateLink is unidirectional by design** -- The consumer initiates connections to the provider; the provider cannot initiate connections back; this provides inherent security by minimizing the blast radius of a compromised provider
11. **Use PrivateLink for API-style access, not full network connectivity** -- It exposes a specific service behind an NLB, not an entire VPC; if you need to reach multiple services in another VPC, you need multiple endpoint services or you should use peering/TGW instead

### Decision Framework
12. **Most enterprises use all three technologies together** -- Transit Gateway for the backbone (VPC-to-VPC and hybrid), PrivateLink for third-party SaaS integration and overlapping-CIDR scenarios, and VPC peering for specific high-throughput connections where TGW processing charges are prohibitive
13. **Cost alone should not drive the decision** -- TGW is more expensive than peering per GB, but it eliminates the operational cost of managing hundreds of peering connections and route table entries; calculate total cost of ownership including engineering time, not just AWS charges
14. **CIDR planning from Day 1 determines your options** -- If every VPC has a unique, non-overlapping CIDR from a hierarchical allocation scheme (yesterday's lesson), you can use any connectivity option; if CIDRs overlap, PrivateLink provides service-level access and Private NAT Gateways with secondary CIDRs (from 100.64.0.0/10) provide full network access as a bridge, but the long-term fix is always re-IPing with IPAM
15. **VPC Lattice is an emerging fourth option at the application layer** -- While Peering, TGW, and PrivateLink operate at the network level (L3/L4), VPC Lattice provides service-to-service connectivity at L7 across VPCs, accounts, and compute types (EC2, ECS, EKS, Lambda) without requiring any of the network plumbing discussed in this doc; it is not a replacement for TGW or peering but a higher-level abstraction for application networking (see the [ECS Advanced doc](../2026-02-17-ecs-advanced.md) for how Lattice fits into service discovery)

---

## Further Reading

- [VPC Peering Basics -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)
- [How Transit Gateways Work -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
- [AWS PrivateLink Concepts -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
- [Building Scalable Multi-VPC Network Infrastructure -- AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html)
- [Field Notes: Working with Route Tables in Transit Gateway -- AWS Blog](https://aws.amazon.com/blogs/architecture/field-notes-working-with-route-tables-in-aws-transit-gateway/)
- [VPC Peering vs Transit Gateway and Beyond -- Ably Engineering Blog](https://ably.com/blog/aws-vpc-peering-vs-transit-gateway-and-beyond)
- [Integrating Transit Gateway with PrivateLink and Route 53 Resolver -- AWS Blog](https://aws.amazon.com/blogs/networking-and-content-delivery/integrating-aws-transit-gateway-with-aws-privatelink-and-amazon-route-53-resolver/)
- [AWS PrivateLink Cross-Region Support Announcement](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-privatelink-cross-region-connectivity-aws-services/)
- [VPC Lattice -- AWS Docs](https://docs.aws.amazon.com/vpc-lattice/latest/ug/what-is-vpc-lattice.html)

---

*Written on February 20, 2026*
