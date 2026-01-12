# EKS Autoscaling - The Staffing & Workspace Management

> Understanding how to automatically scale pods and nodes in EKS using HPA, VPA, Cluster Autoscaler, and Karpenter - continuing the office campus analogy.

---

## TL;DR

| EKS Autoscaling Concept | Office Campus Analogy |
|------------------------|----------------------|
| Horizontal Pod Autoscaler (HPA) | Hiring more workers when workload increases |
| Vertical Pod Autoscaler (VPA) | Giving workers better equipment (faster laptop, more monitors) |
| Cluster Autoscaler | Opening new rooms from predefined floor plans |
| Karpenter | Custom-building the perfect room for each team's needs |
| Pod scaling | Team size adjustment |
| Node scaling | Building/room capacity adjustment |
| CPU/Memory metrics | Workload per worker |
| Pending pods | Workers waiting for a room to work in |
| Underutilized nodes | Empty rooms wasting electricity |

---

## The Problem: Workload Fluctuates

Your office campus (cluster) faces two challenges:

```
CHALLENGE 1: TEAM SIZE (Pod Scaling)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  9 AM:  ┌──┐┌──┐┌──┐           Normal workload            │
│         │👤││👤││👤│           3 workers handling it       │
│         └──┘└──┘└──┘                                       │
│                                                             │
│  1 PM:  ┌──┐┌──┐┌──┐┌──┐┌──┐  Black Friday sale!         │
│         │👤││👤││👤││👤││👤│  Need 5 workers!            │
│         └──┘└──┘└──┘└──┘└──┘                               │
│                                                             │
│  6 PM:  ┌──┐                    Slow evening               │
│         │👤│                    1 worker enough            │
│         └──┘                                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘

CHALLENGE 2: OFFICE SPACE (Node Scaling)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Normal:  [Room A] [Room B]         2 rooms enough         │
│           8 workers                                         │
│                                                             │
│  Peak:    [Room A] [Room B] [🚧]    Need Room C!          │
│           15 workers ───────►  Workers waiting for space!  │
│                                                             │
│  Quiet:   [Room A] [Empty]          Room B wasting $      │
│           3 workers                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Horizontal Pod Autoscaler (HPA) - Hiring More Workers

HPA automatically **adds or removes workers (pods)** based on workload.

```
┌─────────────────────────────────────────────────────────────┐
│                   CUSTOMER SERVICE TEAM                     │
│                                                             │
│  Normal Load (100 tickets/hour)                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Worker 1     Worker 2     Worker 3                │    │
│  │  ┌──┐         ┌──┐         ┌──┐                   │    │
│  │  │👤│ 30%     │👤│ 35%     │👤│ 30%  CPU usage    │    │
│  │  └──┘         └──┘         └──┘                   │    │
│  │                                                    │    │
│  │  Everyone handling their load comfortably ✅      │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  Black Friday (500 tickets/hour!)                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Worker 1     Worker 2     Worker 3                │    │
│  │  ┌──┐         ┌──┐         ┌──┐                   │    │
│  │  │👤│ 90%     │👤│ 95%     │👤│ 92%  CPU spiking! │    │
│  │  └──┘         └──┘         └──┘                   │    │
│  │                                                    │    │
│  │  HPA: "CPU > 80%! Hire more workers!"            │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  HPA Scales to 6 Workers                                   │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Worker 1  Worker 2  Worker 3  Worker 4  Worker 5 │    │
│  │  ┌──┐      ┌──┐      ┌──┐      ┌──┐      ┌──┐    │    │
│  │  │👤│ 45%  │👤│ 50%  │👤│ 48%  │👤│ 52%  │👤│ 47%│    │
│  │  └──┘      └──┘      └──┘      └──┘      └──┘    │    │
│  │                                           Worker 6│    │
│  │                                           ┌──┐    │    │
│  │                                           │👤│ 43%│    │
│  │                                           └──┘    │    │
│  │  Load distributed! Everyone comfortable again ✅  │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### How HPA Works

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: customer-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: customer-service
  minReplicas: 3              # Never go below 3 workers
  maxReplicas: 10             # Never exceed 10 workers
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Keep CPU around 70%
```

**HPA Decision Logic:**

```
Current CPU: 90%
Target CPU:  70%
Current workers: 3

