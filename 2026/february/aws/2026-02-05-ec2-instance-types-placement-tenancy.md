# AWS EC2: Instance Types, Placement Groups, and Tenancy - The Apartment Complex Analogy

> Understanding EC2 instance selection, placement strategies, and tenancy options through a real-world analogy of renting apartments in a massive residential complex.

---

## TL;DR

| AWS EC2 Concept | Real-World Analogy | One-Liner |
|-----------------|-------------------|-----------|
| Instance Type | Apartment model (studio, 1BR, penthouse) | Different sizes and features for different needs |
| Instance Family | Building wing (gym wing, office wing, storage wing) | Optimized for specific activities |
| Instance Generation | Building renovation year (2020 vs 2024 model) | Newer = better amenities, same price |
| Instance Size | Apartment size (small, large, 4xlarge) | Doubles resources with each step up |
| Attribute Suffix (a, g, n, d) | Apartment features (AMD appliances, garden, fiber internet, attached storage) | Special add-ons to base model |
| Cluster Placement Group | All apartments on same floor, same hallway | Sub-millisecond elevator rides between neighbors |
| Spread Placement Group | Each apartment in different building, different block | If one building burns, others unaffected |
| Partition Placement Group | Apartments grouped by wing, wings on separate foundations | Groups isolated, members can coordinate |
| Shared Tenancy | Standard apartment complex | Neighbors share building infrastructure |
| Dedicated Instance | Entire floor reserved for you | No neighbors on your floor, but you don't own it |
| Dedicated Host | You've leased the entire building with full blueprints | You see every room (socket/core), can bring your own furniture (BYOL), control placement, but AWS handles maintenance |

---

## The Big Picture

Imagine AWS operates the world's largest **apartment complex** with millions of units across thousands of buildings. When you need compute, you're essentially renting an apartment. But this isn't a simple "give me a room" situation - you need to decide:

1. **What type of apartment?** (Instance Type) - A studio for light work? A 4-bedroom for heavy processing? A unit with extra storage closets?
2. **Where in the complex?** (Placement Groups) - Do your apartments need to be neighbors for quick communication? Spread across buildings for safety?
3. **What's your lease arrangement?** (Tenancy) - Share infrastructure with others? Have a private floor? Own the entire building?

```
AWS EC2 APARTMENT COMPLEX
=============================================================================

                        ┌─────────────────────────────────────────────┐
                        │           INSTANCE TYPE SELECTION           │
                        │       "What kind of apartment do I need?"   │
                        └─────────────────────────────────────────────┘
                                            │
           ┌────────────────────────────────┼────────────────────────────────┐
           │                                │                                │
           ▼                                ▼                                ▼
    ┌─────────────┐                 ┌─────────────┐                 ┌─────────────┐
    │  M-WING     │                 │  C-WING     │                 │  R-WING     │
    │  (General)  │                 │  (Compute)  │                 │  (Memory)   │
    │             │                 │             │                 │             │
    │ Balanced    │                 │ Extra CPUs  │                 │ Extra RAM   │
    │ living      │                 │ home office │                 │ countertops │
    └─────────────┘                 └─────────────┘                 └─────────────┘

                        ┌─────────────────────────────────────────────┐
                        │          PLACEMENT GROUP SELECTION          │
                        │       "Where should my apartments be?"      │
                        └─────────────────────────────────────────────┘
                                            │
           ┌────────────────────────────────┼────────────────────────────────┐
           │                                │                                │
           ▼                                ▼                                ▼
    ┌─────────────┐                 ┌─────────────┐                 ┌─────────────┐
    │  CLUSTER    │                 │   SPREAD    │                 │  PARTITION  │
    │             │                 │             │                 │             │
    │ Same floor  │                 │ Different   │                 │ Grouped by  │
    │ same hallway│                 │ buildings   │                 │ wing        │
    └─────────────┘                 └─────────────┘                 └─────────────┘

                        ┌─────────────────────────────────────────────┐
                        │            TENANCY SELECTION                │
                        │     "What's my lease arrangement?"          │
                        └─────────────────────────────────────────────┘
                                            │
           ┌────────────────────────────────┼────────────────────────────────┐
           │                                │                                │
           ▼                                ▼                                ▼
    ┌─────────────┐                 ┌─────────────┐                 ┌─────────────┐
    │   SHARED    │                 │  DEDICATED  │                 │  DEDICATED  │
    │  (Default)  │                 │  INSTANCE   │                 │    HOST     │
    │             │                 │             │                 │             │
    │ Standard    │                 │ Private     │                 │ Own the     │
    │ apartment   │                 │ floor       │                 │ building    │
    └─────────────┘                 └─────────────┘                 └─────────────┘
```

