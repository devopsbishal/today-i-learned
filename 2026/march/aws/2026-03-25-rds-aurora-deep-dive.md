# RDS & Aurora Deep Dive -- The Hotel Chain Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock), and equipped your instances with personal workbenches, shared libraries, and specialty studios (EBS, EFS, FSx). But where does the **structured, queryable data** live? Customer orders, financial transactions, user profiles, inventory records -- data that must support complex queries, enforce referential integrity, participate in ACID transactions, and survive failures without losing a single committed row. You could run PostgreSQL or MySQL on an EC2 instance with an EBS volume (and you learned yesterday that RDS uses EBS under the hood for standard deployments), but then **you** are responsible for patching, backups, replication, failover, monitoring, and storage scaling -- the undifferentiated heavy lifting that distracts from building your application. That is the domain of RDS and Aurora -- managed relational database services where AWS handles the operational burden while you focus on schema design and query optimization. And Aurora takes this further by rethinking the database storage layer itself, replacing traditional replication with a distributed, quorum-based architecture that fundamentally changes the performance and resilience equation.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| RDS (Relational Database Service) | A managed hotel chain -- you choose the room style (engine) and room size (instance class), and the chain handles housekeeping, maintenance, and security | Managed relational database supporting six engines (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2), running on EC2 instances with EBS storage, with AWS handling patching, backups, and failover |
| RDS Multi-AZ (Classic) | A hotel with a shadow room that mirrors yours exactly -- if your room floods, the front desk instantly redirects you to the shadow room in the other building | Synchronous replication to a single standby instance in a different AZ; standby cannot serve reads; automatic DNS failover in ~60 seconds; purely an HA mechanism |
| RDS Multi-AZ DB Cluster | A hotel with two shadow rooms in two other buildings -- each shadow room has a reading lamp so guests waiting in the lobby can browse magazines there while the main room handles check-ins | Two readable standby instances across 3 AZs using transaction log replication; standbys serve read traffic via a reader endpoint; failover typically under 35 seconds; MySQL and PostgreSQL only |
| RDS Read Replicas | Opening a chain of read-only branch libraries that photocopy new books from the main library -- copies arrive slightly delayed, but thousands of readers can browse without crowding the main building | Asynchronous replication creating read-only copies for horizontal read scaling; up to 15 per source (both RDS and Aurora); can be cross-region; promotable to standalone |
| Aurora Architecture | A hotel built on a shared underground vault system -- instead of each room having its own safe, all rooms connect to the same distributed vault network that keeps six copies of every valuables across three neighborhoods | Compute-storage separation: Aurora instances (compute) connect to a shared distributed storage layer that replicates data 6 ways across 3 AZs in 10 GB segments with quorum-based reads/writes |
| Aurora 4/6 Write Quorum | The vault needs four of six vault keepers to agree before locking away a new item -- even if an entire neighborhood goes dark (losing 2 keepers), the remaining 4 can still accept deposits | Writes succeed when 4 of 6 storage nodes acknowledge; tolerates the loss of an entire AZ (2 nodes) and still writes; guarantees durability before acknowledging the commit |
| Aurora 3/6 Read Quorum | The vault needs three of six keepers to agree on what is stored -- even if a neighborhood plus one extra keeper is unavailable, three keepers can still retrieve items | Reads succeed when 3 of 6 storage nodes agree; tolerates AZ loss plus one additional node failure and still reads; overlaps with write quorum to guarantee consistency |
| Aurora I/O-Optimized | Switching from a per-use parking meter (pay each time you park) to an unlimited monthly parking pass -- predictable cost when you park frequently | Storage pricing model with no per-I/O charges; 30% higher storage cost but eliminates unpredictable I/O billing; cost-effective when I/O exceeds 25% of total Aurora spend |
| Aurora Serverless v2 | A co-working space that opens and closes desks as workers arrive -- during a conference every desk is occupied, on a quiet Sunday only two desks have lights on, and on holidays the entire floor goes dark and costs nothing until someone walks in | Auto-scaling Aurora compute measured in ACUs (0-256 range); can scale to 0 ACU with auto-pause/resume; scales in 0.5 ACU increments; each instance scales independently; can be mixed with provisioned instances in the same cluster |
| Aurora Global Database | A hotel chain that uses a dedicated underground tunnel to sync its vault to branch locations in other countries -- guests in Tokyo read from the local vault, kept in sync with sub-second replication lag from headquarters, while all deposits route to the headquarters vault in Virginia | Storage-level cross-region replication with typically <1s lag; up to 10 secondary regions; managed planned switchover vs unplanned failover; write forwarding from secondary to primary |
| Aurora Cloning | Photocopying a hotel's guest registry by pointing two managers at the same original pages -- only when one manager edits a page does a physical copy of that page get made | Copy-on-write cloning that completes in minutes regardless of database size; source and clone share data pages until modification; up to 15 clones per source |
| RDS Proxy | A concierge desk in the hotel lobby that holds a pool of room keys -- instead of each guest getting their own key made (expensive), the concierge hands out pre-made keys from the pool and recollects them when guests leave | Managed connection pool sitting between applications and RDS/Aurora; multiplexes thousands of application connections into a smaller set of database connections; improves failover time; integrates with IAM and Secrets Manager |

---

## The Big Picture: RDS as a Managed Hotel Chain, Aurora as the Revolutionary Vault

The best way to understand the RDS-Aurora relationship is to think of two generations of hotel architecture.

**RDS is a traditional managed hotel chain.** You pick the hotel brand (engine: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2), the room size (instance class: db.t3.micro through db.r7g.16xlarge), and the safe type (storage: gp3, io2, or magnetic). The hotel chain (AWS) handles housekeeping (patching), the backup cameras (automated backups), the fire alarm system (monitoring), and the emergency relocation plan (Multi-AZ failover). But under the hood, each hotel room has its own private safe (EBS volume), and when you need a branch library for readers (Read Replica), the entire safe contents must be photocopied to a new location via asynchronous replication.

**Aurora is a next-generation hotel built on a shared vault network.** Instead of each room containing its own safe, Aurora constructs a distributed underground vault system that spans three neighborhoods (Availability Zones). Every piece of valuables (data) is stored in six copies across the three neighborhoods. The hotel rooms (compute instances) are just comfortable workspaces that connect to the vault via high-speed tunnels. Adding a new room for readers (Aurora Replica) does not require photocopying the vault -- the new room simply gets its own tunnel to the same shared vault. This is the fundamental architectural insight: Aurora separates compute from storage, and the storage layer handles replication independently of the database engine.

```
RDS vs AURORA ARCHITECTURE -- THE FUNDAMENTAL DIFFERENCE
════════════════════════════════════════════════════════════════════════════

  TRADITIONAL RDS                          AURORA
  ─────────────                            ─────
  Each instance has its own EBS volume.    All instances share one distributed
  Replication = engine copies data.        storage volume. No engine replication.

  ┌──────────────┐    async repl    ┌──────────────┐
  │ Primary      │ ───────────────▶ │ Read Replica │
  │ (compute)    │    (engine       │ (compute)    │
  │ ┌──────────┐ │     level)       │ ┌──────────┐ │
  │ │ EBS vol  │ │                  │ │ EBS vol  │ │
  │ │ (own     │ │                  │ │ (own     │ │
  │ │  copy)   │ │                  │ │  copy)   │ │
  │ └──────────┘ │                  │ └──────────┘ │
  └──────────────┘                  └──────────────┘

  ┌──────────────┐   ┌──────────────┐
  │ Writer       │   │ Reader(s)    │   ◄── Compute only,
  │ (compute)    │   │ (compute)    │       no local storage
  └──────┬───────┘   └──────┬───────┘
         │                  │
         │    shared        │
         └────────┬─────────┘
                  ▼
  ┌──────────────────────────────────────────────────┐
  │          Aurora Shared Storage Volume             │
  │                                                  │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐        │
  │  │10 GB │  │10 GB │  │10 GB │  │10 GB │  ...   │
  │  │ seg  │  │ seg  │  │ seg  │  │ seg  │        │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘        │
  │     │ 6 copies │ 6 copies│ 6 copies│            │
  │     ▼ across   ▼ across  ▼ across  ▼            │
  │    3 AZs      3 AZs    3 AZs    3 AZs          │
  └──────────────────────────────────────────────────┘
```