Calculation:
  Desired workers = 3 × (90 / 70) = 3.86 → Round up to 4

Action: Scale from 3 to 4 workers
```

---

## Vertical Pod Autoscaler (VPA) - Better Equipment

VPA gives workers **better equipment** instead of hiring more people.

```
┌─────────────────────────────────────────────────────────────┐
│                   DATABASE ADMIN WORKER                     │
│                                                             │
│  Day 1: Worker with basic laptop                           │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Database Admin                                    │    │
│  │  ┌──────────────────────────┐                     │    │
│  │  │ 👤 Basic Laptop          │                     │    │
│  │  │ CPU: 2 cores (100% used!)│  Struggling! 😓     │    │
│  │  │ RAM: 4GB (100% used!)    │                     │    │
│  │  └──────────────────────────┘                     │    │
│  │                                                    │    │
│  │  VPA watching: "This worker needs better tools!"  │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  VPA Action: Upgrade equipment (restart worker)            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Database Admin (same person, new equipment)      │    │
│  │  ┌──────────────────────────┐                     │    │
│  │  │ 👤 Powerful Workstation  │                     │    │
│  │  │ CPU: 8 cores (40% used)  │  Working great! ✅  │    │
│  │  │ RAM: 16GB (50% used)     │                     │    │
│  │  └──────────────────────────┘                     │    │
│  │                                                    │    │
│  │  Better resources = Better performance!           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### VPA Update Modes

| Mode | What Happens | Use When |
|------|-------------|----------|
| **Off** | Just recommends, doesn't change anything. Recommendations stored in VPA `.status` | You want suggestions only, manual control |
| **Initial** | Sets resources **only** when pods are first created. No updates to running pods | Use with HPA, or when restarts are expensive |
| **Recreate** | Evicts pods when recommendations differ significantly, replacement gets new resources | Production workloads that can tolerate restarts |
| **InPlaceOrRecreate** | Updates resources **without restarting** pod when possible, falls back to eviction if needed | Modern clusters with in-place resize support (least disruptive) |
| **Auto** ⚠️ | **DEPRECATED** (v1.4.0+) - Now just an alias for Recreate mode | Use `Recreate` or `InPlaceOrRecreate` instead |

**Key Differences:**
- **Off**: Read-only recommendations, zero automation
- **Initial**: One-time setup at pod creation
- **Recreate**: Active management via pod eviction (brief downtime per eviction)
- **InPlaceOrRecreate**: Best of both worlds - resize without restart when possible (requires Kubernetes 1.27+ with InPlacePodVerticalScaling feature gate)

---

## HPA vs VPA - When to Use Which

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  CUSTOMER SERVICE (Stateless)                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Peak load = more tickets                         │    │
│  │  Solution: Hire more workers!                     │    │
│  │  Use: HPA ✅                                      │    │
│  │                                                    │    │
│  │  Worker 1  Worker 2  Worker 3  Worker 4           │    │
│  │  ┌──┐      ┌──┐      ┌──┐      ┌──┐              │    │
│  │  │👤│      │👤│      │👤│      │👤│              │    │
│  │  └──┘      └──┘      └──┘      └──┘              │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  DATABASE (Stateful)                                       │
│  ┌────────────────────────────────────────────────────┐    │
│  │  1 Master DB (all writes go here)                 │    │
│  │  More workers won't help!                         │    │
│  │  Solution: Give master better equipment!          │    │
│  │  Use: VPA ✅                                      │    │
│  │                                                    │    │
│  │  ┌──────────────────────────┐                     │    │
│  │  │ 👤 Master DB             │                     │    │
│  │  │ More CPU/RAM             │                     │    │
│  │  └──────────────────────────┘                     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Can You Use Both?

