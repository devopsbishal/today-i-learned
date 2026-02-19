# VPC Advanced: CIDR Planning and Subnet Strategy - The City Planning Analogy

> Understanding enterprise-level VPC design through a real-world analogy of a metropolitan city planner who must assign postal codes, zone neighborhoods, design road networks, build private highways, and manage addressing for a growing chain of cities -- all without a single duplicate address or a dead-end street.

---

## TL;DR

| AWS VPC Concept | Real-World Analogy | One-Liner |
|-----------------|-------------------|-----------|
| CIDR Block (/8, /16, /20, etc.) | Postal code range for a city | Defines how many addresses (IPs) a city (VPC) can contain; larger prefix = fewer addresses |
| Non-Overlapping CIDR Planning | Unique postal codes per city | Two cities sharing the same postal codes cannot exchange mail; plan ranges before breaking ground |
| IPAM (IP Address Manager) | National postal authority | Central authority that allocates postal code ranges to regions, then to cities, preventing duplicates organization-wide |
| IPAM Pool Hierarchy | National > Regional > District > Neighborhood postal code allocation | A four-tier system ensuring every city gets a unique, properly-sized code range from a master plan |
| Public Subnet | Neighborhood with a main road entrance | Anyone can drive in from the highway (internet); residents have public-facing addresses |
| Private Subnet | Gated residential neighborhood | Residents can leave through a guarded exit (NAT) but outsiders cannot enter directly |
| Isolated Subnet | Walled compound with no road to the highway | No internet route at all; residents communicate only via internal roads and private couriers |
| Route Table | Road map posted at every neighborhood entrance | Tells drivers how to reach any destination; each neighborhood can have a different map |
| Internet Gateway (IGW) | Highway on-ramp/off-ramp | Two-way connection between your city and the outside world |
| Zonal NAT Gateway (legacy) | One guarded exit per district | Each district (AZ) has its own exit guard; if the guard is sick, that district is stuck |
| Regional NAT Gateway (Nov 2025) | One city-wide exit authority that staffs guards in every district automatically | A single entity that expands and contracts across districts based on where residents live |
| Gateway Endpoint (S3/DynamoDB) | Free private road to the government warehouse | A dedicated road that never touches the public highway; no toll, no limit |
| Interface Endpoint (PrivateLink) | Private telephone line installed in your neighborhood | An ENI with a private address in your subnet that connects to an AWS service without leaving the city |
| Secondary CIDR Block | Annexing adjacent land when the city outgrows its original borders | Add more address space to an existing VPC without rebuilding; some zoning restrictions apply |
| 5 Reserved IPs per Subnet | City infrastructure addresses | The first four and last address in every neighborhood are reserved for roads, fire stations, and utilities |
| Multi-AZ Subnet Architecture | Building the same neighborhood layout in multiple earthquake zones | If one zone has a natural disaster, the other zones keep the city running |

---

## The Big Picture

In the [basic VPC doc from November 2025](../../../2025/november/aws-vpc/2025-11-30-aws-vpc.md), we covered VPC as a secure compound: subnets as buildings, route tables as paths, IGW as the main gate, NAT as the agent handler. That was enough for a single VPC.

Now imagine you are the **chief city planner** for a rapidly growing nation. You do not have one compound -- you have **dozens of cities** (VPCs) across multiple regions, each with their own neighborhoods (subnets), road networks (route tables), and connections to the highway system (internet). Your mandate is:

1. **No two cities can share the same postal codes** -- If City A and City B both use 10.0.0.0/16, mail between them is undeliverable
2. **Every city needs three types of neighborhoods** -- Public-facing shops, gated residential areas, and walled-off vaults
3. **Road maps must be precise** -- A wrong route sends traffic to the internet when it should stay local, or into a dead end
4. **Exit guards must be available in every district** -- If one district loses its guard, residents there cannot reach the outside world
5. **Growth must be planned for** -- A city that runs out of postal codes cannot retroactively get more without significant pain

This document is your **city planning handbook** for enterprise AWS networking.

```
ENTERPRISE VPC LANDSCAPE — THE PLANNING CHALLENGE
=============================================================================

  ORGANIZATION (the nation)
  │
  ├── REGION: us-east-1 (eastern territory)
  │   ├── ACCOUNT: production
  │   │   └── VPC: 10.0.0.0/16     ← City A (65,536 addresses)
  │   ├── ACCOUNT: staging
  │   │   └── VPC: 10.1.0.0/16     ← City B
  │   ├── ACCOUNT: development
  │   │   └── VPC: 10.2.0.0/16     ← City C
  │   └── ACCOUNT: shared-services
  │       └── VPC: 10.3.0.0/16     ← City D
  │
  ├── REGION: eu-west-1 (western territory)
  │   ├── ACCOUNT: production
  │   │   └── VPC: 10.64.0.0/16    ← City E (different range!)
  │   ├── ACCOUNT: staging
  │   │   └── VPC: 10.65.0.0/16    ← City F
  │   └── ...
  │
  └── ON-PREMISES: 172.16.0.0/12   ← Legacy headquarters
      (must NOT overlap with any AWS city)

  THE PROBLEM: Without a master plan, team A picks 10.0.0.0/16,
  team B picks 10.0.0.0/16, and they can never be peered or
  connected via Transit Gateway. This happens in EVERY enterprise
  that tracks CIDRs in spreadsheets.
```

---

## Part 1: CIDR Planning for Non-Overlapping Ranges - The National Postal Authority

### The Analogy

Imagine a nation where every city assigns its own postal codes independently. City A picks codes 10000-19999. City B, unaware, also picks 10000-19999. When someone in City A mails a letter to code 15000, the postal system has no idea which city to deliver to. **The mail is undeliverable.**

The solution is a **National Postal Authority** (AWS IPAM) that owns the entire code space and allocates ranges hierarchically: the nation gets 10.0.0.0/8 (16.7 million addresses), regions get /10 slices, business units get /14 slices, and individual cities (VPCs) get /16 allocations. No city can claim codes that belong to another.

### The Technical Reality

CIDR (Classless Inter-Domain Routing) planning for a multi-VPC, multi-account, multi-region organization requires a hierarchical allocation scheme that prevents overlaps and leaves room for growth.

```
RFC 1918 PRIVATE ADDRESS SPACE — YOUR RAW MATERIAL
=============================================================================

  ┌──────────────────────────────────────────────────────────────────────┐
  │  RANGE              │ CIDR            │ TOTAL ADDRESSES │ COMMON USE│
  ├──────────────────────────────────────────────────────────────────────┤
  │  10.0.0.0/8         │ 10.0.0.0 -      │ 16,777,216     │ Enterprise│
  │                     │ 10.255.255.255  │                │ (largest) │
  ├──────────────────────────────────────────────────────────────────────┤
  │  172.16.0.0/12      │ 172.16.0.0 -    │ 1,048,576      │ Often     │
  │                     │ 172.31.255.255  │                │ on-prem   │
  ├──────────────────────────────────────────────────────────────────────┤
  │  192.168.0.0/16     │ 192.168.0.0 -   │ 65,536         │ Home/lab  │
  │                     │ 192.168.255.255 │                │ networks  │
  └──────────────────────────────────────────────────────────────────────┘

  KEY CONSTRAINTS:
  ├── VPC CIDR must be between /16 (65,536 IPs) and /28 (16 IPs)
  ├── AVOID 172.17.0.0/16 — conflicts with Docker default bridge and
  │   SageMaker default networking
  ├── Each subnet reserves 5 IPs:
  │   ├── .0   = network address
  │   ├── .1   = VPC router
  │   ├── .2   = DNS server
  │   ├── .3   = reserved for future use
  │   └── last IP in the CIDR range = broadcast (e.g., .255 in a /24, .4095 in a /20)
  │   So a /24 subnet (256 IPs) has only 251 usable addresses
  └── Secondary CIDRs must be from the SAME RFC 1918 range as the
      primary (e.g., a 10.x VPC cannot add 172.16.x or 192.168.x).
      You CAN add 100.64.0.0/10 (CGNAT) regardless of the primary range.
```