---

## Part 1: RDS Fundamentals -- Choosing Your Hotel Brand and Room Size

### Supported Engines (The Hotel Brands)

RDS supports six database engines. Aurora supports only two (MySQL and PostgreSQL). This distinction is the first and most common decision fork.

| Engine | Versions | Aurora Support | Key Use Case |
|--------|----------|---------------|-------------|
| **MySQL** | 5.7, 8.0, 8.4 | Yes (Aurora MySQL) | Open-source web applications, WordPress, general OLTP |
| **PostgreSQL** | 13-17 | Yes (Aurora PostgreSQL) | Advanced features (JSON, GIS, full-text search), analytics-heavy OLTP |
| **MariaDB** | 10.4-11.x | No | MySQL fork with additional storage engines, licensing preference |
| **Oracle** | 19c, 21c | No | Enterprise applications with Oracle-specific PL/SQL, migration from on-prem |
| **SQL Server** | 2016-2022 | No | .NET applications, Windows enterprise, SSRS/SSIS |
| **Db2** | 11.5 | No | IBM mainframe modernization, legacy enterprise |

**Critical point**: If you need Oracle, SQL Server, MariaDB, or Db2 -- Aurora is not an option. The engine choice immediately determines whether Aurora is on the table.

### Instance Classes (The Room Sizes)

RDS instance classes follow the same naming convention as EC2 but with a `db.` prefix:

| Class Family | Purpose | Example | Memory | vCPUs |
|---|---|---|---|---|
| **db.t3/t4g** | Burstable, dev/test, small workloads | db.t3.medium | 4 GiB | 2 |
| **db.m5/m6g/m7g** | General purpose, balanced compute-memory | db.m6g.xlarge | 16 GiB | 4 |
| **db.r5/r6g/r7g** | Memory-optimized, large working sets | db.r7g.4xlarge | 128 GiB | 16 |
| **db.x2g** | Memory-intensive, very large working sets and caches | db.x2g.16xlarge | 1024 GiB | 64 |

**The "g" suffix** (e.g., db.r6**g**) indicates ARM-based Graviton processors -- up to 20% better price-performance than Intel equivalents and now the default recommendation for most workloads.

### Storage Types (The Safe Types)

Standard RDS (non-Aurora) uses EBS volumes under the hood. The same volume types you learned yesterday apply:

| Storage Type | IOPS | Throughput | Use Case |
|---|---|---|---|
| **gp3** (default) | 3,000 baseline, up to 80,000 | 125 MiB/s baseline, up to 2,000 MiB/s | Most workloads -- decoupled IOPS and throughput |
| **io2 Block Express** | Up to 256,000 | Up to 4,000 MiB/s | I/O-intensive databases needing guaranteed IOPS |
| **Magnetic** (legacy) | ~100 | ~100 MiB/s | Legacy -- do not use for new workloads |

**Aurora does not use EBS.** Aurora has its own purpose-built distributed storage layer that auto-scales from 10 GiB to **256 TiB** without manual provisioning. You never choose a storage type or size for Aurora -- it just grows.

---

## Part 2: RDS Multi-AZ -- The Shadow Room System

Multi-AZ is RDS's high availability mechanism. It is **not** a read-scaling solution (with one important exception). There are two deployment types, and confusing them is a common exam pitfall.

### Classic Multi-AZ DB Instance (One Shadow Room)

**The analogy**: Your hotel room has a shadow room in a different building. A hidden crew synchronously mirrors everything you put in your room's safe to the shadow room's safe -- every transaction, every commit, in real time. You cannot visit the shadow room or use it for anything. It sits empty and locked. But the instant your room floods (the primary fails), the front desk (DNS endpoint) instantly redirects you to the shadow room, and you continue working as if nothing happened.

**Technical reality:**
- **Synchronous replication** from primary to a single standby in a different AZ
- Standby **cannot serve any traffic** -- not reads, not anything. It exists solely for failover
- Failover is automatic: RDS flips the DNS CNAME record to point at the standby
- **Failover time**: Typically 60-120 seconds (DNS propagation + recovery)
- **Zero data loss**: Because replication is synchronous, the standby has every committed transaction
- Supported for **all six engines** (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2)

**What triggers automatic failover:**
- Primary instance failure
- AZ outage
- Instance type change (planned maintenance)
- Manual failover (for testing: `aws rds reboot-db-instance --force-failover`)
- OS patching on the primary

```
RDS CLASSIC MULTI-AZ -- ONE SHADOW ROOM
════════════════════════════════════════════════════════════════

  Application
      │
      │  Connects via DNS endpoint:
      │  mydb.abc123.us-east-1.rds.amazonaws.com
      │
      ▼
  ┌────────────────────────────────────────────────────────┐
  │                  DNS CNAME (the front desk)            │
  │            Points to active instance                   │
  └────────────────────┬───────────────────────────────────┘
                       │
         ┌─────────────┴──────────────┐
         ▼                            ▼
  ┌──────────────┐  synchronous  ┌──────────────┐
  │   PRIMARY    │ ────────────▶ │   STANDBY    │
  │  (AZ-a)     │  replication   │  (AZ-b)     │
  │              │  (every write) │              │
  │  Serves ALL  │               │  Serves      │
  │  read+write  │               │  NOTHING     │
  │  traffic     │               │  (locked)    │
  │              │               │              │
  │  ┌────────┐  │               │  ┌────────┐  │
  │  │EBS vol │  │               │  │EBS vol │  │
  │  │(gp3)   │  │               │  │(copy)  │  │
  │  └────────┘  │               │  └────────┘  │
  └──────────────┘               └──────────────┘

  On failure: DNS flips to standby in ~60-120 seconds
              Zero data loss (synchronous)
```

### Multi-AZ DB Cluster (Two Readable Shadow Rooms)

**The analogy**: The newer hotel design has **two** shadow rooms in **two** different buildings, and each shadow room has a reading lamp so waiting guests can browse the hotel's catalog there. The main room handles all check-ins (writes), but the shadow rooms can handle guest inquiries (reads). The replication uses a faster, lighter system -- instead of mirroring the entire safe, the main room sends just the transaction log (a journal of what changed) to both shadow rooms.