**⚠️ Careful!** HPA and VPA can conflict if they watch the same metrics (CPU/memory).

```
❌ CONFLICT:
┌─────────────────────────────────────────────────────────────┐
│  Pod CPU spikes to 80%                                     │
│         │                                                   │
│         ├──────────────┬─────────────────┐                 │
│         ▼              ▼                 ▼                 │
│                                                             │
│  HPA:              VPA:                                    │
│  "Add workers!"    "Better equipment!"                     │
│         │              │                                    │
│         ▼              ▼                                    │
│  Creates pod      Restarts pod                            │
│                                                             │
│  They fight each other! 🔄                                │
└─────────────────────────────────────────────────────────────┘

✅ SOLUTION 1: Different Metrics
   HPA: Scale on requests/sec (custom metric)
   VPA: Optimize CPU/memory requests

✅ SOLUTION 2: VPA in "Initial" mode
   VPA: Sets resources when pod created
   HPA: Scales pod count
   No conflict!
```

---

## Cluster Autoscaler - Predefined Floor Plans

When you need **more office space**, open new rooms from **predefined templates**.

```
┌─────────────────────────────────────────────────────────────┐
│                   CLUSTER AUTOSCALER                        │
│                                                             │
│  Setup: Admin creates predefined room templates (ASGs)     │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Template A: Small Room (2 desks, basic equipment)│    │
│  │  Template B: Medium Room (4 desks, good equipment)│    │
│  │  Template C: Large Room (8 desks, premium equipment)   │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  Scenario: 5 new workers hired, need space!                │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Current Rooms:                                    │    │
│  │  [Room A - Full]  [Room B - Full]                 │    │
│  │                                                    │    │
│  │  Workers waiting: 👤👤👤👤👤 (can't schedule!)   │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  Cluster Autoscaler sees pending workers                   │
│  "Need 5 desks... Template B has 4 desks... Pick that!"   │
│                          │                                  │
│                          ▼                                  │
│  Tells ASG: "Build Room B"                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │  ASG launches EC2 instance                         │    │
│  │         ↓                                           │    │
│  │  Instance joins cluster as node (5-10 minutes)     │    │
│  │         ↓                                           │    │
│  │  [Room A - Full]  [Room B - Full]  [Room B - New] │    │
│  │                                     👤👤👤👤         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  ⚠️  Problem: Picked Template B (4 desks)                  │
│      Still 1 worker waiting! Need another room...          │
│      Also wasting 3 desks in new room (only using 1)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### How Cluster Autoscaler Works

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. HPA scales deployment: 3 → 8 pods                      │
│                        │                                    │
│                        ▼                                    │
│  2. 5 pods are Pending (insufficient CPU/memory)           │
│                        │                                    │
│                        ▼                                    │
│  3. Cluster Autoscaler sees pending pods                   │
│     "Which ASG can fit these pods?"                        │
│                        │                                    │
│                        ▼                                    │
│  4. Picks ASG with matching instance type                  │
│     Increases DesiredCapacity: 2 → 3                       │
│                        │                                    │
│                        ▼                                    │
│  5. ASG launches EC2 instance                              │
│     • Bootstraps node                                      │
│     • Joins cluster                                        │
│     Time: 5-10 minutes ⏱️                                  │
│                        │                                    │
│                        ▼                                    │
│  6. Pods get scheduled! ✅                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Scale Down

```
When to remove rooms:
┌─────────────────────────────────────────────────────────────┐
│  Cluster Autoscaler checks every 10 minutes:              │
│                                                             │
│  ✅ Node CPU + Memory < 50% (configurable)                 │
│  ✅ All pods can move to other nodes                       │
│  ✅ No local storage preventing eviction                   │
│  ✅ No system pods blocking removal                        │
│  ✅ Pod Disruption Budgets respected                       │
│  ✅ Been underutilized for 10+ minutes                     │
│                                                             │
│  If ALL conditions met → Remove node (close room)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Karpenter - Custom-Built Rooms