### Hierarchical CIDR Allocation — The Enterprise Pattern

```
FOUR-TIER CIDR HIERARCHY
=============================================================================

  TIER 1: TOP-LEVEL POOL (the entire nation's address space)
  ┌─────────────────────────────────────────────────────────────┐
  │  10.0.0.0/8  (16,777,216 addresses)                        │
  │                                                             │
  │  Owned by: Central network team                             │
  │  Rule: NEVER allocate directly from this pool               │
  └─────────────────────────┬───────────────────────────────────┘
                            │
  TIER 2: REGIONAL POOLS    │  (split by region)
  ┌─────────────────────────┼───────────────────────────────────┐
  │                         │                                   │
  │  ┌──────────────────┐   │   ┌──────────────────┐           │
  │  │ us-east-1        │   │   │ eu-west-1        │           │
  │  │ 10.0.0.0/10      │   │   │ 10.64.0.0/10     │           │
  │  │ (4,194,304 IPs)  │   │   │ (4,194,304 IPs)  │           │
  │  └────────┬─────────┘   │   └────────┬─────────┘           │
  │           │              │            │                     │
  └───────────┼──────────────┼────────────┼─────────────────────┘
              │              │            │
  TIER 3: BU/ENVIRONMENT     │  (split by business unit or environment)
  ┌───────────┼──────────────┼────────────┼─────────────────────┐
  │           │                           │                     │
  │  ┌────────┴────────┐      ┌───────────┴──────────┐         │
  │  │ us-east-1       │      │ eu-west-1            │         │
  │  │ production      │      │ production           │         │
  │  │ 10.0.0.0/14     │      │ 10.64.0.0/14         │         │
  │  │ (262,144 IPs)   │      │ (262,144 IPs)        │         │
  │  └────────┬────────┘      └───────────┬──────────┘         │
  │           │                           │                    │
  │  ┌────────┴────────┐      ┌───────────┴──────────┐        │
  │  │ us-east-1       │      │ eu-west-1            │        │
  │  │ staging         │      │ staging              │        │
  │  │ 10.4.0.0/14     │      │ 10.68.0.0/14         │        │
  │  └────────┬────────┘      └──────────────────────┘        │
  │           │                                                │
  └───────────┼────────────────────────────────────────────────┘
              │
  TIER 4: VPC ALLOCATIONS  (individual cities)
  ┌───────────┼────────────────────────────────────────────────┐
  │           │                                                │
  │  ┌────────┴────────┐  ┌──────────────────┐                │
  │  │ VPC: app-prod   │  │ VPC: data-prod   │                │
  │  │ 10.0.0.0/16     │  │ 10.1.0.0/16      │                │
  │  │ (65,536 IPs)    │  │ (65,536 IPs)     │                │
  │  └─────────────────┘  └──────────────────┘                │
  │                                                            │
  │  Each /14 holds up to 4 VPCs at /16 size                  │
  │  Each /16 VPC holds up to 16 subnets at /20 size          │
  │  (or 64 subnets at /22, etc.)                             │
  └────────────────────────────────────────────────────────────┘


  CONCRETE ALLOCATION TABLE — 10+ VPCs ACROSS 2 REGIONS
  ════════════════════════════════════════════════════════

  Region        Environment    Account            VPC CIDR
  ──────────    ───────────    ────────────────    ────────────────
  us-east-1     production     app-prod           10.0.0.0/16
  us-east-1     production     data-prod          10.1.0.0/16
  us-east-1     production     shared-services    10.2.0.0/16
  us-east-1     staging        app-staging        10.4.0.0/16
  us-east-1     staging        data-staging       10.5.0.0/16
  us-east-1     development    app-dev            10.8.0.0/16
  us-east-1     development    data-dev           10.9.0.0/16
  eu-west-1     production     app-prod-eu        10.64.0.0/16
  eu-west-1     production     data-prod-eu       10.65.0.0/16
  eu-west-1     staging        app-staging-eu     10.68.0.0/16
  eu-west-1     development    app-dev-eu         10.72.0.0/16
  on-premises   —              legacy-dc          172.16.0.0/12

  NOTICE: Gaps between environments (10.0-10.3, then 10.4-10.7,
  then 10.8-10.11) leave room for growth. The on-premises network
  uses an entirely different RFC 1918 range to avoid any overlap.
```

### AWS IPAM — Automating the National Postal Authority

```
IPAM ARCHITECTURE
=============================================================================

  ┌──────────────────────────────────────────────────────────────────────┐
  │                        AWS IPAM                                      │
  │                                                                      │
  │  IPAM SCOPE: private                                                 │
  │                                                                      │
  │  TOP-LEVEL POOL: 10.0.0.0/8                                        │
  │  ├── Allocation rules:                                              │
  │  │   ├── min_netmask_length = /10 (prevent oversized allocations)   │
  │  │   └── max_netmask_length = /10 (force consistent regional size)  │
  │  │                                                                   │
  │  ├── REGIONAL POOL: us-east-1 — 10.0.0.0/10                       │
  │  │   ├── locale = us-east-1                                         │
  │  │   ├── Allocation rules: min /14, max /16                         │
  │  │   │                                                               │
  │  │   ├── ENV POOL: production — 10.0.0.0/14                        │
  │  │   │   ├── Allocation rules: min /16, max /16                     │
  │  │   │   ├── VPC: app-prod → allocated 10.0.0.0/16                 │
  │  │   │   ├── VPC: data-prod → allocated 10.1.0.0/16                │
  │  │   │   └── (1 more /16 allocation available in this /14)                    │
  │  │   │                                                               │
  │  │   ├── ENV POOL: staging — 10.4.0.0/14                           │
  │  │   │   └── VPC: app-staging → allocated 10.4.0.0/16              │
  │  │   │                                                               │
  │  │   └── ENV POOL: development — 10.8.0.0/14                       │
  │  │       └── VPC: app-dev → allocated 10.8.0.0/16                  │
  │  │                                                                   │
  │  └── REGIONAL POOL: eu-west-1 — 10.64.0.0/10                      │
  │      └── (same structure as above)                                   │
  │                                                                      │
  │  SHARING: Pools shared via AWS RAM to member accounts                │
  │  MONITORING: IPAM tracks utilization per pool — alerts when          │
  │  approaching exhaustion so you can plan secondary CIDRs              │
  └──────────────────────────────────────────────────────────────────────┘
```

### Terraform: IPAM Hierarchical Pool Structure