---

## Part 1: Instance Types - Choosing Your Apartment Model

### The Naming Convention - Decoding the Apartment Listing

Every EC2 instance has a name like `m7i.4xlarge`. This isn't random - it's a precise specification:

```
INSTANCE TYPE ANATOMY
=============================================================================

    m    7    i    .    4xlarge
    │    │    │         │
    │    │    │         └── SIZE: How big is the apartment?
    │    │    │             (nano → micro → small → medium → large → xlarge → 2xlarge...)
    │    │    │
    │    │    └── ATTRIBUTES: Special features (optional)
    │    │        a = AMD processors (AMD appliances)
    │    │        g = AWS Graviton/ARM (energy-efficient design)
    │    │        i = Intel processors (Intel appliances)
    │    │        n = Network optimized (fiber internet installed)
    │    │        d = Instance store (attached storage unit - EPHEMERAL!)
    │    │        e = Extra memory or storage
    │    │        z = High frequency CPUs
    │    │
    │    └── GENERATION: Which year's model?
    │        Higher = newer, better price-performance
    │        (5 = ~2020 renovation (approximate), 7 = ~2024 renovation (approximate))
    │
    └── FAMILY: Which wing of the complex?
        M/T = General Purpose (balanced living)
        C   = Compute Optimized (extra home office space)
        R/X = Memory Optimized (huge walk-in closets)
        I/D/H = Storage Optimized (attached warehouses)
        P/G = Accelerated Computing (gaming room, art studio)


EXAMPLES DECODED:
=============================================================================

    c7gn.xlarge
    │ │││  │
    │ │││  └── xlarge: Standard large size
    │ ││└── n: Network optimized (up to 200 Gbps)
    │ │└── g: Graviton (ARM) processor
    │ └── 7: 7th generation (2023-2024)
    └── c: Compute optimized family

    r6id.24xlarge
    │ │││   │
    │ │││   └── 24xlarge: 24x the resources of xlarge
    │ ││└── d: Instance store volumes (NVMe SSDs attached)
    │ │└── i: Intel processor
    │ └── 6: 6th generation
    └── r: Memory optimized family

    m7a.metal
    │ ││  │
    │ ││  └── metal: Bare metal (you get the entire physical server)
    │ │└── a: AMD processor
    │ └── 7: 7th generation
    └── m: General purpose family
```

### Instance Families - The Building Wings

Each wing of the apartment complex is designed for specific lifestyles:

