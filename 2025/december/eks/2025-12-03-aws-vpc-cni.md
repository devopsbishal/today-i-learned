# AWS VPC CNI - The Phone Extension System

> Understanding how pods get IP addresses in EKS using the VPC CNI plugin - continuing the managed office campus analogy.

---

## TL;DR

| VPC CNI Concept | Office Campus Analogy |
|-----------------|----------------------|
| VPC CNI | The phone extension system for the campus |
| VPC Subnet | Building with a pool of phone extension numbers |
| Pod IP Address | Phone extension number assigned to a desk |
| ENI (Elastic Network Interface) | Phone panel on room wall (limited slots) |
| Primary IP on ENI | Room's main extension (used by room itself) |
| Secondary IPs on ENI | Additional extensions for desks (pods) |
| Pod-to-Pod communication | Direct dial between extensions (no switchboard) |
| Pod limit per node | Limited phone extensions available per room |
| Prefix Delegation | Getting a block of 16 extensions per panel slot |
| Subnet exhaustion | Building runs out of extension numbers |
| Custom Networking | Desks use extensions from a different building |
| Secondary CIDR | Adding new buildings with fresh extension numbers |

---

## The Big Picture

In traditional Kubernetes, pods get IPs from a **virtual overlay network** - like a separate phone system that needs translation to talk to the main building network.

With **VPC CNI**, pods get IPs directly from your **VPC subnet** - like every desk getting a real extension from the building's phone pool. No translation needed!

```
TRADITIONAL CNI (Overlay):
  Desk (Pod) ──► Virtual Phone System ──► Translation ──► Building Phone System
  
VPC CNI:
  Desk (Pod) ──► Building Phone System (Direct!)
```

---

## Why This Matters

Since every desk (pod) gets a **real extension from the building's pool**:

| Scenario | With VPC CNI |
|----------|--------------|
| Desk → Desk (same room) | Direct dial ✅ |
| Desk → Desk (different room) | Direct dial ✅ |
| Desk → Desk (different building) | Direct dial ✅ |
| Desk → RDS Database | Direct dial (same campus!) ✅ |
| Desk → ElastiCache | Direct dial (same campus!) ✅ |
| Security Groups on Desks | Possible! ✅ |

**No translation, no overlay = simpler networking, lower latency, easier debugging.**

---

## How Extensions Get Assigned

### The Phone Panel (ENI)

Each room (node) has **phone panels (ENIs)** on the wall. Each panel has limited slots for extensions.

```
┌─────────────────────────────────────────────────────────────────┐
│                         ROOM (Node)                             │
│                                                                 │
│   Phone Panel 1 (Primary ENI)      Phone Panel 2 (Secondary ENI)│
│   ┌─────────────────────────┐      ┌─────────────────────────┐ │
│   │ Main Ext: 10.0.1.5      │      │ Main Ext: 10.0.1.20     │ │
│   │ (Used by ROOM itself) 🚪│      │ (Available for desk) 🪑 │ │
│   │                         │      │                         │ │
│   │ Secondary Extensions:   │      │ Secondary Extensions:   │ │
│   │ • 10.0.1.6 → Desk 🪑   │      │ • 10.0.1.21 → Desk 🪑  │ │
│   │ • 10.0.1.7 → Desk 🪑   │      │ • 10.0.1.22 → Desk 🪑  │ │
│   │ • 10.0.1.8 → Desk 🪑   │      │ • 10.0.1.23 → Desk 🪑  │ │
│   └─────────────────────────┘      └─────────────────────────┘ │
│                                                                 │
│   Room Manager (kubelet) manages extension assignments         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The Rules

1. **Room uses one extension** - Primary IP of primary ENI goes to the node itself
2. **Desks share remaining extensions** - Secondary IPs go to pods
3. **Limited panels per room** - Instance type determines max ENIs
4. **Limited slots per panel** - Instance type determines IPs per ENI

---

## The Desk Limit Problem

### Small Room vs Large Room

Different room sizes (instance types) have different phone panel capacities:

| Room Size | Max Panels (ENIs) | Slots/Panel (IPs) | Total Slots | Desks Possible |
|-----------|-------------------|-------------------|-------------|----------------|
| t3.small | 3 | 4 | 12 | **9** (12 - 3 ENI primary IPs) |
| t3.large | 3 | 12 | 36 | **33** (36 - 3) |
| m5.large | 3 | 10 | 30 | **27** (30 - 3) |
| m5.xlarge | 4 | 15 | 60 | **56** (60 - 4) |

> **Formula**: `(maxENIs × maxIPsPerENI) - maxENIs`. Each ENI's primary IP is reserved for the node itself, so you subtract one per ENI, not just one total.

### The Problem

```
┌─────────────────────────────────────────────────────────────┐
│                    t3.small ROOM                            │
│                                                             │
│   CPU:  ████████░░░░░░░░░░░░ 40% used                      │
│   RAM:  ██████░░░░░░░░░░░░░░ 30% used                      │
│   Extensions: ████████████████████ 100% used (9/9)         │
│                                                             │
│   ❌ Cannot add more desks!                                 │
│   💸 Wasting 60% CPU and 70% RAM                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**You have unused compute but no extensions = No more desks!**