```hcl
# IPAM — the national postal authority
resource "aws_vpc_ipam" "main" {
  description = "Organization-wide IP address manager"

  operating_regions {
    region_name = "us-east-1"
  }

  operating_regions {
    region_name = "eu-west-1"
  }

  tags = {
    Name = "org-ipam"
  }
}

# Top-level pool — 10.0.0.0/8 (the entire national address space)
resource "aws_vpc_ipam_pool" "top_level" {
  address_family = "ipv4"
  ipam_scope_id  = aws_vpc_ipam.main.private_default_scope_id
  description    = "Top-level pool — all private IPv4 space"

  tags = {
    Name = "top-level"
  }
}

resource "aws_vpc_ipam_pool_cidr" "top_level" {
  ipam_pool_id = aws_vpc_ipam_pool.top_level.id
  cidr         = "10.0.0.0/8"
}

# Regional pool — us-east-1 gets 10.0.0.0/10
resource "aws_vpc_ipam_pool" "us_east_1" {
  address_family      = "ipv4"
  ipam_scope_id       = aws_vpc_ipam.main.private_default_scope_id
  source_ipam_pool_id = aws_vpc_ipam_pool.top_level.id
  locale              = "us-east-1"
  description         = "us-east-1 regional pool"

  allocation_min_netmask_length = 14
  allocation_max_netmask_length = 16

  tags = {
    Name   = "us-east-1-pool"
    Region = "us-east-1"
  }
}

resource "aws_vpc_ipam_pool_cidr" "us_east_1" {
  ipam_pool_id = aws_vpc_ipam_pool.us_east_1.id
  cidr         = "10.0.0.0/10"
}

# Environment pool — production in us-east-1 gets 10.0.0.0/14
resource "aws_vpc_ipam_pool" "us_east_1_production" {
  address_family      = "ipv4"
  ipam_scope_id       = aws_vpc_ipam.main.private_default_scope_id
  source_ipam_pool_id = aws_vpc_ipam_pool.us_east_1.id
  locale              = "us-east-1"
  description         = "us-east-1 production environment pool"

  # Force all VPC allocations to be exactly /16
  allocation_min_netmask_length = 16
  allocation_max_netmask_length = 16

  tags = {
    Name        = "us-east-1-production"
    Environment = "production"
  }
}

resource "aws_vpc_ipam_pool_cidr" "us_east_1_production" {
  ipam_pool_id = aws_vpc_ipam_pool.us_east_1_production.id
  cidr         = "10.0.0.0/14"
}

# VPC that allocates its CIDR from IPAM — no hardcoded CIDR needed
resource "aws_vpc" "app_prod" {
  ipv4_ipam_pool_id   = aws_vpc_ipam_pool.us_east_1_production.id
  ipv4_netmask_length = 16 # Request a /16 from the pool

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "app-prod"
    Environment = "production"
    ManagedBy   = "ipam"
  }
}

# Share the production pool with the production OU via RAM
resource "aws_ram_resource_share" "ipam_production" {
  name                      = "ipam-production-pool"
  allow_external_principals = false # Organization only
}

resource "aws_ram_resource_association" "ipam_production" {
  resource_arn       = aws_vpc_ipam_pool.us_east_1_production.arn
  resource_share_arn = aws_ram_resource_share.ipam_production.arn
}

resource "aws_ram_principal_association" "production_ou" {
  principal          = var.production_ou_arn # e.g., arn:aws:organizations::123456789012:ou/o-abc123/ou-def456
  resource_share_arn = aws_ram_resource_share.ipam_production.arn
}
```

### Secondary CIDR Blocks — Annexing Adjacent Land

```
SECONDARY CIDR BLOCKS
=============================================================================

  WHEN YOU NEED THEM:
  ├── Original /16 is running out of IPs (EKS pod networking is a
  │   common culprit — each pod gets a VPC IP with the VPC CNI)
  ├── You need connectivity to a network using a different RFC 1918 range
  └── You want to add 100.64.0.0/10 (carrier-grade NAT) space for
      pods while keeping RFC 1918 for node networking

  HOW IT WORKS:
  ├── aws_vpc_ipv4_cidr_block_association adds a secondary CIDR
  ├── You can add up to 5 CIDRs per VPC (adjustable quota)
  ├── New subnets can be created in the secondary range
  └── Existing subnets are NOT affected — this is additive only

  RESTRICTIONS:
  ├── Secondary CIDRs cannot overlap with existing CIDRs or peered VPCs
  ├── Secondary CIDRs must be from the SAME RFC 1918 range as the primary
  │   (e.g., a 10.x VPC can add another 10.x block, but NOT 172.16.x
  │   or 192.168.x). You CAN always add 100.64.0.0/10 (CGNAT range).
  └── CIDRs cannot be removed if subnets or route table entries use them

  COMMON PATTERN FOR EKS:
  ────────────────────────────────────────────────────
  Primary CIDR:    10.0.0.0/16  → node subnets, control plane
  Secondary CIDR:  100.64.0.0/16 → pod subnets (carrier-grade NAT range)

  This keeps node IPs routable across VPCs and peering while giving
  pods a massive address space that does not consume the main range.

  HOW IT WORKS: Enable EKS Custom Networking and create ENIConfig
  custom resources that map each AZ to a secondary CIDR subnet. This
  tells the VPC CNI plugin to allocate pod IPs from the secondary
  subnets instead of from the node's subnet in the primary CIDR.
  Without custom networking, pods draw IPs from the same subnet as
  the node, defeating the purpose of the secondary block.
```

```hcl
# Add secondary CIDR for EKS pod networking
resource "aws_vpc_ipv4_cidr_block_association" "pods" {
  vpc_id     = aws_vpc.app_prod.id
  cidr_block = "100.64.0.0/16"
}

# Pod subnets in the secondary range
resource "aws_subnet" "pods_az_a" {
  vpc_id     = aws_vpc.app_prod.id
  cidr_block = "100.64.0.0/19" # 8,187 usable IPs for pods in AZ-a
  availability_zone = "us-east-1a"

  tags = {
    Name = "pods-us-east-1a"
    Tier = "pod-networking"
  }

  depends_on = [aws_vpc_ipv4_cidr_block_association.pods]
}
```

---

## Part 2: Subnet Strategy — Three Types of Neighborhoods

### The Analogy

Every well-planned city has three types of neighborhoods:

1. **Commercial districts** (public subnets) — Shops and offices with storefronts on the main road. Anyone can walk in from the highway. The buildings have public-facing addresses and direct highway access.

2. **Gated residential areas** (private subnets) — Residents can leave through a guarded exit to run errands (NAT Gateway), but visitors from outside cannot enter directly. The gate guard translates the resident's private address into a public one for the trip.

3. **Walled-off vaults** (isolated subnets) — High-security areas with no road to the highway at all. The only way in or out is through internal roads (VPC routes) and private couriers (VPC endpoints). Think bank vaults, classified research labs.

The key insight: **what makes a subnet public, private, or isolated is NOT a property on the subnet itself -- it is entirely determined by its route table**. A subnet with a route to an IGW is public. A subnet with a route to a NAT Gateway is private. A subnet with neither is isolated.

### The Technical Reality

