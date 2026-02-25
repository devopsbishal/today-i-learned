# AWS Site-to-Site VPN -- The Armored Convoy Analogy

> Understanding AWS Site-to-Site VPN through the analogy of an armored convoy service -- where encrypted shipments travel on public highways between your warehouse and a logistics hub, dual convoy routes ensure delivery even when one road is blocked, a GPS-enabled dispatch system (BGP) automatically reroutes around closures, and an express toll-road network (Global Accelerator) provides a faster, less congested path for time-sensitive cargo.

---

## TL;DR

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Site-to-Site VPN | Armored convoy on public highways | An encrypted IPSec tunnel over the public internet connecting your on-premises network to AWS; fast to set up, always encrypted, but subject to highway congestion |
| IPSec Tunnel | The armored truck itself | A Layer 3 encrypted tunnel using IKE for key exchange, ESP for payload encryption (AES-256-GCM), providing confidentiality, integrity, and authentication for every packet |
| Customer Gateway (CGW) resource | The pre-filed flight plan | An AWS resource representing your on-premises router -- stores its public IP (departure point), ASN (radio frequency), and device type (vehicle model) so AWS knows who to negotiate the tunnel with |
| Customer gateway device | The actual armored truck depot and loading dock | Your physical on-premises router or firewall (Cisco, Juniper, pfSense, strongSwan) that terminates the IPSec tunnel and handles encryption/decryption |
| Virtual Private Gateway (VGW) | A small receiving dock at one warehouse | An AWS-side VPN termination point attached to a single VPC; simple, limited to one VPC, no ECMP, no traffic aggregation |
| Transit Gateway as VPN endpoint | The central logistics hub with many loading bays | VPN tunnels terminate directly on TGW, enabling access to thousands of VPCs, ECMP bandwidth aggregation, and Accelerated VPN support |
| Static Routing | A fixed paper map given to every driver | You manually specify which IP ranges are reachable on each side; no automatic failover, no ECMP, no route discovery |
| Dynamic Routing (BGP) | A radio dispatch system with periodic check-ins | Routes are exchanged automatically via BGP; the central dispatcher broadcasts road closures to all drivers on periodic check-ins (hold timers), rerouting in ~30-90 seconds |
| Dual-Tunnel Architecture | Two convoy routes through different towns | Every VPN connection provides two tunnels terminating in different AZs; if one route is blocked, the other carries all traffic |
| Redundant CGW Pattern | Two separate truck depots dispatching convoys | A second on-premises device creates a second VPN connection (4 tunnels total); eliminates the single-depot failure scenario |
| VPN CloudHub | A single hub warehouse connecting multiple branch depots | Multiple branch offices each VPN into the same VGW; the VGW re-advertises routes so branches can reach each other through the hub |
| Accelerated VPN | Express toll-road bypassing local traffic | Traffic enters the AWS Global Accelerator network at the nearest edge location, then rides the AWS backbone instead of congested public highways |
| ECMP | Dispatching convoys across all available highways simultaneously | Transit Gateway distributes flows across multiple VPN tunnels using 5-tuple hashing, aggregating bandwidth beyond the 1.25 Gbps single-tunnel limit |
| NAT-Traversal (NAT-T) | Wrapping the armored truck inside a shipping container to fit through a narrow tunnel | Encapsulates ESP packets inside UDP 4500 so IPSec can traverse NAT devices; AWS auto-detects and enables when needed |
| Dead Peer Detection (DPD) | A "check-in" radio signal between the depot and the hub | R_U_THERE messages every 10s; configurable timeout (min 30s); detects unreachable peers and triggers failover |
| CloudWatch TunnelState | The dispatch board showing which convoy routes are active | A metric that reads 1 (UP) or 0 (DOWN) for each tunnel; alarm on this to detect single-tunnel-down states before they become outages |

---

## The Big Picture

In the [Direct Connect Deep Dive](./2026-02-24-aws-direct-connect.md), you built a **private railway** -- dedicated fiber from your data center to the AWS rail network. That railway provides massive bandwidth and consistent latency, but it takes weeks to provision and costs significantly more than internet connectivity. The DX comparison table noted that **Site-to-Site VPN** is the complementary service: encrypted tunnels over the public internet that deploy in minutes, cost less, but sacrifice bandwidth consistency.

Today you learn everything about VPN from the AWS side: how the tunnel is negotiated, where it terminates (VGW vs TGW), how routing works (static vs BGP), how to scale throughput with ECMP, how to accelerate it with Global Accelerator, and how to architect high availability with redundant tunnels and customer gateways. By the end, you will understand the complete hybrid connectivity toolkit: **DX for the highway, VPN for the backup route (or for when you cannot wait weeks for the highway to be built)**.

Think of it this way. Direct Connect is a **private railway** -- expensive infrastructure with predictable schedules. Site-to-Site VPN is an **armored convoy on public highways** -- every shipment is encrypted and locked, but you share the road with everyone else. The convoy is cheap, fast to dispatch, and works immediately. The trade-off is that highway congestion affects delivery times, and a single truck can only carry 1.25 Gbps of cargo (though you can send many trucks at once via ECMP).

```
THE HYBRID CONNECTIVITY TOOLKIT -- WHERE VPN FITS
=============================================================================

  YOUR DATA CENTER                     AWS
  ┌──────────────────┐                 ┌──────────────────────────────────────┐
  │                  │                 │                                      │
  │  Customer        │  DIRECT         │  ┌────────┐                         │
  │  Gateway         │  CONNECT        │  │ DXGW   │──▶ TGW ──▶ VPCs       │
  │  Device(s)       │═══(private═════▶│  └────────┘   (private railway --  │
  │                  │   fiber)         │               consistent, high BW)  │
  │  ┌──────────┐   │                 │                                      │
  │  │ Router/  │   │  SITE-TO-SITE   │  ┌────────┐                         │
  │  │ Firewall │   │  VPN            │  │ VGW or │──▶ VPC(s)              │
  │  │          │   │══(encrypted════▶│  │ TGW    │   (armored convoy --   │
  │  └──────────┘   │  over internet)  │  └────────┘   encrypted, fast      │
  │                  │                 │               setup, variable BW)   │
  │                  │  ACCELERATED    │  ┌────────┐                         │
  │                  │  VPN            │  │ Global │──▶ TGW ──▶ VPCs       │
  │                  │══(encrypted════▶│  │ Accel  │   (express toll-road -- │
  │                  │  via edge loc)   │  └────────┘   less congestion)     │
  └──────────────────┘                 └──────────────────────────────────────┘

  DECISION FRAMEWORK:
  ├── Need it NOW (minutes)?                    → VPN
  ├── Need consistent latency + high bandwidth? → Direct Connect
  ├── Need encryption over DX?                  → MACsec or VPN-over-DX
  ├── Need encrypted backup for DX?             → VPN as standby
  ├── Need VPN + less jitter?                   → Accelerated VPN (TGW only)
  └── Need to scale VPN beyond 1.25 Gbps?      → ECMP on TGW
```

---

## Part 1: VPN Fundamentals -- How the Armored Convoy Works

### The IPSec Tunnel

Every Site-to-Site VPN connection is a pair of **IPSec tunnels**. IPSec (Internet Protocol Security) is not an AWS invention -- it is an industry-standard protocol suite that provides encryption, authentication, and integrity checking at the network layer.

**The Analogy**: An IPSec tunnel is an **armored truck**. Before the truck can leave the depot, both the depot (your router) and the receiving hub (AWS) must agree on a set of security protocols: which lock to use (encryption algorithm), how to verify each other's identity (authentication), and how to exchange keys (IKE negotiation). Once they agree, the truck is loaded, locked, and dispatched. Every packet inside is encrypted -- even if someone intercepts the truck on the highway, they cannot read the cargo.

**The Technical Reality**:

```
IPSec TUNNEL NEGOTIATION -- TWO-PHASE PROCESS
=============================================================================

  PHASE 1: IKE (Internet Key Exchange) -- Establishing Trust
  ────────────────────────────────────────────────────────────
  ├── Purpose: Authenticate both sides, establish a secure channel
  │   for negotiating the actual data encryption parameters
  ├── IKEv2 is the default (IKEv1 also supported for legacy devices)
  ├── Authentication: Pre-shared key (PSK) -- AWS generates this and
  │   includes it in the VPN configuration file you download
  ├── Encryption: AES-128-GCM, AES-256-GCM, AES-128-CBC, AES-256-CBC
  ├── Integrity: SHA2-256, SHA2-384, SHA2-512 (SHA1 deprecated)
  ├── DH Group: 14, 15, 16, 17, 18, 19, 20, 21 (higher = stronger)
  │   NOTE: Group 2 (1024-bit MODP) is technically supported for legacy
  │   devices but is considered INSECURE — never use groups below 14 in production
  └── Result: An IKE Security Association (SA) -- a trusted channel

  PHASE 2: IPSec SA -- The Actual Data Tunnel
  ────────────────────────────────────────────────────────────
  ├── Purpose: Negotiate encryption for the actual traffic
  ├── ESP (Encapsulating Security Payload) protocol
  ├── Encryption: Same options as Phase 1
  ├── Integrity: Same options as Phase 1
  ├── PFS (Perfect Forward Secrecy): New DH key exchange per SA
  │   Even if one SA key is compromised, past/future sessions remain safe
  ├── Tunnel mode: Entire IP packet is encrypted and encapsulated
  │   inside a new IP packet with the tunnel endpoint IPs
  └── Result: Encrypted data flows between your network and AWS

  TUNNEL ADDRESSING:
  ════════════════════════════════════════════════════════
  ├── Outer IP: Your CGW public IP ↔ AWS VPN endpoint public IP
  │   (these are the "truck" addresses visible on the highway)
  ├── Inner IP: 169.254.x.x/30 link-local addresses (for BGP peering)
  │   (these are the "radio frequencies" the depots use to talk)
  ├── AWS assigns two tunnel endpoint IPs (one per tunnel, in different AZs)
  └── You can customize inner tunnel CIDRs if 169.254.x.x conflicts

  WHAT THE VPN CONFIGURATION FILE CONTAINS:
  ════════════════════════════════════════════════════════
  ├── AWS-side tunnel endpoint public IPs (2 endpoints)
  ├── Pre-shared keys for each tunnel
  ├── Inner tunnel IP addresses (169.254.x.x/30 per tunnel)
  ├── BGP ASN and peer IP (if using dynamic routing)
  ├── IKE and IPSec parameter recommendations
  └── Device-specific configuration snippets (Cisco, Juniper, etc.)
```

### The Three Core Components

```
THREE COMPONENTS OF A SITE-TO-SITE VPN
=============================================================================

  COMPONENT 1: CUSTOMER GATEWAY (CGW) RESOURCE
  ────────────────────────────────────────────────────
  ├── An AWS resource (logical, not physical)
  ├── Represents your on-premises router in AWS
  ├── You provide:
  │   ├── Public IP address of your on-prem device
  │   ├── BGP ASN (if using dynamic routing, e.g. 65000)
  │   ├── Certificate ARN (optional, for certificate-based auth)
  │   │   WHEN TO USE: Certificate auth vs PSK (pre-shared key):
  │   │   ├── PSK (default): Simple, AWS generates it, good for most setups
  │   │   ├── Certificate: Use when you need automated key rotation,
  │   │   │   stronger security (asymmetric crypto), or integration with
  │   │   │   your existing PKI infrastructure (e.g., AWS Private CA)
  │   │   └── Certificate eliminates the risk of PSK compromise/leakage
  │   └── Device type (optional, for vendor-specific config download)
  ├── The CGW resource itself does NOT create a tunnel
  └── It is a reference that the VPN connection uses

  COMPONENT 2: VPN TERMINATION POINT ON AWS SIDE
  ────────────────────────────────────────────────────
  ├── Option A: Virtual Private Gateway (VGW)
  │   ├── Attached to a single VPC
  │   ├── No ECMP (limited to 1.25 Gbps per tunnel)
  │   ├── Supports route propagation to VPC route tables
  │   ├── Supports VPN CloudHub
  │   └── Best for: simple single-VPC connectivity
  │
  └── Option B: Transit Gateway (TGW)
      ├── Connects to thousands of VPCs via attachments
      ├── ECMP for bandwidth aggregation (requires BGP)
      ├── Supports Accelerated VPN
      ├── Supports IPv4 and IPv6 inner/outer tunnel addresses
      └── Best for: multi-VPC, high-throughput, production

  COMPONENT 3: VPN CONNECTION
  ────────────────────────────────────────────────────
  ├── The actual tunnel pair linking the CGW to VGW or TGW
  ├── Always creates TWO tunnels (different AZ endpoints)
  ├── You choose: static routing or dynamic (BGP)
  ├── Pricing:
  │   ├── Standard tunnels: $0.05/hr per VPN connection (~$36/month)
  │   ├── Large Bandwidth (5 Gbps) tunnels: $0.60/hr (~$432/month) — 12x more
  │   └── Plus standard data transfer out charges
  └── AWS generates a configuration file for your device
```

---

## Part 2: VGW vs TGW as VPN Termination Points

This decision mirrors the VGW-vs-TGW choice you saw with Direct Connect. The [Transit Gateway Deep Dive](./2026-02-23-transit-gateway-deep-dive.md) covered the five attachment types -- the VPN attachment is one of them. Here is the VPN-specific comparison.

**The Analogy**: A VGW is a **small receiving dock at one warehouse** -- your convoy can deliver to that single building, but nowhere else. A Transit Gateway is the **central logistics hub** -- your convoy pulls into the hub, and from there, packages are distributed to any warehouse in the network.

```
VGW vs TGW FOR VPN TERMINATION
=============================================================================

                              VGW                      TGW
  ──────────────────────────────────────────────────────────────────────
  VPC reach                  1 VPC only               Thousands of VPCs
                             (VGW is attached to       (via TGW VPC
                              a single VPC)             attachments)

  ECMP support               No                        Yes (with BGP)
                             (max 1.25 Gbps per        (aggregate across
                              tunnel, 2 tunnels)        multiple tunnels)

  Max aggregate BW           ~1.25 Gbps                ~50 Gbps
                             (only ONE tunnel active    (many VPN connections
                              at a time; the second      with ECMP enabled)
                              is hot standby)

  Accelerated VPN            No                         Yes

  IPv6 inner tunnel          No                         Yes

  VPN CloudHub               Yes                        N/A (use TGW route
                             (multiple CGWs to           tables for multi-site)
                              same VGW)

  Route propagation          Propagates routes to       Propagates routes to
                             VPC route tables            TGW route tables
                             (enable via VPC RT)         (via attachment
                                                         propagation)

  Static route limit         100 per VPN connection     1,000 per VPN
                                                        connection

  Pricing                    VPN connection-hours       VPN connection-hours
                             only                       + TGW attachment-hours
                                                        + TGW data processing
                                                        ($0.02/GB)

  VPN connections per        10 per VGW                 Up to thousands
  endpoint                                              (account limits apply)

  WHEN TO USE EACH:
  ════════════════════════════════════════════════════════
  VGW:
  ├── Single VPC connectivity (lab, dev, simple prod)
  ├── VPN CloudHub for small multi-site (< 5 branches)
  ├── Cost-sensitive (no TGW data processing charges)
  └── No need for ECMP or Accelerated VPN

  TGW:
  ├── Multi-VPC environment (your network has TGW already)
  ├── Need ECMP for bandwidth beyond 1.25 Gbps
  ├── Need Accelerated VPN for reduced jitter
  ├── Enterprise multi-account architecture
  └── DX + VPN backup pattern (both terminate on TGW)
```

### Terraform: VGW-Based VPN Connection