```
INSTANCE FAMILY COMPARISON
=============================================================================

┌──────────────────────────────────────────────────────────────────────────┐
│                        GENERAL PURPOSE (M, T)                            │
│                     "The Balanced Living Wing"                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  M-Series: "Standard Apartments"                                         │
│  ├── Balanced CPU:Memory ratio (1 vCPU : 4 GB RAM)                      │
│  ├── Good for: web servers, small databases, dev environments           │
│  └── Example: m7i.xlarge = 4 vCPU, 16 GB RAM                            │
│                                                                          │
│  T-Series: "Economy Studios with Burst Mode"                            │
│  ├── Burstable CPU (like a car with turbo boost)                        │
│  ├── Accumulates CPU credits when idle, spends when busy                │
│  ├── Great for: variable workloads, microservices, small websites       │
│  └── CAUTION: Run out of credits = throttled to baseline!               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                       COMPUTE OPTIMIZED (C)                              │
│                    "The Home Office Wing"                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Higher CPU:Memory ratio (1 vCPU : 2 GB RAM)                            │
│  ├── More processor power per dollar                                    │
│  ├── Good for: batch processing, gaming servers, scientific modeling    │
│  ├── NOT for: memory-heavy apps (use R-series instead)                  │
│  └── Example: c7i.2xlarge = 8 vCPU, 16 GB RAM                           │
│                                                                          │
│  Analogy: Apartments with oversized home offices but smaller bedrooms   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                       MEMORY OPTIMIZED (R, X)                            │
│                   "The Executive Suite Wing"                             │
├──────────────────────────────────────────────────────────────────────────┤
│  Apartments with enormous living rooms (RAM) for hosting large           │
│  gatherings - in-memory databases, analytics, and caching workloads.    │
│                                                                          │
│  R-Series: "High Memory-to-vCPU Ratio"                                  │
│  ├── 1 vCPU : 8 GB RAM (double the general purpose ratio)               │
│  ├── Good for: in-memory databases (Redis, Memcached), real-time apps   │
│  ├── NOT just "fast memory" - it's about RATIO of RAM per CPU           │
│  └── Example: r7i.xlarge = 4 vCPU, 32 GB RAM                            │
│                                                                          │
│  X-Series: "The Penthouse Memory Suites"                                │
│  ├── Extreme memory: up to 2 TB RAM on x2idn.32xlarge                   │
│  ├── For 24 TB memory, use High Memory instances (u-24tb1.112xlarge)    │
│  ├── 1 vCPU : 16+ GB RAM ratio                                          │
│  ├── Good for: SAP HANA, large in-memory analytics                      │
│  └── Example: x2idn.32xlarge = 128 vCPU, 2 TB RAM                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                       STORAGE OPTIMIZED (I, D, H)                        │
│                    "The Attached Warehouse Wing"                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  I-Series: "NVMe Speed Demons"                                          │
│  ├── High-speed NVMe instance store (local SSDs)                        │
│  ├── Millions of IOPS, microsecond latency                              │
│  ├── Good for: high-frequency trading, real-time databases              │
│  └── Example: i4i.16xlarge = 64 vCPU, 15 TB NVMe                        │
│                                                                          │
│  D-Series: "Dense Storage Warehouses"                                   │
│  ├── Massive HDD storage (48 TB+ per instance)                          │
│  ├── Good for: data lakes, Hadoop, log processing                       │
│  └── Lower IOPS but huge capacity                                       │
│                                                                          │
│  ⚠️  CRITICAL: Instance store ('d' suffix) is EPHEMERAL!                │
│  ├── Data is LOST when instance stops, terminates, or fails             │
│  ├── Only survives reboots                                              │
│  └── Always replicate important data to EBS or S3!                      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                    ACCELERATED COMPUTING (P, G, Inf, Trn)                │
│                    "The Gaming & Art Studio Wing"                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  P-Series: "GPU Powerhouses"                                            │
│  ├── NVIDIA GPUs (A100, H100) for ML training                           │
│  ├── Good for: deep learning, HPC, rendering                            │
│  └── Example: p5.48xlarge = 8x NVIDIA H100 GPUs                         │
│                                                                          │
│  G-Series: "Graphics Workstations"                                      │
│  ├── GPU for graphics-intensive apps                                    │
│  ├── Good for: video encoding, 3D visualization, game streaming         │
│  └── More cost-effective than P for graphics (not training)             │
│                                                                          │
│  Inf/Trn: "ML Inference & Training Specialists"                         │
│  ├── AWS custom silicon (Inferentia, Trainium)                          │
│  ├── Better price-performance than GPUs for specific ML tasks           │
│  └── Good for: inference at scale, cost-optimized training              │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Instance Sizing - From Studio to Penthouse

Each size doubles the resources from the previous:

```
INSTANCE SIZE PROGRESSION (using m7i family)
=============================================================================

  Size        vCPU    Memory    Network         Analogy
  ─────────────────────────────────────────────────────────────────
  nano         -        -         -             (not available in m7i)
  micro        -        -         -             (not available in m7i)
  small        -        -         -             (not available in m7i)
  medium      1        4 GB      Up to 12.5    Studio apartment
  large       2        8 GB      Up to 12.5    1-bedroom
  xlarge      4       16 GB      Up to 12.5    2-bedroom
  2xlarge     8       32 GB      Up to 12.5    3-bedroom
  4xlarge    16       64 GB      Up to 12.5    4-bedroom duplex
  8xlarge    32      128 GB      15 Gbps       Small house
  12xlarge   48      192 GB      22.5 Gbps     Large house
  16xlarge   64      256 GB      30 Gbps       Mansion
  24xlarge   96      384 GB      45 Gbps       Estate
  metal      128     512 GB      50 Gbps       Own the building
                                               (bare metal server)

  PATTERN: Each step up roughly DOUBLES resources
           xlarge → 2xlarge → 4xlarge → 8xlarge
                ×2        ×2        ×2