```
THREE-TIER SUBNET ARCHITECTURE — STANDARD ENTERPRISE PATTERN
=============================================================================

  VPC: 10.0.0.0/16
  ┌──────────────────────────────────────────────────────────────────────┐
  │                                                                      │
  │  AZ: us-east-1a          AZ: us-east-1b          AZ: us-east-1c    │
  │                                                                      │
  │  PUBLIC SUBNETS (route to IGW)                                       │
  │  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐ │
  │  │ 10.0.0.0/20      │   │ 10.0.16.0/20     │   │ 10.0.32.0/20    │ │
  │  │ 4,091 usable IPs │   │ 4,091 usable IPs │   │ 4,091 usable IPs│ │
  │  │                  │   │                  │   │                  │ │
  │  │ ALB, NAT GW,     │   │ ALB, NAT GW,     │   │ ALB, NAT GW,    │ │
  │  │ bastion hosts     │   │ bastion hosts     │   │ bastion hosts    │ │
  │  └──────────────────┘   └──────────────────┘   └──────────────────┘ │
  │                                                                      │
  │  PRIVATE SUBNETS (route to NAT Gateway)                              │
  │  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐ │
  │  │ 10.0.48.0/20     │   │ 10.0.64.0/20     │   │ 10.0.80.0/20    │ │
  │  │ 4,091 usable IPs │   │ 4,091 usable IPs │   │ 4,091 usable IPs│ │
  │  │                  │   │                  │   │                  │ │
  │  │ ECS tasks, EC2,  │   │ ECS tasks, EC2,  │   │ ECS tasks, EC2, │ │
  │  │ EKS worker nodes │   │ EKS worker nodes │   │ EKS worker nodes│ │
  │  └──────────────────┘   └──────────────────┘   └──────────────────┘ │
  │                                                                      │
  │  ISOLATED SUBNETS (no internet route)                                │
  │  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐ │
  │  │ 10.0.96.0/20     │   │ 10.0.112.0/20    │   │ 10.0.128.0/20   │ │
  │  │ 4,091 usable IPs │   │ 4,091 usable IPs │   │ 4,091 usable IPs│ │
  │  │                  │   │                  │   │                  │ │
  │  │ RDS, ElastiCache,│   │ RDS, ElastiCache,│   │ RDS, ElastiCache│ │
  │  │ internal Lambda  │   │ internal Lambda  │   │ internal Lambda  │ │
  │  └──────────────────┘   └──────────────────┘   └──────────────────┘ │
  │                                                                      │
  │  REMAINING: 10.0.144.0/20 through 10.0.240.0/20 = 7 more /20       │
  │  blocks reserved for future use (EKS pod subnets, TGW subnets,     │
  │  new tiers, etc.)                                                    │
  └──────────────────────────────────────────────────────────────────────┘


  SUBNET SIZING GUIDELINES:
  ════════════════════════════════════════════════════════

  Use Case                  Minimum Size    Recommended    Why
  ─────────────────────     ────────────    ───────────    ──────────────
  Public (ALB, NAT)         /24 (251)       /20 (4,091)    ALBs consume
                                                           IPs per AZ;
                                                           NAT GW needs 1
  Private (workloads)       /22 (1,019)     /20 (4,091)    ECS tasks and
                                                           EC2 instances
                                                           each take an IP
  Private (EKS pods)        /19 (8,187)     /18 (16,379)   VPC CNI assigns
                                                           one IP per pod;
                                                           100s of pods
                                                           per node
  Isolated (databases)      /24 (251)       /22 (1,019)    RDS Multi-AZ
                                                           needs 2 IPs per
                                                           instance
  TGW attachment subnets    /28 (11)        /28 (11)       TGW needs only
                                                           1 IP per AZ;
                                                           /28 is minimum
```

### What Defines Each Subnet Tier

```
WHAT MAKES A SUBNET PUBLIC, PRIVATE, OR ISOLATED
=============================================================================

  PUBLIC SUBNET:
  ├── Route table has: 0.0.0.0/0 → igw-xxxxxxxx
  ├── Resources CAN have public IPs (auto-assign or EIP)
  ├── Bidirectional internet access
  └── Hosts: ALBs, NLBs, NAT Gateways, bastion hosts

  PRIVATE SUBNET:
  ├── Route table has: 0.0.0.0/0 → nat-xxxxxxxx (or tgw-xxxxxxxx)
  ├── Resources have private IPs ONLY
  ├── Outbound internet via NAT; NO inbound from internet
  └── Hosts: application servers, ECS tasks, EKS nodes, Lambda

  ISOLATED SUBNET:
  ├── Route table has: NO 0.0.0.0/0 route at all
  ├── Only local VPC route (10.0.0.0/16 → local)
  ├── Plus VPC endpoint routes (for S3, DynamoDB, Secrets Manager, etc.)
  ├── NO internet access in either direction
  └── Hosts: RDS, ElastiCache, DocumentDB, Redshift, internal Lambda

  THE CRITICAL INSIGHT:
  ════════════════════════════════════════════════════════
  The subnet resource itself has NO "public" or "private" property in
  the AWS API. It is ENTIRELY determined by the route table association.
  Change the route table, and a private subnet becomes public (or
  isolated). This is why route table design is so important.
```

---

## Part 3: Route Table Design — The City Road Map

### The Analogy

Every neighborhood in your city has a **road map** posted at the entrance. When a car (packet) arrives at the intersection, it checks the map: "Where am I going? Take the most specific route listed." If the map says "10.0.0.0/16 -> local roads" and also "0.0.0.0/0 -> highway on-ramp," the car going to 10.0.5.20 takes the local road (more specific match), while the car going to 8.8.8.8 takes the highway.

The critical design decision: **do all neighborhoods share one map, or does each neighborhood get its own?** In the legacy (zonal NAT) model, each AZ's private subnets needed a separate map pointing to that AZ's specific NAT Gateway. With the new Regional NAT Gateway, all private subnets can share a single map.

### The Technical Reality

```
ROUTE TABLE DESIGN — LEGACY (ZONAL NAT) vs MODERN (REGIONAL NAT)
=============================================================================

  LEGACY PATTERN: One route table per tier PER AZ for private subnets
  ────────────────────────────────────────────────────

  Route Table: public-rt (shared across all AZs)
  ┌─────────────────────────────────────────────────┐
  │  Destination       │ Target          │ Note     │
  ├─────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ local           │ VPC CIDR │
  │  0.0.0.0/0         │ igw-abc123      │ Internet │
  │  pl-12345 (S3)     │ vpce-s3-gw      │ S3 GW EP│
  └─────────────────────────────────────────────────┘
  Associated with: public-subnet-a, public-subnet-b, public-subnet-c

  Route Table: private-rt-az-a (AZ-a private subnets only)
  ┌─────────────────────────────────────────────────┐
  │  Destination       │ Target          │ Note     │
  ├─────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ local           │ VPC CIDR │
  │  0.0.0.0/0         │ nat-aaa111      │ AZ-a NAT│
  │  pl-12345 (S3)     │ vpce-s3-gw      │ S3 GW EP│
  └─────────────────────────────────────────────────┘
  Associated with: private-subnet-a ONLY

  Route Table: private-rt-az-b (AZ-b private subnets only)
  ┌─────────────────────────────────────────────────┐
  │  Destination       │ Target          │ Note     │
  ├─────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ local           │ VPC CIDR │
  │  0.0.0.0/0         │ nat-bbb222      │ AZ-b NAT│
  │  pl-12345 (S3)     │ vpce-s3-gw      │ S3 GW EP│
  └─────────────────────────────────────────────────┘
  Associated with: private-subnet-b ONLY

  Route Table: private-rt-az-c (AZ-c private subnets only)
  ┌─────────────────────────────────────────────────┐
  │  Destination       │ Target          │ Note     │
  ├─────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ local           │ VPC CIDR │
  │  0.0.0.0/0         │ nat-ccc333      │ AZ-c NAT│
  │  pl-12345 (S3)     │ vpce-s3-gw      │ S3 GW EP│
  └─────────────────────────────────────────────────┘
  Associated with: private-subnet-c ONLY

  Route Table: isolated-rt (shared across all AZs — no internet route)
  ┌─────────────────────────────────────────────────┐
  │  Destination       │ Target          │ Note     │
  ├─────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ local           │ VPC CIDR │
  │  pl-12345 (S3)     │ vpce-s3-gw      │ S3 GW EP│
  └─────────────────────────────────────────────────┘
  Associated with: isolated-subnet-a, isolated-subnet-b, isolated-subnet-c

  TOTAL: 5 route tables for a 3-tier, 3-AZ VPC (1 public + 3 private + 1 isolated)


  MODERN PATTERN (REGIONAL NAT): One route table per tier, period
  ────────────────────────────────────────────────────

  Route Table: private-rt (shared across ALL AZs)
  ┌─────────────────────────────────────────────────┐
  │  Destination       │ Target          │ Note     │
  ├─────────────────────────────────────────────────┤
  │  10.0.0.0/16       │ local           │ VPC CIDR │
  │  0.0.0.0/0         │ nat-regional-1  │ Regional │
  │  pl-12345 (S3)     │ vpce-s3-gw      │ S3 GW EP│
  └─────────────────────────────────────────────────┘
  Associated with: private-subnet-a, private-subnet-b, private-subnet-c

  TOTAL: 3 route tables (1 public + 1 private + 1 isolated)
  That is 2 fewer route tables and zero per-AZ NAT Gateway management.


  ROUTE MATCHING — LONGEST PREFIX MATCH
  ════════════════════════════════════════════════════════
  IMPORTANT: 0.0.0.0/0 matches EVERY destination — it is not a "fallback
  for unmatched traffic." When multiple routes match, AWS picks the one
  with the longest prefix (most bits). This is why more specific routes
  always win over the default route. Example:

  Routes in table:
  ├── 10.0.0.0/16       → local
  ├── 10.1.0.0/16       → pcx-peer123     (VPC peering to 10.1.0.0/16)
  ├── pl-12345 (S3)     → vpce-s3-gw      (S3 prefix list, e.g., 52.216.0.0/15)
  └── 0.0.0.0/0         → nat-regional-1

  Packet to 10.0.5.20:    matches 10.0.0.0/16 (local) — stays in VPC
  Packet to 10.1.3.50:    matches 10.1.0.0/16 (peering) — crosses to peer
  Packet to 52.216.56.0:  matches pl-12345 (S3 prefix) — goes through GW endpoint
  Packet to 8.8.8.8:      matches 0.0.0.0/0 (default) — goes through NAT

  The S3 prefix list route is MORE SPECIFIC than 0.0.0.0/0, so S3 traffic
  goes through the free gateway endpoint instead of the paid NAT Gateway.
  This is how gateway endpoints save you money even without changing code.
```