```hcl
# ==============================================================
# Customer Gateway — represents your on-premises router
# ==============================================================

resource "aws_customer_gateway" "main" {
  bgp_asn    = 65000            # Your on-premises BGP ASN
  ip_address = "203.0.113.10"   # Public IP of your on-prem device
  type       = "ipsec.1"        # Only supported type

  tags = { Name = "hq-customer-gateway" }
}

# ==============================================================
# Virtual Private Gateway — AWS-side VPN endpoint on a single VPC
# ==============================================================

resource "aws_vpn_gateway" "main" {
  vpc_id          = aws_vpc.main.id
  amazon_side_asn = "64512"     # AWS-side BGP ASN (default 64512)

  tags = { Name = "prod-vgw" }
}

# ==============================================================
# VPN Connection — creates two IPSec tunnels
# ==============================================================

resource "aws_vpn_connection" "hq_to_aws" {
  customer_gateway_id = aws_customer_gateway.main.id
  vpn_gateway_id      = aws_vpn_gateway.main.id
  type                = "ipsec.1"

  # Dynamic routing via BGP (recommended for production)
  static_routes_only = false

  # Tunnel 1 configuration
  tunnel1_ike_versions                 = ["ikev2"]
  tunnel1_phase1_encryption_algorithms = ["AES256-GCM-16"]
  tunnel1_phase1_integrity_algorithms  = ["SHA2-256"]
  tunnel1_phase1_dh_group_numbers      = [20]
  tunnel1_phase2_encryption_algorithms = ["AES256-GCM-16"]
  tunnel1_phase2_integrity_algorithms  = ["SHA2-256"]
  tunnel1_phase2_dh_group_numbers      = [20]

  # Tunnel 2 configuration (same parameters for consistency)
  tunnel2_ike_versions                 = ["ikev2"]
  tunnel2_phase1_encryption_algorithms = ["AES256-GCM-16"]
  tunnel2_phase1_integrity_algorithms  = ["SHA2-256"]
  tunnel2_phase1_dh_group_numbers      = [20]
  tunnel2_phase2_encryption_algorithms = ["AES256-GCM-16"]
  tunnel2_phase2_integrity_algorithms  = ["SHA2-256"]
  tunnel2_phase2_dh_group_numbers      = [20]

  tags = { Name = "hq-to-aws-vpn" }
}

# ==============================================================
# Enable VGW route propagation in VPC route table
# (so VPN-learned routes appear in the VPC automatically)
# ==============================================================

resource "aws_vpn_gateway_route_propagation" "private" {
  vpn_gateway_id = aws_vpn_gateway.main.id
  route_table_id = aws_route_table.private.id
}

# ==============================================================
# Output the VPN configuration for your on-prem device
# ==============================================================

output "vpn_tunnel1_address" {
  value       = aws_vpn_connection.hq_to_aws.tunnel1_address
  description = "AWS-side tunnel 1 public IP"
}

output "vpn_tunnel2_address" {
  value       = aws_vpn_connection.hq_to_aws.tunnel2_address
  description = "AWS-side tunnel 2 public IP"
}

output "vpn_tunnel1_preshared_key" {
  value       = aws_vpn_connection.hq_to_aws.tunnel1_preshared_key
  sensitive   = true
  description = "Pre-shared key for tunnel 1 IKE negotiation"
}
```

### Terraform: TGW-Based VPN Connection with ECMP

```hcl
# ==============================================================
# Transit Gateway — central hub (likely already exists from
# your TGW deployment; shown here for completeness)
# ==============================================================

resource "aws_ec2_transit_gateway" "main" {
  amazon_side_asn                 = 64512
  default_route_table_association = "disable"  # Explicit route tables
  default_route_table_propagation = "disable"
  vpn_ecmp_support                = "enable"   # CRITICAL for ECMP

  tags = { Name = "central-tgw" }
}

# ==============================================================
# Customer Gateway — same resource, different VPN connection target
# ==============================================================

resource "aws_customer_gateway" "hq" {
  bgp_asn    = 65000
  ip_address = "203.0.113.10"
  type       = "ipsec.1"

  tags = { Name = "hq-cgw" }
}

# ==============================================================
# VPN Connection 1 — terminates on TGW (not VGW)
# ==============================================================

resource "aws_vpn_connection" "hq_vpn_1" {
  customer_gateway_id = aws_customer_gateway.hq.id
  transit_gateway_id  = aws_ec2_transit_gateway.main.id
  type                = "ipsec.1"
  static_routes_only  = false  # BGP required for ECMP

  # Enable acceleration (routes through Global Accelerator)
  enable_acceleration = false  # Set true for Accelerated VPN

  tags = { Name = "hq-vpn-connection-1" }
}

# ==============================================================
# VPN Connection 2 — second connection for ECMP aggregation
# (same CGW, same TGW, different tunnel endpoints)
# ==============================================================

resource "aws_vpn_connection" "hq_vpn_2" {
  customer_gateway_id = aws_customer_gateway.hq.id
  transit_gateway_id  = aws_ec2_transit_gateway.main.id
  type                = "ipsec.1"
  static_routes_only  = false

  tags = { Name = "hq-vpn-connection-2" }
}

# ==============================================================
# TGW Route Table — VPN attachment association and propagation
# ==============================================================

resource "aws_ec2_transit_gateway_route_table" "shared" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "shared-services-rt" }
}

# Associate VPN attachment with the shared route table
resource "aws_ec2_transit_gateway_route_table_association" "vpn_1" {
  transit_gateway_attachment_id  = aws_vpn_connection.hq_vpn_1.transit_gateway_attachment_id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.shared.id
}

# Propagate VPN routes (on-prem CIDRs) to spoke route tables
resource "aws_ec2_transit_gateway_route_table_propagation" "vpn_1_to_spoke" {
  transit_gateway_attachment_id  = aws_vpn_connection.hq_vpn_1.transit_gateway_attachment_id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.spoke.id  # (defined elsewhere — see TGW doc)
}

# Same for VPN connection 2 (ECMP requires both connections to
# propagate the same prefixes with the same attributes)
resource "aws_ec2_transit_gateway_route_table_association" "vpn_2" {
  transit_gateway_attachment_id  = aws_vpn_connection.hq_vpn_2.transit_gateway_attachment_id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.shared.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "vpn_2_to_spoke" {
  transit_gateway_attachment_id  = aws_vpn_connection.hq_vpn_2.transit_gateway_attachment_id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.spoke.id
}
```

---

## Part 3: Routing -- Static vs Dynamic (BGP)

### The Analogy

**Static routing** is a **fixed paper map** given to every convoy driver. It tells them exactly which highways to take. If a road closes, nobody updates the map -- the convoy drives into the closure and stops. Someone must manually draw a new map.

**Dynamic routing (BGP)** is a **radio dispatch system** where the central dispatcher broadcasts road closures to all drivers. Drivers do not discover closures themselves in real time -- they rely on periodic check-ins (hold timers, default ~90 seconds) and are rerouted once the dispatcher confirms the closure. It is much faster than paper maps, but not instantaneous (~30-90 second convergence). The dispatch system also lets you influence which routes are preferred by adjusting preferences (LOCAL_PREF), making paths look longer (AS_PATH prepending), or marking route costs (MED).

```
STATIC vs DYNAMIC ROUTING
=============================================================================

                          Static Routing            Dynamic Routing (BGP)
  ──────────────────────────────────────────────────────────────────────────
  Configuration           Manual: specify on-prem   Automatic: BGP peers
                          CIDRs in AWS and          exchange routes
                          AWS CIDRs on-prem         dynamically

  Failover                Manual intervention        Automatic: BGP detects
                          (change routes by hand)    dead peer, reroutes

  ECMP (on TGW)           NOT supported              Supported (mandatory
                                                     requirement for ECMP)

  Route limit (VGW)       100 static routes          100 BGP-learned routes

  Route limit (TGW)       1,000 static routes        5,000 BGP routes (per
                                                     route table, increasing)

  On-prem route changes   Must update AWS config     Automatically advertised
                          manually                   via BGP

  AWS route changes        Must update on-prem        Automatically advertised
                          device manually             via BGP

  VPN CloudHub            Not supported               Required (unique ASN
                          (requires BGP)              per site so VGW can
                                                      distinguish route ads
                                                      from different branches)

  Path influence           None                       AS_PATH prepending,
                                                     LOCAL_PREF (communities),
                                                     MED

  Dead Peer Detection     DPD only (tunnel level)    DPD + BGP hold timer +
                                                     optional BFD-like behavior

  AWS RECOMMENDATION:
  ════════════════════════════════════════════════════════
  ALWAYS use BGP for production VPN connections.
  ├── Automatic failover when a tunnel goes down
  ├── Required for ECMP bandwidth aggregation
  ├── Required for VPN CloudHub multi-site connectivity
  ├── Enables path preference via AS_PATH prepending
  └── Reduces operational burden (no manual route updates)

  Use static routing ONLY when:
  ├── Your on-premises device does not support BGP
  ├── Simple lab/dev environment with a single route
  └── You accept the risk of manual failover


ROUTE PRIORITY ON VGW (when both static and BGP routes exist):
  ════════════════════════════════════════════════════════
  1. Static routes (manually configured) — HIGHEST priority
  2. Longest prefix match (most specific route wins)
  3. Direct Connect BGP routes (if DX and VPN coexist on VGW)
  4. VPN BGP routes — LOWEST priority

  KEY INSIGHT: On a VGW, if you have a static route 10.0.0.0/16
  and a BGP-learned route 10.0.0.0/16, the static route wins.
  This is the opposite of what some routing protocols do
  (where dynamic is preferred). On TGW, same behavior:
  static routes always take precedence over propagated routes.
```

### BGP Path Selection for VPN

When you have multiple VPN connections (or VPN + DX) advertising the same prefix, BGP determines the winning path. This directly extends the BGP evaluation order from the [Direct Connect doc](./2026-02-24-aws-direct-connect.md):

