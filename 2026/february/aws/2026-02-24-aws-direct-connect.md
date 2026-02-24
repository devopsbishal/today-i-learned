# AWS Direct Connect -- The Private Railway Analogy

> Understanding AWS Direct Connect through the analogy of building a private railway from your corporate campus to a major logistics hub -- where dedicated tracks replace shared highways, virtual platforms let you reach different destinations from one station, a central dispatcher routes trains across the national rail network, and redundant lines at separate junctions keep freight moving even when a station goes dark.

---

## TL;DR

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Direct Connect | Private railway from your campus to an AWS hub | A dedicated physical link that bypasses the public internet, providing consistent latency, higher bandwidth, and lower data transfer costs |
| DX Location (colocation facility) | The junction station where your railway meets the AWS rail network | A physical facility (Equinix, CoreSite, etc.) where your fiber cross-connects to AWS infrastructure |
| Dedicated Connection (1/10/100/400 Gbps) | Owning your own track at the junction station | A physical port reserved exclusively for you on AWS equipment; you control the full bandwidth |
| Hosted Connection (50 Mbps-10 Gbps) | Leasing space on a shared track from a rail operator | An AWS Partner provisions a connection on their existing infrastructure and sub-allocates bandwidth to you |
| Virtual Interface (VIF) | A platform at the junction station | Each platform serves a different destination type; a Dedicated connection supports up to 51 VIFs (50 private/public + 4 transit, combined max 51) |
| Private VIF | Platform for freight trains to a single warehouse district | Connects to one VPC (via VGW) or multiple VPCs (via DXGW); carries private traffic to your AWS resources |
| Public VIF | Platform for trains to the public marketplace | Reaches all AWS public service endpoints (S3, DynamoDB, STS) using public IP addresses -- destinations are public, but your trains arrive on private tracks instead of the public highway |
| Transit VIF | Platform for express trains to the central interchange | Connects through DXGW to Transit Gateway, reaching thousands of VPCs across regions through one BGP session |
| Direct Connect Gateway (DXGW) | A national rail switchboard | A globally available routing junction that permanently connects your local tracks to destination tracks in any city -- but only handles traffic entering and leaving the network (north-south), not transfers between destinations (east-west) |
| Link Aggregation Group (LAG) | Bundling multiple parallel tracks into one high-capacity corridor | Combines 2-4 DX connections at the same location into a single logical link for increased bandwidth and port-level redundancy |
| BGP (Border Gateway Protocol) | The signal system that tells trains which routes exist | The routing protocol that advertises reachable destinations between your router and AWS; every DX connection requires BGP |
| BGP Communities | Priority and jurisdiction labels on the shipping manifest | Tags stamped on route advertisements that tell every switching station whether this cargo gets express handling (local preference) and how far across the network it should be advertised (local, continental, global) |
| MACsec Encryption | An armored railcar that encrypts cargo | Layer 2 encryption on 10/100/400 Gbps Dedicated connections, protecting data in transit between your router and the AWS device |
| SiteLink | Direct track between two junction stations | Lets traffic flow between DX locations over the AWS backbone without routing through a region, enabling site-to-site connectivity |
| Resiliency Model | How many independent railways you build | From one track at one station (dev/test) to separate tracks at separate stations with separate junction points (maximum resiliency, 99.99% SLA) |

---

## The Big Picture

In the [Transit Gateway Deep Dive](./2026-02-23-transit-gateway-deep-dive.md), you saw how TGW acts as the central interchange for VPC-to-VPC traffic within AWS. One of its five attachment types was the **Direct Connect Gateway attachment** -- a brief mention that a DX connection could plug into TGW via a Transit VIF and DXGW. Today you learn everything about what happens on the **other side of that attachment**: the physical fiber, the colocation facility, the virtual interfaces, the BGP sessions, the failover mechanics, and the resiliency architecture.

Think of it this way. All your VPCs are warehouses connected by the Transit Gateway interchange. The interchange handles traffic beautifully inside the AWS rail network. But your corporate headquarters is in a different city entirely -- your on-premises data center. To connect your headquarters to the AWS rail network, you have two options:

1. **Ship freight on public highways (the internet via VPN)** -- Cheaper upfront, works immediately, but speed varies with traffic, lanes are shared with everyone, and capacity is 1.25 Gbps per tunnel (scalable to ~50 Gbps aggregate via ECMP on TGW).
2. **Build a private railway to a junction station (Direct Connect)** -- Significant upfront investment, takes weeks to provision, but you get **dedicated bandwidth** (up to 400 Gbps), **consistent latency** (no contention), **lower data transfer costs** (cheaper than internet egress), and a direct connection to the AWS rail network.

Direct Connect is option 2. You are building private rail infrastructure.

```
THE HYBRID CONNECTIVITY LANDSCAPE -- WHERE DIRECT CONNECT FITS
=============================================================================

  YOUR DATA CENTER                  AWS
  ┌──────────────┐                  ┌──────────────────────────────────────┐
  │              │                  │                                      │
  │  On-Prem     │   OPTION 1:     │  ┌──────────┐                       │
  │  Servers     │═══VPN (IPsec)══▶│  │ VGW/TGW  │  (encrypted, internet│
  │              │   over internet  │  └──────────┘   path, 1.25 Gbps/    │
  │              │                  │                  tunnel, variable    │
  │              │                  │                  latency)            │
  │              │                  │                                      │
  │              │   OPTION 2:      │  ┌──────────┐                       │
  │  Your        │═══Direct═══════▶│  │ DX       │  (dedicated fiber,    │
  │  Router      │   Connect       │  │ Location │   up to 400 Gbps,     │
  │  (BGP)       │   (private      │  └─────┬────┘   consistent latency, │
  │              │    fiber)        │        │        lower $/GB)          │
  └──────────────┘                  │        ▼                            │
                                    │  ┌──────────┐                       │
                                    │  │ VIFs     │→ Private VIF → VGW/DXGW │
                                    │  │          │→ Transit VIF → DXGW → TGW│
                                    │  │          │→ Public VIF → AWS public │
                                    │  └──────────┘                       │
                                    └──────────────────────────────────────┘
```

---

## Part 1: The Physical Connection -- Building the Railway

### Connection Types: Dedicated vs Hosted

The first decision is whether you own your own track or lease space on someone else's.

**Dedicated Connection -- Owning your own track at the junction station.** You request a port directly from AWS (via the console or API). AWS assigns you a physical port on their router at a DX location. You (or your colocation provider) run a fiber cross-connect from your router cage to the AWS cage in the same facility. You own the full bandwidth of that port.