### Terraform: Complete Route Table Design

```hcl
# ──────────────────────────────────────────────────────────
# VPC and Internet Gateway
# ──────────────────────────────────────────────────────────

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "app-prod"
    Environment = "production"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "app-prod-igw"
  }
}

# ──────────────────────────────────────────────────────────
# Subnets: 3 tiers x 3 AZs = 9 subnets
# ──────────────────────────────────────────────────────────

locals {
  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]

  # Public subnets: 10.0.0.0/20, 10.0.16.0/20, 10.0.32.0/20
  public_cidrs = ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20"]

  # Private subnets: 10.0.48.0/20, 10.0.64.0/20, 10.0.80.0/20
  private_cidrs = ["10.0.48.0/20", "10.0.64.0/20", "10.0.80.0/20"]

  # Isolated subnets: 10.0.96.0/20, 10.0.112.0/20, 10.0.128.0/20
  isolated_cidrs = ["10.0.96.0/20", "10.0.112.0/20", "10.0.128.0/20"]
}

resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_cidrs[count.index]
  availability_zone = local.azs[count.index]

  map_public_ip_on_launch = true # Public subnet — auto-assign public IPs

  tags = {
    Name = "public-${local.azs[count.index]}"
    Tier = "public"
  }
}

resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = {
    Name = "private-${local.azs[count.index]}"
    Tier = "private"
  }
}

resource "aws_subnet" "isolated" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.isolated_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = {
    Name = "isolated-${local.azs[count.index]}"
    Tier = "isolated"
  }
}

# ──────────────────────────────────────────────────────────
# Route Tables: 3 tables (public, private, isolated)
# Using Regional NAT Gateway — one private RT for all AZs
# ──────────────────────────────────────────────────────────

# PUBLIC route table — route to IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-rt"
    Tier = "public"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# PRIVATE route table — route to Regional NAT Gateway
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.regional.id
  }

  tags = {
    Name = "private-rt"
    Tier = "private"
  }
}

resource "aws_route_table_association" "private" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

# ISOLATED route table — no internet route, only local + endpoints
resource "aws_route_table" "isolated" {
  vpc_id = aws_vpc.main.id

  # No routes added — only the implicit local route exists
  # Gateway endpoint routes are added automatically when the endpoint
  # is associated with this route table

  tags = {
    Name = "isolated-rt"
    Tier = "isolated"
  }
}

resource "aws_route_table_association" "isolated" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.isolated[count.index].id
  route_table_id = aws_route_table.isolated.id
}
```

---

## Part 4: NAT Gateway — Zonal vs Regional

### The Analogy

In the old model, every district (AZ) in your city had **its own guarded exit** (zonal NAT Gateway) stationed in the commercial district (public subnet). Each exit needed its own ID badge (Elastic IP), its own guard house (NAT Gateway resource), and a separate road map entry in that district's private neighborhoods. If the guard in District A called in sick, all residents of District A were stuck indoors even though Districts B and C were fine.

The new Regional NAT Gateway (November 2025) is like replacing all the district-level guards with a **city-wide exit authority**. You create one entity, and it automatically stations guards in whichever districts have residents. When a new district opens up, a guard appears within 15-20 minutes. When a district empties out, the guard is recalled. Crucially, the city-wide authority does not even need to own a guard house in the commercial district -- it manages its own facilities independently, meaning **public subnets are no longer required**.

### The Technical Reality

```
ZONAL vs REGIONAL NAT GATEWAY
=============================================================================

                          ZONAL (legacy)            REGIONAL (Nov 2025)
                          ──────────────            ───────────────────
  Count needed            1 per AZ                  1 per VPC
  Public subnet required  Yes (one per AZ)          No
  Elastic IPs             1 per NAT GW              Automatic (or manual)
  Max EIPs per AZ         8                         32
  Route tables            1 per AZ (private)        1 shared (private)
  AZ expansion            Manual (create new        Automatic (15-20 min)
                          NAT GW + update routes)
  AZ failure behavior     Only that AZ affected     Traffic reroutes cross-AZ
                                                    until expansion completes
  Cost (3-AZ VPC)         3 x $0.045/hr = $0.135/hr  $0.045/hr per AZ used
                          + 3 EIPs if idle           (same per-AZ rate, but
                                                     contracts when AZs unused)
  Terraform complexity    3 NAT GWs + 3 EIPs +     1 NAT GW + 1 route table
                          3 route tables

  OPERATING MODES (Regional only):
  ────────────────────────────────────────────────────
  AUTOMATIC MODE:
  ├── AWS allocates and manages Elastic IPs
  ├── AWS expands/contracts across AZs based on workload presence
  ├── Simplest to operate — recommended for most workloads
  └── You cannot predict which IPs will be used (harder to whitelist)

  MANUAL MODE:
  ├── You provide Elastic IPs and specify which AZs to cover
  ├── Full control over IP addresses (important for whitelisting)
  ├── You manage AZ presence — add/remove AZs explicitly
  └── Use when downstream services require known source IPs


  WHEN TO STILL USE ZONAL:
  ────────────────────────────────────────────────────
  ├── You need to guarantee zero cross-AZ data transfer charges
  │   (regional NAT may route cross-AZ during expansion)
  ├── You need deterministic per-AZ source IPs for whitelisting
  │   (manual mode regional NAT also supports this, but behavior
  │   is different during AZ expansion)
  └── Compliance requires AZ-isolated egress paths
```