```
BGP PATH SELECTION -- VPN-SPECIFIC SCENARIOS
=============================================================================

  SCENARIO 1: ACTIVE/PASSIVE VPN TUNNELS (VGW)
  ────────────────────────────────────────────────────
  VGW provides 2 tunnels but does NOT load-balance between them.
  ├── AWS uses one tunnel at a time for outbound traffic from the VPC.
  │   Which tunnel is active depends on route attributes and the AZ
  │   of the originating instance — there is no deterministic "tunnel 1 preferred" rule.
  ├── The other tunnel is standby (hot standby, established but idle)
  ├── You can influence which tunnel is active by:
  │   ├── AS_PATH prepending on the less-preferred tunnel
  │   │   (add extra hops to make it look "farther away")
  │   └── Setting MED to a higher value on the backup tunnel
  └── Failover: When active tunnel drops, BGP reroutes to standby
      (seconds with aggressive timers, ~30s with defaults)

  SCENARIO 2: ECMP ON TGW (ACTIVE/ACTIVE)
  ────────────────────────────────────────────────────
  TGW with vpn_ecmp_support = "enable" and multiple VPN connections:
  ├── All tunnels that advertise the SAME prefix with the SAME
  │   attributes (AS_PATH length, MED, origin) are ECMP candidates
  ├── TGW distributes FLOWS (not packets) across tunnels
  │   using a 5-tuple hash: src IP, dst IP, src port, dst port, protocol
  ├── A single flow is ALWAYS pinned to one tunnel
  │   (single flow max = tunnel bandwidth, regardless of ECMP)
  ├── More connections = more aggregate bandwidth:
  │   STANDARD TUNNELS (1.25 Gbps each):
  │     2 VPN connections (4 tunnels) = up to 5 Gbps aggregate
  │     4 VPN connections (8 tunnels) = up to 10 Gbps aggregate
  │   LARGE BANDWIDTH TUNNELS (5 Gbps each, TGW/Cloud WAN only):
  │     2 VPN connections (4 tunnels) = up to 20 Gbps aggregate
  │     4 VPN connections (8 tunnels) = up to 40 Gbps aggregate
  └── Requirements:
      ├── BGP dynamic routing (static does NOT support ECMP)
      ├── Same prefix advertised from all connections
      ├── Same AS_PATH length from all connections
      └── vpn_ecmp_support enabled on TGW at creation time

  SCENARIO 3: DX PRIMARY + VPN BACKUP (TGW)
  ────────────────────────────────────────────────────
  This is the enterprise pattern from yesterday's DX doc.
  TWO SEPARATE MECHANISMS FOR TWO DIRECTIONS:

  ON-PREM → AWS (BGP communities control preference):
  ├── DX routes tagged with community 7224:7300 → high LOCAL_PREF on AWS side
  ├── VPN routes tagged with community 7224:7100 → low LOCAL_PREF on AWS side
  ├── On-prem router also sets LOCAL_PREF higher for DX-learned AWS routes
  └── Result: on-prem always sends traffic via DX when healthy

  AWS → ON-PREM (TGW built-in route priority, no config needed):
  ├── 1. Static routes (highest)
  ├── 2. Longest prefix match
  ├── 3. DX propagated routes (preferred over VPN for equal prefix)
  └── 4. VPN propagated routes (lowest)

  FAILOVER SEQUENCE (when DX fails):
  ├── BGP hold timer expires (~90s default, faster with BFD if configured)
  ├── DX BGP session tears down, DX routes withdrawn from TGW route table
  ├── VPN routes (already present but previously shadowed) become sole routes
  ├── TGW immediately forwards via VPN attachment — no recalculation needed
  ├── Simultaneously: on-prem router's DX session drops, VPN-learned AWS
  │   routes become preferred on the on-prem routing table
  └── Both directions converge independently through parallel BGP events

  KEY INSIGHT: Failover is fast because VPN routes are already in the
  TGW route table before DX fails. Nothing needs to be discovered or
  negotiated — only preference shifts.

  ├── DX recovery: BGP re-establishes, higher-preference DX routes
  │   re-installed, traffic returns to DX automatically
  └── No manual intervention in either direction
```

---

## Part 4: VPN CloudHub -- Connecting Branch Offices

### The Analogy

Imagine a company with a headquarters and three branch offices in different cities. Instead of building direct roads between every pair of offices (expensive, complex), you designate one **central hub warehouse** (the VGW). Each branch dispatches armored convoys to the hub. The hub's dispatch board re-advertises each branch's location to all other branches. Now Branch A's convoy can deliver to Branch B by routing through the hub -- the hub serves as a relay point.

```
VPN CLOUDHUB -- MULTI-SITE HUB-AND-SPOKE VIA VGW
=============================================================================

                        ┌──────────────────────────┐
                        │    Virtual Private        │
                        │    Gateway (VGW)          │
                        │                           │
                        │  Re-advertises routes     │
                        │  from each site to all    │
                        │  other sites via BGP      │
                        │                           │
                        │  ASN: 64512 (AWS-side)    │
                        └─────┬───────┬───────┬─────┘
                              │       │       │
                    VPN 1     │       │       │     VPN 3
                   ┌──────────┘       │       └──────────┐
                   │                  │                   │
                   ▼            VPN 2 ▼                   ▼
          ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
          │ Branch A       │  │ Branch B       │  │ HQ (also DX)   │
          │ CGW: 198.51.1.1│  │ CGW: 198.51.2.1│  │ CGW: 198.51.3.1│
          │ ASN: 65001     │  │ ASN: 65002     │  │ ASN: 65003     │
          │ CIDR: 10.1.0/24│  │ CIDR: 10.2.0/24│  │ CIDR: 10.3.0/24│
          └────────────────┘  └────────────────┘  └────────────────┘

  HOW CLOUDHUB WORKS:
  ────────────────────────────────────────────────────
  1. Branch A advertises 10.1.0.0/24 to VGW via BGP (ASN 65001)
  2. Branch B advertises 10.2.0.0/24 to VGW via BGP (ASN 65002)
  3. HQ advertises 10.3.0.0/24 to VGW via BGP (ASN 65003)
  4. VGW re-advertises ALL routes to ALL sites:
     ├── Branch A learns: 10.2.0.0/24 (Branch B), 10.3.0.0/24 (HQ)
     ├── Branch B learns: 10.1.0.0/24 (Branch A), 10.3.0.0/24 (HQ)
     └── HQ learns: 10.1.0.0/24 (Branch A), 10.2.0.0/24 (Branch B)
  5. Branch A → Branch B traffic flows: A → VPN → VGW → VPN → B

  REQUIREMENTS:
  ├── Each site MUST have a unique BGP ASN
  ├── BGP dynamic routing is mandatory (no static)
  ├── IP ranges must NOT overlap between sites
  ├── All VPN connections to the SAME VGW
  └── VGW can also have a DX connection (mix VPN + DX)

  LIMITATIONS:
  ├── Branch-to-branch traffic goes through AWS (added latency)
  │   WHY IT IS SLOW: Traffic crosses the public internet TWICE —
  │   Branch A → internet → VGW → internet → Branch B. Each leg is
  │   subject to independent internet congestion, so latency is additive.
  │   This is an inherent architectural problem (double internet traversal
  │   through a relay point), NOT a shared bandwidth cap issue.
  │   NOTE: The 1.25 Gbps limit is per-tunnel, not shared across all
  │   VPN connections on the VGW. Each branch has its own tunnel pair.
  ├── No ECMP between branches (VGW limitation)
  ├── All branches share the VGW's single VPC connectivity
  ├── Scales to ~10 sites; beyond that, use TGW
  └── Bandwidth limited by VPN tunnel capacity (1.25 Gbps/tunnel)

  COST-EFFECTIVE ALTERNATIVE TO TGW FOR SMALL DEPLOYMENTS:
  ├── CloudHub: N VPN connections to 1 VGW. Cost = N x $0.05/hr
  ├── TGW: N VPN attachments + TGW hourly + data processing.
  │   Cost = TGW($0.05/hr) + N x attachment($0.05/hr) + $0.02/GB
  └── For < 5 sites with low bandwidth, CloudHub wins on cost
```

### VPN Concentrator (November 2025) -- The Modern Multi-Site Solution

AWS launched **Site-to-Site VPN Concentrator** as the newer, scalable alternative to CloudHub for connecting many branch offices. While CloudHub works for < 5 sites on a VGW, the Concentrator is purpose-built for 25-100+ low-bandwidth remote sites through a single TGW attachment.