```

---

## Part 2: Placement Groups - Deciding Where Your Apartments Should Be

Now that you've chosen your apartment type, where should it be located in the complex?

### Cluster Placement Group - "Same Floor, Same Hallway"

```
CLUSTER PLACEMENT GROUP
=============================================================================

THE ANALOGY:
All your apartments are on the SAME FLOOR of the SAME BUILDING,
in the SAME HALLWAY. Neighbors can literally shout to each other.

TECHNICAL REALITY:
├── All instances on same rack or adjacent racks
├── Same Availability Zone (single AZ only!)
├── Network latency: sub-millisecond (often <100 microseconds)
├── Network throughput: up to 100 Gbps between instances
└── Uses Enhanced Networking with Elastic Fabric Adapter (EFA)


    ┌─────────────────────────────────────────────────────────────┐
    │                    BUILDING A, FLOOR 7                       │
    │                    (Single Rack / AZ)                        │
    │                                                              │
    │     ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐       │
    │     │ i-1 │───│ i-2 │───│ i-3 │───│ i-4 │───│ i-5 │       │
    │     └─────┘   └─────┘   └─────┘   └─────┘   └─────┘       │
    │        │         │         │         │         │           │
    │        └─────────┴─────────┴─────────┴─────────┘           │
    │                   Sub-millisecond latency                   │
    │                   Up to 100 Gbps throughput                 │
    └─────────────────────────────────────────────────────────────┘

IDEAL FOR:
├── HPC (High Performance Computing) - weather modeling, CFD
├── MPI (Message Passing Interface) workloads
├── Distributed ML training where nodes constantly sync
└── Any tightly-coupled application needing minimal latency

LIMITATIONS:
├── Single AZ = single point of failure
├── Capacity constraints - might not get all instances you need
├── All instances should be same type for best results
└── Start all instances together for best rack placement

LAUNCH STRATEGY:
├── Request all instances in single launch request if possible
├── Stop/start may land instance on different rack (avoid if possible)
└── Use capacity reservations for guaranteed placement
```

### Spread Placement Group - "Each Apartment in Different Buildings"

```
SPREAD PLACEMENT GROUP
=============================================================================

THE ANALOGY:
Each of your apartments is in a DIFFERENT BUILDING on a DIFFERENT BLOCK.
If one building has a fire, your other apartments are completely unaffected.

TECHNICAL REALITY:
├── Each instance on separate underlying hardware (different rack)
├── Can span multiple AZs
├── Maximum 7 instances per AZ (hard limit!)
├── AWS guarantees hardware isolation
└── If one physical server fails, only one instance affected


    AZ-A                    AZ-B                    AZ-C
    ┌─────────┐            ┌─────────┐            ┌─────────┐
    │ Bldg 1  │            │ Bldg 4  │            │ Bldg 7  │
    │ ┌─────┐ │            │ ┌─────┐ │            │ ┌─────┐ │
    │ │ i-1 │ │            │ │ i-4 │ │            │ │ i-7 │ │
    │ └─────┘ │            │ └─────┘ │            │ └─────┘ │
    └─────────┘            └─────────┘            └─────────┘

    ┌─────────┐            ┌─────────┐            ┌─────────┐
    │ Bldg 2  │            │ Bldg 5  │            │ Bldg 8  │
    │ ┌─────┐ │            │ ┌─────┐ │            │ ┌─────┐ │
    │ │ i-2 │ │            │ │ i-5 │ │            │ │ i-8 │ │
    │ └─────┘ │            │ └─────┘ │            │ └─────┘ │
    └─────────┘            └─────────┘            └─────────┘

    ┌─────────┐            ┌─────────┐            ┌─────────┐
    │ Bldg 3  │            │ Bldg 6  │            │ Bldg 9  │
    │ ┌─────┐ │            │ ┌─────┐ │            │ ┌─────┐ │
    │ │ i-3 │ │            │ │ i-6 │ │            │ │ i-9 │ │
    │ └─────┘ │            │ └─────┘ │            │ └─────┘ │
    └─────────┘            └─────────┘            └─────────┘

    Max 7 per AZ           Max 7 per AZ           Max 7 per AZ
                    Total: up to 21 instances

IDEAL FOR:
├── Critical instances that must be isolated (HA for small clusters)
├── Primary/replica database pairs
├── Zookeeper, etcd, or other quorum-based systems
└── Any workload where correlated failure is unacceptable