Instead of picking from **predefined templates**, Karpenter **custom-builds** the perfect room!

```
┌─────────────────────────────────────────────────────────────┐
│                        KARPENTER                            │
│                                                             │
│  Scenario: Need room for 3 workers with specific needs     │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Worker 1: Needs high-end graphics workstation    │    │
│  │  Worker 2: Needs lots of desk space               │    │
│  │  Worker 3: Budget-friendly desk is fine           │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  Karpenter analyzes ALL available options:                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Option A: 2 small desks, basic    = Too small    │    │
│  │  Option B: 4 medium desks, good    = Wasted space │    │
│  │  Option C: 3 premium desks, perfect = Too expensive   │
│  │  Option D: 3 mixed desks, optimized = PERFECT! ✅  │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  Karpenter calls EC2 RunInstances directly                 │
│  (No ASG! Direct to AWS!)                                  │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Instance launches: c5.xlarge                      │    │
│  │         ↓                                           │    │
│  │  Joins cluster (1-2 minutes) ⚡                    │    │
│  │         ↓                                           │    │
│  │  [Custom Room D]                                   │    │
│  │   Graphics desk 🖥️  Large desk 📊  Basic desk 💻  │    │
│  │       👤              👤            👤              │    │
│  │                                                    │    │
│  │  Perfect fit! No wasted space! Optimal cost! ✅   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Karpenter's Intelligence

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        # Karpenter can pick from any of these
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m5.large
            - m5.xlarge
            - c5.large
            - c5.xlarge
            - r5.large
      # Karpenter chooses based on:
      # • Pod requirements
      # • Cost (prefers cheaper)
      # • Availability
```

### How Karpenter Chooses

```
Pod needs: 3 vCPU, 10GB RAM

Karpenter evaluates 600+ instance types:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  m5.large:     2 vCPU, 8GB   → Too small (both) ❌         │
│  c5.xlarge:    4 vCPU, 8GB   → Not enough RAM ❌           │
│  m5.xlarge:    4 vCPU, 16GB  → Perfect fit! ($0.192/hr) ✅ │
│  r5.xlarge:    4 vCPU, 32GB  → Too much RAM ($0.252/hr)   │
│  m5.2xlarge:   8 vCPU, 32GB  → Overkill ($0.384/hr)       │
│                                                             │
│  Winner: m5.xlarge (cheapest that meets requirements)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Karpenter's Superpowers

### 1. Consolidation - Automatic Room Optimization

```
WITHOUT CONSOLIDATION (3 rooms, underutilized):
┌─────────────────────────────────────────────────────────────┐
│  Room A          Room B          Room C                     │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐              │
│  │ 👤 20%    │  │ 👤 15%    │  │ 👤 25%    │              │
│  │ Wasted    │  │ Wasted    │  │ Wasted    │              │
│  └───────────┘  └───────────┘  └───────────┘              │
│                                                             │
│  Cost: $$$  (3 rooms running)                              │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼ Karpenter consolidates
┌─────────────────────────────────────────────────────────────┐
│  Room D (larger, better utilized)                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 👤 👤 👤                                            │    │
│  │ 60% utilized                                       │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  Cost: $  (1 room running)                                 │
│  Savings: 66%! 💰                                          │
└─────────────────────────────────────────────────────────────┘