```
VPN CONCENTRATOR vs CLOUDHUB
=============================================================================

                          CloudHub (VGW)             VPN Concentrator (TGW)
  ──────────────────────────────────────────────────────────────────────────
  Target scale             < 5-10 sites              25-100+ sites
  Termination              VGW                        Single TGW attachment
  Max aggregate BW         1.25 Gbps (1 tunnel)       5 Gbps per concentrator
  Routing                  BGP (unique ASN/site)      BGP only (no static)
  Pricing                  N x $0.05/hr               ~$1.95/hr per concentrator
                           (per VPN connection)        + $0.01/hr per connection
  TGW data processing      N/A (VGW)                  Included in concentrator
  Use case                 Small branch network        Large branch network
                           budget-sensitive             where individual VPN
                                                       attachments per site
                                                       would be too expensive

  WHEN TO CHOOSE CONCENTRATOR:
  ├── You have 25+ branch offices with low bandwidth needs
  ├── Individual TGW VPN attachments per site are cost-prohibitive
  ├── You want a single TGW attachment to manage (simpler routing)
  └── All sites support BGP (no static routing option)
```

---

## Part 5: Accelerated Site-to-Site VPN

### The Analogy

A standard VPN tunnel is a convoy on **regular public highways** -- traffic jams, construction zones, and unpredictable delays. Accelerated VPN puts that same convoy on an **express toll-road network** (AWS Global Accelerator). The convoy enters the toll-road at the nearest on-ramp (the nearest AWS edge location), then travels on private, congestion-free toll roads (the AWS global backbone) directly to the destination. The cargo is still encrypted the same way -- the difference is the road quality.

```
ACCELERATED VPN vs STANDARD VPN -- PACKET PATH
=============================================================================

  STANDARD VPN:
  ────────────────────────────────────────────────────
  On-prem → ISP → public internet (hop, hop, hop, hop, hop)
  → AWS Region VPN endpoint

  Path quality: Variable. Each hop is a different ISP's network.
  Jitter, packet loss, and latency fluctuate with internet congestion.

         On-Prem           Public Internet            AWS Region
         ┌──────┐     ┌─── ─── ─── ─── ───┐        ┌──────────┐
         │ CGW  │────▶│ ISP → ISP → ISP → │───────▶│ TGW      │
         │      │     │ ISP → ISP → ISP   │        │ VPN      │
         └──────┘     └─── ─── ─── ─── ───┘        │ endpoint │
                       (unpredictable path)          └──────────┘


  ACCELERATED VPN:
  ────────────────────────────────────────────────────
  On-prem → ISP → nearest AWS edge location (Global Accelerator
  anycast IP) → AWS global backbone → AWS Region VPN endpoint

  Path quality: Consistent. Only the first hop (to the edge) is on
  the public internet. Everything after rides the AWS backbone.

         On-Prem      Public     Edge Location    AWS Backbone
         ┌──────┐     Internet   ┌────────────┐  ┌──────────────┐
         │ CGW  │────▶(1 hop)──▶│ GA Anycast  │─▶│ Private AWS  │
         │      │               │ IP          │  │ network      │
         └──────┘               └────────────┘  │              │
                                                 │    ┌────────┐│
                                                 │    │TGW VPN ││
                                                 │    │endpoint││
                                                 │    └────────┘│
                                                 └──────────────┘

  KEY DETAILS:
  ════════════════════════════════════════════════════════
  ├── ONLY works with Transit Gateway (not VGW)
  ├── AWS creates 2 accelerators (1 per tunnel) — managed, invisible
  │   in the Global Accelerator console
  ├── Uses anycast IPs instead of regional VPN endpoint IPs
  │   (your CGW device connects to the anycast IPs)
  ├── Cannot be enabled/disabled after creation — you must create
  │   a new VPN connection with enable_acceleration = true
  ├── PRICING (on top of standard VPN charges):
  │   ├── Global Accelerator fixed fee: ~$0.025/hr per accelerator
  │   │   (2 accelerators per VPN = ~$0.05/hr additional)
  │   └── Data Transfer Premium: $0.015-$0.091/GB depending on region
  │       (this is ON TOP of standard data transfer charges)
  └── WHEN TO USE:
      ├── On-prem is geographically far from the AWS Region
      ├── Internet path quality is poor (high jitter, packet loss)
      ├── Applications sensitive to latency variability (VoIP, RDP)
      └── NOT worth it if on-prem is near the AWS Region (minimal benefit)
```

---

## Part 6: High Availability -- Eliminating Single Points of Failure

### Dual-Tunnel Architecture (Built-in)

Every VPN connection creates **two tunnels**, each terminating at a different AWS endpoint in a different AZ. This is automatic -- you do not configure it.

**The Analogy**: Every time you dispatch an armored convoy, you actually send **two convoys on two different highways through two different towns**. If a bridge collapses on Highway A, the convoy on Highway B delivers the cargo. You always have two routes active.

```
DUAL-TUNNEL BUILT-IN REDUNDANCY
=============================================================================

  On-Premises                          AWS Region (us-east-1)
  ┌──────────────┐                     ┌──────────────────────────────────┐
  │              │     Tunnel 1        │                                  │
  │  Customer    │════════════════════▶│  VPN Endpoint A (AZ-1)          │
  │  Gateway     │                     │        │                         │
  │  Device      │     Tunnel 2        │        ▼                         │
  │  (single)    │════════════════════▶│  VPN Endpoint B (AZ-2)          │
  │              │                     │        │                         │
  └──────────────┘                     │        ▼                         │
                                       │  VGW or TGW                     │
                                       │        │                         │
                                       │        ▼                         │
                                       │  VPC(s)                         │
                                       └──────────────────────────────────┘

  BEHAVIOR:
  ├── Both tunnels are established simultaneously
  ├── With VGW: active/passive — AWS sends traffic on one tunnel,
  │   standby on the other (you can influence with AS_PATH/MED)
  ├── With TGW + ECMP: active/active — both tunnels carry traffic
  ├── Tunnel health: DPD (Dead Peer Detection) sends R_U_THERE every 10 seconds
  │   ├── DPD timeout is configurable (minimum 30 seconds)
  │   ├── After timeout expires without response, peer is considered dead → tunnel DOWN
  │   └── DPD action options: "clear" (close tunnel), "restart" (renegotiate IKE), "none" (do nothing)
  ├── TUNNEL ENDPOINT MAINTENANCE:
  │   AWS periodically replaces VPN tunnel endpoints for security patching.
  │   ├── During replacement, ONE tunnel goes down for a few minutes
  │   ├── This is expected behavior — NOT a sign of misconfiguration
  │   ├── If you configured BOTH tunnels, traffic fails over seamlessly
  │   ├── If you only configured ONE tunnel, you get a full outage
  │   └── INTERVIEW Q: "Why does my VPN tunnel go down periodically even
  │       though nothing changed on my side?" → AWS endpoint maintenance
  └── YOUR RESPONSIBILITY: configure BOTH tunnels on your device
      (a common mistake is configuring only one tunnel, losing all
       redundancy)

  SINGLE POINT OF FAILURE REMAINING:
  ════════════════════════════════════════════════════════
  The on-premises Customer Gateway device itself.
  If your single router fails, BOTH tunnels go down.
  Solution: redundant CGW devices (next section).
```

### Redundant Customer Gateways (4-Tunnel Pattern)

```
REDUNDANT CUSTOMER GATEWAYS -- MAXIMUM VPN RESILIENCE
=============================================================================

  On-Premises                              AWS Region
  ┌──────────────────────────────┐         ┌──────────────────────────┐
  │                              │         │                          │
  │  ┌─────────────┐            │  VPN 1  │                          │
  │  │ CGW Device A│────Tunnel 1A════════▶│ Endpoint AZ-1            │
  │  │ 203.0.113.10│────Tunnel 1B════════▶│ Endpoint AZ-2            │
  │  └─────────────┘            │         │       │                   │
  │                              │         │       ▼                   │
  │  ┌─────────────┐            │  VPN 2  │  VGW or TGW              │
  │  │ CGW Device B│────Tunnel 2A════════▶│ Endpoint AZ-3            │
  │  │ 203.0.113.20│────Tunnel 2B════════▶│ Endpoint AZ-4            │
  │  └─────────────┘            │         │       │                   │
  │                              │         │       ▼                   │
  │  (two separate physical      │         │  VPC(s)                  │
  │   routers/firewalls)         │         │                          │
  └──────────────────────────────┘         └──────────────────────────┘

  HOW TO BUILD THIS:
  ════════════════════════════════════════════════════════
  1. Create CGW Resource A (ip_address = 203.0.113.10, bgp_asn = 65000)
  2. Create CGW Resource B (ip_address = 203.0.113.20, bgp_asn = 65000)
     NOTE: Both devices can use the SAME ASN (they are in the same network)
  3. Create VPN Connection 1: CGW-A → VGW/TGW (2 tunnels)
  4. Create VPN Connection 2: CGW-B → VGW/TGW (2 tunnels)
  5. Total: 4 tunnels, 2 physical devices

  FAILOVER WITH BGP:
  ├── Both devices advertise the same on-prem prefixes
  ├── If Device A fails, BGP detects dead peer on both Tunnel 1A and 1B
  ├── Traffic automatically shifts to Device B (Tunnel 2A and 2B)
  ├── No manual intervention required
  └── Recovery: When Device A comes back, BGP re-establishes and
      traffic can be rebalanced (with ECMP on TGW) or stays on
      the preferred device (with AS_PATH/MED on VGW)

  FAILOVER WITH STATIC ROUTING:
  ├── Requires MANUAL route table updates when a device fails
  ├── No automatic detection or rerouting
  └── This is why AWS strongly recommends BGP for redundant CGW setups

  THIS MIRRORS THE DX RESILIENCY PATTERN:
  ════════════════════════════════════════════════════════
  From yesterday's DX doc, the Maximum Resiliency model uses
  4 connections at 2 locations with 2 devices each. The VPN
  equivalent is 2 CGW devices creating 2 VPN connections (4 tunnels).
  The design principle is identical: eliminate every single point
  of failure — the device, the connection, and the AZ.
```