**Hosted Connection -- Leasing capacity from a rail operator.** An AWS Partner Network (APN) partner already has Dedicated connections to AWS. They carve out a sub-allocation of bandwidth and provision a Hosted connection on your behalf. You never touch the physical equipment -- the partner manages the cross-connect.

```
DEDICATED vs HOSTED CONNECTIONS
=============================================================================

  DEDICATED CONNECTION
  ────────────────────────────────────────────────────
  Speeds:        1 Gbps, 10 Gbps, 100 Gbps, 400 Gbps
  Port fee:      $0.30/hr (1G), $2.25/hr (10G), $22.50/hr (100G)
  Provisioning:  You order the port, receive an LOA-CFA (Letter of
                 Authorization / Connecting Facility Assignment), then
                 hand it to your colo provider to complete the cross-connect
  Lead time:     Days to weeks (depends on colo provider)
  VIFs:          Up to 50 private/public VIFs (shared pool) + up to 4 transit
                 VIFs per connection (combined max 51)
  MACsec:        Supported on 10 Gbps, 100 Gbps, and 400 Gbps
  LAG:           Can be added to a Link Aggregation Group
  Best for:      High-bandwidth workloads, enterprises with colo presence

  HOSTED CONNECTION
  ────────────────────────────────────────────────────
  Speeds:        50 Mbps, 100 Mbps, 200 Mbps, 300 Mbps, 400 Mbps, 500 Mbps,
                 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps
  Port fee:      Varies by partner
  Provisioning:  Partner provisions it; you accept in the AWS console
  Lead time:     Hours to days (partner-dependent)
  VIFs:          1 VIF per Hosted connection (private, public, OR transit)
  MACsec:        Not supported
  LAG:           Cannot be added to LAG
  Best for:      First-time DX users, lower bandwidth needs, no colo presence

  CRITICAL EXAM DISTINCTION:
  ├── Dedicated: multiple VIFs on one connection (multiplexed via 802.1Q VLANs)
  ├── Hosted: exactly ONE VIF per connection
  ├── Hosted > 1 Gbps supports jumbo frames; Hosted < 1 Gbps does not
  └── Jumbo frames require end-to-end MTU compatibility (your router,
      DX connection, VGW/TGW, EC2 instance) — any hop below 9001/8500
      MTU causes fragmentation or drops
```

### Physical Requirements