### Terraform: Regional NAT Gateway

```hcl
# Regional NAT Gateway — automatic mode
# Requires terraform-provider-aws v6.24.0+ (December 2025)
# No subnet_id required — Regional NAT GWs are standalone resources.
resource "aws_nat_gateway" "regional" {
  connectivity_type = "public"
  availability_mode = "regional" # The key attribute — makes it regional

  # Automatic mode: omit allocation_id and AWS manages EIPs.
  # Manual mode: provide allocation_id + availability_zone_address blocks
  # to control which EIPs are used in which AZs.

  tags = {
    Name = "app-prod-regional-nat"
    Type = "regional"
  }
}

# Manual mode example (when you need predictable source IPs for whitelisting):
#
# resource "aws_nat_gateway" "regional_manual" {
#   connectivity_type = "public"
#   availability_mode = "regional"
#
#   availability_zone_address {
#     availability_zone = "us-east-1a"
#     allocation_id     = aws_eip.nat_az_a.id
#   }
#
#   availability_zone_address {
#     availability_zone = "us-east-1b"
#     allocation_id     = aws_eip.nat_az_b.id
#   }
#
#   availability_zone_address {
#     availability_zone = "us-east-1c"
#     allocation_id     = aws_eip.nat_az_c.id
#   }
# }

# With Regional NAT, a SINGLE route table serves ALL private subnets
# across ALL AZs (already shown in Part 3 above)
```

```
COST COMPARISON: 3-AZ VPC, 1 TB/MONTH EGRESS
=============================================================================

  ZONAL NAT (3 gateways):
  ├── 3 NAT GWs x $0.045/hr x 730 hrs     = $98.55/mo
  ├── 1 TB data processing x $0.045/GB      = $46.08/mo
  ├── 3 Elastic IPs (attached, no charge)    = $0.00/mo
  └── TOTAL                                  = $144.63/mo

  REGIONAL NAT (1 gateway, spanning 3 AZs):
  ├── $0.045/hr per AZ x 3 AZs x 730 hrs   = $98.55/mo (same as zonal!)
  ├── 1 TB data processing x $0.045/GB      = $46.08/mo
  ├── Cross-AZ data transfer (varies)        = ~$5-15/mo
  └── TOTAL                                  = ~$149-160/mo

  IMPORTANT: Regional NAT Gateway is NOT cheaper per hour when spanning
  the same number of AZs. The per-AZ rate is identical ($0.045/hr/AZ).

  WHERE REGIONAL NAT SAVES MONEY:
  ├── Dynamic AZ contraction — if workloads leave an AZ, regional NAT
  │   stops charging for that AZ; zonal NAT GWs keep running regardless
  ├── Fewer EIPs to manage (cost impact when EIPs are unattached)
  └── Reduced operational overhead (fewer resources = less management)

  THE REAL VALUE IS OPERATIONAL, NOT HOURLY COST:
  ├── 1 resource to manage instead of 3
  ├── 3 route tables instead of 5
  ├── Auto-expansion when you add AZs (no Terraform changes)
  └── No more "forgot to create NAT GW in the new AZ" outages
```

---

## Part 5: VPC Endpoints — Private Couriers and Free Highways

### The Analogy

Your city's private neighborhoods need supplies from the government warehouse (S3) and the post office (SQS, Secrets Manager, etc.). The default route is: leave through the guarded exit (NAT Gateway), drive on the public highway (internet), arrive at the government facility. This works, but you pay a toll (NAT data processing charges) and your packages travel on a public road (security risk).

VPC endpoints are **private alternatives**:

- **Gateway Endpoints** are like a **free private highway** built directly from your city to specific government warehouses (S3 and DynamoDB only). No toll, no public road, no bandwidth limit. The city's road map is updated to include a new route: "For packages going to the S3 warehouse, take the private highway instead of the toll road."

- **Interface Endpoints** are like **private telephone lines** installed inside your neighborhoods. Each line (ENI) has a local phone number (private IP) that connects directly to a specific government office. There is a monthly line rental ($0.01/hr per AZ) and a per-call charge ($0.01/GB), but it is cheaper than the NAT toll for high-volume services and works from peered VPCs and on-premises networks.

### The Technical Reality

```
GATEWAY vs INTERFACE ENDPOINTS — DECISION FRAMEWORK
=============================================================================

                          GATEWAY ENDPOINT          INTERFACE ENDPOINT
                          ────────────────          ──────────────────
  Supported services      S3, DynamoDB only         180+ AWS services
  Cost                    FREE                      $0.01/hr per AZ +
                                                    $0.01/GB processed
  How it works            Route table entry          ENI with private IP
                          (prefix list)             in your subnet
  Throughput limit        None                      10 Gbps per ENI
                                                    (bursts to 100 Gbps)
  DNS                     Uses public S3 DNS        Private DNS via
                          (route handles traffic)   Route 53 Resolver
  Access from peered VPC  No                        Yes (via private IP)
  Access from on-prem     No                        Yes (via DX/VPN)
  Security                VPC endpoint policy       VPC endpoint policy +
                                                    security groups on ENI
  VPCs supported          Only the VPC where        Same VPC + peered VPCs
                          it is created             + on-premises

  ENTERPRISE S3 STRATEGY (the common pattern):
  ────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  WITHIN THE VPC:                                               │
  │  Use GATEWAY endpoint (free, no limit)                         │
  │  ├── All route tables include the S3 prefix list route         │
  │  ├── S3 traffic stays on AWS backbone                          │
  │  └── Saves NAT data processing charges ($0.045/GB)             │
  │                                                                 │
  │  FROM PEERED VPCs / ON-PREMISES:                               │
  │  Use INTERFACE endpoint in a shared-services VPC               │
  │  ├── Create interface endpoint in shared-services VPC          │
  │  ├── On-prem resolves S3 DNS to the interface endpoint IP      │
  │  ├── Traffic flows: on-prem → DX/VPN → shared VPC → S3        │
  │  └── Avoids routing S3 traffic through the internet            │
  │                                                                 │
  └─────────────────────────────────────────────────────────────────┘


  COST EXAMPLE: 2 TB/month S3 traffic from private subnets
  ────────────────────────────────────────────────────

  Without endpoint (through NAT):
  ├── NAT data processing: 2,048 GB x $0.045 = $92.16/mo
  └── Plus NAT hourly charge shared across all traffic

  With gateway endpoint:
  └── $0.00 (free)

  ANNUAL SAVINGS: $1,105 per VPC just for S3 traffic
  This is why gateway endpoints for S3 should be in EVERY VPC, no exceptions.


  CRITICAL FOR ISOLATED SUBNETS:
  ────────────────────────────────────────────────────
  Isolated subnets (databases) have NO default route — they cannot reach
  ANY AWS service without an endpoint. Common scenarios where this matters:

  ├── RDS S3 integration (aws_s3 extension) — exporting/importing data
  ├── RDS snapshot exports to S3
  ├── RDS audit logs to CloudWatch → needs CloudWatch Logs Interface Endpoint
  ├── RDS IAM database authentication → needs STS Interface Endpoint
  └── RDS Enhanced Monitoring → needs Monitoring Interface Endpoint

  Add the S3 gateway endpoint to the isolated route table. This gives RDS
  a private path to S3 without breaking isolation — gateway endpoints are
  private paths, not internet paths. The security guarantee is maintained.
```