**Technical reality:**
- **One writer + two reader instances** across **three AZs**
- Replication via **transaction log shipping** (semisynchronous -- the primary waits for at least one standby to acknowledge *receiving* the transaction log entry before confirming the commit to the client, but does not wait for the standby to fully *apply* it. This is faster than classic Multi-AZ's full synchronous block-level replication)
- Reader instances **can serve read traffic** via a dedicated **reader endpoint**
- **Failover time**: Typically under **35 seconds** (significantly faster than classic Multi-AZ)
- **Commit latency**: Up to **2x faster** than classic Multi-AZ because transaction log replication is lighter than full block-level synchronous replication
- **Currently supports only MySQL and PostgreSQL** (not Oracle, SQL Server, MariaDB, or Db2)

**The three endpoints:**
- **Cluster endpoint** (writer): Routes to the writer instance -- use for all writes and reads that need the latest data
- **Reader endpoint**: Load-balances across the two reader instances -- use for read-heavy queries that can tolerate milliseconds of replication lag
- **Instance endpoints**: Direct connection to a specific instance -- use for debugging or specialized routing

```
RDS MULTI-AZ DB CLUSTER -- TWO READABLE SHADOW ROOMS
════════════════════════════════════════════════════════════════

  Application
      │
      ├── writes ──▶ Cluster Endpoint (writer)
      │                     │
      └── reads ───▶ Reader Endpoint (load-balanced)
                            │
         ┌──────────────────┼──────────────────┐
         ▼                  ▼                  ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   WRITER     │  │   READER 1   │  │   READER 2   │
  │  (AZ-a)     │  │  (AZ-b)     │  │  (AZ-c)     │
  │              │  │              │  │              │
  │  read+write  │  │  read-only   │  │  read-only   │
  │              │  │              │  │              │
  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
  │  │EBS vol │  │  │  │EBS vol │  │  │  │EBS vol │  │
  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │
  └──────┬───────┘  └──────▲───────┘  └──────▲───────┘
         │                 │                 │
         │  Transaction    │                 │
         └─── log ─────────┴─────────────────┘
              shipping (semisynchronous)

  Failover: ~35 seconds (reader promoted to writer)
  Engines: MySQL and PostgreSQL only
```

### Classic Multi-AZ vs Multi-AZ DB Cluster Comparison

| Feature | Classic Multi-AZ (DB Instance) | Multi-AZ DB Cluster |
|---|---|---|
| **Standby count** | 1 | 2 |
| **Standbys serve reads?** | No | Yes (reader endpoint) |
| **Replication method** | Synchronous block-level | Semisynchronous transaction log |
| **Failover time** | ~60-120 seconds | ~35 seconds |
| **Commit latency** | Higher (full sync) | Lower (up to 2x faster) |
| **Engines supported** | All 6 | MySQL, PostgreSQL only |
| **AZs used** | 2 | 3 |
| **Use case** | HA for any engine | HA + read capacity for MySQL/PostgreSQL |

---

## Part 3: RDS Read Replicas -- The Branch Library System

**The analogy**: Your hotel's main library (primary database) is overwhelmed with readers. Instead of turning guests away, you open branch libraries in other parts of the city -- or even in other cities entirely. The main library photocopies new books and sends them to each branch. The copies arrive slightly delayed (asynchronous -- the main library does not wait for branches to confirm receipt before accepting new books). Branches serve read-only traffic; if a guest at a branch wants to add a book, they must go to the main library. If the main library burns down, you can **promote** a branch to become the new main library -- but this is a manual, one-way operation that permanently breaks the photocopying link.

**Technical reality:**
- **Asynchronous replication**: The source DB streams changes to replicas without waiting for acknowledgment. This means replicas may lag behind the source by seconds to minutes depending on write volume and network conditions.
- **Read-only**: Replicas serve only SELECT queries. All writes must go to the source.
- **Up to 15 replicas per source** for all RDS engines (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server). This limit was increased from 5 -- older documentation and some exam prep materials may still reference the legacy 5-replica cap.
- **Cross-region replicas**: You can create a replica in a different AWS region for geographic read locality and DR.
- **Promotable**: A read replica can be promoted to a standalone DB instance. This breaks replication permanently -- the promoted instance becomes an independent, writable database.
- **Replica of a replica (daisy-chaining)**: For MySQL and MariaDB, you can create a replica of a replica. Each hop adds replication lag.
- **Supported engines**: MySQL, MariaDB, PostgreSQL, Oracle (with Active Data Guard license), SQL Server (limited)

### Read Replicas vs Multi-AZ -- The Critical Distinction

This comparison is one of the most frequently tested concepts in AWS certifications:

| Dimension | Multi-AZ (Classic) | Read Replicas |
|---|---|---|
| **Purpose** | High availability (automatic failover) | Horizontal read scaling |
| **Replication** | Synchronous | Asynchronous |
| **Serves traffic?** | No (standby is locked) | Yes (read-only) |
| **Cross-region?** | No (same region, different AZ) | Yes |
| **Failover** | Automatic (DNS flip) | Manual (promotion to standalone) |
| **Data loss risk** | Zero (synchronous) | Possible (async lag during promotion) |
| **Can combine?** | Yes -- a read replica can itself be Multi-AZ enabled | Yes |
| **Cost** | ~2x (you pay for the standby instance) | Per-replica instance cost + cross-region data transfer if applicable |

**The combination pattern**: You can make a cross-region read replica Multi-AZ-enabled. This gives you a read-scaling replica in another region that itself has a standby for HA -- combining both patterns for maximum resilience.

```
READ REPLICA PATTERNS
════════════════════════════════════════════════════════════════

  Pattern 1: Same-region read scaling
  ┌──────────┐   async   ┌──────────┐
  │ Primary  │ ────────▶ │ Replica 1│ ─── reads
  │ (writer) │ ────────▶ │ Replica 2│ ─── reads
  │          │ ────────▶ │ Replica 3│ ─── reads
  └──────────┘           └──────────┘
       │                 (up to 15 for RDS)
       │
  Multi-AZ standby
  (synchronous, no reads)

  Pattern 2: Cross-region DR + read locality
  ┌────────── us-east-1 ──────────┐    ┌────────── eu-west-1 ──────────┐
  │ ┌──────────┐                  │    │ ┌──────────────────┐          │
  │ │ Primary  │ ──── async ──────┼───▶│ │ Cross-Region     │          │
  │ │ (writer) │   replication    │    │ │ Read Replica     │          │
  │ └──────────┘                  │    │ │ (Multi-AZ enabled│          │
  │                               │    │ │  for HA)         │          │
  │                               │    │ └──────────────────┘          │
  └───────────────────────────────┘    └───────────────────────────────┘

  Pattern 3: Promotion for DR
  Primary fails ──▶ Promote cross-region replica to standalone
                    (manual, breaks replication, minutes of downtime)
```

### Terraform Example: RDS Multi-AZ with Read Replica

```hcl
# Primary RDS instance with Multi-AZ enabled
resource "aws_db_instance" "primary" {
  identifier     = "orders-db-primary"
  engine         = "postgres"
  engine_version = "16.4"
  instance_class = "db.r7g.xlarge"

  allocated_storage     = 100
  max_allocated_storage = 500  # Enable storage autoscaling up to 500 GiB
  storage_type          = "gp3"
  storage_throughput    = 250  # MiB/s (above 125 baseline)
  iops                  = 6000 # Above 3000 baseline

  db_name  = "orders"
  username = "admin"
  manage_master_user_password = true  # Secrets Manager integration

  # Multi-AZ: creates a synchronous standby (classic mode)
  multi_az = true

  # Encryption at rest with KMS (connects to your Mar 9 KMS knowledge)
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  # Backup configuration (lighter coverage -- deeper in Week 5)
  backup_retention_period = 14          # Days to retain automated backups
  backup_window           = "03:00-04:00"
  copy_tags_to_snapshot   = true

  # Maintenance window
  maintenance_window = "sun:05:00-sun:06:00"

  # Network
  db_subnet_group_name   = aws_db_subnet_group.private.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  # Enhanced monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn

  # Performance Insights (free for 7-day retention)
  performance_insights_enabled          = true
  performance_insights_retention_period = 7

  # Deletion protection
  deletion_protection = true

  tags = {
    Name        = "orders-db-primary"
    Environment = "production"
  }
}

# Read Replica in the same region
resource "aws_db_instance" "read_replica" {
  identifier     = "orders-db-replica-1"
  instance_class = "db.r7g.xlarge"

  # Key: replicate_source_db links this to the primary
  # For same-region: identifier works. For cross-region: use the source's ARN instead.
  replicate_source_db = aws_db_instance.primary.identifier

  # Replica inherits engine, storage type, encryption from source
  # You can override instance_class for a different size

  # Replica-specific: can enable Multi-AZ on the replica itself
  multi_az = false  # Set true for HA on the replica

  # Replicas get their own backup configuration
  backup_retention_period = 0  # Disable backups on replica (source handles it)

  # Same network placement
  vpc_security_group_ids = [aws_security_group.rds.id]

  performance_insights_enabled          = true
  performance_insights_retention_period = 7

  tags = {
    Name = "orders-db-replica-1"
    Role = "read-replica"
  }
}

# Subnet group: private subnets across AZs
resource "aws_db_subnet_group" "private" {
  name       = "orders-db-private"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name = "orders-db-private-subnets"
  }
}

# Security group: allow PostgreSQL (5432) from app layer only
resource "aws_security_group" "rds" {
  name_prefix = "rds-orders-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }

  # Default egress allows all outbound; restrict as needed for compliance
}
```

---

## Part 4: Aurora Architecture -- The Shared Vault Network

Aurora is Amazon's re-invention of the relational database. It is compatible with MySQL and PostgreSQL (you connect with the same drivers, run the same SQL), but the storage layer is completely different. This is not a marketing claim -- it is an architectural revolution that changes how replication, failover, scaling, and backups work.

### The Core Insight: Compute-Storage Separation

**The analogy**: In a traditional hotel (RDS), each room has its own safe, and replication means a crew physically copies the contents of one safe to another safe in another room. This is slow, bandwidth-intensive, and means each new room doubles the storage requirement.

Aurora builds a **shared underground vault network** beneath all the rooms. The vault spans three neighborhoods (AZs) and stores six copies of everything. Hotel rooms are just comfortable workspaces with high-speed tunnel access to the vault. When you want to add a new room for readers, you simply dig a new tunnel to the same vault -- zero data copying needed.

**What this means in practice:**
- **Adding a read replica takes minutes**, not hours, because no data is copied. The new compute instance connects to the existing shared storage.
- **Storage auto-scales** from 10 GiB to 256 TiB without provisioning. You never specify a volume size.
- **Replication lag between writer and readers is typically single-digit milliseconds** because readers read directly from the shared storage. They just need to apply redo log records to update their buffer caches -- no data transfer between instances.
- **Backups are continuous and incremental** -- Aurora does not need periodic snapshot windows because the storage layer continuously backs up data to S3.

### The Quorum System: 4/6 Write, 3/6 Read

This is the intellectual core of Aurora's architecture and a guaranteed interview question for any database-related role.

**The analogy**: The underground vault employs six vault keepers, two in each of the three neighborhoods (AZs). When you deposit something (write), you need four keepers to confirm they have locked it away before you get a receipt (write acknowledged). When you want to retrieve something (read), you need three keepers to agree on what is stored. Why these specific numbers? Because any group of four (write quorum) and any group of three (read quorum) must overlap by at least one keeper -- guaranteeing that every read includes data from the most recent write.

**The math that makes this work:**
- **6 copies across 3 AZs** (2 copies per AZ, stored in 10 GB segments)
- **Write quorum**: 4/6 -- writes succeed when 4 of 6 storage nodes acknowledge
- **Read quorum**: 3/6 -- reads succeed when 3 of 6 storage nodes agree
- **Overlap guarantee**: 4 + 3 = 7 > 6, so any write quorum and read quorum share at least one node, ensuring reads always see the latest committed data

**Failure tolerance:**
- **Lose an entire AZ** (2 nodes gone): 4 remain. Write quorum (4/6) still met. Read quorum (3/6) still met. **Full read+write availability.**
- **Lose an AZ + 1 more node** (3 nodes gone): 3 remain. Write quorum (4/6) NOT met. Read quorum (3/6) still met. **Read-only availability.**
- **Lose an AZ + 2 more nodes** (4 nodes gone): 2 remain. Neither quorum met. **Outage.** But this scenario is statistically near-impossible because of segment repair speed.

**Why 10 GB segments matter:**
Each 10 GB segment is independently replicated. On a 10 Gbps network, re-replicating a 10 GB segment to a healthy node takes under 10 seconds. This keeps the Mean Time To Repair (MTTR) so low that the probability of a second failure occurring during the repair window is negligible -- making the "AZ+2" catastrophic failure scenario astronomically unlikely.

**Peer-to-peer gossip repair**: Storage nodes use a gossip protocol to detect and repair inconsistencies without involving the database engine. If a node discovers it is missing a segment or has stale data, it pulls the update from peers. This self-healing runs continuously in the background.

```
AURORA QUORUM-BASED STORAGE
════════════════════════════════════════════════════════════════

  Writer Instance                    Reader Instance(s)
  (sends redo log                    (read from shared
   records to storage)                storage directly)
       │                                    │
       ▼                                    ▼
  ┌──────────────────────────────────────────────────────────┐
  │              Aurora Shared Storage Volume                  │
  │                                                          │
  │  ┌─── AZ-a ────┐  ┌─── AZ-b ────┐  ┌─── AZ-c ────┐    │
  │  │ ┌──┐  ┌──┐  │  │ ┌──┐  ┌──┐  │  │ ┌──┐  ┌──┐  │    │
  │  │ │S1│  │S2│  │  │ │S3│  │S4│  │  │ │S5│  │S6│  │    │
  │  │ └──┘  └──┘  │  │ └──┘  └──┘  │  │ └──┘  └──┘  │    │
  │  │  2 copies   │  │  2 copies   │  │  2 copies   │    │
  │  └─────────────┘  └─────────────┘  └─────────────┘    │
  │                                                          │
  │  Write: need 4/6 ack ──▶ tolerate AZ loss (lose 2)      │
  │  Read:  need 3/6 agree ─▶ tolerate AZ+1 loss (lose 3)   │
  │                                                          │
  │  Each segment = 10 GB                                    │
  │  Segment repair < 10 seconds on 10 Gbps network          │
  │  Gossip protocol: nodes self-heal without engine help     │
  └──────────────────────────────────────────────────────────┘

  FAILURE SCENARIOS:
  ═══════════════════════════════════════
  AZ-b fails (lose S3, S4):
    Remaining: S1, S2, S5, S6 (4 nodes)
    Write quorum 4/6: ✓ (4 available)
    Read quorum 3/6:  ✓ (4 available)
    Result: FULL READ+WRITE ✓

  AZ-b fails + S5 fails (lose 3):
    Remaining: S1, S2, S6 (3 nodes)
    Write quorum 4/6: ✗ (only 3)
    Read quorum 3/6:  ✓ (3 available)
    Result: READ-ONLY ✓

  AZ-b fails + S5 + S6 fail (lose 4):
    Remaining: S1, S2 (2 nodes)
    Write quorum 4/6: ✗
    Read quorum 3/6:  ✗
    Result: OUTAGE (astronomically unlikely)
```

### Aurora Read Replicas vs RDS Read Replicas

| Feature | Aurora Read Replica | RDS Read Replica |
|---|---|---|
| **Max replicas** | 15 | 15 |
| **Replication mechanism** | Shared storage (no data copy) | Engine-level async replication (full data copy) |
| **Replication lag** | Typically <10ms | Seconds to minutes |
| **Failover target?** | Yes (automatic promotion) | No (manual promotion, becomes standalone) |
| **Storage cost per replica** | Zero (shared volume) | Full (each replica has its own EBS) |
| **Time to add a replica** | Minutes | Minutes to hours (depends on DB size) |

### Aurora Endpoints

Aurora provides multiple endpoints for different traffic patterns:

| Endpoint | Resolves To | Use For |
|---|---|---|
| **Cluster endpoint** (writer) | Current writer instance | All writes, reads needing latest data |
| **Reader endpoint** | Load-balanced across readers | Read-heavy queries that tolerate ms-level lag |
| **Custom endpoints** | User-defined subset of instances | Analytics queries on larger instances, specific routing |
| **Instance endpoints** | Specific instance | Debugging, direct access |

---

## Part 5: Aurora I/O-Optimized vs Standard Pricing

**The analogy**: Standard Aurora pricing is like a parking meter -- you pay every time you park (per I/O request). If you park twice a day, it is cheap. If you park 200 times a day, the meter charges add up to more than a monthly parking pass. Aurora I/O-Optimized is the **unlimited monthly parking pass** -- 30% higher base rent, but no per-park charges.

| Feature | Aurora Standard | Aurora I/O-Optimized |
|---|---|---|
| **Storage cost** | $0.10/GB/month | $0.225/GB/month (2.25x higher) |
| **I/O cost** | $0.20 per million read requests, $0.20 per million write requests | **$0 (included)** |
| **Compute cost** | Same | Same |
| **When to choose** | I/O spend < 25% of total Aurora cost | I/O spend > 25% of total Aurora cost |
| **Switching** | Can switch to I/O-Optimized once every 30 days | Can switch back to Standard anytime |

**The decision rule is simple**: Look at your Aurora bill. If I/O charges exceed 25% of total Aurora spending, switch to I/O-Optimized. AWS introduced this in 2023 specifically because unpredictable I/O billing was the number one complaint from Aurora customers.

---

## Part 6: Aurora Serverless v2 -- The Elastic Hotel

**The analogy**: A traditional provisioned Aurora cluster is like booking a 100-room hotel for the year. You pay for all 100 rooms whether they are occupied or not. Aurora Serverless v2 is a hotel that **dynamically builds and demolishes rooms in real time** -- during a convention (traffic spike), it expands to 200 rooms within seconds; on a quiet Tuesday, it shrinks to 10. You are billed per room-minute, not per year.

### How ACU Scaling Works

- **1 ACU (Aurora Capacity Unit)** = approximately 2 GiB RAM + proportional CPU and networking
- Scaling range: **0 ACU to 256 ACU** (0 GiB to 512 GiB RAM)
- **Scale to zero**: Since November 2024, Serverless v2 supports scaling to 0 ACUs with automatic pause and resume. When idle, the instance pauses and incurs no compute charges. On the next connection attempt, it resumes (adds a few seconds of cold-start latency). Set `min_capacity = 0` for dev/test; use `min_capacity = 0.5` or higher for always-on production instances where resume latency is unacceptable.
- Scaling increments: **0.5 ACU** (very granular, unlike Serverless v1 which doubled in coarse steps)
- Scaling speed: **Seconds** -- capacity adjusts continuously based on CPU utilization, connections, and available memory
- Each instance scales **independently** -- writer and readers can be at different ACU levels

### Promotion Tiers and Scaling Behavior

Aurora Serverless v2 instances have promotion tiers (0-15) that control two things: failover priority and scaling behavior.

| Tier | Failover Priority | Scaling Behavior |
|---|---|---|
| **0-1** | Highest (promoted first) | **Scales with the writer** -- if writer scales to 64 ACU, tier 0-1 readers scale to at least 64 ACU too, ensuring they can handle promotion without cold-start lag |
| **2-15** | Lower | **Scales independently** based on own read load -- can sit at minimum ACU when idle |

**Why tier 0-1 matters**: If a reader is your failover target, it must be ready to handle the writer's workload immediately upon promotion. A reader sitting at 2 ACU that suddenly becomes the writer for a 64-ACU workload would collapse. Tier 0-1 prevents this by keeping failover candidates scaled in sync with the writer.

### Mixed-Configuration Clusters

You can combine provisioned and Serverless v2 instances in the **same cluster**:

```
MIXED AURORA CLUSTER -- PROVISIONED WRITER + SERVERLESS READERS
════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │              Aurora Cluster                               │
  │                                                          │
  │  ┌─────────────────────┐    ┌────────────────────────┐   │
  │  │  WRITER             │    │  READER 1              │   │
  │  │  Provisioned        │    │  Serverless v2         │   │
  │  │  db.r7g.4xlarge     │    │  Min: 2 ACU            │   │
  │  │  (steady baseline)  │    │  Max: 64 ACU           │   │
  │  │                     │    │  Tier: 0               │   │
  │  │  $1.84/hr fixed     │    │  (scales with writer)  │   │
  │  └─────────────────────┘    └────────────────────────┘   │
  │                                                          │
  │  ┌────────────────────────┐  ┌────────────────────────┐  │
  │  │  READER 2              │  │  READER 3              │  │
  │  │  Serverless v2         │  │  Serverless v2         │  │
  │  │  Min: 0.5 ACU          │  │  Min: 0.5 ACU          │  │
  │  │  Max: 128 ACU          │  │  Max: 128 ACU          │  │
  │  │  Tier: 2               │  │  Tier: 15              │  │
  │  │  (independent scaling) │  │  (independent scaling) │  │
  │  └────────────────────────┘  └────────────────────────┘  │
  │                                                          │
  │  All instances share the same Aurora storage volume       │
  └──────────────────────────────────────────────────────────┘

  Use case: Provisioned writer for predictable baseline cost,
            Serverless readers for unpredictable read spikes
```

**When to use Serverless v2:**
- Dev/test environments (scale to 0.5 ACU when idle)
- Applications with unpredictable traffic patterns (seasonal spikes, event-driven)
- New applications where you cannot forecast capacity requirements
- Mixed clusters where readers handle variable query loads

**When to use provisioned:**
- Steady, predictable workloads where you can forecast capacity
- Cost-sensitive workloads where provisioned per-hour pricing is cheaper than ACU-minute pricing at sustained high utilization
- Writer instances with consistent load (provisioned writer + Serverless readers is a common pattern)

---

## Part 7: Aurora Global Database -- The International Vault Network

**The analogy**: Your hotel chain opens branches in five other countries. Instead of each international branch maintaining its own independent vault (which would require expensive, slow photocopying of every transaction), you build a dedicated underground tunnel system that replicates your vault contents at the **storage layer** to each international branch. The tunnel is so fast that branches are typically less than one second behind headquarters. Guests in Tokyo read from their local vault instantly. But deposits (writes) still route to headquarters in Virginia -- you can optionally let Tokyo guests drop off deposits locally, and the branch forwards them through the tunnel to headquarters (write forwarding).

### Architecture

- **Primary region**: One Aurora cluster (writer + optional readers) with full read-write access
- **Secondary regions**: Up to **10 secondary regions**, each with a read-only Aurora cluster (up to 16 readers per cluster). Note: with more than 5 secondary clusters, the primary cluster is limited to 5 reader instances instead of 15
- **Replication**: At the **storage layer**, not the engine layer. The Aurora storage subsystem replicates redo log records across regions. This is fundamentally different from RDS cross-region read replicas, which replicate at the engine level.
- **Replication lag**: Typically **under 1 second** (compared to seconds-to-minutes for engine-level RDS cross-region replicas)
- **Performance impact on primary**: **Near zero**, because the storage layer handles replication independently of the database engine's CPU and memory

### Switchover vs Failover

| Operation | Managed Planned Switchover | Unplanned Failover |
|---|---|---|
| **When** | Planned maintenance, DR drills, region rotation | Region outage, primary failure |
| **Data loss** | **Zero** (waits for secondary to catch up) | **RPO in seconds** (replication lag = potential data loss) |
| **Downtime** | Typically 1-2 minutes | **RTO in minutes** |
| **Process** | Automated: demotes primary to secondary, promotes secondary to primary | Automated: detaches secondary, promotes to standalone primary |
| **Replication after** | Automatically re-establishes | Must manually re-add the old primary region as secondary |

### Write Forwarding

Write forwarding allows applications connected to a **secondary region** to execute write operations. The secondary region's reader instances transparently forward write SQL statements to the primary cluster's writer instance. Aurora handles the cross-region networking and transmits session and transactional context with each forwarded statement. This is a separate mechanism from the storage replication channel (which flows primary→secondary only for redo log records).

**Use cases:**
- Global applications that want to use a single connection string per region
- Reducing application complexity by avoiding separate write and read endpoints per region

**Trade-offs:**
- **Higher write latency** from secondary regions (cross-region round trip added)
- Not suitable for write-heavy workloads from secondary regions
- Best for occasional writes from regions that are primarily read-heavy

```
AURORA GLOBAL DATABASE
════════════════════════════════════════════════════════════════

  ┌──── us-east-1 (PRIMARY) ────────────────────────────────┐
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
  │  │ Writer   │  │ Reader   │  │ Reader   │              │
  │  │ Instance │  │ Instance │  │ Instance │              │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
  │       └──────────────┴──────────────┘                    │
  │                      │                                   │
  │         ┌────────────┴────────────┐                      │
  │         │  Aurora Storage Volume   │ ──── storage-level  │
  │         │  (6 copies, 3 AZs)      │      replication     │
  │         └────────────┬────────────┘      (<1s lag)       │
  │                      │                       │           │
  └──────────────────────┼───────────────────────┼───────────┘
                         │                       │
              ┌──────────┘                       └──────────┐
              ▼                                              ▼
  ┌──── eu-west-1 (SECONDARY) ──────┐   ┌──── ap-northeast-1 ──────┐
  │                                  │   │                           │
  │  ┌──────────┐  ┌──────────┐     │   │  ┌──────────┐            │
  │  │ Reader   │  │ Reader   │     │   │  │ Reader   │            │
  │  │ Instance │  │ Instance │     │   │  │ Instance │            │
  │  └────┬─────┘  └────┬─────┘     │   │  └────┬─────┘            │
  │       └──────────────┘           │   │       │                  │
  │              │                   │   │       │                  │
  │  ┌───────────┴────────────┐      │   │  ┌────┴──────────────┐   │
  │  │ Aurora Storage Volume  │      │   │  │ Storage Volume    │   │
  │  │ (read-only replica)    │      │   │  │ (read-only)       │   │
  │  └────────────────────────┘      │   │  └───────────────────┘   │
  │                                  │   │                           │
  │  Write forwarding: enabled ──────┼──▶│  Writes forward to       │
  │  (writes route to us-east-1)     │   │  primary region           │
  └──────────────────────────────────┘   └───────────────────────────┘

  Switchover (planned): Zero data loss, ~1-2 min downtime
  Failover (unplanned): RPO in seconds, RTO in minutes
  Up to 10 secondary regions, 16 readers each
```

---

## Part 8: Aurora Cloning -- The Copy-on-Write Photocopier

**The analogy**: Imagine a 10,000-page encyclopedia. Instead of photocopying all 10,000 pages to create a second copy, you give two librarians pointers to the **same** original pages. Both librarians can read any page instantly. When one librarian needs to edit a page, **only that page** is photocopied -- the librarian edits the copy while the other continues reading the original. After heavy editing, perhaps 200 pages have been copied. The other 9,800 pages are still shared, consuming zero additional space.

**Technical reality:**
- Uses the **copy-on-write protocol** at the storage layer
- **Completes in minutes** regardless of database size (a 64 TiB database clones just as fast as a 10 GiB one)
- **Zero additional storage** at creation time -- storage grows only as pages diverge between source and clone
- Clone is a fully independent, writable Aurora cluster
- **Up to 15 clones** per source cluster (clones of clones count toward the limit)
- Must be in the **same AWS Region** as the source
- Works with both provisioned and Serverless v2 instances

**Use cases:**
- **Dev/test from production data**: Clone production, run destructive tests, delete the clone
- **Analytics without impacting production**: Clone, run heavy analytical queries, discard
- **Blue/green deployments**: Clone, apply schema changes to clone, test, then switch traffic
- **Debugging**: Clone production to reproduce and debug an issue without touching prod

**Clone vs Snapshot restore:**

| Feature | Aurora Clone | Snapshot Restore |
|---|---|---|
| **Speed** | Minutes (any size) | Scales with DB size (minutes to hours) |
| **Initial storage** | Zero (shared pages) | Full copy of all data |
| **Cost at creation** | $0 additional storage | Full storage cost immediately |
| **Cross-region** | No (same region only) | Yes (copy snapshot cross-region) |
| **Cross-account** | Yes (with RAM sharing) | Yes (share snapshot) |

### Blue/Green Deployments

RDS and Aurora support **managed Blue/Green Deployments** for zero-downtime engine upgrades, major version upgrades, and schema changes. This is distinct from cloning:

- **Blue environment**: Your current production database (unchanged, still serving traffic)
- **Green environment**: A staging copy created by AWS that replicates from Blue via logical replication
- **Process**: Make changes to Green (engine upgrade, schema migration, parameter changes) → test Green → initiate switchover → AWS promotes Green to production in typically under a minute with built-in rollback guardrails
- **Key difference from cloning**: Blue/Green maintains replication between environments until switchover, whereas a clone is immediately independent. Blue/Green is purpose-built for production upgrades; cloning is for dev/test and analytics.
- **Supported engines**: Aurora MySQL, Aurora PostgreSQL, RDS MySQL, RDS MariaDB, RDS PostgreSQL

### Parameter Groups and Option Groups

Every RDS and Aurora instance is associated with a **parameter group** that controls database engine configuration (equivalent to `postgresql.conf` or `my.cnf`):

- **DB parameter groups** (RDS) / **DB cluster parameter groups** (Aurora): Tune settings like `max_connections`, `shared_buffers`, `innodb_buffer_pool_size`, `work_mem`, etc.
- **Default parameter groups** are read-only -- you must create a custom parameter group to change any setting
- Changes to **static parameters** require a reboot; **dynamic parameters** apply immediately
- **Option groups** (RDS only, not Aurora): Enable optional engine features like Oracle TDE, SQL Server audit, MySQL memcached plugin

### Encryption at Rest -- The Creation-Time Constraint

**Critical caveat**: Encryption must be enabled at cluster/instance creation time. You **cannot encrypt an existing unencrypted RDS instance or Aurora cluster in place**. The migration path is:

1. Take a snapshot of the unencrypted instance
2. Copy the snapshot with encryption enabled (specify KMS key)
3. Restore from the encrypted snapshot to a new instance
4. Update DNS / application connection strings to point to the new encrypted instance
5. Delete the old unencrypted instance

This is the same pattern you learned in the EBS deep dive (Mar 24) -- encryption cannot be added retroactively to existing resources.

---

## Part 9: RDS Proxy -- The Hotel Concierge

**The analogy**: In a busy hotel, imagine if every guest had to get their own room key fabricated from scratch (establishing a database connection) every time they walked through the lobby. Key fabrication takes time and resources -- the locksmith can only make so many keys at once. An RDS Proxy is a **concierge desk** that holds a pool of pre-fabricated keys. When a guest arrives, the concierge hands them an available key from the pool. When they leave, the key goes back into the pool for the next guest. The locksmith only needs to make a small fixed number of keys, and hundreds of guests can cycle through them efficiently.

### Why Connection Pooling Matters

Relational databases have a hard limit on concurrent connections. For RDS, the default `max_connections` is calculated from instance memory (e.g., `{DBInstanceClassMemory/9531392}` for PostgreSQL), so a db.r7g.2xlarge will have thousands of connections while a db.t3.micro may have under 100. Each connection consumes memory (5-10 MB). Serverless architectures exacerbate this because each Lambda invocation opens a new connection and may not close it gracefully.

**Without RDS Proxy:**
```
  Lambda functions (hundreds concurrent)
  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ... ┌──┐
  └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘     └─┬┘
    │    │    │    │    │    │         │
    └────┴────┴────┴────┴────┴────┬────┘
         200+ simultaneous connections
                    │
              ┌─────┴─────┐
              │  RDS/Aurora│ ◄── overwhelmed, max_connections hit
              │  Database  │     "too many connections" errors
              └───────────┘
```

**With RDS Proxy:**
```
  Lambda functions (hundreds concurrent)
  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ... ┌──┐
  └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘     └─┬┘
    │    │    │    │    │    │         │
    └────┴────┴────┴────┴────┴────┬────┘
         200+ application connections
                    │
              ┌─────┴─────┐
              │ RDS Proxy  │ ◄── multiplexes into ~20 DB connections
              │ (concierge)│     from a pre-warmed pool
              └─────┬─────┘
                    │
              ~20 database connections
                    │
              ┌─────┴─────┐
              │  RDS/Aurora│ ◄── comfortable, well within limits
              │  Database  │
              └───────────┘
```

### Key RDS Proxy Features

| Feature | Description |
|---|---|
| **Connection pooling** | Multiplexes thousands of application connections into a smaller pool of database connections |
| **IAM authentication** | Applications authenticate to Proxy using IAM roles instead of database credentials; Proxy fetches DB credentials from Secrets Manager |
| **Failover acceleration** | During Multi-AZ failover, Proxy routes connections to the new primary without applications needing to re-resolve DNS; reduces failover impact from minutes to seconds |
| **Pinning detection** | Detects session-level state (prepared statements, temp tables) that requires connection pinning and manages it transparently |
| **Lambda integration** | The primary use case -- Lambda functions connect to Proxy instead of directly to the database, solving the connection storm problem |
| **Supported engines** | MySQL, PostgreSQL, MariaDB (RDS and Aurora where applicable), SQL Server (RDS) |

### Terraform Example: RDS Proxy for Lambda

```hcl
# RDS Proxy for Lambda-to-Aurora connection pooling
resource "aws_db_proxy" "app" {
  name                   = "orders-db-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800  # 30 minutes
  require_tls            = true
  vpc_security_group_ids = [aws_security_group.proxy.id]
  vpc_subnet_ids         = var.private_subnet_ids

  auth {
    auth_scheme = "SECRETS"
    description = "Aurora master credentials"
    iam_auth    = "REQUIRED"  # Force IAM authentication
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }

  role_arn = aws_iam_role.proxy.arn

  tags = {
    Name = "orders-db-proxy"
  }
}

# Associate Proxy with the Aurora cluster
resource "aws_db_proxy_default_target_group" "app" {
  db_proxy_name = aws_db_proxy.app.name

  connection_pool_config {
    connection_borrow_timeout    = 120  # seconds to wait for a connection
    max_connections_percent      = 100  # % of max_connections Proxy can use
    max_idle_connections_percent = 50   # keep 50% of pool warm when idle
  }
}

resource "aws_db_proxy_target" "app" {
  db_proxy_name          = aws_db_proxy.app.name
  target_group_name      = aws_db_proxy_default_target_group.app.name
  db_cluster_identifier  = aws_rds_cluster.aurora.id
}

# Proxy endpoint for read-only traffic (optional)
resource "aws_db_proxy_endpoint" "read_only" {
  db_proxy_name          = aws_db_proxy.app.name
  db_proxy_endpoint_name = "orders-db-proxy-readonly"
  vpc_subnet_ids         = var.private_subnet_ids
  vpc_security_group_ids = [aws_security_group.proxy.id]
  target_role            = "READ_ONLY"
}
```

---

## Part 10: RDS Automated Backups vs Manual Snapshots

This section provides a foundational overview -- deeper backup and DR strategies will be covered in Week 5.

**The analogy**: Automated backups are like a security camera system that records continuously -- you can rewind to any second within the retention window. Manual snapshots are like pressing the "photograph" button at a specific moment -- you get a permanent photo that exists until you explicitly delete it.

| Feature | Automated Backups | Manual Snapshots |
|---|---|---|
| **Created by** | AWS automatically (daily window + continuous transaction logs) | You (via console, CLI, or Terraform) |
| **Retention** | 0-35 days (set at creation; 0 disables backups) | Indefinite (until you delete them) |
| **Granularity** | **Point-in-time recovery (PITR)** -- any second within retention window | Specific point in time when snapshot was taken |
| **Storage cost** | Free up to 100% of DB size; excess charged per-GB | Charged per-GB from first snapshot |
| **Deleted with instance?** | Yes (automated backups deleted when DB is deleted) | No (manual snapshots persist independently) |
| **Cross-region** | Yes (cross-region automated backup replication supported for MySQL, PostgreSQL, MariaDB, Oracle, SQL Server -- enables cross-region PITR) | Yes (can copy to other regions) |

**Critical interview point**: When you delete an RDS instance, automated backups are deleted too. If you need to preserve a backup beyond instance deletion, take a **manual snapshot** before deleting. For Aurora, you can also choose to retain the automated backup as a final snapshot during deletion.

**PITR for Aurora is continuous**: Because Aurora's storage layer continuously backs up data to S3, Aurora PITR does not depend on a daily snapshot window. You can restore to any point within the retention period (up to 35 days) with per-second granularity.

---

## Part 11: Aurora vs RDS Decision Framework

```
AURORA vs RDS DECISION TREE
════════════════════════════════════════════════════════════════════════════

  Do you need Oracle, SQL Server, MariaDB, or Db2?
       │
       ├── YES ──▶ RDS (Aurora only supports MySQL and PostgreSQL)
       │
       └── NO (MySQL or PostgreSQL)
             │
             ├── Small/simple workload, cost is the top priority?
             │     │
             │     ├── YES ──▶ RDS
             │     │           (Aurora has higher minimum costs; db.t3.micro
             │     │            on RDS is cheaper than minimum Serverless v2)
             │     │
             │     └── NO ──▶ Continue evaluation
             │
             ├── Need exact open-source MySQL/PostgreSQL compatibility
             │   with zero cloud-native modifications?
             │     │
             │     ├── YES ──▶ RDS (vanilla engine, no Aurora-specific behavior)
             │     │
             │     └── NO ──▶ Continue evaluation
             │
             └── Evaluate Aurora for these scenarios:
                   │
                   ├── Need replicas with sub-10ms lag and automatic failover?
                   │     ──▶ Aurora (shared storage, automatic replica promotion)
                   │
                   ├── Need replicas without additional storage cost?
                   │     ──▶ Aurora (shared volume, zero per-replica storage)
                   │
                   ├── Need sub-second cross-region replication?
                   │     ──▶ Aurora Global Database (<1s lag)
                   │         vs RDS cross-region replica (seconds to minutes)
                   │
                   ├── Need automatic storage scaling without provisioning?
                   │     ──▶ Aurora (10 GiB to 256 TiB auto-scale)
                   │         vs RDS (manual resize or max_allocated_storage)
                   │
                   ├── Need fast, space-efficient cloning for dev/test?
                   │     ──▶ Aurora (copy-on-write cloning, minutes, zero storage)
                   │
                   ├── Need auto-scaling compute for variable workloads?
                   │     ──▶ Aurora Serverless v2
                   │
                   ├── Need faster failover?
                   │     ──▶ Aurora (~30s automatic replica promotion)
                   │         vs RDS Classic Multi-AZ (~60-120s DNS flip)
                   │
                   └── Need higher write throughput?
                         ──▶ Aurora (claims up to 5x MySQL, 3x PostgreSQL
                              throughput due to reducing write amplification --
                              only redo logs are written to storage, not
                              full data pages)
```

### Quick Comparison Table

| Feature | RDS | Aurora |
|---|---|---|
| **Engines** | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2 | MySQL, PostgreSQL only |
| **Storage** | EBS volumes (gp3, io2) -- manual sizing | Distributed, auto-scaling (10 GiB - 256 TiB) |
| **Replication** | Engine-level (sync for Multi-AZ, async for replicas) | Storage-level (6 copies, quorum-based) |
| **Max read replicas** | 15 | 15 |
| **Replica lag** | Seconds to minutes | Typically <10ms |
| **Failover time** | 60-120s (classic), ~35s (DB cluster) | ~30s (automatic replica promotion) |
| **Cross-region** | Read replicas (engine-level, seconds lag) | Global Database (storage-level, <1s lag) |
| **Cloning** | Not available (use snapshot restore) | Copy-on-write (minutes, zero initial storage) |
| **Serverless** | Not available | Serverless v2 (0-256 ACU, scale-to-zero supported) |
| **Backups** | Daily snapshot + transaction logs (PITR) | Continuous to S3 (PITR, per-second granularity) |
| **Minimum cost** | Lower (db.t3.micro ~$12/month) | Higher (~$30/month for smallest always-on Serverless v2; scale-to-zero reduces idle cost to near $0) |
| **Performance claim** | Standard engine performance | Up to 5x MySQL, 3x PostgreSQL throughput |

---

## Terraform Example: Aurora Cluster with Serverless v2 Readers

```hcl
# Aurora PostgreSQL cluster with provisioned writer + Serverless v2 readers
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "orders-aurora"
  engine             = "aurora-postgresql"
  engine_version     = "16.4"
  database_name      = "orders"
  master_username    = "admin"

  # Secrets Manager integration for password management
  manage_master_user_password = true

  # Storage configuration
  storage_type      = "aurora-iopt1"  # I/O-Optimized (no per-I/O charges)
  storage_encrypted = true
  kms_key_id        = aws_kms_key.aurora.arn

  # Serverless v2 scaling range (applies to Serverless instances in cluster)
  serverlessv2_scaling_configuration {
    min_capacity = 0.5   # Minimum ACU when active (0 enables scale-to-zero pause)
    max_capacity = 128   # Maximum ACU (256 GiB RAM)
  }

  # Network
  db_subnet_group_name   = aws_db_subnet_group.private.name
  vpc_security_group_ids = [aws_security_group.aurora.id]

  # Backup
  backup_retention_period      = 14
  preferred_backup_window      = "03:00-04:00"
  preferred_maintenance_window = "sun:05:00-sun:06:00"
  copy_tags_to_snapshot        = true

  # Deletion protection
  deletion_protection = true

  # Skip final snapshot only for dev/test -- always create one in production
  skip_final_snapshot = false
  final_snapshot_identifier = "orders-aurora-final"

  tags = {
    Name        = "orders-aurora"
    Environment = "production"
  }
}

# Writer instance: provisioned for predictable baseline
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "orders-aurora-writer"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r7g.2xlarge"  # Provisioned: 64 GiB RAM, 8 vCPUs
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  # Performance Insights
  performance_insights_enabled          = true
  performance_insights_retention_period = 7

  # Enhanced Monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn

  # Promotion tier: N/A for writer (but set for consistency)
  promotion_tier = 0

  tags = {
    Name = "orders-aurora-writer"
    Role = "writer"
  }
}

# Reader 1: Serverless v2, failover target (tier 0 -- scales with writer)
resource "aws_rds_cluster_instance" "reader_failover" {
  identifier         = "orders-aurora-reader-failover"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"  # Serverless v2
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  promotion_tier = 0  # Tier 0: scales with writer, highest failover priority

  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  monitoring_interval                   = 60
  monitoring_role_arn                   = aws_iam_role.rds_monitoring.arn

  tags = {
    Name = "orders-aurora-reader-failover"
    Role = "reader-failover-target"
  }
}

# Reader 2: Serverless v2, independent scaling (tier 2)
resource "aws_rds_cluster_instance" "reader_general" {
  identifier         = "orders-aurora-reader-general"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  promotion_tier = 2  # Tier 2: scales independently, lower failover priority

  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  monitoring_interval                   = 60
  monitoring_role_arn                   = aws_iam_role.rds_monitoring.arn

  tags = {
    Name = "orders-aurora-reader-general"
    Role = "reader-general"
  }
}

# Security group for Aurora cluster
resource "aws_security_group" "aurora" {
  name_prefix = "aurora-orders-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }

  ingress {
    description     = "Allow RDS Proxy"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.proxy.id]
  }
}
```

---

## Complete Architecture: Production Aurora with Global Database and RDS Proxy

```
PRODUCTION AURORA ARCHITECTURE
════════════════════════════════════════════════════════════════════════════

  ┌──── us-east-1 (PRIMARY REGION) ──────────────────────────────────────┐
  │                                                                       │
  │  ┌─── Application Layer ──────────────────────────────────────────┐   │
  │  │                                                                │   │
  │  │  Lambda Functions ──▶ RDS Proxy (IAM auth, connection pool)    │   │
  │  │  ECS/EKS Services ──▶      │                                  │   │
  │  │                             │                                  │   │
  │  │                    ┌────────┴────────┐                         │   │
  │  │                    │ Cluster Endpoint │ (writes)               │   │
  │  │                    │ Reader Endpoint  │ (reads)                │   │
  │  │                    └────────┬────────┘                         │   │
  │  └─────────────────────────────┼──────────────────────────────────┘   │
  │                                │                                      │
  │  ┌─── Aurora Cluster ─────────┼──────────────────────────────────┐   │
  │  │                             │                                  │   │
  │  │  ┌──────────────────┐  ┌──────────────────┐                   │   │
  │  │  │ WRITER           │  │ READER (Tier 0)  │                   │   │
  │  │  │ db.r7g.2xlarge   │  │ Serverless v2    │                   │   │
  │  │  │ (provisioned)    │  │ 0.5-128 ACU      │                   │   │
  │  │  └────────┬─────────┘  │ (scales w/writer)│                   │   │
  │  │           │            └────────┬─────────┘                   │   │
  │  │           │  ┌──────────────────┐                              │   │
  │  │           │  │ READER (Tier 2)  │                              │   │
  │  │           │  │ Serverless v2    │                              │   │
  │  │           │  │ 0.5-128 ACU     │                              │   │
  │  │           │  │ (independent)    │                              │   │
  │  │           │  └────────┬─────────┘                              │   │
  │  │           │           │                                        │   │
  │  │  ┌────────┴───────────┴─────────────────────────────────────┐  │   │
  │  │  │          Aurora Shared Storage Volume                     │  │   │
  │  │  │  I/O-Optimized | Auto-scales 10 GiB - 256 TiB          │  │   │
  │  │  │  6 copies across 3 AZs | 4/6 write, 3/6 read quorum    │  │   │
  │  │  │  Continuous backup to S3 | PITR up to 35 days            │  │   │
  │  │  └──────────────────────────┬───────────────────────────────┘  │   │
  │  │                             │                                  │   │
  │  └─────────────────────────────┼──────────────────────────────────┘   │
  │                                │                                      │
  │                   Storage-level replication (<1s lag)                  │
  │                                │                                      │
  └────────────────────────────────┼──────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                              ▼
  ┌──── eu-west-1 (SECONDARY) ──────┐  ┌──── ap-northeast-1 ──────────┐
  │                                  │  │                               │
  │  Aurora Read-Only Cluster        │  │  Aurora Read-Only Cluster     │
  │  ┌──────────┐  ┌──────────┐     │  │  ┌──────────┐                │
  │  │ Reader   │  │ Reader   │     │  │  │ Reader   │                │
  │  │ (Sless)  │  │ (Sless)  │     │  │  │ (Sless)  │                │
  │  └──────────┘  └──────────┘     │  │  └──────────┘                │
  │                                  │  │                               │
  │  Write forwarding: enabled       │  │  Write forwarding: enabled    │
  │  (writes → us-east-1 primary)    │  │  Switchover target for DR     │
  │                                  │  │                               │
  │  Local reads for EU users        │  │  Local reads for APAC users   │
  └──────────────────────────────────┘  └───────────────────────────────┘
```

---

## Key Takeaways

- **RDS is managed database operations; Aurora is a re-architected storage layer.** RDS runs standard engines on EBS volumes with engine-level replication. Aurora separates compute from a distributed storage layer that replicates independently, enabling faster failover, cheaper replicas, auto-scaling storage, and cloning. The trade-off is Aurora only supports MySQL and PostgreSQL.

- **Multi-AZ is HA; Read Replicas are read scaling. Do not confuse them.** Classic Multi-AZ creates a synchronous standby that serves zero traffic -- it exists solely for automatic failover. Read Replicas use asynchronous replication for read scaling and can be cross-region. The newer Multi-AZ DB Cluster blurs this line (two readable standbys), but it is still fundamentally an HA mechanism limited to MySQL and PostgreSQL.

- **Aurora's 4/6 write quorum and 3/6 read quorum are the interview answer.** Any write quorum (4) and read quorum (3) overlap (4+3=7 > 6), guaranteeing reads see the latest write. Aurora tolerates losing an entire AZ for read+write, and AZ+1 for read-only. The 10 GB segment size enables sub-10-second repair, making multi-failure scenarios statistically negligible.

- **Aurora replicas cost zero additional storage** because they connect to the same shared storage volume. RDS replicas each maintain a full copy of the data via engine-level replication. While both now support up to 15 replicas, Aurora replicas have zero storage overhead and sub-10ms lag versus RDS replicas' full data copy and seconds-to-minutes lag -- fundamentally different economics and performance.

- **Aurora I/O-Optimized eliminates surprise I/O bills.** If I/O charges exceed 25% of your total Aurora cost, switch to I/O-Optimized. It has 2.25x higher storage cost but zero per-I/O charges -- making costs predictable.

- **Aurora Serverless v2 promotion tiers control failover readiness.** Tier 0-1 readers scale in sync with the writer so they can handle promotion without a cold-start collapse. Tier 2-15 readers scale independently based on their own load. Mixed clusters (provisioned writer + Serverless readers) are the most cost-effective pattern for production workloads with variable read traffic.

- **Aurora Global Database replicates at the storage layer, not the engine layer.** This results in sub-second cross-region lag (versus seconds-to-minutes for RDS cross-region read replicas) with zero performance impact on the primary. Managed planned switchover provides zero-data-loss region rotation; unplanned failover has RPO in seconds and RTO in minutes.

- **Aurora cloning uses copy-on-write and completes in minutes regardless of DB size.** It creates zero additional storage initially -- pages are only copied when either the source or clone modifies them. Use it for dev/test, analytics, blue/green deployments. Limit: 15 clones per source, same region only.

- **RDS Proxy solves the Lambda connection storm problem.** It multiplexes thousands of application connections into a smaller pool of database connections, authenticates via IAM + Secrets Manager, and accelerates failover by maintaining persistent connections to the database and transparently routing to the new primary.

- **The Aurora vs RDS decision starts with the engine.** If you need Oracle, SQL Server, MariaDB, or Db2 -- the answer is RDS, full stop. For MySQL and PostgreSQL, evaluate Aurora when you need: more than 5 replicas, sub-second cross-region replication, auto-scaling storage, Serverless v2 compute, copy-on-write cloning, or higher write throughput. Choose RDS when cost is the primary constraint, the workload is small, or you need exact vanilla engine compatibility.