Every Direct Connect connection, whether Dedicated or Hosted, terminates at a **DX location** -- a colocation facility where AWS has equipment racks. Your router must be at the same facility (or connected to it via a partner's network). The physical specifications:

- **Single-mode fiber** (1000BASE-LX for 1G, 10GBASE-LR for 10G, 100GBASE-LR4 for 100G)
- **802.1Q VLAN encapsulation** for multiplexing virtual interfaces on a single port
- **BGP** (IPv4 and/or IPv6) with MD5 authentication on every VIF
- **BFD strongly recommended** (Bidirectional Forwarding Detection) -- provides sub-second failure detection (~300ms typical) compared to BGP hold-timer default of 90 seconds. Without BFD, a DX failure could take 90 seconds to detect. Enable on all production connections.

### LOA-CFA Process

When you order a Dedicated connection, AWS provisions a port and generates a **Letter of Authorization and Connecting Facility Assignment (LOA-CFA)**. This document specifies the AWS rack, port, and VLAN assignment at the DX location. You download it from the AWS console and hand it to your colocation provider, who completes the physical cross-connect (fiber patch) from your router cage to the AWS cage. This is a common interview question about the DX provisioning workflow.

If you are not physically present at a DX location, you have three options: use a Hosted connection through a partner, lease dark fiber from the DX location to your data center, or use a partner who extends the connection to your premises.

---

## Part 2: Virtual Interfaces -- The Three Platforms

A physical DX connection is just raw fiber. You need **Virtual Interfaces (VIFs)** to actually reach AWS resources. Each VIF is a tagged 802.1Q VLAN on the physical connection with its own BGP peering session.

Think of the DX connection as a junction station, and each VIF as a **platform** at that station. Different platforms serve different train services to different destinations.

```
THREE VIF TYPES -- THE THREE PLATFORMS
=============================================================================

  PHYSICAL DX CONNECTION (the station)
  ┌────────────────────────────────────────────────────────────────────────┐
  │                                                                        │
  │  PLATFORM 1: PRIVATE VIF (VLAN 100)                                   │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  Destination: VPC private resources                              │  │
  │  │  Connects to: Virtual Private Gateway (VGW) or DX Gateway        │  │
  │  │  BGP: Your ASN (e.g. 65000) ↔ Amazon-side ASN (default 64512)   │  │
  │  │  Addresses: Private IPs                                          │  │
  │  │  Jumbo frames: Yes (9001 MTU)                                    │  │
  │  │  Prefixes from AWS: VPC CIDR(s)                                  │  │
  │  │  Prefixes from you: On-prem CIDRs                                │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  │                                                                        │
  │  PLATFORM 2: PUBLIC VIF (VLAN 200)                                    │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  Destination: AWS public service endpoints (S3, DynamoDB, etc.)  │  │
  │  │  Connects to: AWS public routing infrastructure directly          │  │
  │  │  BGP: Your ASN ↔ AWS ASN (7224)                                  │  │
  │  │  Addresses: Public IPs (you must own and advertise them)          │  │
  │  │  Jumbo frames: No (1500 MTU)                                     │  │
  │  │  Prefixes from AWS: All AWS public IP ranges (or scoped by       │  │
  │  │                     BGP communities to region/continent)          │  │
  │  │  Prefixes from you: Your public IP prefixes                       │  │
  │  │                                                                    │  │
  │  │  USE CASE: Access S3, DynamoDB, STS, CloudFormation endpoints     │  │
  │  │  over DX instead of the internet. Also required for certain       │  │
  │  │  patterns like VPN-over-DX (VPN endpoints are public IPs).        │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  │                                                                        │
  │  PLATFORM 3: TRANSIT VIF (VLAN 300)                                   │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  Destination: VPCs via Transit Gateway (thousands of VPCs)       │  │
  │  │  Connects to: Direct Connect Gateway → Transit Gateway            │  │
  │  │  BGP: Your ASN ↔ AWS ASN (64512 default or custom)               │  │
  │  │  Addresses: Private IPs                                           │  │
  │  │  Jumbo frames: Yes (8500 MTU -- note: NOT 9001 for transit VIF)  │  │
  │  │  Prefixes from AWS: Up to 200 from TGW (route summarization      │  │
  │  │                     critical!)                                     │  │
  │  │  Prefixes from you: On-prem CIDRs (up to 100 by default)         │  │
  │  │                                                                    │  │
  │  │  THIS IS THE SCALABLE OPTION. One Transit VIF + DXGW + TGW       │  │
  │  │  replaces dozens of Private VIFs.                                  │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────────────────┘

  VIF SELECTION DECISION:
  ├── Need to reach 1-3 VPCs with high bandwidth?      → Private VIF to VGW
  ├── Need to reach many VPCs across regions?           → Transit VIF to DXGW → TGW
  ├── Need to reach AWS public services (S3, etc.)?     → Public VIF
  └── Need all of the above?                            → All three VIF types
      (A Dedicated connection supports all simultaneously)
```

### Private VIF vs Transit VIF -- When to Use Each

This is one of the most important architectural decisions and a common exam topic. You covered Transit Gateway's five attachment types yesterday -- here is the DX-side perspective on why you might still use Private VIF even when TGW exists.

```
PRIVATE VIF vs TRANSIT VIF -- COST AND ARCHITECTURE TRADE-OFFS
=============================================================================

                          Private VIF               Transit VIF
  ──────────────────────────────────────────────────────────────────────
  Connects to             VGW (1 VPC) or            DXGW → TGW
                          DXGW (up to 20 VPCs)      (thousands of VPCs)

  BGP sessions needed     1 per VIF                 1 per VIF
  for N VPCs              (but need N VIFs           (1 VIF serves all)
                          if not using DXGW)

  Max MTU                 9001 bytes                8500 bytes

  TGW data processing     None ($0)                 $0.02/GB
  charge

  Prefix limit from AWS   100 per BGP session       200 from TGW per DXGW

  VPC-to-VPC transit      No (DXGW does NOT         Yes (TGW handles
  via same path           support east-west)         east-west)

  Route summarization     Less critical              Critical (200 prefix
                                                     limit from TGW)

  COST EXAMPLE — 10 TB/month OUT from AWS to on-prem (one VPC):
  (Note: data transfer IN to AWS over DX is free)
  ├── Private VIF: DX data transfer out only = ~$200/mo
  ├── Transit VIF: DX data transfer out + TGW processing = ~$200 + $200 = ~$400/mo
  └── Delta: $200/mo extra for the TGW hop

  SCALING CEILING (the fundamental scalability difference beyond cost):
  ├── Private VIF + DXGW: Hard cap of 20 VPCs (max 20 VGW associations per DXGW)
  └── Transit VIF + DXGW + TGW: Scales to thousands of VPCs (TGW supports
      5,000 attachments; multiple TGWs can share the same DXGW)

  RECOMMENDATION (from the AWS whitepaper):
  ├── DEFAULT: Transit VIF for most VPCs (operational simplicity, one BGP session)
  ├── EXCEPTION: Private VIF for high-bandwidth VPCs where TGW processing
  │   costs are prohibitive (e.g., 50 TB/month data lake = $1,000/mo in TGW fees)
  └── BOTH: A Dedicated connection can have Private VIFs AND Transit VIFs
      simultaneously — use Transit VIF for most traffic, Private VIF for
      specific high-volume VPCs
```

---

## Part 3: Direct Connect Gateway -- The National Rail Switchboard

### The Analogy

Without a central switchboard, each platform at your junction station can only reach one warehouse district in one city. If you have VPCs in us-east-1, eu-west-1, and ap-southeast-1, you would need separate platforms (VIFs) for each. The **Direct Connect Gateway (DXGW)** is a **national rail switchboard** -- a routing junction that permanently connects your local tracks to destination tracks in any city. It handles traffic entering and leaving the network, but does not make dynamic decisions or route transfers between destinations.

### The Technical Reality

```
DIRECT CONNECT GATEWAY -- GLOBAL ROUTING LAYER
=============================================================================

  WITHOUT DXGW (the old way):
  ────────────────────────────────────────────────────

  On-prem → Private VIF 1 → VGW in us-east-1 VPC-A
  On-prem → Private VIF 2 → VGW in us-east-1 VPC-B
  On-prem → Private VIF 3 → VGW in eu-west-1 VPC-C
  On-prem → Private VIF 4 → VGW in ap-southeast-1 VPC-D

  4 VPCs = 4 VIFs = 4 BGP sessions = operational nightmare at scale
  (and a Hosted connection only supports 1 VIF!)


  WITH DXGW + PRIVATE VIFs (multi-VPC, multi-region):
  ────────────────────────────────────────────────────

  On-prem → 1 Private VIF → DXGW → VGW in VPC-A (us-east-1)
                                  → VGW in VPC-B (us-east-1)
                                  → VGW in VPC-C (eu-west-1)
                                  → VGW in VPC-D (ap-southeast-1)

  1 VIF, 1 BGP session, up to 20 VGWs across any regions
  LIMIT: 20 VGW associations per DXGW


  WITH DXGW + TRANSIT VIF (the scalable architecture):
  ────────────────────────────────────────────────────

  On-prem → 1 Transit VIF → DXGW → TGW us-east-1 → 100s of VPCs
                                  → TGW eu-west-1 → 100s of VPCs
                                  → TGW ap-southeast-1 → 100s of VPCs

  1 VIF, 1 BGP session, up to 6 TGW associations across any regions
  Each TGW connects to thousands of VPCs
  LIMIT: 6 TGW associations per DXGW, 200 prefixes from TGW to on-prem
```

### DXGW Critical Limitation: North-South Only

The DXGW is a dispatcher for trains going **between your campus and AWS warehouses** (north-south traffic). It does NOT dispatch trains between warehouses (east-west, VPC-to-VPC). If VPC-A and VPC-B are both associated with the same DXGW, they **cannot communicate through it**.

```
DXGW TRAFFIC FLOW RESTRICTION
=============================================================================

  ✅ WORKS:     On-prem → DXGW → VPC-A     (north-south)
  ✅ WORKS:     On-prem → DXGW → VPC-B     (north-south)
  ❌ BLOCKED:   VPC-A   → DXGW → VPC-B     (east-west — NOT SUPPORTED)

  For VPC-to-VPC traffic, you still need:
  ├── Transit Gateway (hub-and-spoke within a region)
  ├── VPC Peering (point-to-point)
  └── TGW Peering (cross-region via the TGW path you studied yesterday)

  This is why Transit VIF + DXGW + TGW is the recommended architecture:
  DXGW handles on-prem ↔ AWS traffic, TGW handles VPC ↔ VPC traffic.
```

### Cross-Account DXGW Associations

The DXGW owner (typically the network services account in a landing zone) can accept **association proposals** from other AWS accounts. This enables a shared DX infrastructure model:

- Network account owns the DX connection, DXGW, and Transit VIF
- Spoke accounts own their VGWs or TGWs
- Spoke accounts send an association proposal to the DXGW
- Network account accepts or rejects the proposal
- Traffic flows without the spoke account needing its own DX connection

This maps directly to the RAM sharing pattern from yesterday's TGW doc -- centralized network infrastructure shared across accounts.

### Allowed Prefixes -- The Route Filter You Must Understand

When you associate a DXGW with a TGW or VGW, you configure **allowed prefixes**. This is a common exam trap: allowed prefixes are **not** pass-through filters that selectively forward individual VPC routes -- they are **replacements**. The DXGW never sends the individual VPC CIDRs (e.g., 10.0.0.0/16, 10.1.0.0/16) to your on-premises router. Instead, it advertises only the supernet(s) you specify in the allowed prefixes list (e.g., 10.0.0.0/8). If a VPC CIDR does not fall within any allowed prefix, that VPC is simply unreachable from on-prem -- the route is dropped, not filtered through.

```
ALLOWED PREFIXES -- HOW THEY FILTER ROUTE ADVERTISEMENTS
=============================================================================

  TGW has routes for:         Allowed prefixes on        What on-prem
  (from VPC attachments)      DXGW-TGW association:      actually sees:
  ────────────────────────    ──────────────────────     ──────────────
  10.0.0.0/16 (VPC-A)        10.0.0.0/8                 10.0.0.0/8
  10.1.0.0/16 (VPC-B)                                   (single supernet)
  10.2.0.0/16 (VPC-C)
  172.16.0.0/16 (VPC-D)      (not listed)               NOT ADVERTISED!

  KEY POINTS:
  ├── Allowed prefixes are NOT pass-through filters -- they REPLACE
  │   the individual routes with the supernet you specify
  ├── Maximum 20 allowed prefixes per DXGW-to-TGW/VGW association
  │   (separate from the 200-prefix limit from TGW to on-prem)
  ├── Use broad supernets (e.g. 10.0.0.0/8) to cover all VPC CIDRs
  └── If you forget to include a CIDR range, that traffic will not
      be routable from on-prem
```

### DXGW Migration Constraint

If a DXGW was previously associated with a VGW via a **Private VIF**, you cannot simultaneously associate that same DXGW with a **TGW** via a Transit VIF. This is a subtle but important constraint when migrating from a Private VIF + VGW architecture to Transit VIF + TGW. The workaround is to create a new DXGW for the TGW association, then migrate VIFs over.

---

## Part 4: The Full Hybrid Architecture -- DX + DXGW + TGW

This is the reference architecture that ties together everything from the past week. It is the most common enterprise pattern and the one you will see on the exam.

```
PRODUCTION HYBRID ARCHITECTURE -- THE COMPLETE PICTURE
=============================================================================

  ON-PREMISES DATA CENTER
  ┌──────────────────────────────────────────────────────────────────────┐
  │                                                                      │
  │  Customer Router (BGP-capable, your ASN e.g. 65000)                 │
  │  ├── Interface 1 → Fiber to DX Location A                           │
  │  └── Interface 2 → Fiber to DX Location B                           │
  │                                                                      │
  └──────────────┬──────────────────────────────────────┬────────────────┘
                 │ Dedicated fiber                       │ Dedicated fiber
                 ▼                                       ▼
  ┌──────────────────────┐              ┌──────────────────────┐
  │  DX LOCATION A       │              │  DX LOCATION B       │
  │  (e.g., Equinix DC1) │              │  (e.g., CoreSite NY1)│
  │                      │              │                      │
  │  DX Connection 1     │              │  DX Connection 2     │
  │  (10 Gbps Dedicated) │              │  (10 Gbps Dedicated) │
  │                      │              │                      │
  │  ┌── Transit VIF 1   │              │  ┌── Transit VIF 2   │
  │  │   VLAN 300        │              │  │   VLAN 300        │
  │  │   BGP: 65000↔64512│              │  │   BGP: 65000↔64512│
  │  └──────┬────────────│              │  └──────┬────────────│
  │         │            │              │         │            │
  │  ┌── Public VIF 1    │              │  ┌── Public VIF 2    │
  │  │   VLAN 200        │              │  │   VLAN 200        │
  │  │   BGP: 65000↔7224 │              │  │   BGP: 65000↔7224 │
  │  └──────┬────────────│              │  └──────┬────────────│
  └─────────┼────────────┘              └─────────┼────────────┘
            │                                     │
            ▼                                     ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │                   DIRECT CONNECT GATEWAY                           │
  │                   (globally available)                              │
  │                                                                    │
  │  Associations:                                                     │
  │  ├── TGW us-east-1  (allowed prefixes: 10.0.0.0/8)               │
  │  ├── TGW eu-west-1  (allowed prefixes: 10.0.0.0/8)               │
  │  └── TGW ap-southeast-1 (allowed prefixes: 10.0.0.0/8)           │
  └────────┬─────────────────────────┬──────────────────────┬──────────┘
           │                         │                      │
           ▼                         ▼                      ▼
  ┌──────────────────┐    ┌──────────────────┐   ┌──────────────────┐
  │ TGW us-east-1    │    │ TGW eu-west-1    │   │ TGW ap-south-1   │
  │ ┌──────────────┐ │    │ ┌──────────────┐ │   │ ┌──────────────┐ │
  │ │ Prod VPCs    │ │    │ │ Prod VPCs    │ │   │ │ Prod VPCs    │ │
  │ │ Dev VPCs     │ │    │ │ Dev VPCs     │ │   │ │ Dev VPCs     │ │
  │ │ Shared Svcs  │ │    │ │ Shared Svcs  │ │   │ │ Shared Svcs  │ │
  │ │ Egress VPC   │ │    │ │ Egress VPC   │ │   │ │ Egress VPC   │ │
  │ └──────────────┘ │    │ └──────────────┘ │   │ └──────────────┘ │
  └──────────────────┘    └──────────────────┘   └──────────────────┘

  TRAFFIC FLOW: On-prem server (172.16.5.10) → Prod VPC (10.0.5.20) in us-east-1
  ═══════════════════════════════════════════════════════════════════════
  1. On-prem router has BGP-learned route: 10.0.0.0/8 via Transit VIF 1
  2. Packet → DX Location A → Transit VIF 1 → DXGW
  3. DXGW evaluates: dest 10.0.5.20 matches TGW us-east-1 association
  4. DXGW forwards → TGW us-east-1
  5. TGW route table lookup: 10.0.0.0/16 → prod-vpc-attachment
  6. TGW forwards → Prod VPC ENI → destination instance

  RETURN PATH:
  1. Instance 10.0.5.20 → VPC route table: 172.16.0.0/12 → tgw
  2. TGW route table: 172.16.0.0/12 → dxgw-attachment (propagated from BGP)
  3. TGW → DXGW → Transit VIF → DX Location → on-prem router → 172.16.5.10
```

### The 200-Prefix Limit and Route Summarization

When TGW advertises routes back to your on-premises network through DXGW, you are limited to **200 prefixes**. If you have 300 VPCs each with a /16, you cannot advertise 300 individual routes. You must summarize.

```
ROUTE SUMMARIZATION STRATEGY
=============================================================================

  BAD (exceeds 200-prefix limit):
  ├── 10.0.0.0/16  → VPC-1
  ├── 10.1.0.0/16  → VPC-2
  ├── 10.2.0.0/16  → VPC-3
  ├── ... (300 more entries)
  └── RESULT: BGP session drops when limit exceeded

  GOOD (summarized):
  ├── 10.0.0.0/8   → all VPCs (single supernet covers everything)
  └── RESULT: 1 prefix, on-prem knows to send all 10.x traffic over DX

  BETTER (segmented summaries for policy routing):
  ├── 10.0.0.0/14  → Production VPCs (10.0-10.3.x)
  ├── 10.4.0.0/14  → Staging VPCs (10.4-10.7.x)
  ├── 10.8.0.0/14  → Dev VPCs (10.8-10.11.x)
  └── RESULT: 3 prefixes, on-prem can apply different policies per environment

  THIS IS WHY CIDR PLANNING MATTERS (from the Feb 19 VPC doc):
  If you assigned VPC CIDRs randomly, you cannot summarize them cleanly.
  Contiguous, hierarchical CIDR allocation enables route summarization.
```

---

## Part 5: BGP Fundamentals for Direct Connect

### The Signal System

BGP is the signal system that controls which routes your trains follow. Every VIF requires a BGP peering session between your router (your ASN, e.g. 65000) and AWS (ASN 7224 fixed for Public VIF, or your chosen Amazon-side ASN -- default 64512, set on the DXGW or VGW -- for Private/Transit VIF).

### Route Evaluation Order

When multiple paths exist to the same destination (e.g., two DX connections for redundancy), BGP determines which path wins:

```
BGP ROUTE EVALUATION ORDER FOR DIRECT CONNECT
=============================================================================

  PRIORITY  ATTRIBUTE                    HOW TO CONTROL IT
  ────────────────────────────────────────────────────────────────────
  1 (HIGH)  LOCAL PREFERENCE              BGP community tags:
            (set by AWS based on          7224:7100 = Low preference
             community tags you send)     7224:7200 = Medium (default same-region)
                                          7224:7300 = High preference

  2         AS_PATH LENGTH                Prepend your ASN to make a path
            (shorter = preferred)         less preferred (longer path = backup)

  3 (LOW)   MED (Multi-Exit              Lowest MED wins; less commonly used
            Discriminator)                with DX

  FAILOVER EXAMPLE -- ACTIVE/PASSIVE WITH BGP COMMUNITIES:
  ════════════════════════════════════════════════════════

  DX Connection 1 (primary, DX Location A):
  ├── Advertise on-prem prefixes with community 7224:7300 (HIGH preference)
  └── AWS will prefer this path for traffic TO your data center

  DX Connection 2 (backup, DX Location B):
  ├── Advertise on-prem prefixes with community 7224:7100 (LOW preference)
  └── AWS will only use this path if Connection 1 goes down

  RESULT: Deterministic active/passive failover
  ├── Normal: 100% traffic on Connection 1
  ├── Failover: 100% traffic shifts to Connection 2
  └── Recovery: Traffic returns to Connection 1 when BGP re-establishes

  ACTIVE/ACTIVE WITH ECMP:
  ════════════════════════════════════════════════════════
  ├── Both connections advertise the SAME prefixes
  ├── Both use the SAME community tag (7224:7200)
  ├── Same AS_PATH length, same MED
  └── TGW or VGW performs ECMP: traffic is load-balanced across both
      (flow-level hashing, not packet-level — same flow stays on same path)
```

### Public VIF Scope Communities

Public VIF routes use different community tags to control geographic scope:

```
PUBLIC VIF SCOPE COMMUNITIES
=============================================================================

  Community     Scope           What it means
  ────────────────────────────────────────────────────
  7224:9100     Local region    Only the region where the DX location resides
                                sees your advertised prefixes
  7224:9200     Continental     All regions on the same continent
  7224:9300     Global          All AWS regions worldwide

  EXAMPLE: You have a DX connection in us-east-1 and advertise your
  public IP prefix 203.0.113.0/24 for accessing S3:

  ├── Tag with 7224:9100 → Only us-east-1 routes S3 traffic to you over DX
  ├── Tag with 7224:9200 → All US regions route S3 traffic to you over DX
  └── Tag with 7224:9300 → All global regions route S3 traffic to you over DX

  AWS TAGGING: All routes learned from AWS on a Public VIF are tagged with
  NO_EXPORT community — this prevents you from accidentally advertising
  AWS public IP ranges to the broader internet via your other BGP peers.
```

---

## Part 6: Link Aggregation Groups (LAG) -- Bundling Tracks

### The Analogy

If one railway track (DX connection) gives you 10 Gbps, bundling four parallel tracks into a single corridor gives you 40 Gbps. A **Link Aggregation Group** combines multiple physical DX connections at the same location into a single logical connection using the IEEE 802.3ad (LACP) standard.

### Technical Details

```
LINK AGGREGATION GROUPS (LAG)
=============================================================================

  WHAT A LAG IS:
  ├── 2-4 Dedicated connections at the SAME DX location
  ├── Same bandwidth per connection (all must be same speed)
  ├── Combined into one logical interface using LACP (802.3ad)
  ├── VIFs are created on the LAG, not on individual connections
  └── Single BGP session for the entire LAG

  WHAT A LAG IS NOT:
  ├── NOT cross-location redundancy (all connections in same location)
  ├── NOT a substitute for multi-location resiliency
  └── NOT supported with Hosted connections

  LAG BEHAVIOR:
  ┌──────────────────────────────────────────────────────────────────┐
  │  LAG: 4 x 10 Gbps Dedicated connections at DX Location A       │
  │                                                                  │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                        │
  │  │ DX-1 │  │ DX-2 │  │ DX-3 │  │ DX-4 │  Physical connections  │
  │  │ 10G  │  │ 10G  │  │ 10G  │  │ 10G  │                        │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘                        │
  │     │         │         │         │                              │
  │     └─────────┴─────────┴─────────┘                              │
  │                    │                                              │
  │             LACP Bundle                                          │
  │             (40 Gbps aggregate)                                  │
  │                    │                                              │
  │              ┌─────┴─────┐                                       │
  │              │ Transit   │   VIFs are created on the LAG         │
  │              │ VIF       │   not on individual connections       │
  │              └───────────┘                                       │
  │                                                                  │
  │  MINIMUM LINKS THRESHOLD:                                        │
  │  ├── You set a minimum (e.g., 3 out of 4)                       │
  │  ├── If fewer than 3 links are active, the entire LAG goes down  │
  │  ├── This prevents degraded-bandwidth operation                  │
  │  └── BGP failover triggers, traffic shifts to backup path        │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Resiliency Models -- How Many Railways Do You Build?

This is where the physical infrastructure decisions directly impact your SLA. AWS defines three resiliency models through the **Direct Connect Resiliency Toolkit**.

```
THREE RESILIENCY MODELS
=============================================================================

  MODEL 1: DEVELOPMENT AND TEST (no SLA)
  ────────────────────────────────────────────────────
  ┌──────────────┐        ┌─────────────────────┐
  │  On-Prem     │───DX1──│  DX Location A      │
  │              │───DX2──│  (separate devices)  │
  └──────────────┘        └─────────────────────┘

  ├── 2 connections on SEPARATE DEVICES at ONE location
  ├── Protects against: single device failure
  ├── Vulnerable to: location-level failure (fiber cut to facility,
  │   power outage, natural disaster)
  ├── SLA: None
  └── Use for: Dev/test environments, non-critical workloads


  MODEL 2: HIGH RESILIENCY (99.9% SLA)
  ────────────────────────────────────────────────────
  ┌──────────────┐        ┌─────────────────────┐
  │              │───DX1──│  DX Location A      │
  │  On-Prem     │        └─────────────────────┘
  │              │        ┌─────────────────────┐
  │              │───DX2──│  DX Location B      │
  └──────────────┘        └─────────────────────┘

  ├── 1 connection at EACH of 2 DIFFERENT locations
  ├── Protects against: device failure + single location failure
  ├── Vulnerable to: simultaneous failure of both locations (unlikely)
  ├── SLA: 99.9%
  └── Use for: Production workloads with moderate availability needs


  MODEL 3: MAXIMUM RESILIENCY (99.99% SLA)
  ────────────────────────────────────────────────────
  ┌──────────────┐        ┌─────────────────────┐
  │              │───DX1──│  DX Location A      │
  │              │───DX2──│  (separate devices)  │
  │  On-Prem     │        └─────────────────────┘
  │              │        ┌─────────────────────┐
  │              │───DX3──│  DX Location B      │
  │              │───DX4──│  (separate devices)  │
  └──────────────┘        └─────────────────────┘

  ├── 2 connections on SEPARATE DEVICES at EACH of 2 DIFFERENT locations
  ├── Minimum 4 connections total
  ├── Protects against: device failure + complete location failure
  ├── SLA: 99.99%
  └── Use for: Mission-critical production, financial, healthcare

  THE RESILIENCY TOOLKIT:
  ├── Connection wizard provisions connections on SEPARATE DEVICES (guaranteed)
  ├── LAG creation is a separate, deliberate choice (not created automatically)
  └── FAILOVER TESTING: Brings down a BGP session on demand so you can
      verify traffic reroutes correctly BEFORE a real outage hits
      (this is critical — never trust failover you have not tested)
```

---

## Part 8: MACsec Encryption and SiteLink

### MACsec -- The Armored Railcar

Direct Connect traffic is **not encrypted by default**. The fiber is private (not shared with other customers), but data traverses physical infrastructure that could theoretically be tapped. MACsec adds **Layer 2 encryption** (IEEE 802.1AE) directly on the wire.

```
MACsec ENCRYPTION
=============================================================================

  AVAILABILITY:
  ├── 10 Gbps, 100 Gbps, and 400 Gbps Dedicated connections ONLY
  ├── Not available on 1 Gbps or Hosted connections
  └── Not available at all DX locations (check AWS region table)

  HOW IT WORKS:
  ├── Key agreement: MACsec Key Agreement (MKA) protocol using
  │   pre-shared CAK/CKN (Connectivity Association Key / Key Name)
  ├── Encryption: AES-256-GCM (same standard as TLS 1.3)
  ├── Scope: Encrypts frames between YOUR router and the AWS DX device
  │   (Layer 2 — does not extend beyond the DX location)
  ├── Overhead: ~40 bytes per frame for MACsec header/trailer
  └── Performance: Line-rate encryption, no bandwidth penalty

  WHEN TO USE:
  ├── Compliance requirements (HIPAA, PCI-DSS, financial regulations)
  ├── Defense-in-depth alongside application-level TLS
  └── When IPsec VPN overhead is unacceptable but encryption is mandatory

  ALTERNATIVE IF MACsec IS NOT AVAILABLE:
  ├── VPN over DX: Create a Site-to-Site VPN using the Public VIF
  │   to establish IPsec tunnels over the DX connection
  ├── Private IP VPN: IPsec over Transit VIF (no public IPs needed)
  └── Both add encryption but reduce effective throughput due to
      IPsec overhead (~1.25 Gbps per tunnel, can ECMP for more)
```

### SiteLink -- Direct Track Between Stations

```
SITELINK -- INTER-SITE CONNECTIVITY
=============================================================================

  WITHOUT SITELINK:
  ────────────────────────────────────────────────────
  Site A (Tokyo) → DX Location Tokyo → AWS ap-northeast-1 →
    TGW peering → AWS us-east-1 → DX Location Virginia → Site B (NYC)

  Traffic must traverse an AWS region even to go between two DX locations.
  Adds latency, costs TGW data processing, requires TGW peering setup.


  WITH SITELINK:
  ────────────────────────────────────────────────────
  Site A (Tokyo) → DX Location Tokyo → AWS backbone → DX Location Virginia → Site B (NYC)

  Traffic flows directly between DX locations over the AWS global backbone.
  No region traversal. No TGW needed for the inter-site hop.

  HOW TO ENABLE:
  ├── Enable SiteLink on the VIF (per-VIF setting)
  ├── Both VIFs must be associated with the same DXGW
  ├── Both VIFs must have SiteLink enabled
  └── BGP routes from one site are advertised to the other site

  COST:
  ├── SiteLink data transfer charges apply (per-GB, varies by region pair)
  ├── Separate from standard DX data transfer charges
  └── Can be cheaper than routing through a region (no TGW processing)

  USE CASES:
  ├── Global enterprise WAN: connect offices via the AWS backbone
  ├── Disaster recovery: replicate between sites without region dependency
  └── Low-latency inter-site traffic where regional routing adds delay
```

---

## Part 9: Direct Connect vs Site-to-Site VPN

This comparison appears on almost every networking exam and interview. The two services are complementary, not competitive -- most production architectures use both.

```
DIRECT CONNECT vs SITE-TO-SITE VPN
=============================================================================

                         Direct Connect              Site-to-Site VPN
  ──────────────────────────────────────────────────────────────────────────
  Connection type        Dedicated private fiber      Encrypted tunnel over
                                                      public internet

  Bandwidth              1 Gbps - 400 Gbps           1.25 Gbps per tunnel
                         (per connection)             (~50 Gbps aggregate via
                                                      ECMP on TGW)

  Latency                Consistent, predictable      Variable (internet path)

  Encryption             Not by default               Always (IPsec)
                         (MACsec optional)

  Setup time             Weeks to months              Minutes to hours

  Cost model             Port-hours + data transfer   VPN connection-hours +
                         (lower $/GB than internet)   data transfer (internet
                                                      rates)

  Redundancy             You architect it              Built-in (2 tunnels
                         (multiple connections)        per VPN connection)

  Routing                BGP required                 BGP or static

  Connection to TGW      Transit VIF → DXGW → TGW    VPN attachment directly
                                                      on TGW

  Max throughput to TGW  Up to 50 Gbps (per DXGW     Up to ~50 Gbps with
                         attachment)                   ECMP (many tunnels)

  Use case               Primary path for production  Backup path, quick setup,
                         (high bandwidth, consistent  lower-bandwidth workloads,
                         latency, lower cost/GB)      encryption requirement

  COMMON PATTERN -- DX PRIMARY + VPN BACKUP:
  ════════════════════════════════════════════════════════
  ├── DX Connection with community 7224:7300 (high local preference)
  ├── VPN connection with community 7224:7100 (low local preference)
  ├── Normal operation: 100% traffic flows over DX (preferred)
  ├── DX failure: BGP detects loss, VPN takes over in seconds (with BFD)
  │   or minutes (default BGP timers)
  └── DX recovery: Traffic automatically returns to DX (higher preference)

  ENCRYPTED DX OPTIONS (when you need DX bandwidth + encryption):
  ├── MACsec on DX (Layer 2, line-rate, 10/100 Gbps only)
  │   NOTE: MACsec only encrypts the DX link segment (your router ↔ AWS DX
  │   device at the colocation). Traffic beyond the DX device — across the AWS
  │   backbone to your VPCs — is NOT encrypted by MACsec. For true end-to-end
  │   encryption, layer application-level TLS/mTLS on top.
  ├── VPN over Public VIF (IPsec tunnels using DX as transport)
  └── Private IP VPN over Transit VIF (IPsec without public IPs)
```

---

## Part 10: Terraform -- Direct Connect Resources

While you cannot provision the physical connection via Terraform (that requires AWS console or API for port ordering), you manage the logical resources -- VIFs, DXGW, and associations -- entirely in code.

```hcl
# ==============================================================
# Direct Connect Gateway
# ==============================================================

resource "aws_dx_gateway" "main" {
  name            = "enterprise-dxgw"
  amazon_side_asn = "64512"  # AWS-side ASN for BGP sessions
}

# ==============================================================
# Transit VIF — connects DX to DXGW (the scalable option)
# ==============================================================

resource "aws_dx_transit_virtual_interface" "primary" {
  connection_id = "dxcon-abc123def"  # Your Dedicated connection ID
  dx_gateway_id = aws_dx_gateway.main.id
  name          = "primary-transit-vif"

  vlan          = 300                # 802.1Q VLAN tag
  address_family = "ipv4"
  bgp_asn       = 65000             # Your on-prem ASN
  mtu           = 8500              # Max MTU for Transit VIF (not 9001!)

  tags = { Name = "primary-transit-vif" }
}

# ==============================================================
# Private VIF — for high-bandwidth VPCs (skip TGW processing)
# ==============================================================

resource "aws_dx_private_virtual_interface" "high_bandwidth" {
  connection_id = "dxcon-abc123def"
  dx_gateway_id = aws_dx_gateway.main.id  # Can also point to a VGW directly
  name          = "data-lake-private-vif"

  vlan          = 100
  address_family = "ipv4"
  bgp_asn       = 65000
  mtu           = 9001              # Private VIF supports jumbo frames

  tags = { Name = "data-lake-private-vif" }
}

# ==============================================================
# DXGW ↔ Transit Gateway Association
# ==============================================================

resource "aws_dx_gateway_association" "tgw_east" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = aws_ec2_transit_gateway.east.id

  allowed_prefixes {
    cidr = "10.0.0.0/8"    # Summarized prefix advertised from TGW to on-prem
  }
  allowed_prefixes {
    cidr = "172.16.0.0/12"  # Additional prefix for on-prem networks
  }
}