LIMITATIONS:
├── Max 7 instances per AZ (not 7 total - 7 PER AZ)
├── Not suitable for large clusters
├── Higher network latency than cluster placement
└── Each instance request may fail if no isolated hardware available
```

### Partition Placement Group - "Apartments Grouped by Wing"

```
PARTITION PLACEMENT GROUP
=============================================================================

THE ANALOGY:
Your apartments are organized into WINGS. Each wing has its own foundation
and utilities. If Wing A has a foundation crack, Wings B and C are fine.
Within each wing, neighbors can still coordinate closely.

TECHNICAL REALITY:
├── Instances divided into logical partitions (max 7 per AZ)
├── Each partition on separate rack (isolated failure domain)
├── Unlimited instances PER partition
├── Partition metadata exposed via instance metadata service
└── Applications can be partition-aware for intelligent replication


    AZ-A
    ┌────────────────────────────────────────────────────────────┐
    │  PARTITION 1       PARTITION 2       PARTITION 3          │
    │  (Rack A)          (Rack B)          (Rack C)             │
    │  ┌──────────┐      ┌──────────┐      ┌──────────┐        │
    │  │ i-1  i-2 │      │ i-5  i-6 │      │ i-9  i-10│        │
    │  │ i-3  i-4 │      │ i-7  i-8 │      │ i-11 i-12│        │
    │  │    ...   │      │    ...   │      │    ...   │        │
    │  │ (100s ok)│      │ (100s ok)│      │ (100s ok)│        │
    │  └──────────┘      └──────────┘      └──────────┘        │
    │       │                 │                 │               │
    │       ▼                 ▼                 ▼               │
    │   Isolated          Isolated          Isolated           │
    │   Failure           Failure           Failure            │
    │   Domain            Domain            Domain             │
    └────────────────────────────────────────────────────────────┘

              Max 7 partitions per AZ
              UNLIMITED instances per partition


PARTITION METADATA ACCESS:
=============================================================================

# From within an EC2 instance, query the metadata service:
$ curl http://169.254.169.254/latest/meta-data/placement/partition-number

# Response: "1" or "2" or "3" etc.

# This allows applications like Cassandra and HDFS to:
# - Know which partition they're in
# - Place replicas in DIFFERENT partitions
# - Avoid correlated failures automatically


IDEAL FOR:
├── Cassandra - place replicas across partitions for fault tolerance
├── HDFS - ensure data blocks replicated across racks
├── Kafka - partition leaders on separate failure domains
├── HBase - region servers distributed across partitions
└── Any distributed database needing topology-aware replication

KEY INSIGHT:
├── Spread = max 7 instances per AZ (7 isolated instances)
├── Partition = max 7 partitions per AZ (unlimited instances, 7 failure domains)
└── Partition is for LARGE clusters with topology awareness
```

### Placement Group Comparison

```
PLACEMENT GROUP DECISION MATRIX
=============================================================================

                    CLUSTER         SPREAD          PARTITION
                    ───────         ──────          ─────────
Instances/AZ        Unlimited*      Max 7           Unlimited
                                                    (7 partitions)

AZ Span             Single AZ       Multi-AZ        Multi-AZ

Latency             Lowest          Higher          Medium
                    (<100 us)

Failure Domain      Same rack       Each instance   Each partition
                    (shared risk)   isolated        isolated

Best For            HPC, ML         Critical small  Large distributed
                    Training        clusters        databases

Topology Aware      No              No              YES (metadata API)

* Capacity permitting
```

### When NOT to Use Placement Groups

```
DON'T OVER-ENGINEER:
=============================================================================