---

## Part 7: ECMP -- Scaling VPN Throughput on Transit Gateway

### The Analogy

A single armored truck carries 1.25 Gbps of cargo. That is the hard limit -- you cannot make the truck bigger. But you can **dispatch multiple trucks simultaneously on different highways**. With ECMP (Equal-Cost Multi-Path), the logistics hub (TGW) distributes shipments across all available trucks based on the shipment's label (5-tuple hash). More trucks = more aggregate cargo capacity.

```
ECMP BANDWIDTH AGGREGATION ON TGW
=============================================================================

  WITHOUT ECMP (VGW or TGW with ECMP disabled):
  ────────────────────────────────────────────────────
  1 VPN connection → 2 tunnels → active/passive → 1.25 Gbps max

  WITH ECMP (TGW, standard 1.25 Gbps tunnels):
  ────────────────────────────────────────────────────
  N VPN connections → 2N tunnels → all active → N x 1.25 Gbps aggregate*

  WITH ECMP (TGW, Large Bandwidth 5 Gbps tunnels):
  ────────────────────────────────────────────────────
  N VPN connections → 2N tunnels → all active → N x 5 Gbps aggregate*

  * Per-flow bandwidth is still limited to one tunnel's capacity.
    ECMP distributes FLOWS, not packets. A single TCP connection
    between two hosts maxes out at 1.25 Gbps (standard) or 5 Gbps
    (Large Bandwidth) regardless of how many tunnels you have.


  SCALING EXAMPLE:
  ════════════════════════════════════════════════════════

  ┌─────────────────────────────┐      ┌──────────────────────┐
  │  On-Premises CGW Device     │      │  Transit Gateway     │
  │                             │      │  (ECMP enabled)      │
  │  VPN Connection 1:          │      │                      │
  │  ├── Tunnel 1A ═══════════════════▶│  VPN Endpoint 1A    │
  │  └── Tunnel 1B ═══════════════════▶│  VPN Endpoint 1B    │
  │                             │      │                      │
  │  VPN Connection 2:          │      │                      │
  │  ├── Tunnel 2A ═══════════════════▶│  VPN Endpoint 2A    │
  │  └── Tunnel 2B ═══════════════════▶│  VPN Endpoint 2B    │
  │                             │      │                      │
  │  VPN Connection 3:          │      │                      │
  │  ├── Tunnel 3A ═══════════════════▶│  VPN Endpoint 3A    │
  │  └── Tunnel 3B ═══════════════════▶│  VPN Endpoint 3B    │
  │                             │      │                      │
  │  VPN Connection 4:          │      │                      │
  │  ├── Tunnel 4A ═══════════════════▶│  VPN Endpoint 4A    │
  │  └── Tunnel 4B ═══════════════════▶│  VPN Endpoint 4B    │
  │                             │      │                      │
  └─────────────────────────────┘      └──────────────────────┘

  Standard tunnels: 8 × 1.25 Gbps = 10 Gbps aggregate
  Large BW tunnels: 8 × 5 Gbps = 40 Gbps aggregate
  (assuming well-distributed flows across the 5-tuple hash)

  ECMP REQUIREMENTS (all must be true):
  ├── TGW created with vpn_ecmp_support = "enable"
  ├── BGP dynamic routing on ALL VPN connections
  ├── All connections advertise the SAME prefix (e.g., 10.0.0.0/8)
  ├── Same AS_PATH length from all connections
  ├── Same MED values (or leave unset for all)
  └── Same origin attribute (IGP, EGP, or incomplete)

  IF ANY ATTRIBUTE DIFFERS:
  ├── TGW treats connections as non-equal-cost
  ├── Traffic goes to the BEST path only (no distribution)
  └── Other connections become standby (backup, not ECMP)

  5-TUPLE HASH:
  ├── src_ip + dst_ip + src_port + dst_port + protocol
  ├── Same 5-tuple = same tunnel (flow pinning)
  ├── Different 5-tuple = potentially different tunnel
  └── Implication: many small flows distribute well;
      one elephant flow is stuck on one tunnel (1.25 Gbps max)

  5 Gbps LARGE BANDWIDTH TUNNELS:
  ════════════════════════════════════════════════════════
  AWS launched 5 Gbps VPN tunnels in November 2025.
  ├── ONLY available with TGW or Cloud WAN (NOT VGW)
  ├── Must be explicitly configured (not the default)
  ├── Higher per-flow throughput for large transfers (4x standard)
  ├── PRICING: $0.60/hr per VPN connection (~$432/month)
  │   vs standard $0.05/hr (~$36/month) — a 12x cost increase
  │   This is a common interview question for cost optimization scenarios
  ├── 4 VPN connections with 5 Gbps tunnels = up to 40 Gbps aggregate
  └── Available in most regions — check AWS VPN pricing page for your region
```

---

## Part 8: Direct Connect vs VPN -- The Complete Decision Framework

Yesterday's [Direct Connect doc](./2026-02-24-aws-direct-connect.md) covered this comparison from the DX perspective. Here it is from the VPN side, synthesized with the full week's networking knowledge.

```
HYBRID CONNECTIVITY DECISION MATRIX
=============================================================================

                       Site-to-Site     Accelerated    Direct        DX + VPN
                       VPN              VPN            Connect       Backup
  ───────────────────────────────────────────────────────────────────────────
  Setup time           Minutes          Minutes        Weeks/months  Weeks + min

  Bandwidth/tunnel     1.25 Gbps        1.25 Gbps      1-400 Gbps   DX + 1.25G
                       (5 Gbps option)  (5 Gbps opt)   (per conn)   VPN fallback

  Max aggregate        ~50 Gbps         ~50 Gbps       400 Gbps+    DX BW +
  (via ECMP on TGW)    (many tunnels)   (many tunnels) (via LAG)    VPN ECMP

  Latency              Variable         More stable    Consistent    DX normally,
                       (internet path)  (AWS backbone  (dedicated    VPN during
                                         after edge)   fiber)       failover

  Encryption           Always           Always         Not by        DX: optional
                       (IPSec)          (IPSec)        default       VPN: always
                                                       (MACsec opt)

  Traverses            Public internet  Edge + AWS     Private       Both
                                        backbone       fiber

  Pricing              $0.05/hr/conn    $0.05/hr/conn  Port-hours   Both costs
                       ($0.60/hr for    + GA premium    + data xfer  combined
                        5 Gbps LBT)     + data xfer     (lower $/GB)

  Routing options      Static or BGP    Static or BGP  BGP only     BGP for both

  Termination          VGW or TGW       TGW only       DXGW → TGW  Both → TGW
                                                       or VGW

  HA model             Built-in dual    Built-in dual  You build    DX HA +
                       tunnels +        tunnels +      (2-4 conns   VPN auto-
                       redundant CGW    redundant CGW  at 2 locs)   failover

  USE CASES:
  ════════════════════════════════════════════════════════
  VPN only:
  ├── Quick connectivity while waiting for DX provisioning
  ├── Branch offices with low bandwidth needs
  ├── Dev/test environments
  └── Backup connectivity for DX

  Accelerated VPN:
  ├── Remote offices far from AWS Region
  ├── Latency-sensitive apps over VPN (VoIP, remote desktop)
  └── Poor internet quality between on-prem and Region

  Direct Connect only:
  ├── High-bandwidth workloads (data replication, ETL)
  ├── Latency-sensitive production (financial, gaming)
  └── High-volume data transfer (lower $/GB than internet)

  DX + VPN backup (the production pattern):
  ├── Enterprise environments requiring maximum availability
  ├── DX as primary (7224:7300 high preference)
  ├── VPN as encrypted backup (7224:7100 low preference)
  ├── Both terminate on the same TGW for unified routing
  └── Automatic failover via BGP when DX goes down

  PRIVATE IP VPN (advanced pattern):
  ════════════════════════════════════════════════════════
  IPSec tunnels running over DX instead of the public internet.

  PACKET PATH:
  On-prem → DX → Transit VIF → DXGW → TGW → IPSec tunnel endpoints
  → TGW VPN attachment → VPC(s)

  WHY Transit VIF? A Private VIF connects to a single VPC's VGW.
  A Transit VIF connects to TGW via DXGW, which is where the Private
  IP VPN tunnel endpoints live. You cannot run Private IP VPN over a
  Private VIF — it MUST be a Transit VIF.

  KEY DETAILS:
  ├── Provides encryption ON TOP of the DX connection
  ├── No public IP addresses needed (uses inner tunnel private IPs)
  ├── Available ONLY with TGW (not VGW)
  ├── Route limits: 5,000 outbound routes (vs 200 for DX alone via
  │   DXGW allowed_prefixes) — a major scaling advantage
  ├── Alternative to MACsec for encrypting DX traffic
  │   MACsec = Layer 2 encryption (faster, less overhead, DX-port-level)
  │     BUT: MACsec only encrypts the physical DX link — traffic on the
  │     AWS backbone between DX endpoint and VPC is unencrypted
  │   Private IP VPN = Layer 3 encryption (more flexible, per-tunnel control)
  │     Provides true end-to-end encryption from on-prem to TGW VPN endpoint
  │
  │   CRITICAL DISTINCTION FOR COMPLIANCE:
  │   ├── "Encrypt the DX link" → MACsec is sufficient (link-level)
  │   ├── "End-to-end encryption on-prem to VPC" → Private IP VPN required
  │   └── Always ask: does the mandate require link or end-to-end encryption?
  └── Use when: compliance requires end-to-end encryption AND you
      already have DX AND you need TGW-scale routing
```