# ==============================================================
# DXGW ↔ Transit Gateway Association (second region)
# ==============================================================

resource "aws_dx_gateway_association" "tgw_europe" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = aws_ec2_transit_gateway.europe.id

  allowed_prefixes {
    cidr = "10.0.0.0/8"
  }
}

# ==============================================================
# Cross-Account DXGW Association Proposal
# (from a spoke account that owns a TGW)
# ==============================================================

resource "aws_dx_gateway_association_proposal" "spoke_account" {
  dx_gateway_id               = "dxgw-abc123"  # DXGW ID from network account
  dx_gateway_owner_account_id = "111111111111"  # Network account ID
  associated_gateway_id       = aws_ec2_transit_gateway.spoke.id

  allowed_prefixes {
    cidr = "10.0.0.0/8"
  }
}

# In the network account, accept the proposal:
resource "aws_dx_gateway_association" "accept_spoke" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = aws_ec2_transit_gateway.spoke.id  # From proposal
  proposal_id           = "proposal-abc123"

  allowed_prefixes {
    cidr = "10.0.0.0/8"
  }
}

# ==============================================================
# Link Aggregation Group
# ==============================================================

resource "aws_dx_lag" "primary_lag" {
  name                  = "primary-lag"
  connections_bandwidth = "10Gbps"
  location              = "EqDC2"         # DX location identifier
  number_of_connections = 2               # Creates 2 NEW connections (not just metadata)
  force_destroy         = false

  tags = { Name = "primary-lag" }
}