❌ DON'T use Cluster for:
   ├── Stateless web servers (they don't need low latency to each other)
   ├── Independent batch jobs (no inter-node communication)
   └── Anything that can tolerate normal network latency

❌ DON'T use Spread for:
   ├── Large clusters (you'll hit the 7/AZ limit)
   ├── Tightly-coupled HPC (you need cluster instead)
   └── Instances that don't need hardware isolation

❌ DON'T use Partition for:
   ├── Applications that aren't partition-aware
   ├── Small clusters (spread is simpler)
   └── HPC workloads (cluster gives better latency)

✅ DEFAULT: No placement group is fine for most workloads!
```

---

## Part 3: Tenancy - Your Lease Agreement

Now you've chosen your apartment type and location. What's your lease arrangement with the building owner?

### Shared Tenancy - "Standard Apartment"

```
SHARED TENANCY (Default)
=============================================================================

THE ANALOGY:
Standard apartment living. You have your own locked unit with your own key,
but you share the building's foundation, plumbing, and electrical systems
with other tenants. The landlord manages everything.

TECHNICAL REALITY:
├── Your instances share physical hardware with other AWS customers
├── Hardware-level isolation via hypervisor (Nitro System)
├── You cannot see or affect other tenants
├── Most cost-effective option
└── Suitable for 99% of workloads


    ┌──────────────────────────────────────────────────────────┐
    │               PHYSICAL SERVER (Host)                      │
    │                                                           │
    │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
    │   │ Your EC2    │  │ Customer B  │  │ Customer C  │     │
    │   │ Instance    │  │ EC2         │  │ EC2         │     │
    │   └─────────────┘  └─────────────┘  └─────────────┘     │
    │   ═══════════════════════════════════════════════════   │
    │                   AWS Nitro Hypervisor                   │
    │          (Isolates tenants at hardware level)            │
    │                                                           │
    │   CPU Cores    │    Memory    │    Network    │   NVMe   │
    └──────────────────────────────────────────────────────────┘

WHEN TO USE:
├── Default choice for most workloads
├── No compliance requirements mandating dedicated hardware
├── Cost is a primary concern
└── You trust AWS's hypervisor isolation (you should - it's battle-tested)

COST: Standard EC2 pricing
```

### Dedicated Instances - "Private Floor"

```
DEDICATED INSTANCES
=============================================================================

THE ANALOGY:
You've rented an ENTIRE FLOOR of the building. No other tenants on your floor,
but you don't own the building and can't see the floor plan. The landlord
still manages everything, you just know you're isolated.

TECHNICAL REALITY:
├── Your instances run on hardware dedicated to your account
├── May share hardware with OTHER instances from YOUR account
├── You have NO visibility into which host you're on
├── No control over instance placement on hosts
└── Cannot bring your own licenses (no socket/core visibility)


    ┌──────────────────────────────────────────────────────────┐
    │               PHYSICAL SERVER (Host)                      │
    │                      YOUR FLOOR                           │
    │                                                           │
    │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
    │   │ Your EC2    │  │ Your EC2    │  │ Your EC2    │     │
    │   │ Instance 1  │  │ Instance 2  │  │ Instance 3  │     │
    │   └─────────────┘  └─────────────┘  └─────────────┘     │
    │   ═══════════════════════════════════════════════════   │
    │                                                           │
    │   ✅ No other AWS customers on this hardware             │
    │   ❌ No visibility into host details                     │
    │   ❌ Cannot bring per-socket licenses                    │
    │                                                           │
    └──────────────────────────────────────────────────────────┘

PRICING:
├── Per-instance hourly rate (same as shared, or slight premium)
├── PLUS: $2/hour regional dedicated fee (per region, not per instance!)
└── Example: 10 instances in us-east-1 = instance costs + $2/hr

    Cost = (Instance Hours × Instance Price) + ($2/hr × Region Hours)

    Running 5 × m5.large for 720 hours:
    Shared:    5 × $0.096 × 720 = $345.60
    Dedicated: 5 × $0.096 × 720 + $2 × 720 = $1,785.60
                                   ↑
                          Regional fee adds up!

WHEN TO USE:
├── Compliance requires hardware isolation (but not BYOL)
├── Security policies mandate dedicated hardware
├── Regulations (HIPAA, PCI-DSS) interpreted to require isolation
└── You want isolation but don't need host-level control
```

### Dedicated Hosts - "Own the Building"

```
DEDICATED HOSTS
=============================================================================

THE ANALOGY:
You've purchased the ENTIRE BUILDING. You own it, see the blueprints,
know exactly how many rooms there are, and can bring your own furniture
(software licenses). You can sublease rooms however you want.

TECHNICAL REALITY:
├── You allocate an entire physical server
├── Full visibility: socket count, core count, host ID
├── Control over instance placement (host affinity)
├── CAN use BYOL (Bring Your Own License) for per-socket/per-core software
└── YOU manage capacity (how many instances fit on the host)


    ┌──────────────────────────────────────────────────────────┐
    │               PHYSICAL SERVER (Host)                      │
    │                     YOU OWN THIS                          │
    │                                                           │
    │   Host ID: h-0abc123def456                               │
    │   Sockets: 2                                              │
    │   Cores per Socket: 24                                    │
    │   Total Cores: 48                                         │
    │                                                           │
    │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
    │   │ Your m5.4xl │  │ Your m5.2xl │  │ Available       │ │
    │   │ (16 cores)  │  │ (8 cores)   │  │ (24 cores left) │ │
    │   └─────────────┘  └─────────────┘  └─────────────────┘ │
    │                                                           │
    │   ✅ Full visibility into hardware                       │
    │   ✅ Host affinity (pin instances to this host)          │
    │   ✅ BYOL: Windows Server, SQL Server, RHEL, SUSE        │
    │   ✅ You control placement                               │
    │                                                           │
    └──────────────────────────────────────────────────────────┘


WHY BYOL REQUIRES DEDICATED HOSTS:
=============================================================================

Microsoft licenses Windows Server PER PHYSICAL CORE.

Shared Tenancy:
├── AWS doesn't tell you which cores your instance uses
├── Cores are shared/oversubscribed across customers
├── You CANNOT prove to Microsoft which cores you're licensing
└── ❌ BYOL impossible

Dedicated Instance:
├── You know it's dedicated hardware
├── But you don't know socket/core details
└── ❌ Still can't prove core count to Microsoft

Dedicated Host:
├── AWS tells you: "2 sockets, 24 cores each"
├── You can tell Microsoft: "I need licenses for 48 cores"
├── You control which instances run on those licensed cores
└── ✅ BYOL works!


PRICING COMPARISON:
=============================================================================

Dedicated Host Pricing (m5 family example):
├── On-Demand: ~$4.00/hour for entire host (48 cores worth)
├── Reservation: ~$2.50/hour (1-year) or ~$1.75/hour (3-year)
└── You decide how to slice it into instances

LICENSE COST SAVINGS EXAMPLE:
├── SQL Server Enterprise: ~$14,000/core/year × 48 cores = $672,000/year
├── WITH BYOL + Dedicated Host: $1.75/hr × 8760 = $15,330/year
├── Savings: $656,670/year (assuming you have existing licenses)
└── This is why large enterprises use Dedicated Hosts


HOST AFFINITY:
=============================================================================

# Ensure instance always runs on specific host (for licensing compliance)

resource "aws_instance" "licensed_sql" {
  ami           = "ami-windows-sql-2019"
  instance_type = "m5.4xlarge"

  host_id       = "h-0abc123def456"  # Pin to this host
  tenancy       = "host"

  # If host fails, instance WON'T auto-recover to different host
  # (preserves licensing compliance)
}
```

### Tenancy Comparison

```
TENANCY DECISION MATRIX
=============================================================================

                    SHARED          DEDICATED        DEDICATED
                    (Default)       INSTANCE         HOST
                    ─────────       ─────────        ─────────

Hardware            Multi-tenant    Single-tenant    Single-tenant
Isolation

Host Visibility     None            None             Full (ID, cores,
                                                     sockets)

BYOL Support        No              No               YES

Instance            AWS             AWS              You (or AWS
Placement                                            with auto-place)

Cost                Lowest          Medium           Highest (but
                                                     saves on BYOL)

Compliance          Basic           Hardware         Hardware +
                                    isolation        audit trail

Use Case            General         Compliance       BYOL (Windows,
                    workloads       without BYOL     SQL, RHEL, SUSE)


DECISION TREE:
=============================================================================

Do you need BYOL for per-socket/per-core licenses?
    │
    ├── YES → Dedicated Host (only option)
    │
    └── NO → Do you need dedicated hardware for compliance?
                │
                ├── YES → Dedicated Instance (simpler than Host)
                │
                └── NO → Shared Tenancy (default, most cost-effective)
```

---

## Terraform Examples

### Instance Type Selection

```hcl
# General purpose web server
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "m7i.large"  # M = general, 7 = latest gen, i = Intel

  tags = {
    Name = "web-server"
  }
}

