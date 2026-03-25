# Block & File Storage (EBS, EFS, FSx) -- The Workshop, Library, and Specialty Studio Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), and organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock). But your EC2 instances still need somewhere to store their **operating system, databases, and application state** -- data that must look like a disk drive, not a file in a bucket. Your containerized microservices need a **shared filesystem** that multiple instances can mount simultaneously. Your HPC cluster needs a **parallel filesystem** that can deliver hundreds of gigabytes per second to thousands of compute nodes processing a genome sequence. And your Windows enterprise needs **SMB shares integrated with Active Directory** so that migrating from on-premises feels seamless. That is the domain of EBS, EFS, and FSx -- the block and file storage services that complement S3's object storage to complete your AWS storage toolkit.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| EBS (Elastic Block Store) | A personal workbench with dedicated drawers -- only one craftsman uses it, but they can swap drawer units for bigger or faster ones without stopping work | Block storage volumes attached to a single EC2 instance (or up to 16 with Multi-Attach on io2); the OS sees them as raw disk devices for filesystems, databases, or boot volumes |
| EBS gp3 | A standard workbench with a guaranteed set of basic tools included free, plus a catalog to order specialty tools a la carte | Default SSD volume with 3,000 IOPS and 125 MiB/s included at no extra cost, independently scalable to 80,000 IOPS and 2,000 MiB/s -- you only pay for what you provision beyond the baseline |
| EBS gp2 | The older workbench where drawer size determined how many tools you got -- want more tools? Buy a bigger bench | Legacy SSD volume where IOPS scale with volume size (3 IOPS/GiB) up to 16,000; small volumes rely on burst credits; replaced by gp3 for new workloads |
| EBS io2 Block Express | A precision-engineered surgical workbench with vibration dampening, guaranteed sub-millisecond tool retrieval, and a 99.999% reliability certification | High-performance SSD with up to 256,000 IOPS, 4,000 MiB/s, sub-millisecond latency, and 99.999% durability (100x better than gp3); the only EBS type supporting Multi-Attach |
| EBS st1 | A warehouse conveyor belt -- it moves massive quantities in a stream, but you cannot pick individual items off it quickly | Throughput-optimized HDD for large sequential workloads (MapReduce, Kafka, log processing); up to 500 MiB/s throughput but cannot be a boot volume |
| EBS sc1 | A cold storage warehouse -- cheapest rent, items arrive slowly on a small conveyor, but it costs almost nothing to keep them there | Cold HDD for infrequently accessed sequential data; up to 250 MiB/s; lowest cost EBS volume type; cannot be a boot volume |
| EBS Snapshots | Photocopying only the pages that changed since the last copy -- the first copy is the full document, subsequent copies are just the deltas | Incremental backups stored in S3; only changed blocks are saved after the first snapshot; can be copied cross-region for DR; Fast Snapshot Restore eliminates first-access latency |
| EBS Multi-Attach | A shared workbench bolted to the floor that two to sixteen craftsmen can use simultaneously -- but they must coordinate who uses which drawer or they will overwrite each other's work | io2 Block Express only; up to 16 Nitro instances in the same AZ; requires a cluster-aware filesystem (GFS2, OCFS2) to prevent data corruption |
| EBS Encryption | A locksmith (KMS) who encrypts every drawer's contents and the conveyor between the bench and the craftsman -- once enabled, everything is transparent | AES-256 encryption of data at rest, in transit (instance-to-volume), and all snapshots; uses KMS keys; cannot directly encrypt an existing unencrypted volume |
| EFS (Elastic File System) | A shared library with unlimited shelf space -- any number of readers can walk in from any branch location, the building automatically expands as more books arrive, and rarely-read books are moved to cheaper basement shelves | Managed NFS filesystem that scales automatically, supports thousands of concurrent EC2/ECS/Lambda connections across AZs, and offers lifecycle tiering to IA and Archive classes |
| EFS General Purpose mode | The main library floor with fast checkout counters and a high throughput ceiling | Default performance mode; lowest latency (~1ms read); up to 2.5 million read IOPS with Elastic throughput; recommended for all workloads |
| EFS Max I/O mode | The old industrial library with more checkout counters but longer lines at each one -- higher aggregate throughput at the cost of per-operation latency | Legacy mode; higher latency; deprecated for new workloads since General Purpose with Elastic throughput matches or exceeds it |
| EFS Elastic throughput | A library that hires more checkout clerks during rush hour and sends them home when it is quiet -- you only pay for the minutes they work | Default throughput mode; automatically scales to 60 GiBps read / 5 GiBps write; pay only for throughput consumed; recommended for most workloads |
| EFS storage classes | Different floors in the library -- ground floor (Standard) is instant, basement (IA) requires a small retrieval fee, sub-basement (Archive) is cheapest for rarely-read books | Standard, Infrequent Access (up to 92% cheaper), Archive (lowest cost); One Zone variants available for Standard and IA (47% cheaper, single AZ) -- Archive is Regional only |
| FSx for Windows File Server | A branch of the city's filing department that speaks the government's language (SMB), checks employee badges (Active Directory), and follows all the bureaucratic filing rules Windows admins expect | Managed Windows-native file shares with SMB protocol, AD integration, DFS Namespaces, shadow copies, data deduplication; zero code changes for Windows application migration |
| FSx for Lustre | A high-speed assembly line that feeds raw materials (S3 objects) to hundreds of factory workers (compute nodes) simultaneously at 1,000+ GBps | Managed parallel filesystem for HPC, ML training, media rendering; S3 data repository association for lazy-loading; scratch (temporary) vs persistent (durable) deployment types |
| FSx for NetApp ONTAP | A universal translator filing system that speaks every language (NFS, SMB, iSCSI, NVMe) simultaneously, with built-in photocopier (SnapMirror), automatic basement storage (tiering), and instant carbon copies (FlexClone) | Multi-protocol NAS with automatic SSD-to-capacity-pool tiering, cross-region SnapMirror replication, zero-copy clones, compression, deduplication; ideal for on-premises NetApp migration |
| FSx for OpenZFS | A Linux craftsman's personal workshop with instant snapshot capability, built-in compression, and the ability to clone entire project setups in seconds without using extra space | NFS-based managed filesystem with ZFS features: snapshots, clones, compression, up to 1M IOPS; ideal for Linux workloads migrating from on-premises ZFS |
| RAID 0 on EBS | Bolting two conveyor belts side by side so twice as many boxes move simultaneously -- but if either belt breaks, everything on both belts is lost | Striping across multiple EBS volumes for additive IOPS and throughput; no redundancy; the only RAID level AWS recommends on EBS |

---

## The Big Picture: Workshop, Library, and Specialty Studios

AWS block and file storage maps to three real-world models:

**EBS is your personal workbench** -- a dedicated workspace attached to one craftsman (EC2 instance). The workbench has drawers (volumes) that store the craftsman's tools and materials. Only that craftsman uses it (with rare Multi-Attach exceptions). You choose drawer types based on whether you need fast random access to small items (SSD for databases) or high-volume sequential flow of materials (HDD for log processing). When the craftsman goes home (instance stops), the drawers stay locked and intact. If the craftsman moves to a different workshop in the same building (same AZ), the drawers can be unbolted and moved. But they cannot be carried to a different building (different AZ) -- you would need to photocopy the contents (snapshot) and recreate the drawers at the new location.

**EFS is the shared library** -- a building that anyone in the organization can walk into from any branch location (any AZ in the region). The library automatically expands its shelves as more books arrive (elastic scaling). It has a checkout system (NFS protocol) that thousands of readers can use simultaneously. Rarely-requested books are moved to cheaper basement shelves (Infrequent Access and Archive tiers). The library costs more per book than a personal workbench drawer, but the sharing and auto-scaling make it essential for workloads where multiple instances need the same data.

**FSx is a collection of specialty studios** -- each built for a specific craft. The Windows calligraphy studio (FSx for Windows) has all the tools a Windows artisan expects. The high-speed factory floor (FSx for Lustre) moves materials at massive throughput for industrial production runs. The universal workshop (FSx for NetApp ONTAP) speaks every craft language and includes a built-in photocopier and material compressor. The Linux woodworking shop (FSx for OpenZFS) has instant snapshot capability and built-in sawdust compression.

```
AWS BLOCK & FILE STORAGE -- THE COMPLETE PICTURE
================================================================================

  ┌─────────────────── BLOCK STORAGE ───────────────────┐
  │                                                     │
  │  EBS (Personal Workbench)                           │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │
  │  │   gp3    │ │   io2    │ │ st1/sc1  │           │
  │  │   SSD    │ │ Block Exp│ │   HDD    │           │
  │  │ Default  │ │ Extreme  │ │ Throughp │           │
  │  │ 80K IOPS│ │ 256K IOPS│ │ 500 MBps │           │
  │  └──────────┘ └──────────┘ └──────────┘           │
  │  Single instance (Multi-Attach: io2 only)          │
  │  Same-AZ only  |  Snapshots for backup/DR          │
  └─────────────────────────────────────────────────────┘

  ┌─────────────────── FILE STORAGE ────────────────────┐
  │                                                     │
  │  EFS (Shared Library)          FSx (Specialty Studios)
  │  ┌──────────────────┐         ┌────────────────────┐│
  │  │ NFS / POSIX      │         │ Windows File Server││
  │  │ Multi-AZ elastic │         │ (SMB + AD)         ││
  │  │ Standard/IA/Arch │         ├────────────────────┤│
  │  │ 60 GiBps read    │         │ Lustre             ││
  │  │ Thousands of     │         │ (HPC + S3 bridge)  ││
  │  │ concurrent       │         ├────────────────────┤│
  │  │ clients          │         │ NetApp ONTAP       ││
  │  └──────────────────┘         │ (NFS+SMB+iSCSI)    ││
  │                               ├────────────────────┤│
  │                               │ OpenZFS            ││
  │                               │ (NFS + snapshots)  ││
  │                               └────────────────────┘│
  └─────────────────────────────────────────────────────┘

  ┌─────────────────── OBJECT STORAGE ──────────────────┐
  │  S3 (City Archive -- yesterday's deep dive)         │
  │  HTTP API  |  11 9s durability  |  8 storage classes│
  └─────────────────────────────────────────────────────┘
```

---

## Part 1: EBS Volume Types -- Choosing the Right Workbench Drawer

EBS offers six volume types organized along two axes: **IOPS-optimized SSD** for random read/write workloads (databases, boot volumes) and **throughput-optimized HDD** for sequential read/write workloads (big data, log processing). The single most important concept is understanding when you need high IOPS (thousands of small random reads per second, like a database doing index lookups) versus high throughput (streaming gigabytes per second sequentially, like MapReduce scanning a dataset).

### The Complete EBS Volume Type Comparison

| Volume Type | Category | Size Range | Max IOPS | Max Throughput | Latency | Durability | Boot Volume? | Multi-Attach? | Cost Model |
|---|---|---|---|---|---|---|---|---|---|
| **gp3** | SSD | 1 GiB - 64 TiB | 80,000 | 2,000 MiB/s | Single-digit ms | 99.8-99.9% | Yes | No | $0.08/GB + $0.005/provisioned IOPS above 3K + $0.04/provisioned MiB/s above 125 |
| **gp2** (legacy) | SSD | 1 GiB - 16 TiB | 16,000 | **250 MiB/s** | Single-digit ms | 99.8-99.9% | Yes | No | $0.10/GB (IOPS tied to size) |
| **io2 Block Express** | SSD | 4 GiB - 64 TiB | 256,000 | 4,000 MiB/s | Sub-millisecond | **99.999%** | Yes | **Yes** (up to 16) | $0.125/GB + $0.065/provisioned IOPS (first 32K) + $0.046 (32K-64K) + $0.032 (64K+) |
| **st1** | HDD | 125 GiB - 16 TiB | 500 | 500 MiB/s | ~5-10 ms | 99.8-99.9% | **No** | No | $0.045/GB |
| **sc1** | HDD | 125 GiB - 16 TiB | 250 | 250 MiB/s | ~5-10 ms | 99.8-99.9% | **No** | No | $0.015/GB |

### gp3: The Default Choice (The Standard Workbench)

gp3 is the default EBS volume type and the correct starting point for almost every workload. Its defining innovation over gp2 is **decoupling IOPS and throughput from volume size**.

**The analogy**: gp2 was an older workbench where the size of your bench determined how many tools you got -- want 16,000 IOPS? You had to buy a 5,334 GiB bench (16,000 / 3 IOPS-per-GiB), even if you only needed 100 GiB of storage. gp3 is the modern workbench that comes with a standard set of tools (3,000 IOPS, 125 MiB/s) regardless of size, and you can order additional specialty tools a la carte without buying a bigger bench.

**Key gp3 numbers for interviews:**
- **3,000 IOPS and 125 MiB/s included free** with every gp3 volume, regardless of size
- Scale IOPS up to **80,000** (provisioning formula: up to 500 IOPS per GiB of volume size)
- Scale throughput up to **2,000 MiB/s** (provisioning formula: up to 0.25 MiB/s per provisioned IOPS)
- IOPS and throughput are **independently adjustable** -- you can have high IOPS with low throughput or vice versa
- **20% cheaper per-GB** than gp2 ($0.08 vs $0.10), and most workloads never need to provision beyond the free baseline