# Associate an existing connection with the LAG
resource "aws_dx_connection_association" "dx1_to_lag" {
  connection_id = "dxcon-abc123def"
  lag_id        = aws_dx_lag.primary_lag.id
}

# ==============================================================
# BGP Peer (example for Private VIF with IPv6)
# ==============================================================

resource "aws_dx_bgp_peer" "ipv6" {
  virtual_interface_id = aws_dx_private_virtual_interface.high_bandwidth.id
  address_family       = "ipv6"
  bgp_asn              = 65000
}
```

---

## Part 11: Connecting Everything -- DX + TGW Integration

Yesterday you studied Transit Gateway from the inside -- route tables, associations, propagations, attachments. Here is how DX plugs into that architecture from the outside.

```
DX → TGW INTEGRATION -- THE COMPLETE ATTACHMENT CHAIN
=============================================================================

  Physical Layer:
  ┌────────────────────────────────────────────────────────┐
  │  Your Router ←──fiber──→ AWS Router at DX Location     │
  │  (BGP peer)               (BGP peer)                   │
  └────────────────────────────────────────────────────────┘
                            │
  Logical Layer:            ▼
  ┌────────────────────────────────────────────────────────┐
  │  Transit VIF (802.1Q VLAN on the physical connection)  │
  │  BGP session: your ASN ↔ 64512                         │
  │  Routes advertised TO AWS: your on-prem CIDRs          │
  │  Routes learned FROM AWS: summarized VPC CIDRs         │
  └────────────────────────────────────────────────────────┘
                            │
  Global Layer:             ▼
  ┌────────────────────────────────────────────────────────┐
  │  Direct Connect Gateway (globally available)            │
  │  Associates Transit VIF with TGW(s)                     │
  │  Allowed prefixes control what is advertised            │
  └────────────────────────────────────────────────────────┘
                            │
  Regional Layer:           ▼
  ┌────────────────────────────────────────────────────────┐
  │  Transit Gateway (regional)                             │
  │  DX Gateway attachment in the TGW route table           │
  │  ├── Association: DX attachment → shared-rt             │
  │  ├── Propagation: DX attachment propagates on-prem      │
  │  │   CIDRs to spoke route tables                        │
  │  └── Spoke VPCs propagate their CIDRs to shared-rt     │
  │      (so TGW knows to send on-prem-bound traffic to DX) │
  └────────────────────────────────────────────────────────┘
                            │
  VPC Layer:                ▼
  ┌────────────────────────────────────────────────────────┐
  │  Spoke VPC route table: 172.16.0.0/12 → tgw            │
  │  (on-prem CIDRs routed to TGW, which forwards to DX)   │
  └────────────────────────────────────────────────────────┘

  CONFIGURATION CHECKLIST FOR DX + TGW:
  ════════════════════════════════════════════════════════
  □ Physical DX connection provisioned and active
  □ Transit VIF created on the DX connection, pointing to DXGW
  □ BGP session established (verify with BGP state = "available")
  □ DXGW created with appropriate Amazon-side ASN
  □ DXGW associated with TGW (allowed_prefixes configured)
  □ TGW route table: DX attachment associated with a route table
  □ TGW route table: DX attachment propagates on-prem routes
  □ TGW route table: spoke VPCs propagate to the DX route table
  □ Spoke VPC route tables: on-prem CIDRs → TGW
  □ On-prem router: AWS VPC CIDRs learned via BGP (or static routes)
  □ BFD enabled for sub-second failover detection
  □ Failover tested using the Resiliency Toolkit

  COMMON TROUBLESHOOTING:
  ════════════════════════════════════════════════════════
  ├── VIF stuck in "verifying" → BGP has not come up; check VLAN tag,
  │   MD5 auth key, and BGP ASN match on both sides
  ├── Routes not propagating → Check TGW route table associations and
  │   propagations; verify allowed_prefixes on DXGW-TGW association
  │   include the relevant CIDRs
  └── Asymmetric routing → Ensure return-path route tables in VPCs
      point to TGW, and TGW route table sends on-prem-bound traffic
      to the DX attachment