### Terraform: VPC Endpoints

```hcl
# ──────────────────────────────────────────────────────────
# Gateway Endpoint for S3 — FREE, add to EVERY route table
# ──────────────────────────────────────────────────────────

resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"

  vpc_endpoint_type = "Gateway"

  # Associate with ALL route tables — public, private, and isolated
  route_table_ids = [
    aws_route_table.public.id,
    aws_route_table.private.id,
    aws_route_table.isolated.id,
  ]

  # Endpoint policy — restrict which buckets can be accessed
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowAccessToSpecificBuckets"
        Effect    = "Allow"
        Principal = "*"
        Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [
          "arn:aws:s3:::app-prod-data-*",
          "arn:aws:s3:::app-prod-data-*/*",
          # Always include the ECR and ECS buckets for container pulls
          "arn:aws:s3:::prod-us-east-1-starport-layer-bucket/*",
        ]
      }
    ]
  })

  tags = {
    Name = "s3-gateway-endpoint"
  }
}

# Gateway Endpoint for DynamoDB — also FREE
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.dynamodb"

  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.public.id,
    aws_route_table.private.id,
    aws_route_table.isolated.id,
  ]

  tags = {
    Name = "dynamodb-gateway-endpoint"
  }
}

# ──────────────────────────────────────────────────────────
# Interface Endpoints — for services that need private access
# ──────────────────────────────────────────────────────────

# Security group for interface endpoints
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "vpc-endpoints-"
  vpc_id      = aws_vpc.main.id
  description = "Allow HTTPS from VPC to interface endpoints"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block] # Allow from entire VPC
    description = "HTTPS from VPC"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "vpc-endpoints-sg"
  }
}

# Common interface endpoints for ECS/Fargate workloads
#
# NOTE ON ECR: Pulling container images requires THREE endpoints:
#   1. ecr.api — for authentication and image metadata (DescribeImages, GetAuthorizationToken)
#   2. ecr.dkr — for the actual Docker image pulls
#   3. S3     — for image layer downloads (layers are stored in S3)
# The S3 gateway endpoint (free) handles #3, so it does double duty:
# serving your app's S3 traffic AND ECR image layer downloads.
locals {
  interface_endpoints = {
    # Container orchestration
    "ecr-api"          = "com.amazonaws.us-east-1.ecr.api"
    "ecr-dkr"          = "com.amazonaws.us-east-1.ecr.dkr"
    "ecs"              = "com.amazonaws.us-east-1.ecs"
    "ecs-agent"        = "com.amazonaws.us-east-1.ecs-agent"
    "ecs-telemetry"    = "com.amazonaws.us-east-1.ecs-telemetry"
    # Secrets and configuration
    "secretsmanager"   = "com.amazonaws.us-east-1.secretsmanager"
    "ssm"              = "com.amazonaws.us-east-1.ssm"
    # Logging and monitoring
    "logs"             = "com.amazonaws.us-east-1.logs"
    "monitoring"       = "com.amazonaws.us-east-1.monitoring"
    # Networking
    "ec2"              = "com.amazonaws.us-east-1.ec2"
  }
}

resource "aws_vpc_endpoint" "interface" {
  for_each = local.interface_endpoints

  vpc_id              = aws_vpc.main.id
  service_name        = each.value
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true # Override public DNS with private endpoint IPs

  # Place in private subnets — one ENI per AZ
  subnet_ids = aws_subnet.private[*].id

  security_group_ids = [aws_security_group.vpc_endpoints.id]

  tags = {
    Name = "${each.key}-endpoint"
  }
}

# COST NOTE: 10 interface endpoints x 3 AZs x $0.01/hr x 730 hrs
# = $219/mo. Compare this to NAT Gateway DATA PROCESSING charges
# ($0.045/GB) to determine if the endpoints pay for themselves.
# The NAT hourly charge is NOT saved — you still pay it for other traffic.
#
# Rule of thumb: if a SINGLE service generates > 500 GB/mo through NAT,
# an interface endpoint for THAT service saves money ($0.045 x 500 GB =
# $22.50/mo vs $7.20/mo x 3 AZs = $21.60/mo). For low-volume services,
# NAT is cheaper than the endpoint hourly charge — but security (keeping
# traffic off the public internet) may justify the cost regardless.
```

---

## Part 6: Multi-AZ Architecture — Building for Earthquakes

### The Analogy

A responsible city planner builds the same neighborhood layout in multiple earthquake zones. If Zone A is hit by a disaster, Zones B and C keep the city running. But this means you need the same amenities in each zone: shops (public subnets), residences (private subnets), and vaults (isolated subnets). You also need exit guards (NAT Gateways) in each zone so that a single guard's absence does not strand an entire zone.

### Complete Multi-AZ Architecture Diagram