# Memory-optimized Redis cache
resource "aws_instance" "cache" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "r7i.xlarge"  # R = memory optimized, 1:8 CPU:RAM ratio

  tags = {
    Name = "redis-cache"
  }
}

# Compute-optimized batch processor
resource "aws_instance" "batch" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "c7i.2xlarge"  # C = compute optimized

  tags = {
    Name = "batch-processor"
  }
}

# Storage-optimized with instance store (EPHEMERAL!)
resource "aws_instance" "analytics" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "i4i.xlarge"  # I = storage optimized, has NVMe instance store

  # WARNING: Instance store data is lost on stop/terminate!
  # Use only for scratch data, replicate important data to S3/EBS

  tags = {
    Name = "analytics-node"
  }
}
```

### Placement Groups

```hcl
# Cluster placement group for HPC
resource "aws_placement_group" "hpc_cluster" {
  name     = "hpc-cluster"
  strategy = "cluster"  # Same rack, lowest latency
}

resource "aws_instance" "hpc_node" {
  count = 10

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "c7i.8xlarge"

  placement_group = aws_placement_group.hpc_cluster.id

  tags = {
    Name = "hpc-node-${count.index}"
  }
}

# Spread placement group for critical instances
resource "aws_placement_group" "critical" {
  name     = "critical-spread"
  strategy = "spread"  # Each instance on separate hardware
}