```

---

## Key Takeaways

- **Direct Connect is physical infrastructure** -- unlike most AWS services, it involves real fiber optic cables at colocation facilities. Provisioning takes weeks, not minutes. Plan ahead.

- **Three VIF types serve three purposes**: Private VIF for VPC resources (via VGW or DXGW), Transit VIF for scalable multi-VPC access (via DXGW to TGW), Public VIF for AWS public service endpoints. A Dedicated connection can run all three simultaneously.

- **DXGW is the global glue** that connects your DX connection to VPCs and TGWs in any region. Without it, each VIF reaches only one VGW in one region. With it, one VIF reaches up to 20 VGWs or 6 TGWs globally. Critical limitation: north-south only, no VPC-to-VPC transit.

- **Transit VIF is the default for most architectures** because it scales to thousands of VPCs through TGW. Use Private VIF selectively for high-bandwidth VPCs where TGW processing costs ($0.02/GB) are prohibitive.

- **The 200-prefix limit from TGW to on-prem** makes CIDR planning and route summarization essential. This is why the hierarchical CIDR allocation from the Feb 19 doc matters for hybrid connectivity.

- **BGP communities give you deterministic failover**: tag your primary DX with 7224:7300 (high preference) and backup with 7224:7100 (low preference). For active/active, use identical attributes and let ECMP distribute traffic.

- **Resiliency is your responsibility**: single connection = no SLA. Two connections at two locations = 99.9%. Two connections on separate devices at two locations (4 total) = 99.99%. Always test failover with the Resiliency Toolkit before you need it.

- **DX is not encrypted by default**: use MACsec (10/100/400 Gbps Dedicated only) for line-rate Layer 2 encryption, or VPN-over-DX for IPsec encryption at the cost of throughput.

- **DX + VPN is the standard enterprise pattern**: DX as primary (high bandwidth, consistent latency, lower cost/GB), VPN as encrypted backup (quick failover, always available).

- **SiteLink enables direct inter-site traffic** over the AWS backbone without routing through a region -- useful for global WAN architectures where DX locations serve as network hubs.

---

## Further Reading

- [AWS Direct Connect User Guide](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html) -- The definitive reference for all DX features and configuration
- [Building a Scalable Multi-VPC Network -- Direct Connect section](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/direct-connect.html) -- The architecture decision guide for DX in multi-VPC environments
- [Transit Gateway Deep Dive (yesterday's doc)](./2026-02-23-transit-gateway-deep-dive.md) -- The TGW architecture that DX plugs into via Transit VIF and DXGW
- [VPC Peering vs Transit Gateway](./2026-02-20-vpc-peering-vs-transit-gateway.md) -- Understanding when TGW vs VGW determines your VIF type choice
- [VPC Advanced: CIDR Planning](./2026-02-19-vpc-advanced-cidr-subnet-strategy.md) -- Why hierarchical CIDR allocation enables the route summarization DX requires