---

## Part 9: VPN Monitoring and Troubleshooting

### CloudWatch Metrics

```
VPN CLOUDWATCH METRICS
=============================================================================

  METRIC              DESCRIPTION                   ALARM ON
  ──────────────────────────────────────────────────────────────────────────
  TunnelState         0 = DOWN, 1 = UP              < 1 for > 1 minute
                      (per tunnel)                   (detect single-tunnel
                                                     failure — you still
                                                     have connectivity but
                                                     lost redundancy)

  TunnelDataIn        Bytes received from on-prem    N/A (informational)
                      (after decryption)

  TunnelDataOut       Bytes sent to on-prem          N/A (informational)
                      (before encryption)

  DIMENSIONS:
  ├── VpnId: The VPN connection ID (vpn-xxxxxxxx)
  └── TunnelIpAddress: The outside IP of the specific tunnel

  CRITICAL ALARM STRATEGY:
  ════════════════════════════════════════════════════════

  ALARM 1: "Single Tunnel Down" (WARNING)
  ├── Metric: TunnelState < 1 for tunnel 1 OR tunnel 2
  ├── Threshold: < 1 for 1 minute (1 datapoint)
  ├── Action: SNS notification to ops team
  ├── WHY: Traffic still flows on the surviving tunnel, but you have
  │   lost redundancy. If the second tunnel fails, you are offline.
  └── Response: Investigate and restore before a second failure

  ALARM 2: "Both Tunnels Down" (CRITICAL)
  ├── Metric: TunnelState < 1 for BOTH tunnels
  ├── Threshold: < 1 for 1 minute
  ├── Action: SNS + PagerDuty + auto-remediation
  └── WHY: Complete VPN connection failure. If no redundant
      CGW/connection exists, on-prem is disconnected.

  ALARM 3: "VPN Throughput Approaching Limit" (WARNING)
  ├── Metric: TunnelDataIn + TunnelDataOut approaching 1.25 Gbps
  ├── Threshold: > 1 Gbps sustained for 5 minutes
  ├── Action: Plan ECMP scaling (add more VPN connections)
  └── WHY: Nearing single-tunnel capacity. Risk of packet drops.
```

### VPN Connection Logs

```
VPN CONNECTION LOGS -- WHAT TO LOOK FOR
=============================================================================

  VPN logs are delivered to CloudWatch Logs and capture tunnel lifecycle
  events. Enable logging when creating the VPN connection:

  LOG EVENTS:
  ────────────────────────────────────────────────────
  ├── IKE Phase 1 negotiation: initiation, success, failure
  │   COMMON FAILURE: Mismatched pre-shared key, mismatched IKE
  │   parameters (encryption algo, DH group), or wrong peer IP
  │
  ├── IKE Phase 2 (IPSec SA) negotiation: success, failure, rekey
  │   COMMON FAILURE: Mismatched Phase 2 parameters, PFS group mismatch
  │
  ├── DPD (Dead Peer Detection) events: timeout, recovery
  │   COMMON ISSUE: CGW behind NAT with UDP 500/4500 blocked
  │
  │   NAT-TRAVERSAL (NAT-T) — IMPORTANT INTERVIEW CONCEPT:
  │   When a CGW device sits behind a NAT (common in smaller offices
  │   or when sharing a single public IP), standard IPSec (ESP, IP
  │   protocol 50) cannot traverse the NAT because NAT modifies headers.
  │   NAT-T solves this by encapsulating ESP packets inside UDP port 4500.
  │   ├── AWS VPN automatically detects and enables NAT-T when needed
  │   ├── Requires UDP 500 (IKE) AND UDP 4500 (NAT-T) open on firewalls
  │   ├── Adds ~20 bytes overhead per packet (UDP encapsulation)
  │   └── If UDP 4500 is blocked, the tunnel will fail to establish
  │
  └── Tunnel state transitions: UP, DOWN, with reason codes
      COMMON CAUSES: Internet path failure, CGW device reboot,
      IKE rekey failure, AWS endpoint maintenance

  TROUBLESHOOTING CHECKLIST:
  ════════════════════════════════════════════════════════

  SYMPTOM: Tunnel stuck DOWN, never comes UP
  ├── Check: Is your CGW initiating IKE? (AWS side waits for initiation
  │   by default — your device must be the initiator)
  ├── Check: UDP ports 500 (IKE) and 4500 (NAT-T) open in firewalls?
  ├── Check: ESP protocol (IP protocol 50) allowed?
  ├── Check: Pre-shared key matches exactly? (case-sensitive)
  ├── Check: IKE version, encryption, integrity, DH group all match?
  └── Check: Correct peer IP configured on your device?

  SYMPTOM: Tunnel comes UP but no traffic flows
  ├── Check: CloudWatch TunnelDataIn — is traffic reaching AWS at all?
  │   If zero, the problem is on-prem routing, not AWS-side.
  ├── Check: On-prem device has route to VPC CIDR via tunnel interface?
  ├── Check: TRAFFIC SELECTORS / PROXY IDs — the most commonly missed cause.
  │   Tunnel can be UP (IKE established) but data DROPPED if the on-prem
  │   device's IPSec encryption domains (source/dest CIDRs) do not match
  │   the actual traffic. The tunnel negotiation succeeds, but packets
  │   outside the agreed traffic selectors are silently discarded.
  ├── Check: VPC route table has route to on-prem CIDR → VGW/TGW?
  │   NOTE: VGW route propagation works with BOTH static and BGP routing.
  │   If using static routing, the VPN's static routes are propagated to
  │   VPC route tables when propagation is enabled — you do NOT need BGP
  │   for route propagation to work.
  ├── Check: Correct route table is associated with the EC2 instance's subnet?
  │   (VPCs can have multiple route tables — verify the right one)
  ├── Check: Security groups allow traffic from on-prem CIDR?
  ├── Check: NACLs allow traffic from on-prem CIDR (BOTH directions — stateless)?
  ├── Check: BGP session established? (if dynamic routing)
  │   Run: aws ec2 describe-vpn-connections — look for BGP status
  ├── Check: Static routes configured on AWS side? (if static routing)
  └── Check: Test BOTH directions — can EC2 reach on-prem? Isolates which
      direction is broken and narrows diagnosis significantly.

  SYMPTOM: Tunnel flaps (UP/DOWN repeatedly)
  ├── Check: DPD timeout values — aggressive timers on a lossy link
  │   cause unnecessary flaps
  ├── Check: MTU — if path MTU is lower than 1500, fragmentation issues
  │   can cause DPD failures (set TCP MSS clamping to 1379)
  ├── Check: IKE SA lifetime mismatch — if Phase 1/2 lifetimes differ,
  │   rekey storms can occur
  └── Check: Overlapping tunnel traffic selectors (proxy IDs) on
      the CGW device
```