Karpenter:
1. Identifies underutilized nodes
2. Moves pods to fewer, better-sized nodes
3. Terminates unused nodes
4. Continuously optimizes!
```

### 2. Spot + On-Demand Mix

```
┌─────────────────────────────────────────────────────────────┐
│  Karpenter intelligently mixes cheap and reliable:         │
│                                                             │
│  Critical workers (databases, stateful):                   │
│  └─► On-Demand rooms (guaranteed, won't disappear)        │
│                                                             │
│  Batch jobs, dev workloads:                                │
│  └─► Spot rooms (70% cheaper! Can be reclaimed)           │
│                                                             │
│  If spot unavailable:                                      │
│  └─► Automatically falls back to on-demand                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. Speed Comparison

| Cluster Autoscaler | Karpenter |
|-------------------|-----------|
| Check pods → Pick ASG → Call ASG API → ASG launches instance → Instance joins cluster | Check pods → Pick instance type → EC2 RunInstances → Instance joins cluster |
| **5-10 minutes** | **1-2 minutes** ⚡ |

---

## The Complete Autoscaling Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLETE FLOW                                │
│                                                                 │
│  1. Traffic spike!                                              │
│     Website gets 10x requests                                   │
│                        │                                        │
│                        ▼                                        │
│  2. HPA detects high CPU                                        │
│     "Workers at 95% CPU! Need more!"                           │
│                        │                                        │
│                        ▼                                        │
│  3. HPA scales: 3 → 10 pods                                    │
│     Creates 7 new pods                                          │
│                        │                                        │
│                        ▼                                        │
│  4. Scheduler tries to place pods                              │
│     5 pods scheduled ✅                                        │
│     2 pods pending ⏸️ (no room!)                               │
│                        │                                        │
│                        ▼                                        │
│  5. Karpenter sees pending pods                                │
│     "Need room for 2 workers with 2 vCPU, 4GB each"           │
│                        │                                        │
│                        ▼                                        │
│  6. Karpenter provisions perfect node                          │
│     c5.xlarge (4 vCPU, 8GB) - fits both! ⚡                   │
│     Time: 90 seconds                                            │
│                        │                                        │
│                        ▼                                        │
│  7. Pods scheduled on new node ✅                              │
│     All 10 pods running!                                        │
│                        │                                        │
│                        │ (2 hours later, traffic normalizes)    │
│                        ▼                                        │
│  8. HPA scales down: 10 → 3 pods                               │
│     Removes 7 pods                                              │
│                        │                                        │
│                        ▼                                        │
│  9. Karpenter sees underutilized node                          │
│     "This room is 15% used for 10+ minutes"                    │
│                        │                                        │
│                        ▼                                        │
│  10. Karpenter drains and removes node                         │
│      Moves remaining pods, terminates node 💰                  │
│      Back to baseline!                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Decision Guide

### HPA vs VPA

```
Stateless application (API, web server)?
└─► Use HPA ✅
    More instances handle more requests

Stateful application (database, cache)?
└─► Use VPA ✅
    Better resources for single instance

Can tolerate pod restarts?
├─► YES → VPA Auto mode possible
└─► NO  → VPA Initial/Off mode only

Want both?
└─► Use HPA + VPA (Initial mode) ✅
    VPA sets initial resources, HPA scales count
```

---

### Cluster Autoscaler vs Karpenter

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Choose CLUSTER AUTOSCALER when:                           │
│  ✅ Existing ASG infrastructure                            │
│  ✅ Multi-cloud (GKE, AKS) - not AWS-only                  │
│  ✅ Self-managed Kubernetes on EC2                         │
│  ✅ Strict instance type requirements                      │
│  ✅ Conservative, battle-tested approach                   │
│                                                             │
│  Choose KARPENTER when:                                    │
│  ✅ New EKS cluster (AWS recommended)                      │
│  ✅ Dynamic/unpredictable workloads                        │
│  ✅ Cost optimization is priority                          │
│  ✅ Heavy spot instance usage                              │
│  ✅ Need fast scaling (1-2 min vs 5-10 min)               │
│  ✅ Want simplicity (no ASG management)                    │
│  ✅ Diverse pod requirements (CPU/mem/GPU mix)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Real-World Example: E-Commerce Site

```
┌─────────────────────────────────────────────────────────────────┐
│                   E-COMMERCE PLATFORM                           │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    WEB TIER                               │ │
│  │  Deployment: web-frontend                                 │ │
│  │  HPA: Min 5, Max 50                                       │ │
│  │  Trigger: CPU > 70% OR requests/sec > 1000               │ │
│  │                                                           │ │
│  │  Normal:  5 pods   (50 req/sec)                          │ │
│  │  Peak:    30 pods  (3000 req/sec - Black Friday!)        │ │
│  │  Night:   5 pods   (10 req/sec)                          │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    API TIER                               │ │
│  │  Deployment: api-backend                                  │ │
│  │  HPA: Min 3, Max 20                                       │ │
│  │  VPA: Initial mode (sets optimal resources)              │ │
│  │                                                           │ │
│  │  VPA learned: Each pod needs 500m CPU, 1GB RAM           │ │
│  │  HPA scales count based on load                          │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                  DATABASE TIER                            │ │
│  │  StatefulSet: postgres (1 master, 2 replicas)            │ │
│  │  VPA: Auto mode (continuously optimizes)                 │ │
│  │                                                           │ │
│  │  VPA adjusts master resources based on usage:            │ │
│  │  • Busy season: 4 CPU, 16GB RAM                          │ │
│  │  • Quiet: 2 CPU, 8GB RAM (restarts acceptable)           │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                  NODE TIER (Karpenter)                    │ │
│  │                                                           │ │
│  │  Normal: 3 nodes (m5.large)                              │ │
│  │  Peak: Karpenter adds 10 nodes mix:                      │ │
│  │        • 7 spot (cheap for web tier)                     │ │
│  │        • 3 on-demand (reliable for critical pods)        │ │
│  │  Post-peak: Consolidates to 3 nodes                     │ │
│  │                                                           │ │
│  │  Result: 60% cost savings vs fixed capacity! 💰          │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Pitfalls

| Problem | Solution |
|---------|----------|
| **HPA + VPA on same metric** | Use different metrics OR VPA Initial mode |
| **Cluster Autoscaler slow** | Switch to Karpenter for faster scaling |
| **Pods stuck pending** | Check node autoscaler (CA/Karpenter) is installed |
| **Aggressive scale-down** | Tune `scale-down-delay-after-add` (CA) or `consolidationPolicy` (Karpenter) |
| **Cost explosion** | Set HPA `maxReplicas` and resource limits properly |
| **Database restarts from VPA** | Use VPA Initial/Off mode for databases |

---

## Quick Reference

| Scaling Need | Use | Why |
|--------------|-----|-----|
| More pod replicas | HPA | Distribute load across instances |
| Better pod resources | VPA | Optimize single instance performance |
| More nodes (old way) | Cluster Autoscaler | Works with existing ASGs |
| More nodes (modern) | Karpenter | Faster, smarter, cheaper |
| Stateless app scaling | HPA | Add replicas easily |
| Stateful app scaling | VPA | Better resources for single instance |
| Cost optimization | Karpenter + HPA | Smart node provisioning + pod scaling |

---

## Key Takeaways

1. **HPA = hire more workers** - Scales pod count horizontally
2. **VPA = better equipment** - Optimizes pod resources (restarts pods!)
3. **Cluster Autoscaler = predefined rooms** - Uses ASGs, 5-10 min
4. **Karpenter = custom rooms** - Direct EC2, 1-2 min, smarter
5. **HPA + VPA = possible with Initial mode** - Different metrics or Initial mode
6. **Karpenter is AWS recommended** - For new EKS clusters
7. **Consolidation saves money** - Karpenter automatically optimizes

---

## Related Reading

- [AWS VPC - The Compound Analogy](../november/2025-11-30-aws-vpc.md)
- [AWS EKS - The Managed Office Building Analogy](./2025-12-02-aws-eks.md)
- [AWS VPC CNI - The Phone Extension System](./2025-12-03-aws-vpc-cni.md)
- [EKS Networking - The Mail & Delivery System](./2025-12-09-eks-networking.md)
- [EKS IAM - The Employee Badge System](./2025-12-10-eks-iam-irsa-pod-identity.md)
- [EKS Storage - The Locker & Storage Room System](./2025-12-11-eks-storage.md)

---

*Written on December 12, 2025*