**gp2 burst credits -- why they matter for migration decisions:**

gp2 volumes smaller than 1,000 GiB receive fewer than 3,000 baseline IOPS (since it is 3 IOPS/GiB). A 100 GiB gp2 volume has only 300 baseline IOPS. It gets a burst credit balance that allows temporary bursts to 3,000 IOPS, but if your workload sustains high IOPS, the credits deplete and performance drops to 300 IOPS. gp3 eliminates this problem -- even a 1 GiB gp3 volume gets 3,000 baseline IOPS with no burst credit mechanic.

### io2 Block Express: The Precision Workbench

io2 Block Express is for workloads that need **extreme IOPS, sub-millisecond latency, or 99.999% durability** (five 9s, compared to gp3's three 9s). As of November 2023, all io2 volumes created are automatically Block Express -- there is no separate selection.

**When to use io2:**
- Databases requiring more than 80,000 IOPS (the gp3 ceiling): SAP HANA, Oracle RAC, large SQL Server
- Workloads requiring **sub-millisecond latency** (gp3 is single-digit millisecond)
- **Multi-Attach** shared storage (io2 is the only EBS type supporting this)
- Compliance requirements mandating **99.999% volume durability** (100x better than gp3)

**Provisioning formula**: Up to 1,000 IOPS per GiB of volume size, maximum 256,000 IOPS. A 256 GiB io2 volume can provision up to 256,000 IOPS. A 16 GiB io2 volume can only provision up to 16,000 IOPS (16 x 1,000).

### st1 and sc1: The Warehouse Conveyor Belts

HDD volumes exist for a fundamentally different workload pattern: **large sequential I/O** where throughput matters more than latency.

**The analogy**: SSD volumes are like grabbing individual items from a well-organized toolbox (random access). HDD volumes are like a warehouse conveyor belt -- terrible for picking individual items, but excellent for moving large quantities in a continuous stream.

| Feature | st1 (Throughput Optimized) | sc1 (Cold) |
|---------|---------------------------|------------|
| **Analogy** | An active conveyor belt running all shift | A slow belt in cold storage, started only when needed |
| **Baseline throughput** | 40 MiB/s per TiB | 12 MiB/s per TiB |
| **Burst throughput** | 250 MiB/s per TiB | 80 MiB/s per TiB |
| **Max throughput** | 500 MiB/s | 250 MiB/s |
| **Use case** | MapReduce, Kafka, log processing, data warehouses | Infrequently accessed sequential data, backups |
| **Boot volume?** | **No** | **No** |
| **Cost** | $0.045/GB | $0.015/GB |

**Critical interview point**: HDD volumes (st1, sc1) **cannot be boot volumes**. The OS must boot from an SSD volume (gp3, gp2, or io2). This is one of the most common EBS trick questions.

### EBS Volume Type Decision Framework

```
EBS VOLUME TYPE DECISION TREE
════════════════════════════════════════════════════════════════════

  What is the I/O pattern?
       │
       ├── Random I/O (database, boot volume, application server)
       │     │
       │     ├── Need > 80,000 IOPS or sub-ms latency?
       │     │     YES ──▶ io2 Block Express
       │     │             (up to 256K IOPS, sub-ms, 99.999% durability)
       │     │
       │     ├── Need Multi-Attach (shared block storage)?
       │     │     YES ──▶ io2 Block Express (only option)
       │     │
       │     └── Standard database / boot / application?
       │           ──▶ gp3 (default, 3K IOPS free, scale as needed)
       │               Cost tip: Check if current gp2 volumes can
       │               be migrated to gp3 via Elastic Volumes for
       │               immediate 20% savings
       │
       └── Sequential I/O (big data, logs, streaming, ETL)
             │
             ├── Frequently accessed (daily processing)?
             │     ──▶ st1 (Throughput Optimized HDD)
             │         40 MiB/s per TiB baseline, 500 MiB/s max
             │
             └── Infrequently accessed (cold data, backups)?
                   ──▶ sc1 (Cold HDD)
                       12 MiB/s per TiB baseline, cheapest EBS
```

---

## Part 2: EBS Features -- Snapshots, Multi-Attach, Encryption, and RAID

### EBS Snapshots -- The Incremental Photocopier

Think of an EBS snapshot as **photocopying a document**: the first copy reproduces every page, but subsequent copies only photocopy the pages that changed. This incremental approach means a 1 TiB volume with 50 GiB of changes since the last snapshot only stores 50 GiB of new data -- dramatically reducing storage cost and snapshot creation time.

**Application-consistent vs crash-consistent snapshots**: EBS snapshots capture a point-in-time view of the block device, but if the application (e.g., MySQL) has data buffered in memory that has not been flushed to disk, the snapshot is only crash-consistent -- restoring it requires the database to run crash recovery (e.g., InnoDB replay), which adds startup time and risk. For a truly application-consistent snapshot, flush the application's buffers first (e.g., `FLUSH TABLES WITH READ LOCK` in MySQL) before initiating the snapshot, then release the lock immediately after.

**How snapshots work technically:**
- Snapshots are stored in S3 (managed by AWS, not visible in your S3 console)
- The first snapshot captures all blocks with data (not empty space)
- Subsequent snapshots capture only blocks that changed since the previous snapshot
- Each snapshot is **independently restorable** -- you do not need the full chain; AWS manages the block-level deduplication internally
- Snapshots are **regional** but can be **copied cross-region** for DR (this connects to your DR patterns study -- cross-region snapshot copy enables Pilot Light architecture)

**Fast Snapshot Restore (FSR):**

When you create an EBS volume from a snapshot, blocks are **lazily loaded** -- the first read of each block must fetch it from S3, which adds latency. For boot volumes or databases where first-access latency is unacceptable, FSR pre-initializes the volume so all blocks are immediately available at full performance.

- FSR is enabled per snapshot per AZ
- Costs ~$0.75/FSR-enabled snapshot-AZ/hour (expensive -- enable only for critical volumes)
- Creates up to 10 volumes per FSR-enabled snapshot-AZ before throttling

**Snapshot Archive:**

For long-term retention (compliance, yearly backups), Snapshot Archive stores snapshots at **75% lower cost** than standard snapshots. The trade-off: restoring from archive takes 24-72 hours. This mirrors the S3 storage class tiering you studied yesterday -- same economic principle of trading retrieval speed for storage cost.

**Data Lifecycle Manager (DLM):**

Automates snapshot creation, retention, and cross-region copy on a schedule. Think of it as a cron job for snapshots with policy-based lifecycle management.

```hcl
# Terraform: Automated EBS snapshot lifecycle policy
resource "aws_dlm_lifecycle_policy" "daily_snapshots" {
  description        = "Daily EBS snapshots with 30-day retention"
  execution_role_arn = aws_iam_role.dlm.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]

    schedule {
      name = "daily-snapshot"

      create_rule {
        interval      = 24
        interval_unit = "HOURS"
        times         = ["03:00"]  # UTC -- during low-traffic window
      }

      retain_rule {
        count = 30  # Keep last 30 snapshots
      }

      tags_to_add = {
        SnapshotType = "DLM-automated"
      }

      copy_tags = true  # Inherit volume tags

      # Cross-region copy for DR
      cross_region_copy_rule {
        target    = "eu-west-1"
        encrypted = true
        cmk_arn   = "arn:aws:kms:eu-west-1:123456789012:key/dr-key-id"

        retain_rule {
          interval      = 14
          interval_unit = "DAYS"
        }
      }
    }

    target_tags = {
      Backup = "true"  # Only snapshot volumes with this tag
    }
  }
}
```

### Multi-Attach -- The Shared Workbench

Multi-Attach allows a single io2 Block Express volume to be attached to **up to 16 Nitro-based EC2 instances simultaneously** within the same AZ.

**The critical constraint**: EBS provides a raw block device. If two instances mount the same block device with a standard filesystem (ext4, XFS), they will **corrupt each other's data** because standard filesystems are not designed for concurrent writers. You must use a **cluster-aware filesystem** like GFS2 (Red Hat) or OCFS2 (Oracle) that coordinates access across nodes.

**Use cases:**
- Oracle RAC (Real Application Clusters)
- Shared storage for clustered applications that manage their own concurrency
- Achieving shared-disk architecture without the overhead of a network filesystem

**When NOT to use Multi-Attach:**
- If you need shared file access for general applications -- use EFS instead
- If instances are in different AZs -- Multi-Attach is same-AZ only
- If you are using gp3 -- Multi-Attach is io2-only

### EBS Encryption -- Transparent Locksmith Service

EBS encryption uses the KMS infrastructure you studied on [Mar 9](2026-03-09-aws-kms-encryption-deep-dive.md) to provide transparent AES-256 encryption:

- **Data at rest** on the volume is encrypted
- **Data in transit** between the instance and the volume is encrypted
- **All snapshots** created from an encrypted volume are encrypted
- **All volumes** created from an encrypted snapshot are encrypted

**The "cannot encrypt in place" trap:**

You **cannot directly encrypt an existing unencrypted volume**. The migration path is:
1. Create a snapshot of the unencrypted volume
2. Copy the snapshot with encryption enabled (specify KMS key)
3. Create a new encrypted volume from the encrypted snapshot
4. Detach the old volume and attach the new one

**Default encryption**: You can enable default EBS encryption per region per account, which automatically encrypts all new volumes and snapshots using the specified KMS key. This is a best practice that eliminates the risk of unencrypted volumes being created by accident.

### Elastic Volumes -- Live Modification Without Downtime

EBS Elastic Volumes let you modify a volume's type, size, IOPS, and throughput **while it remains attached and in use** -- no detach, no downtime. This is the mechanism behind gp2-to-gp3 migration: you change the volume type from gp2 to gp3 via the console or CLI, and the modification happens live.

**Key constraints:**
- You can **increase** volume size but **never shrink** it (to shrink, create a smaller volume and migrate data)
- After a modification, you must wait **6 hours** before making another modification to the same volume
- Type changes (e.g., gp2 → gp3, gp3 → io2) are supported
- Filesystem resizing (e.g., `resize2fs`, `xfs_growfs`) must be done after the volume expansion completes

### Per-Instance EBS Bandwidth Limits -- The Hidden Bottleneck

Each EC2 instance type has a **maximum EBS bandwidth** independent of the volume's capabilities. You can provision 80,000 IOPS on a gp3 volume, but if your instance type only supports 4,750 Mbps of EBS throughput (e.g., m5.xlarge), the instance becomes the bottleneck, not the volume.

**Common gotcha**: Provisioning high-IOPS volumes on undersized instances. Always check the [instance type EBS specifications](https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html) to ensure your instance can saturate the volume. EBS-optimized instances (default on most current-gen types) dedicate network bandwidth to EBS traffic.

### EC2 Instance Store -- The Ephemeral Speed Demon

Instance store volumes are **physically attached NVMe SSDs** on the host server. They deliver millions of IOPS with the lowest possible latency because there is no network hop. The trade-off: **all data is lost** when the instance stops, terminates, or the underlying hardware fails.

**When to use instance store:**
- Temporary scratch space, caches, buffers
- Distributed systems that replicate data across nodes (Kafka, Cassandra, HDFS)
- Workloads needing millions of random IOPS that no EBS volume can match

**When NOT to use instance store:**
- Boot volumes, databases of record, anything requiring durability
- Data that must survive instance stop/start cycles

### RAID on EBS -- Striping for Performance

AWS only recommends **RAID 0** (striping) on EBS. The other RAID levels waste EBS IOPS:

| RAID Level | AWS Recommendation | Why |
|---|---|---|
| **RAID 0** (striping) | **Recommended** | Additive IOPS and throughput across volumes; if you need 160,000 IOPS, stripe two gp3 volumes at 80,000 IOPS each |
| **RAID 1** (mirroring) | Not recommended | Writes go to both volumes, consuming twice the IOPS with no throughput benefit; EBS already replicates within the AZ |
| **RAID 5/6** (parity) | **Never** | Parity writes consume 20-30% of available IOPS; EBS's network latency amplifies the penalty; AWS explicitly warns against this |

**When to use RAID 0:**
When a single EBS volume cannot deliver enough IOPS or throughput, or when the **per-instance EBS bandwidth limit** is not the bottleneck. For example, if your workload needs 160,000 IOPS (gp3 max per volume is 80,000), stripe two gp3 volumes. But understand the trade-off: **if any volume in the stripe fails, all data is lost**. Always pair RAID 0 with snapshots and the understanding that the application or replication layer provides durability.

```bash
# Create RAID 0 across two EBS volumes
sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/xvdb /dev/xvdc

# Create filesystem
sudo mkfs.xfs /dev/md0

# Mount
sudo mkdir /data
sudo mount /dev/md0 /data
```

---

## Part 3: EFS -- The Shared Library

EFS is a **managed NFS filesystem** that scales automatically and supports concurrent access from thousands of clients across multiple AZs. If EBS is a personal workbench dedicated to one craftsman, EFS is the shared library where everyone in the organization can check out books simultaneously.

### Performance Modes

| Mode | Analogy | Max IOPS | Latency | Supports One Zone? | Status |
|---|---|---|---|---|---|
| **General Purpose** | Main library floor with fast checkout | Up to 2.5M read IOPS (with Elastic throughput) | ~1ms read / ~2.7ms write | Yes | **Default, recommended** |
| **Max I/O** | Old industrial library -- more counters but longer lines | Higher aggregate (legacy) | Higher latency | **No** | **Deprecated for new workloads** |

**Why Max I/O is deprecated**: General Purpose mode with Elastic throughput now matches or exceeds Max I/O's parallelism capabilities without the latency penalty. AWS recommends General Purpose for all new file systems. Max I/O is not available for One Zone file systems.

### Throughput Modes -- The Staffing Model

This is where EFS economics get interesting, and where the old model (Bursting) created real operational pain.

| Mode | Analogy | How It Works | Best For | Gotcha |
|---|---|---|---|---|
| **Elastic** | Hire temporary staff during rush hour, pay only for hours worked | Auto-scales to 60 GiBps read / 5 GiBps write; pay per-MiB consumed | Most workloads (default in console) | Higher per-MiBps cost than Provisioned for sustained high throughput |
| **Provisioned** | Hire a fixed number of full-time staff regardless of foot traffic | You specify a throughput level (MiB/s) independent of storage size | Predictable, sustained high-throughput workloads | You pay for provisioned throughput whether you use it or not |
| **Bursting** | Library staff size determined by building size -- small building = few staff, even during rush hour | Throughput scales with stored data: 50 KiB/s per GiB baseline, bursts to 100 MiB/s per TiB | **Legacy -- avoid for new file systems** | Small file systems (< 1 TiB) exhaust burst credits and throttle to unusably low throughput |

**The Bursting throughput trap (critical for interviews):**

With Bursting mode, a 100 GiB EFS filesystem gets only 5 MiB/s baseline throughput (100 GiB x 50 KiB/s). It can burst to ~100 MiB/s using credits, but once credits deplete, it drops to 5 MiB/s -- often too slow for practical use. The legacy workaround was to **store dummy data** to inflate the filesystem size and increase baseline throughput. Elastic throughput mode eliminated this problem entirely.

**Critical detail**: The Bursting baseline is calculated from **Standard storage class data only** (the `ValueInStandard` metric). Data in IA and Archive classes does **not** contribute to the baseline. This creates a dangerous interaction with lifecycle management: as files tier out of Standard over time, the Bursting baseline silently shrinks. An 80 GiB filesystem where 90% of files move to IA drops from 4 MiB/s to 0.4 MiB/s baseline -- a progressive performance collapse with no errors, only throttling visible in CloudWatch. This is another reason Elastic throughput is essential: it decouples performance from storage class distribution entirely.

### EFS Storage Classes and Lifecycle Policies

EFS offers the same economic principle as S3 -- tiering data by access frequency -- but implemented at the file level rather than the object level.

| Storage Class | Latency | Cost (us-east-1 approx.) | Best For |
|---|---|---|---|
| **Standard** | Single-digit ms | ~$0.30/GB-month | Frequently accessed files |
| **Infrequent Access (IA)** | Low double-digit ms | ~$0.025/GB-month + per-access fee | Files not accessed for 30+ days |
| **Archive** | Low double-digit ms | ~$0.008/GB-month + per-access fee | Files accessed a few times per year |
| **One Zone Standard** | Single-digit ms | ~$0.16/GB-month | Frequently accessed, recreatable data (single AZ) |
| **One Zone IA** | Low double-digit ms | ~$0.013/GB-month + per-access fee | Infrequently accessed, recreatable data (single AZ) |

**Lifecycle policies** automatically transition files between tiers based on access recency:
- Default: Move to IA after 30 days without access, Archive after 90 days
- Configurable: 1, 7, 14, 30, 60, 90, 180, 270, or 365 days
- Files smaller than **128 KB are never transitioned** -- the same threshold as S3 lifecycle transitions you studied yesterday
- Files can automatically move **back to Standard** when accessed (transition back on first access)

```
EFS STORAGE CLASS TIERING
════════════════════════════════════════════════════════════════

  File created ──▶ Standard (or One Zone Standard)
                      │
                      │ Not accessed for 30 days (configurable)
                      ▼
                   Infrequent Access
                      │
                      │ Not accessed for 90 days (configurable)
                      ▼
                   Archive
                      │
                      │ File accessed ──▶ Automatically moves
                      │                   back to Standard
                      │                   (transition on access)
                      │
                   NOTE: Files < 128 KB always stay in Standard
```

### EFS Terraform Example

```hcl
# EFS filesystem with lifecycle tiering and cross-AZ access
resource "aws_efs_file_system" "shared" {
  creation_token = "shared-app-data"
  encrypted      = true
  kms_key_id     = aws_kms_key.efs.arn

  performance_mode = "generalPurpose"
  throughput_mode  = "elastic"

  # Lifecycle tiering: IA after 30 days, Archive after 90 days
  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }

  lifecycle_policy {
    transition_to_archive = "AFTER_90_DAYS"
  }

  # Transition back to Standard on first access
  lifecycle_policy {
    transition_to_primary_storage_class = "AFTER_1_ACCESS"
  }

  tags = {
    Name = "shared-app-data"
  }
}

# Mount target in each subnet (one per AZ)
resource "aws_efs_mount_target" "az" {
  for_each = toset(var.private_subnet_ids)

  file_system_id  = aws_efs_file_system.shared.id
  subnet_id       = each.value
  security_groups = [aws_security_group.efs.id]
}

# Security group: allow NFS (port 2049) from application instances
resource "aws_security_group" "efs" {
  name_prefix = "efs-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 2049
    to_port         = 2049
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}

# EFS access point for containerized workloads (ECS/EKS)
resource "aws_efs_access_point" "app" {
  file_system_id = aws_efs_file_system.shared.id

  posix_user {
    uid = 1000
    gid = 1000
  }

  root_directory {
    path = "/app-data"
    creation_info {
      owner_uid   = 1000
      owner_gid   = 1000
      permissions = "0755"
    }
  }

  tags = {
    Name = "app-access-point"
  }
}
```

---

## Part 4: FSx -- The Specialty Studios

FSx provides four managed file systems, each purpose-built for a specific workload family. The decision tree is straightforward: **match the protocol and workload pattern to the FSx variant**.

### FSx for Windows File Server -- The Government Filing Department

**The analogy**: If your organization runs Windows workloads that expect SMB file shares, Active Directory authentication, DFS Namespaces, and Windows ACLs, FSx for Windows File Server is the branch of the city's filing department that speaks the government's language. Windows applications connect to it exactly as they would to an on-premises Windows file server -- zero code changes.

**Key capabilities:**
- **SMB protocol** (Server Message Block) -- the native Windows file sharing protocol
- **Active Directory integration** -- join to AWS Managed Microsoft AD or self-managed AD; user permissions flow through existing AD group policies
- **DFS Namespaces** -- present multiple file systems under a single namespace, scaling to hundreds of petabytes
- **Shadow copies** -- Windows-native point-in-time snapshots accessible via the "Previous Versions" tab in Windows Explorer
- **Data deduplication** -- reduces storage consumption for datasets with repeated data patterns
- **Single-AZ and Multi-AZ** deployment options -- Multi-AZ provides automatic failover with an active and standby file server in different AZs
- **Storage types**: SSD (sub-ms latency) or HDD (single-digit ms latency, lower cost) with optional SSD cache for HDD deployments

**When to choose FSx for Windows:**
- Windows application migration from on-premises (IIS, SQL Server file shares, .NET applications)
- Home directories and department shares for Windows users authenticated via AD
- Any workload requiring SMB protocol -- EFS does not support SMB

### FSx for Lustre -- The High-Speed Factory Floor

**The analogy**: Imagine a factory where hundreds of assembly line workers need raw materials simultaneously. A standard library (EFS) could not deliver materials fast enough -- the checkout counter would be a bottleneck. FSx for Lustre is a **high-speed assembly line** that feeds materials to hundreds of workers in parallel at up to 1,000+ GBps aggregate throughput.

**The S3 integration -- the killer feature:**

FSx for Lustre can link to an S3 bucket via a **data repository association**. When the filesystem is created, it lazily loads S3 object metadata (file names, sizes, permissions) as filesystem entries. When a compute node reads a file, Lustre fetches the actual object data from S3 on first access. Results can be written back to S3 via data repository tasks. This means S3 is the **durable data layer** and Lustre is the **high-performance compute layer** -- you do not need to maintain two copies of your data.

```
FSx FOR LUSTRE + S3 DATA FLOW
════════════════════════════════════════════════════════════════

  ┌──────────────┐     Data Repository      ┌──────────────┐
  │              │     Association           │              │
  │   S3 Bucket  │◀────────────────────────▶│  FSx Lustre  │
  │  (Durable    │  1. Metadata lazy-loaded  │  (High-perf  │
  │   data lake) │     at FS creation        │   compute    │
  │              │  2. Data fetched on first  │   layer)     │
  │              │     access by compute node │              │
  │              │  3. Results exported back   │              │
  │              │     to S3 via tasks         │              │
  └──────────────┘                           └──────┬───────┘
                                                    │
                                              ┌─────┴─────┐
                                              │           │
                                         ┌────┴──┐  ┌────┴──┐
                                         │ EC2   │  │ EC2   │
                                         │ HPC   │  │ HPC   │  ... hundreds
                                         │ node  │  │ node  │      of nodes
                                         └───────┘  └───────┘
```

**Deployment types -- scratch vs persistent:**

| Feature | Scratch | Persistent 2 (Current Gen) | Persistent 1 (Previous Gen) |
|---|---|---|---|
| **Analogy** | Temporary assembly line for a one-day production run -- tear it down when done | Permanent factory floor with backup generators | Older factory floor |
| **Data replication** | None | Within the AZ | Within the AZ |
| **Failover** | Data lost if file server fails | Automatic to replacement server (minutes) | Automatic |
| **Burst throughput** | 200 MBps/TiB with 6x burst | 125-1,000 MBps/TiB (varies by storage) | 50-200 MBps/TiB |
| **Storage classes** | SSD only | SSD, HDD, Intelligent-Tiering | SSD, HDD |
| **Use case** | Short-lived HPC jobs where source data is in S3 | Long-running workloads requiring durability | Legacy |

**When to choose FSx for Lustre:**
- HPC (genomics, weather modeling, financial simulations)
- ML training with massive datasets stored in S3
- Media rendering pipelines needing hundreds of GBps throughput
- Any POSIX workload needing throughput that EFS cannot match

### FSx for NetApp ONTAP -- The Universal Translator

**The analogy**: Most file systems speak one language (NFS or SMB). FSx for NetApp ONTAP is a **universal translator filing system** that speaks NFS, SMB, iSCSI, and NVMe over TCP simultaneously. A Linux application and a Windows application can access the **same data** through their native protocols. On top of that, it has a built-in photocopier (SnapMirror), automatic basement storage (tiering), and an instant carbon-copy machine (FlexClone).

**Key capabilities:**

| Feature | Description |
|---|---|
| **Multi-protocol** | NFS + SMB + iSCSI + NVMe over TCP -- the only FSx type supporting all four simultaneously |
| **Automatic tiering** | Two-tier storage: high-performance SSD primary + elastic capacity pool (cheaper); data auto-tiers based on access patterns, similar to S3 Intelligent-Tiering |
| **SnapMirror** | Cross-region or cross-account replication with RPO as low as 5 minutes; the NetApp equivalent of S3 CRR |
| **FlexClone** | Instant, space-efficient clones using copy-on-write; a 10 TiB dataset clone consumes zero additional space until data diverges; ideal for dev/test |
| **Deduplication + Compression** | Can reduce storage consumption by 65%+ for many workloads |
| **SnapLock** | WORM compliance (like S3 Object Lock) for financial and regulatory requirements |
| **Multi-AZ** | Active-standby deployment with automatic failover across AZs |
| **FlexGroup volumes** | Scale a single volume across multiple aggregates for up to 300 PiB capacity |

**When to choose FSx for NetApp ONTAP:**
- Migrating from on-premises NetApp -- applications require zero code changes
- Multi-protocol access needed (Linux NFS + Windows SMB to the same data)
- Need automatic SSD-to-capacity-pool tiering (reduces cost for mixed access patterns)
- Need SnapMirror for cross-region DR with tight RPO
- Dev/test environments that benefit from FlexClone (instant zero-copy clones)

### FSx for OpenZFS -- The Linux Craftsman's Workshop

**The analogy**: A Linux woodworking shop with built-in instant-snapshot cameras (photograph your project state, revert if you make a mistake), a sawdust compressor (data compression), and the ability to clone an entire workshop setup in seconds without needing extra materials.

**Key capabilities:**
- **NFS protocol** -- Linux-native
- **Snapshots** -- point-in-time, instant, space-efficient
- **Clones** -- instant zero-copy clones from snapshots (like FlexClone but without the NetApp ecosystem)
- **Compression** -- LZ4 or ZSTD, reducing storage consumption
- **Up to 1 million IOPS** -- suitable for low-latency workloads
- **No multi-protocol** -- NFS only (unlike ONTAP)

**When to choose FSx for OpenZFS:**
- Migrating Linux workloads from on-premises ZFS
- Need powerful snapshot and clone capabilities without the full NetApp feature set
- NFS-only is sufficient (no SMB or iSCSI requirement)
- Want ZFS-style compression and data management in a managed service

### FSx Decision Framework

```
FSx VARIANT DECISION TREE
════════════════════════════════════════════════════════════════

  What protocol(s) do you need?
       │
       ├── SMB (Windows)
       │     └──▶ FSx for Windows File Server
       │           (AD integration, DFS Namespaces, shadow copies)
       │
       ├── POSIX with massive parallel throughput (100s GBps)
       │     └──▶ FSx for Lustre
       │           (HPC, ML training, S3 data repository)
       │
       ├── Multiple protocols (NFS + SMB + iSCSI)
       │     └──▶ FSx for NetApp ONTAP
       │           (multi-protocol NAS, SnapMirror, FlexClone)
       │
       ├── NFS only, need ZFS features (snapshots, clones, compression)
       │     │
       │     ├── Migrating from on-premises NetApp?
       │     │     YES ──▶ FSx for NetApp ONTAP
       │     │     NO  ──▶ FSx for OpenZFS
       │     │
       │
       └── NFS only, general shared file access
             │
             ├── Need auto-scaling, serverless integration, simplicity?
             │     ──▶ EFS (not FSx -- simpler, fully elastic)
             │
             └── Need specific ZFS or NetApp features?
                   ──▶ FSx for OpenZFS or NetApp ONTAP
```

---

## Part 5: The Grand Storage Selection Decision Framework

This is the master decision tree that ties together everything: EBS (yesterday's EC2 deep dives), EFS, FSx, and S3 (yesterday's object storage deep dive).

```
AWS STORAGE SELECTION -- MASTER DECISION FRAMEWORK
════════════════════════════════════════════════════════════════════════════

  What is the access pattern?
       │
       ├── BLOCK (raw disk device for OS, database, application state)
       │     │
       │     ├── Single instance needs a disk?
       │     │     ──▶ EBS
       │     │         gp3 (default), io2 (extreme), st1/sc1 (sequential HDD)
       │     │
       │     ├── Shared block across instances (same AZ, cluster-aware FS)?
       │     │     ──▶ EBS io2 Block Express with Multi-Attach
       │     │
       │     └── Highest possible I/O (millions of IOPS, ephemeral OK)?
       │           ──▶ EC2 Instance Store (not EBS -- data lost on stop)
       │
       ├── FILE (shared filesystem with directory/file semantics)
       │     │
       │     ├── NFS, shared across AZs, auto-scaling, simple?
       │     │     ──▶ EFS
       │     │
       │     ├── SMB / Windows / Active Directory?
       │     │     ──▶ FSx for Windows File Server
       │     │
       │     ├── Massive parallel throughput (HPC, ML, rendering)?
       │     │     ──▶ FSx for Lustre
       │     │
       │     ├── Multi-protocol (NFS + SMB + iSCSI), on-prem NetApp migration?
       │     │     ──▶ FSx for NetApp ONTAP
       │     │
       │     └── NFS + ZFS features (snapshots, compression, clones)?
       │           ──▶ FSx for OpenZFS
       │
       └── OBJECT (HTTP API, unstructured data, infinite scale)
             │
             └── ──▶ S3
                     (8 storage classes, lifecycle tiering, replication,
                      Object Lock -- see yesterday's deep dive)
```

### Quick Comparison Table

| Feature | EBS | EFS | FSx Windows | FSx Lustre | FSx ONTAP | FSx OpenZFS | S3 |
|---|---|---|---|---|---|---|---|
| **Type** | Block | File (NFS) | File (SMB) | File (POSIX) | File (Multi) | File (NFS) | Object |
| **Protocol** | Block device | NFSv4.1 | SMB 2/3 | POSIX/Lustre | NFS+SMB+iSCSI+NVMe | NFS | HTTP/HTTPS |
| **Multi-AZ** | No (same AZ) | Yes (Regional) | Optional | No (single AZ) | Optional | Optional | Yes (11 9s) |
| **Scaling** | Manual (Elastic Volumes) | Automatic | Manual | Manual | Manual | Manual | Automatic |
| **Max throughput** | 4,000 MiB/s (io2) | 60 GiBps read | 12 GBps | 1,000+ GBps | 72 GBps | 12.5 GBps | N/A |
| **Shared access** | io2 Multi-Attach only | Thousands of clients | Yes | Thousands of nodes | Yes | Yes | Unlimited |
| **Lifecycle tiering** | No | Standard/IA/Archive | No | Intelligent-Tiering (Persistent 2) | SSD + Capacity Pool | No | 8 classes |
| **Encryption** | KMS (transparent) | KMS (transparent) | KMS | KMS | KMS | KMS | SSE-S3/KMS/C |
| **Backup** | Snapshots, DLM | AWS Backup | AWS Backup | S3 export | SnapMirror + Backup | AWS Backup | Versioning, CRR |
| **Cost model** | Per-GB + IOPS/throughput | Per-GB (class-dependent) | Per-GB + throughput | Per-GB (deployment type) | Per-GB (tier-dependent) | Per-GB + IOPS | Per-GB + requests |

---

## Complete Architecture: Multi-Tier Storage for a Production Application

```
PRODUCTION STORAGE ARCHITECTURE
════════════════════════════════════════════════════════════════════════════

  ┌─────────────────── us-east-1 ───────────────────────────────────┐
  │                                                                  │
  │  ┌── Application Tier ──────────────────────────────────────┐   │
  │  │                                                          │   │
  │  │  ┌─── AZ-a ───────────────┐  ┌─── AZ-b ───────────────┐│   │
  │  │  │                        │  │                        │ │   │
  │  │  │  EC2 (web server)      │  │  EC2 (web server)      │ │   │
  │  │  │  ├── gp3 boot vol      │  │  ├── gp3 boot vol      │ │   │
  │  │  │  └── EFS mount ────────┼──┼──┘  (shared content)   │ │   │
  │  │  │                        │  │                        │ │   │
  │  │  │  EC2 (database primary)│  │  EC2 (database replica) │ │   │
  │  │  │  ├── io2 data volume   │  │  ├── io2 data volume   │ │   │
  │  │  │  │   (64K IOPS, 1 GBps)│  │  │   (32K IOPS)        │ │   │
  │  │  │  └── gp3 WAL volume    │  │  └── gp3 WAL volume    │ │   │
  │  │  │      (sequential write) │  │                        │ │   │
  │  │  └────────────────────────┘  └────────────────────────┘ │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                  │
  │  ┌── Shared File Storage ───────────────────────────────────┐   │
  │  │                                                          │   │
  │  │  EFS (shared-app-data)                                   │   │
  │  │  ├── Performance: General Purpose                        │   │
  │  │  ├── Throughput: Elastic                                 │   │
  │  │  ├── Lifecycle: Standard ─30d─▶ IA ─90d─▶ Archive       │   │
  │  │  ├── Mount targets in each AZ subnet                     │   │
  │  │  └── Encrypted with KMS CMK                              │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                  │
  │  ┌── HPC / ML Pipeline ─────────────────────────────────────┐   │
  │  │                                                          │   │
  │  │  FSx for Lustre (Persistent 2, SSD)                      │   │
  │  │  ├── S3 data repository: s3://ml-training-data           │   │
  │  │  ├── 200 MBps/TiB throughput                             │   │
  │  │  └── Results exported back to S3 via data repository task│   │
  │  │                                                          │   │
  │  │  EC2 p5 instances (GPU) ──Lustre client──▶ Lustre FS         │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                  │
  │  ┌── Windows Enterprise ────────────────────────────────────┐   │
  │  │                                                          │   │
  │  │  FSx for Windows File Server (Multi-AZ, SSD)             │   │
  │  │  ├── Joined to AWS Managed Microsoft AD                  │   │
  │  │  ├── DFS Namespace: \\corp.example.com\shares            │   │
  │  │  └── Shadow copies enabled for self-service restore      │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                  │
  │  ┌── Object Storage (yesterday's deep dive) ────────────────┐   │
  │  │                                                          │   │
  │  │  S3 Buckets                                              │   │
  │  │  ├── app-assets: Static content via CloudFront OAC       │   │
  │  │  ├── ml-training-data: Linked to FSx Lustre              │   │
  │  │  ├── db-backups: Lifecycle to Glacier after 90d          │   │
  │  │  └── compliance-records: Object Lock, Compliance Mode    │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                  │
  │  ┌── Backup & DR ──────────────────────────────────────────┐    │
  │  │                                                          │   │
  │  │  EBS Snapshots (DLM: daily, 30-day retention)            │   │
  │  │  ├── Cross-region copy to eu-west-1 (Pilot Light DR)     │   │
  │  │  └── Fast Snapshot Restore on database volumes            │   │
  │  │                                                          │   │
  │  │  AWS Backup                                              │   │
  │  │  ├── EFS daily backups                                   │   │
  │  │  ├── FSx Windows daily backups                           │   │
  │  │  └── Cross-region vault copy                             │   │
  │  │                                                          │   │
  │  │  FSx ONTAP SnapMirror (if used)                          │   │
  │  │  └── Cross-region replication, 5-min RPO                 │   │
  │  └──────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **gp3 is the default EBS volume type for nearly every workload**. It decouples IOPS from volume size, includes 3,000 IOPS and 125 MiB/s free, and is 20% cheaper per-GB than gp2. Always consider migrating gp2 volumes to gp3 via Elastic Volumes (zero-downtime, zero-detach).

- **IOPS vs throughput is the fundamental EBS distinction**: Databases doing thousands of small random reads need high IOPS (SSD: gp3 or io2). Data warehouses scanning terabytes sequentially need high throughput (HDD: st1). Confusing the two is the most common interview mistake.

- **HDD volumes (st1, sc1) cannot be boot volumes**. The OS always boots from SSD (gp3 or io2).

- **io2 Block Express is the only EBS type supporting Multi-Attach** (up to 16 Nitro instances, same AZ). It requires a cluster-aware filesystem -- standard ext4/XFS will corrupt data with concurrent writers.

- **You cannot encrypt an existing unencrypted EBS volume in place**. The path is: snapshot, copy with encryption, create new volume. Enable default encryption per-region to prevent this problem proactively.

- **AWS recommends only RAID 0 on EBS**. RAID 5/6 waste IOPS on parity writes. RAID 1 wastes bandwidth. EBS already replicates within the AZ for durability.

- **EFS Elastic throughput mode eliminated the Bursting trap** where small filesystems got throttled. Use Elastic (default) for most workloads, Provisioned for sustained predictable throughput, and avoid Bursting for new file systems.

- **EFS files under 128 KB are never transitioned** to IA or Archive -- the same threshold as S3 lifecycle policies. EFS can automatically transition files back to Standard on first access, unlike S3 where you must restore and copy.

- **FSx variant selection is protocol-driven**: SMB + Active Directory = Windows File Server. Massive parallel throughput + S3 bridge = Lustre. Multi-protocol NAS + NetApp features = ONTAP. NFS + ZFS features = OpenZFS. General NFS shared access = EFS (not FSx).

- **FSx for Lustre's S3 data repository association is the HPC secret weapon**: It presents S3 as a POSIX filesystem with lazy data loading, letting you use S3 as the durable layer and Lustre as the compute-speed layer without duplicating data.

- **FSx for NetApp ONTAP is the only AWS file service supporting four protocols simultaneously** (NFS, SMB, iSCSI, NVMe over TCP). It is the migration path for on-premises NetApp with zero application code changes.

- **The master storage decision starts with access pattern**: block (single-instance disk) = EBS, file (shared filesystem) = EFS or FSx, object (HTTP API, infinite scale) = S3. Instance store beats all of these for raw performance but data is ephemeral.

- **EBS snapshots are incremental but independently restorable** -- you do not need the full snapshot chain. Fast Snapshot Restore eliminates first-access latency but costs ~$0.75/snapshot-AZ/hour. Snapshot Archive provides 75% cost savings for long-term retention at the cost of 24-72 hour restore times.

---

## Further Reading

- [EBS Volume Types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html) -- Complete specification tables for all six volume types
- [EBS Snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html) -- Incremental snapshots, FSR, Snapshot Archive, and DLM
- [EBS RAID Configuration](https://docs.aws.amazon.com/ebs/latest/userguide/raid-config.html) -- Why only RAID 0 is recommended
- [EFS Performance](https://docs.aws.amazon.com/efs/latest/ug/performance.html) -- Performance modes and throughput modes
- [EFS Storage Classes](https://docs.aws.amazon.com/efs/latest/ug/lifecycle-management-efs.html) -- Standard, IA, Archive, and lifecycle policies
- [FSx Choosing Guide](https://aws.amazon.com/fsx/when-to-choose-fsx/) -- Decision matrix for all four FSx types
- [FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/using-fsx-lustre.html) -- Deployment types and S3 integration
- [FSx for NetApp ONTAP](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/what-is-fsx-ontap.html) -- Multi-protocol NAS and data management
- Related docs in this repo: [S3 Advanced Deep Dive](2026-03-23-s3-advanced-deep-dive.md), [KMS & Encryption](2026-03-09-aws-kms-encryption-deep-dive.md), [EC2 Instance Types](../../february/aws/2026-02-05-ec2-instance-types-placement-tenancy.md), [VPC Advanced](../../february/aws/2026-02-19-vpc-advanced-cidr-subnet-strategy.md)