resource "aws_instance" "zookeeper" {
  count = 3  # Max 7 per AZ!

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "m7i.large"

  placement_group = aws_placement_group.critical.id
  availability_zone = element(data.aws_availability_zones.available.names, count.index)

  tags = {
    Name = "zookeeper-${count.index}"
  }
}

# Partition placement group for Cassandra
resource "aws_placement_group" "cassandra" {
  name            = "cassandra-partitioned"
  strategy        = "partition"
  partition_count = 3  # Max 7 partitions per AZ
}

resource "aws_instance" "cassandra_node" {
  count = 9  # 3 nodes per partition

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "i4i.2xlarge"  # Storage optimized for database

  placement_group = aws_placement_group.cassandra.id

  placement {
    partition_number = (count.index % 3) + 1  # Distribute across partitions
  }

  tags = {
    Name = "cassandra-${count.index}"
    Partition = (count.index % 3) + 1
  }
}
```

### Tenancy Options

```hcl
# Shared tenancy (default - no special config needed)
resource "aws_instance" "shared" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "m7i.large"
  # tenancy defaults to "default" (shared)
}

# Dedicated instance
resource "aws_instance" "dedicated" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "m5.xlarge"
  tenancy       = "dedicated"  # Runs on dedicated hardware

  # Note: $2/hr regional fee applies to your account
  tags = {
    Name = "compliant-workload"
  }
}

# Dedicated host allocation
resource "aws_ec2_host" "windows_host" {
  instance_type     = "m5.large"  # Family/size determines host capacity
  availability_zone = "us-east-1a"

  # For BYOL
  auto_placement = "off"  # Manual placement for license tracking

  tags = {
    Name    = "windows-byol-host"
    Purpose = "Windows Server BYOL"
  }
}

# Instance on dedicated host
resource "aws_instance" "windows_byol" {
  ami           = "ami-windows-server-2022"  # Your BYOL AMI
  instance_type = "m5.large"

  host_id = aws_ec2_host.windows_host.id  # Pin to specific host
  tenancy = "host"

  tags = {
    Name    = "windows-byol-instance"
    License = "BYOL-Windows-Server-2022"
  }
}
```

---

## Key Takeaways

### Instance Types
1. **Decode every instance name** - Family (M/C/R), Generation (5/6/7), Attributes (a/g/i/n/d), Size (xlarge/2xlarge)
2. **Choose family by workload** - General (M/T), Compute (C), Memory (R/X), Storage (I/D), Accelerated (P/G)
3. **Always use latest generation** - m7i beats m6i beats m5 at the same price
4. **Instance store ('d') is ephemeral** - Data lost on stop/terminate, only survives reboot
5. **R-series is about ratio** - 1:8 CPU:RAM, not "faster" memory

### Placement Groups
6. **Cluster = low latency** - Same rack, single AZ, for HPC/ML training
7. **Spread = max isolation** - Different hardware, max 7 instances per AZ
8. **Partition = topology-aware** - Max 7 partitions per AZ, unlimited instances per partition
9. **Partition metadata is exposed** - Query instance metadata for topology-aware replication
10. **Don't over-engineer** - No placement group is fine for most workloads

### Tenancy
11. **Shared is the default** - AWS hypervisor isolation is battle-tested
12. **Dedicated Instance = compliance** - Hardware isolation without host control, $2/hr regional fee
13. **Dedicated Host = BYOL** - Required for per-socket/per-core licenses (Windows Server, SQL Server)
14. **Host affinity pins instances** - For license compliance, won't migrate on host failure

---

*Written on February 5, 2026*