---

## Part 10: Putting It All Together -- Enterprise VPN Architecture

This diagram shows how VPN integrates with the Transit Gateway architecture you studied on [Feb 23](./2026-02-23-transit-gateway-deep-dive.md) and the Direct Connect setup from [yesterday](./2026-02-24-aws-direct-connect.md).

```
ENTERPRISE HYBRID ARCHITECTURE -- DX PRIMARY + VPN BACKUP + BRANCH VPN
=============================================================================

  HEADQUARTERS                         AWS (us-east-1)
  ┌──────────────────────────┐         ┌──────────────────────────────────┐
  │                          │         │                                  │
  │  ┌────────────────────┐  │   DX    │  ┌──────────────────────────┐   │
  │  │ Core Router        │══════════▶ │  │ DX Location → DXGW      │   │
  │  │ (BGP: 65000)       │  │(primary)│  │ → Transit VIF           │   │
  │  │                    │  │         │  │ → TGW DX attachment      │   │
  │  │  ┌──────────────┐  │  │         │  └──────────┬───────────────┘   │
  │  │  │ Firewall A   │  │  │   VPN   │             │                   │
  │  │  │ CGW: .10     │──────────────▶│  ┌──────────┴───────────────┐   │
  │  │  └──────────────┘  │  │(backup) │  │                          │   │
  │  │  ┌──────────────┐  │  │   VPN   │  │   TRANSIT GATEWAY        │   │
  │  │  │ Firewall B   │──────────────▶│  │   (vpn_ecmp = enabled)   │   │
  │  │  │ CGW: .20     │  │  │(backup) │  │                          │   │
  │  │  └──────────────┘  │  │         │  │  ┌─────────────────────┐ │   │
  │  └────────────────────┘  │         │  │  │ Shared RT           │ │   │
  └──────────────────────────┘         │  │  │ DX att → assoc      │ │   │
                                       │  │  │ VPN att → assoc     │ │   │
  BRANCH OFFICE A                      │  │  │ VPC routes → prop   │ │   │
  ┌──────────────────────────┐         │  │  └─────────────────────┘ │   │
  │  ┌──────────────┐       │  VPN    │  │                          │   │
  │  │ Branch FW    │───────────────▶ │  │  ┌─────────────────────┐ │   │
  │  │ CGW: .30     │       │         │  │  │ Prod RT             │ │   │
  │  │ ASN: 65001   │       │         │  │  │ DX/VPN prop → here  │ │   │
  │  └──────────────┘       │         │  │  │ 0.0.0.0/0 → egress │ │   │
  └──────────────────────────┘         │  │  └─────────────────────┘ │   │
                                       │  │                          │   │
  BRANCH OFFICE B                      │  │  ┌─────────────────────┐ │   │
  ┌──────────────────────────┐         │  │  │ Dev RT              │ │   │
  │  ┌──────────────┐       │  VPN    │  │  │ shared-svc → prop   │ │   │
  │  │ Branch FW    │───────────────▶ │  │  │ 0.0.0.0/0 → egress │ │   │
  │  │ CGW: .40     │       │         │  │  └─────────────────────┘ │   │
  │  │ ASN: 65002   │       │         │  │                          │   │
  │  └──────────────┘       │         │  └──────────┬───────────────┘   │
  └──────────────────────────┘         │             │                   │
                                       │    ┌────────┼────────┐          │
                                       │    ▼        ▼        ▼          │
                                       │  Prod     Shared    Dev         │
                                       │  VPCs     Svc VPC   VPCs        │
                                       └──────────────────────────────────┘

  ROUTING DESIGN:
  ════════════════════════════════════════════════════════

  HQ → AWS (traffic TO AWS from headquarters):
  ├── DX primary: BGP community 7224:7300 → high LOCAL_PREF in AWS
  ├── VPN backup: BGP community 7224:7100 → low LOCAL_PREF in AWS
  └── Result: AWS always prefers DX path; VPN is standby

  AWS → HQ (traffic FROM AWS to headquarters):
  ├── DX route propagates to TGW route table (172.16.0.0/12)
  ├── VPN route also propagates (same prefix, 172.16.0.0/12)
  ├── TGW route priority: DX propagated > VPN propagated
  └── If DX attachment disappears: VPN propagated route takes over

  Branch → AWS:
  ├── Each branch VPN propagates its CIDR to the shared RT
  ├── Shared RT propagates back to prod/dev RTs as needed
  └── Branch-to-branch: both routes in shared RT, TGW forwards

  Branch → HQ (through AWS):
  ├── Branch sends 172.16.0.0/12 traffic to TGW
  ├── TGW shared RT has 172.16.0.0/12 → DX attachment (propagated)
  ├── Traffic goes HQ via DX (preferred) or VPN (failover)
  └── This is similar to CloudHub but with TGW-scale routing control
```

---

## Key Takeaways

- **Site-to-Site VPN is the encrypted, quick-deploy complement to Direct Connect.** VPN deploys in minutes over the public internet with always-on IPSec encryption. DX takes weeks but provides dedicated bandwidth and consistent latency. Most production architectures use both.

- **Always configure BOTH tunnels on your CGW device.** Each VPN connection provides two tunnels in different AZs. Configuring only one tunnel is the most common mistake -- you lose all redundancy and are one maintenance event away from an outage.

- **Use BGP dynamic routing for every production VPN.** Static routing disables automatic failover, blocks ECMP, and prevents VPN CloudHub. The only reason to use static is when your on-prem device physically cannot speak BGP.

- **VGW is for simple single-VPC connectivity; TGW is for everything else.** If you already have a Transit Gateway (and you should for any multi-VPC environment), terminate VPN on TGW to get ECMP, Accelerated VPN, multi-VPC reach, and unified routing with DX.

- **ECMP on TGW scales VPN throughput far beyond 1.25 Gbps.** Multiple VPN connections with BGP advertising identical prefixes and attributes allow TGW to distribute flows across all tunnels. But remember: a single flow is always pinned to one tunnel.

- **Redundant CGW devices eliminate the on-premises single point of failure.** Two physical devices creating two VPN connections (4 tunnels total) mirrors the DX Maximum Resiliency model. With BGP, failover is automatic.

- **Accelerated VPN is worth the premium when geography or internet quality matters.** It uses Global Accelerator to minimize time on the public internet, reducing jitter and packet loss. Only works with TGW. The cost adder is real -- evaluate based on your latency sensitivity.

- **CloudWatch TunnelState is your most important VPN alarm.** A single-tunnel-down state means you still have connectivity but zero redundancy. Alarm on this before it becomes a dual-tunnel failure.

- **The DX + VPN backup pattern is the standard enterprise design.** DX as primary (BGP community 7224:7300), VPN as backup (7224:7100), both on TGW. Automatic failover via BGP when DX goes down. This synthesizes everything from this week's networking block.

- **VPN CloudHub is the budget multi-site solution.** For fewer than 5 branch offices with low bandwidth needs, CloudHub on a VGW avoids TGW costs. For anything larger, TGW provides better scalability, ECMP, and routing control.

---

## Further Reading

- [AWS Site-to-Site VPN User Guide](https://docs.aws.amazon.com/vpn/latest/s2svpn/how_it_works.html) -- The definitive reference for all VPN features, tunnel options, and configuration
- [Building Scalable Multi-VPC Network Infrastructure -- VPN Section](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/vpn.html) -- Architecture decision guide for VPN in multi-VPC environments
- [Scaling VPN Throughput Using Transit Gateway (AWS Blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/scaling-vpn-throughput-using-aws-transit-gateway/) -- ECMP deep-dive with step-by-step architecture
- [Transit Gateway Deep Dive (Feb 23)](./2026-02-23-transit-gateway-deep-dive.md) -- The TGW architecture that VPN plugs into as an attachment type
- [AWS Direct Connect Deep Dive (Feb 24)](./2026-02-24-aws-direct-connect.md) -- The DX architecture that VPN complements as an encrypted backup
- [VPC Advanced: CIDR Planning (Feb 19)](./2026-02-19-vpc-advanced-cidr-subnet-strategy.md) -- Why hierarchical CIDR allocation matters for route summarization in hybrid connectivity
- [AWS Site-to-Site VPN Quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html) -- Route limits and connection limits per VGW/TGW (frequently tested in interviews)