---

## Solution 1: Prefix Delegation

Instead of getting **one extension per slot**, get a **block of 16 extensions** per slot!

### Without Prefix Delegation

```
Phone Panel:
┌─────────────────────────────────────┐
│ Slot 1: 10.0.1.5  → 1 extension    │
│ Slot 2: 10.0.1.6  → 1 extension    │
│ Slot 3: 10.0.1.7  → 1 extension    │
│ Slot 4: 10.0.1.8  → 1 extension    │
│                                     │
│ Total: 4 slots = 4 extensions       │
└─────────────────────────────────────┘
```

### With Prefix Delegation

```
Phone Panel:
┌─────────────────────────────────────────────────────────────┐
│ Slot 1: 10.0.1.0/28  → 16 extensions (10.0.1.0 - 10.0.1.15)│
│ Slot 2: 10.0.1.16/28 → 16 extensions (10.0.1.16 - 10.0.1.31)│
│ Slot 3: 10.0.1.32/28 → 16 extensions (10.0.1.32 - 10.0.1.47)│
│ Slot 4: 10.0.1.48/28 → 16 extensions (10.0.1.48 - 10.0.1.63)│
│                                                             │
│ Total: 4 slots × 16 = 64 extensions!                       │
└─────────────────────────────────────────────────────────────┘
```

### The Math (t3.small)

| Mode | Calculation | Max Desks |
|------|-------------|-----------|
| **Default** | (3 ENIs × 4 IPs) - 3 | **9 desks** |
| **Prefix Delegation** | (3 ENIs × 4 slots × 16 IPs) - 3 | **~189 desks** |

> **Note**: With prefix delegation, `max-pods` and CPU/memory will likely be the actual bottleneck before IP addresses.

**10x more desks on the same room!** 🚀

### Trade-off

| Benefit | Drawback |
|---------|----------|
| Way more desks per room | Uses extensions in blocks of 16 |
| Better resource utilization | Even if desk needs 1, you "reserve" 16 |
| | Building can exhaust faster if many rooms |

---

## Solution 2: Custom Networking

What if a building runs out of extensions? Use extensions from a **different building**!

```
┌─────────────────────────────────────────────────────────────────┐
│                         YOUR CAMPUS (VPC)                       │
│                                                                 │
│   Building A (Subnet: 10.0.1.0/24)    Building B (Subnet: 10.0.2.0/24)
│   ┌─────────────────────────────┐    ┌─────────────────────────────┐
│   │                             │    │                             │
│   │   Room 1 (Node)             │    │   Extension Pool:           │
│   │   ┌─────────────────────┐   │    │   10.0.2.1 - 10.0.2.254    │
│   │   │ Room Ext: 10.0.1.5  │   │    │   (Available for desks!)   │
│   │   │                     │   │    │                             │
│   │   │ Desks use extensions│───┼────┼─► 10.0.2.10 → Desk        │
│   │   │ from Building B!    │   │    │   10.0.2.11 → Desk        │
│   │   └─────────────────────┘   │    │   10.0.2.12 → Desk        │
│   │                             │    │                             │
│   └─────────────────────────────┘    └─────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Nodes stay in original building, but pods get extensions from a different building.**

---

## Solution 3: Secondary CIDRs

If the whole campus is running out of extensions, **add new buildings**!

```
┌─────────────────────────────────────────────────────────────────┐
│                         YOUR CAMPUS (VPC)                       │
│                                                                 │
│   Original Buildings: 10.0.0.0/16 (65,536 extensions)          │
│                     │                                           │
│                     │  Running out!                             │
│                     ▼                                           │
│   New Buildings: 100.64.0.0/16 (65,536 more extensions!)       │
│                                                                 │
│   Total available: 131,072 extensions!                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Note**: Secondary CIDRs often use `100.64.0.0/10` (RFC 6598 range) to avoid conflicts.