```
PRODUCTION VPC — COMPLETE MULTI-AZ ARCHITECTURE
=============================================================================

                            ┌──────────────┐
                            │   INTERNET    │
                            └──────┬───────┘
                                   │
                            ┌──────┴───────┐
                            │     IGW      │
                            └──────┬───────┘
                                   │
  ┌────────────────────────────────┼────────────────────────────────────┐
  │                    VPC: 10.0.0.0/16                                 │
  │                                │                                    │
  │  ┌─────────────────────────────┼─────────────────────────────────┐ │
  │  │              PUBLIC TIER    │                                  │ │
  │  │                             │                                  │ │
  │  │  ┌──────────┐    ┌─────────┴──┐    ┌──────────┐              │ │
  │  │  │ AZ-a     │    │ AZ-b       │    │ AZ-c     │              │ │
  │  │  │10.0.0/20 │    │10.0.16/20  │    │10.0.32/20│              │ │
  │  │  │          │    │            │    │          │              │ │
  │  │  │ [ALB]    │    │ [ALB]      │    │ [ALB]    │              │ │
  │  │  └──────────┘    └────────────┘    └──────────┘              │ │
  │  └──────────────────────────────────────────────────────────────┘ │
  │                                                                    │
  │                    ┌──────────────────┐                            │
  │                    │  Regional NAT GW │ (auto-expands to all AZs) │
  │                    │  nat-regional-1  │                            │
  │                    └────────┬─────────┘                            │
  │                             │                                      │
  │  ┌──────────────────────────┼────────────────────────────────────┐ │
  │  │             PRIVATE TIER │                                    │ │
  │  │                          │   0.0.0.0/0 → nat-regional-1      │ │
  │  │  ┌──────────┐    ┌──────┴─────┐    ┌──────────┐             │ │
  │  │  │ AZ-a     │    │ AZ-b       │    │ AZ-c     │             │ │
  │  │  │10.0.48/20│    │10.0.64/20  │    │10.0.80/20│             │ │
  │  │  │          │    │            │    │          │             │ │
  │  │  │[ECS][EC2]│    │[ECS][EC2]  │    │[ECS][EC2]│             │ │
  │  │  └──────────┘    └────────────┘    └──────────┘             │ │
  │  └──────────────────────────────────────────────────────────────┘ │
  │                                                                    │
  │  ┌──────────────────────────────────────────────────────────────┐ │
  │  │             ISOLATED TIER (no internet route)                 │ │
  │  │                                                              │ │
  │  │  ┌──────────┐    ┌────────────┐    ┌──────────┐             │ │
  │  │  │ AZ-a     │    │ AZ-b       │    │ AZ-c     │             │ │
  │  │  │10.0.96/20│    │10.0.112/20 │    │10.0.128/20│            │ │
  │  │  │          │    │            │    │          │             │ │
  │  │  │[RDS pri] │    │[RDS stdby] │    │[ElCache] │             │ │
  │  │  └──────────┘    └────────────┘    └──────────┘             │ │
  │  └──────────────────────────────────────────────────────────────┘ │
  │                                                                    │
  │  VPC ENDPOINTS:                                                    │
  │  ├── Gateway: S3 (free) — all route tables                        │
  │  ├── Gateway: DynamoDB (free) — all route tables                  │
  │  ├── Interface: ECR, ECS, Secrets Manager, CloudWatch Logs        │
  │  └── Interface endpoints placed in private subnets (3 AZs)        │
  │                                                                    │
  └────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Well-Architected Best Practices for IP Planning

```
WELL-ARCHITECTED CHECKLIST — IP ADDRESS MANAGEMENT (REL02-BP05)
=============================================================================

  DO:
  ├── [1] Use IPAM to centrally manage all CIDR allocations
  │       → Prevents the "spreadsheet of CIDRs" anti-pattern
  │
  ├── [2] Plan CIDR ranges BEFORE creating VPCs
  │       → Retroactive changes require migrations or Private NAT GW
  │
  ├── [3] Leave gaps between environment allocations for growth
  │       → production=10.0.0.0/14, staging=10.4.0.0/14, dev=10.8.0.0/14
  │       → The gap between /14 blocks provides expansion room
  │
  ├── [4] Use 100.64.0.0/10 (CGNAT range) for EKS pod networking
  │       → Preserves RFC 1918 space for nodes; pods get massive range
  │
  ├── [5] Deploy gateway endpoints for S3 and DynamoDB in every VPC
  │       → Free, no performance impact, saves NAT processing charges
  │
  ├── [6] Monitor VPC IP utilization with IPAM or custom CloudWatch metrics
  │       → Running out of IPs in a subnet is a production-down event
  │
  ├── [7] Use Regional NAT Gateway for new deployments
  │       → Simpler, cheaper, auto-expands across AZs
  │
  └── [8] Document the CIDR allocation scheme in a living architecture doc
        → The scheme should be discoverable by any team member

  DO NOT:
  ├── [1] Use the same CIDR range on-premises and in AWS
  │       → Hybrid connectivity becomes impossible
  │
  ├── [2] Use 172.17.0.0/16 for VPCs
  │       → Conflicts with Docker default bridge network
  │
  ├── [3] Track CIDR allocations in spreadsheets
  │       → Inevitably leads to overlaps, stale data, human error
  │
  ├── [4] Allocate /16 to every VPC "just in case"
  │       → 10.0.0.0/8 only contains 256 /16 blocks — you run out fast
  │
  ├── [5] Skip isolated subnets for databases
  │       → Databases should never have a route to the internet, even
  │         through NAT — reduces attack surface and meets compliance
  │
  └── [6] Ignore the 5 reserved IPs per subnet in capacity planning
        → A /24 has 251 usable IPs, not 256; a /28 has 11, not 16
```

---

## Key Takeaways

### CIDR Planning
1. **Start with IPAM, not spreadsheets** -- AWS IPAM provides a hierarchical pool structure with allocation rules that physically prevent teams from requesting overlapping or oversized CIDRs; the four-tier hierarchy (top-level, regional, environment, VPC) scales to hundreds of VPCs
2. **Reserve 10.0.0.0/8 for AWS and a different RFC 1918 range for on-premises** -- This eliminates overlap risk entirely between cloud and data center; if on-prem already uses 10.x, plan secondary CIDRs carefully or use 100.64.0.0/10
3. **Leave gaps between environment allocations** -- Assign production at 10.0.0.0/14, staging at 10.4.0.0/14, development at 10.8.0.0/14; the gaps between /14 blocks leave room for future VPCs without reorganizing
4. **Avoid 172.17.0.0/16** -- It conflicts with Docker's default bridge network and SageMaker; this is a surprisingly common production gotcha

### Subnet Strategy
5. **Three tiers are the enterprise standard: public, private, isolated** -- Public for load balancers and NAT Gateways, private for application workloads, isolated for databases; the tier is defined entirely by the route table, not by any subnet property
6. **Size EKS subnets at /19 or larger** -- The VPC CNI assigns one IP per pod; a node with 100 pods needs 100 subnet IPs; a /24 (251 usable) fills up with just 2-3 nodes; underprovisioned subnets cause pod scheduling failures that are painful to fix in production
7. **Every subnet loses 5 IPs to AWS infrastructure** -- Network address, VPC router, DNS server, reserved, and broadcast; account for this in capacity planning, especially for small subnets

### Route Tables
8. **Longest prefix match determines routing** -- A gateway endpoint route for S3 (specific prefix list) always wins over the 0.0.0.0/0 NAT route, which is exactly how gateway endpoints save you money without code changes
9. **Regional NAT Gateway reduces route table count from 5 to 3 in a standard 3-tier, 3-AZ VPC** -- One public, one private (shared across all AZs), one isolated; this simplification eliminates an entire class of per-AZ configuration bugs

### NAT Gateway
10. **Use Regional NAT Gateway for new deployments** -- The per-AZ hourly rate is the same as zonal ($0.045/hr/AZ), but the operational simplification is massive: one resource instead of three, 3 route tables instead of 5, automatic AZ expansion, and no public subnet requirement; it also saves money when workloads dynamically contract to fewer AZs
11. **Use zonal NAT Gateways only when you need deterministic per-AZ egress IPs or must avoid cross-AZ data transfer charges** -- These are valid requirements for compliance-heavy or cost-sensitive workloads, but they are the exception

### VPC Endpoints
12. **Deploy S3 and DynamoDB gateway endpoints in every VPC, no exceptions** -- They are free, have no bandwidth limit, and save $0.045/GB in NAT data processing charges; for 2 TB/month of S3 traffic, that is $1,100/year per VPC
13. **Interface endpoints pay for themselves above ~500 GB/month per service** -- At $7.20/month per AZ for the hourly charge, an interface endpoint across 3 AZs costs $21.60/month; NAT processing on 500 GB costs $22.50/month, so the endpoint breaks even and also provides better security

### Enterprise Planning
14. **Use secondary CIDRs with 100.64.0.0/10 for EKS pod networking** -- This keeps node IPs in the primary RFC 1918 range (routable across peered VPCs) while giving pods a massive non-overlapping address space
15. **A /16 VPC comfortably holds 9 subnets at /20 with 7 blocks of /20 reserved for growth** -- This gives you 3 tiers across 3 AZs with 4,091 usable IPs per subnet, and room for future tiers (TGW attachment subnets, pod subnets, etc.)
16. **Plan your CIDR scheme before creating any VPCs** -- Retroactively fixing overlapping ranges requires either VPC migration, Private NAT Gateway (for connectivity between overlapping VPCs), or AWS PrivateLink; all three are significantly more complex than planning ahead

---

## Further Reading

- [VPC CIDR Blocks -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)
- [Building a Scalable and Secure Multi-VPC AWS Network Infrastructure -- AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html)
- [Amazon VPC IPAM Best Practices -- AWS Blog](https://aws.amazon.com/blogs/networking-and-content-delivery/amazon-vpc-ip-address-manager-best-practices/)
- [Regional NAT Gateways -- AWS Docs](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html)
- [Choosing Your VPC Endpoint Strategy for S3 -- AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/choosing-your-vpc-endpoint-strategy-for-amazon-s3/)
- [Well-Architected: Enforce Non-Overlapping IP Ranges (REL02-BP05)](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_network_topology_non_overlap_ip.html)

---

*Written on February 19, 2026*