---

## Communication Flow

Since every desk has a **real campus extension**, communication is direct:

```
┌─────────────────────────────────────────────────────────────────┐
│                         YOUR CAMPUS (VPC)                       │
│                                                                 │
│   Building A                         Building B                 │
│   ┌─────────────────────┐           ┌─────────────────────┐    │
│   │ Room 1              │           │ Room 3              │    │
│   │ ┌─────────────────┐ │           │ ┌─────────────────┐ │    │
│   │ │Desk: 10.0.1.10  │─┼───────────┼─│Desk: 10.0.2.20  │ │    │
│   │ └─────────────────┘ │  Direct   │ └─────────────────┘ │    │
│   │                     │  Dial!    │                     │    │
│   └─────────────────────┘           └─────────────────────┘    │
│                                                                 │
│                              │                                  │
│                              ▼                                  │
│                    ┌─────────────────┐                         │
│                    │ RDS Database    │                         │
│                    │ (10.0.3.50)     │ ← Also direct dial!     │
│                    └─────────────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

No translation! No overlay! Just native campus routing.
```

---

## Solutions Summary

| Problem | Solution | How It Works |
|---------|----------|--------------|
| **Room hits extension limit** | Prefix Delegation | Get 16 extensions per slot instead of 1 |
| **Building runs out of extensions** | Custom Networking | Use extensions from a different building |
| **Campus runs out of extensions** | Secondary CIDRs | Add new buildings with fresh extension pools |

---

## Choosing the Right Solution

```
Problem: Room can't fit more desks (but has compute)
└── Solution: Prefix Delegation (more extensions per room)

Problem: Building subnet is exhausted
├── Option A: Custom Networking (pods use different subnet)
└── Option B: Secondary CIDRs (add more subnets to VPC)

Problem: Need security isolation between pod groups
└── Solution: Custom Networking (separate subnets for different workloads)
```

---

## VPC CNI vs Other CNIs

| Aspect | VPC CNI | Overlay CNI (Calico, Flannel) |
|--------|---------|-------------------------------|
| Pod IPs from | VPC Subnet (real IPs) | Virtual overlay network |
| Talk to AWS resources | Direct ✅ | Needs NAT/translation |
| Security Groups on Pods | Possible ✅ | Not possible |
| Network debugging | Easier (real IPs) | Harder (overlay IPs) |
| IP consumption | Uses VPC IPs | Doesn't use VPC IPs |
| Best for | AWS-native workloads | Multi-cloud, IP conservation |

---

## Key Takeaways

1. **VPC CNI gives pods real VPC IPs** - Direct communication with all VPC resources
2. **ENIs are phone panels** - Each node has limited panels with limited slots
3. **Instance type matters** - Larger instances = more ENIs = more pods
4. **Prefix Delegation = 10x more pods** - Get blocks of 16 IPs per slot
5. **Custom Networking** - Pods can use IPs from different subnets than nodes
6. **Plan your IP space** - VPC CNI consumes real IPs, plan subnets accordingly
7. **No overlay = simpler debugging** - Pod IPs are real, traceable VPC IPs

---

## Related Reading

- [AWS VPC - The Compound Analogy](../november/2025-11-30-aws-vpc.md)
- [AWS EKS - The Managed Office Building Analogy](./2025-12-02-aws-eks.md)

---

*Written on December 3, 2025*
