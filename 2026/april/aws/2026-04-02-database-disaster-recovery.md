# Database Disaster Recovery Patterns -- The Emergency Room Triage Analogy

> You have built the managed hotel chain with its revolutionary shared vault system (RDS and Aurora), organized the giant filing cabinet with drawer labels and folder tabs (DynamoDB), installed the deli counter for lightning-fast repeat orders (ElastiCache), mapped out the four-tier hospital emergency preparedness spectrum (Backup & Restore through Active-Active DR), orchestrated the centralized records department (AWS Backup), and designed multi-region architectures with data-plane-first routing. But here is a question that separates architects who have designed DR from architects who have actually **operated** it: when your primary database goes down at 2 AM, what is the exact sequence of actions? Not "promote the secondary" -- but **which CLI command**, **which endpoint changes**, **how long does each step take**, and **what happens to the 47 in-flight transactions that were mid-commit**? This is the domain of database disaster recovery patterns -- the operational mechanics of how databases back up, replicate, fail over, and recover. You already know the four DR strategies and where Aurora Global Database and RDS fit on that spectrum. Today goes deeper: the internal machinery of RDS backups (how PITR actually reconstructs your database second by second), the precise failover procedures for Aurora Global Database (managed switchover vs unplanned failover with the exact CLI commands and what changes), RDS Proxy's role in smoothing the chaos of failover, and a decision framework that maps RPO/RTO requirements to the right database DR pattern. Think of this as emergency room triage -- every database failure has a severity level, and the right response depends on knowing your tools cold, not figuring them out under pressure.

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| RDS Automated Backups | A hospital's continuous vital-sign recorder that writes to a rolling 35-day tape -- the daily snapshot is a full body scan, and transaction logs captured every 5 minutes are the second-by-second heart monitor | Daily snapshots + 5-minute transaction log archival to S3; enables PITR to any second within retention window (0-35 days); deleted when instance is deleted |
| RDS Manual Snapshots | A doctor pressing "print screenshot" on the vital-sign monitor -- the printout exists forever until someone shreds it, independent of whether the patient is still in the hospital | User-initiated, persist indefinitely, survive instance deletion, shareable cross-account; no PITR capability -- just a frozen point-in-time photo |
| RDS PITR Mechanics | Reconstructing a patient's state at 2:47:33 PM by loading their morning body scan and replaying every heartbeat from the monitor tape until 2:47:33 | Restores the most recent snapshot before target time, then replays transaction logs forward; always creates a NEW instance with a new endpoint |
| RDS Cross-Region Backup Replication | Faxing the vital-sign tapes AND the body scans to a sister hospital in another city in real time -- so they can reconstruct any patient state independently | Native RDS feature that replicates both snapshots AND transaction logs to another Region, enabling cross-region PITR (unlike AWS Backup which copies snapshots only) |
| Aurora Continuous Backups | A hospital where the building itself has walls that are always taking their own X-rays -- no scheduled body scan needed, no performance impact | Aurora's storage layer continuously backs up to S3 with zero performance impact; no backup window, no I/O suspension; up to 35-day retention |
| Aurora Backtrack | Rewinding a patient's vital-sign tape in-place and saying "pretend it is 3 PM again" -- same hospital room, same patient, no transfer needed | In-place rewind of an Aurora cluster to a prior point in time without creating a new instance; MySQL only, 72-hour max window, no cross-region support |
| Aurora Global DB Managed Switchover | A planned patient transfer between two fully operational hospitals where both teams coordinate to ensure every chart page arrives before the transfer | RPO=0 planned promotion; synchronizes all data before switching; preserves global cluster topology; ~1-2 minutes downtime |
| Aurora Global DB Unplanned Failover | An emergency helicopter evacuation -- get the patient to the backup hospital NOW; some chart pages from the last few seconds may be lost | RPO in seconds; immediately promotes secondary; old primary must be manually cleaned up; write fencing prevents split-brain |
| Global Writer Endpoint | A single ambulance dispatch number that automatically knows which hospital is the active trauma center -- no need to update the dispatcher after a transfer | Global-level endpoint that auto-updates to point to the current writer after switchover/failover; eliminates post-failover endpoint reconfiguration |
| RDS Cross-Region Read Replica Promotion | Permanently converting a branch clinic into an independent hospital -- cut the information pipeline, it is on its own now | Manual promotion breaks replication permanently; becomes standalone writable instance; requires DNS/app reconfiguration |
| RDS Proxy Failover Acceleration | A hospital switchboard operator who holds all incoming calls during the 10-second room transfer instead of disconnecting everyone | Maintains client connections during Multi-AZ failover; reduces disruption from 60-120s to under 10s by absorbing DNS propagation delay |
| Connection Pinning | A patient who brought their own IV drip and special pillow -- they cannot be moved to a different room without disconnecting everything | Session state (prepared statements, temp tables, SET variables) that prevents connection reuse; pinned connections cannot be multiplexed and are disrupted during failover |

---

## RDS Backup Mechanics -- The Vital-Sign Recorder

You know from the [RDS & Aurora deep dive](../../march/aws/2026-03-25-rds-aurora-deep-dive.md) that automated backups exist and that retention ranges from 0-35 days. But understanding the **internal mechanics** is critical for DR planning because it determines your actual RPO and RTO.

### How RDS Automated Backups Actually Work

**The analogy**: A hospital's vital-sign monitoring system operates on two levels simultaneously. Once a day, the patient gets a full-body scan (the daily snapshot). But between scans, a heart monitor continuously records every beat to a rolling tape (transaction logs). If you need to reconstruct the patient's exact state at 2:47:33 PM, you load the morning body scan and then replay the heart monitor tape forward, beat by beat, until you reach 2:47:33.

**The technical reality:**

1. **Daily storage-level snapshot**: RDS takes a full snapshot of the DB instance during the backup window. For **single-AZ deployments**, this causes a brief I/O suspension (typically seconds to minutes depending on data size). For **Multi-AZ deployments**, the snapshot is taken from the standby, so the primary experiences no I/O impact.

2. **Transaction log archival every 5 minutes**: RDS continuously uploads transaction logs (WAL for PostgreSQL, binlog for MySQL) to S3 every 5 minutes. This is why the `LatestRestorableTime` is typically within the last 5 minutes -- you can restore to any second between the oldest retained snapshot and 5 minutes ago.

3. **Backup retention**: 0-35 days. Setting retention to 0 **disables automated backups entirely** (no PITR capability). The default is 7 days for console-created instances; **1 day for CLI/API-created instances** (including Terraform). Always set `backup_retention_period` explicitly.

```
RDS AUTOMATED BACKUP INTERNALS
════════════════════════════════════════════════════════════════════════════

  Timeline: what RDS does behind the scenes

  03:00 AM          03:05    03:10    03:15         ...        2:47 PM
     │                │        │        │                        │
     ▼                ▼        ▼        ▼                        ▼
  ┌─────────┐    ┌──────┐ ┌──────┐ ┌──────┐              ┌──────────┐
  │  Daily   │    │ TLog │ │ TLog │ │ TLog │    ...       │  Latest  │
  │ Snapshot │    │  #1  │ │  #2  │ │  #3  │              │  TLog    │
  │ (full    │    │(5min)│ │(5min)│ │(5min)│              │  (5min)  │
  │  copy)   │    └──────┘ └──────┘ └──────┘              └──────────┘
  └─────────┘         │        │        │                       │
       │              ▼        ▼        ▼                       ▼
       └──────────────┴────────┴────────┴──────── all in S3 ───┘

  PITR to 2:47:33 PM:
  1. Find most recent snapshot BEFORE 2:47:33 PM → the 03:00 AM snapshot
  2. Restore that snapshot to a NEW DB instance
  3. Replay ALL transaction logs from 03:00 AM through 2:47:33 PM
  4. New instance is ready with data as of 2:47:33 PM

  ⚠️  Restore time = snapshot restore time + transaction log replay time
      More hours of logs to replay = longer restore (this is the hidden cost)

  ⚠️  PITR ALWAYS creates a NEW instance with a NEW endpoint
      Your application must be reconfigured to point to the new instance
```

### The I/O Suspension Gotcha

For **RDS single-AZ instances**, the daily snapshot can cause I/O latency spikes. The recommendation is to schedule the backup window during low-traffic periods. For **Multi-AZ instances**, snapshots are taken from the standby -- no primary impact. For **Aurora**, this entire problem does not exist because Aurora's storage layer continuously backs up to S3 without any snapshot window or I/O suspension.

```
BACKUP WINDOW I/O IMPACT
════════════════════════════════════════════════════════════════════════════

  RDS Single-AZ:    I/O suspended briefly during snapshot → schedule wisely
  RDS Multi-AZ:     Snapshot from standby → no primary impact
  RDS DB Cluster:   Snapshot from standby → no primary impact
  Aurora:           Continuous backup → no window, no suspension, no impact
```

---

## Manual Snapshots vs Automated Backups

You covered the basics in the [RDS deep dive](../../march/aws/2026-03-25-rds-aurora-deep-dive.md). Here is the operational detail that matters for DR:

| Dimension | Automated Backups | Manual Snapshots |
|-----------|-------------------|------------------|
| **Creation** | AWS-managed, daily + 5-min logs | User-initiated (console, CLI, Terraform, AWS Backup) |
| **Retention** | 0-35 days (auto-deleted after) | Indefinite until explicitly deleted |
| **Deleted with instance?** | **Yes** -- gone when DB is deleted | **No** -- persist independently |
| **PITR support** | Yes (any second in retention window) | No (frozen point-in-time only) |
| **Cross-account sharing** | No | **Yes** (share snapshot ARN with other accounts) |
| **Cross-region copy** | Via cross-region backup replication (see below) | Yes (manual `copy-db-snapshot --source-region`) |
| **Automatic cleanup** | Yes (managed by retention policy) | **No** -- accumulates if you do not delete; can inflate storage costs |
| **Cost** | Free up to 100% of allocated storage | Per-GB-month from the first snapshot (no free tier for manual snapshots) |

**The DR-critical insight**: When you delete an RDS instance, automated backups are deleted too. If your DR plan depends on restoring from a backup after instance deletion, you **must** take a manual snapshot first. Aurora offers a "create final snapshot" option during deletion, but you should not rely on remembering this during an emergency.

### CLI: Manual Snapshot Before Deletion

```bash
# Take a manual snapshot before any destructive operation
aws rds create-db-snapshot \
  --db-instance-identifier orders-db-primary \
  --db-snapshot-identifier orders-db-pre-deletion-2026-04-02

# Verify snapshot completion (wait for "available" status)
aws rds wait db-snapshot-available \
  --db-snapshot-identifier orders-db-pre-deletion-2026-04-02

# Copy snapshot to DR region for cross-region protection
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier orders-db-pre-deletion-2026-04-02 \
  --target-db-snapshot-identifier orders-db-pre-deletion-2026-04-02 \
  --source-region us-east-1 \
  --region us-west-2 \
  --kms-key-id arn:aws:kms:us-west-2:111122223333:key/mrk-abc123  # Multi-region KMS key
```

---

## RDS Cross-Region Automated Backup Replication

This is the feature that most architects confuse with AWS Backup cross-region copies. The distinction is critical:

| Feature | RDS Cross-Region Backup Replication | AWS Backup Cross-Region Copy |
|---------|--------------------------------------|------------------------------|
| **What gets copied** | Snapshots **AND** transaction logs | Snapshots only |
| **PITR in DR region?** | **Yes** (full cross-region PITR) | **No** (restore to snapshot point only) |
| **RPO** | 5 minutes (same as source PITR) | Depends on backup schedule (hours) |
| **Supported services** | RDS only (not Aurora) | 20+ services including Aurora |
| **Setup** | Per-instance configuration | Centralized backup plan |

**The analogy**: RDS cross-region backup replication is like faxing both the body scans AND the heart monitor tapes to the sister hospital in real time. AWS Backup cross-region copy is like FedExing just the body scans on a schedule -- if you need to reconstruct minute-by-minute, you cannot.

**When to use which:**
- Use **RDS cross-region backup replication** when you need cross-region PITR for an RDS database (Backup & Restore tier DR with tight RPO)
- Use **AWS Backup** when you need centralized backup orchestration across many resource types, or when snapshot-level granularity is sufficient
- Use **both** when you want centralized governance (AWS Backup) plus fine-grained PITR capability (native replication) -- they are complementary, not competing

### Enabling Cross-Region Backup Replication

```bash
# Enable cross-region backup replication for an RDS PostgreSQL instance
aws rds start-db-instance-automated-backups-replication \
  --source-db-instance-arn arn:aws:rds:us-east-1:111122223333:db:orders-db-primary \
  --kms-key-id arn:aws:kms:us-west-2:111122223333:key/mrk-abc123 \
  --region us-west-2  # Destination region

# Verify replication status
aws rds describe-db-instance-automated-backups \
  --db-instance-automated-backups-arn \
    arn:aws:rds:us-west-2:111122223333:auto-backup:ab-abc123 \
  --region us-west-2
```

**Key limitations:**
- Not supported for **Aurora** (Aurora uses Global Database for cross-region DR)
- Not supported for **Multi-AZ DB Clusters** (the newer 3-AZ read-writable standby mode)
- Maximum **20 cross-region backup replications** per source account
- Destination region incurs storage costs for the replicated backups

---

## Aurora Backup Differences -- The Self-Imaging Building

Aurora's backup architecture is fundamentally different from RDS because of the compute-storage separation you studied in the [RDS & Aurora deep dive](../../march/aws/2026-03-25-rds-aurora-deep-dive.md).

**The analogy**: While an RDS hospital schedules daily body scans that briefly interrupt the patient, Aurora is a hospital where the walls themselves continuously take X-rays. No scheduled scan, no patient interruption, no backup window. The building's structure IS the backup system.

### How Aurora Continuous Backups Work

1. **Continuous incremental backups to S3**: Aurora's storage layer continuously backs up data as it writes. There is no discrete "backup job" that runs at 3 AM. Every write to the storage volume is backed up to S3 in parallel, incrementally.

2. **No performance impact**: Because backups are handled at the storage layer (not the database engine), there is zero CPU/memory/I/O impact on your Aurora instances. No I/O suspension. No backup window to schedule around.

3. **Up to 35-day retention**: Same maximum as RDS, but with per-second granularity (RDS PITR is per-second too, but depends on 5-minute transaction log intervals).

4. **Restore creates a new cluster**: Like RDS, Aurora PITR creates a **new cluster** with a new endpoint. The restore-to-new-instance constraint applies here too.

### Aurora Backtrack -- The In-Place Rewind

Backtrack is Aurora's unique capability that no other AWS database offers. It rewinds the cluster **in place** without creating a new instance.

**The analogy**: Instead of transferring the patient to a new room reconstructed to match their state at 3 PM, you rewind the clock on the vital-sign monitor in their current room. The patient stays in the same bed (same endpoint), the same room (same cluster), and everything rolls back to 3 PM. No new hospital (instance) needed.

| Feature | Aurora PITR | Aurora Backtrack |
|---------|------------|-----------------|
| **Creates new instance?** | Yes (new cluster, new endpoint) | **No** (in-place on existing cluster) |
| **Endpoint changes?** | Yes (must reconfigure application) | **No** (same endpoint) |
| **Max window** | 35 days | **72 hours** |
| **Engine support** | MySQL and PostgreSQL | **MySQL only** |
| **Cross-region** | Yes (if Global Database) | **No** (local cluster only) |
| **Speed** | Minutes to hours (depends on data size) | **Seconds to minutes** |
| **Use case** | True DR, long-ago recovery | Quick rollback of recent mistakes |

**Backtrack critical gotchas:**
- Cannot be added to an **existing running cluster** via modify -- but CAN be enabled when restoring from a snapshot to a new cluster
- Costs based on the number of change records stored (more writes = higher backtrack cost)
- Backtracks the **entire cluster**, not individual tables
- Does not work with Aurora PostgreSQL -- MySQL only

### Aurora Cloning for DR Testing

Aurora cloning (covered in the [RDS deep dive](../../march/aws/2026-03-25-rds-aurora-deep-dive.md)) is not a DR mechanism itself, but it is invaluable for **DR testing**. Clone your production database, run destructive failover tests against the clone, and delete it when done. Copy-on-write means the clone is ready in minutes regardless of database size, and costs nothing until data diverges.

---

## Aurora Global Database Failover Deep Dive

The [DR Strategies doc](../../march/aws/2026-03-30-disaster-recovery-strategies.md) covered the conceptual difference between managed switchover and unplanned failover. Here we go into the **exact operational procedures**.

### Managed Planned Switchover (RPO=0)

**The analogy**: A coordinated patient transfer between two hospitals. Both surgical teams get on a conference call. The sending hospital says "we are transferring Patient X -- let us confirm all chart pages are synchronized." They verify every page, then simultaneously the sending hospital stops admitting new patients for Patient X while the receiving hospital activates their admission desk. The transfer takes 1-2 minutes of "quiet time" where neither hospital is admitting, but zero chart pages are lost.

**When to use**: Planned Region migration, DR drills, compliance rotation, follow-the-sun operations.

**What happens step by step:**
1. You initiate switchover via CLI or console
2. Aurora **pauses writes** on the primary cluster
3. Aurora waits for all secondary clusters to **fully catch up** (replication lag drops to zero)
4. Aurora **demotes** the primary cluster to secondary (read-only)
5. Aurora **promotes** the target secondary cluster to primary (read-write)
6. The **global cluster topology is preserved** -- both clusters remain in the global database with roles swapped
7. The **global writer endpoint** automatically updates to point to the new primary

**CLI command:**
```bash
# Managed planned switchover (RPO=0, no data loss)
aws rds switchover-global-cluster \
  --global-cluster-identifier ecommerce-global \
  --target-db-cluster-identifier arn:aws:rds:us-west-2:111122223333:cluster:ecommerce-dr

# Monitor progress
aws rds describe-global-clusters \
  --global-cluster-identifier ecommerce-global \
  --query 'GlobalClusters[0].GlobalClusterMembers[*].{Cluster:DBClusterIdentifier,Writer:IsWriter,State:GlobalWriteForwardingStatus}'
```

**Pre-switchover checklist:**
- Engine versions must be compatible across primary and secondary
- Monitor `AuroraGlobalDBReplicationLag` CloudWatch metric -- should be consistently low (<100ms) before initiating
- Parameter groups should be consistent across regions
- Ensure secondary cluster has sufficient compute capacity to handle write workload (if DR reader is db.r6g.large and primary writer is db.r6g.2xlarge, scale up the DR instance **before** switchover)

### Managed Unplanned Failover (RPO = Seconds)

**The analogy**: An emergency helicopter evacuation. The primary hospital just lost power. There is no time to verify chart synchronization. The backup hospital immediately activates its emergency department and starts treating patients. Some chart pages from the last few seconds before the power failure may be missing (the RPO), but the patient survives. The old hospital is quarantined until inspectors clear it -- it cannot simply rejoin the network without cleanup.

**When to use**: Primary Region is degraded or unavailable. This is the "break glass" operation.

**What happens step by step:**
1. You initiate unplanned failover via CLI or console
2. Aurora **immediately promotes** the target secondary cluster to primary (does NOT wait for replication to catch up)
3. **Write fencing** activates via an epoch-based mechanism: the new primary increments an epoch counter that is embedded in all storage operations. If the old primary somehow comes back online, its lower epoch number causes all its writes to be rejected -- preventing split-brain
4. Aurora takes an automatic snapshot of the promoted cluster (named `rds:unplanned-global-failover-*`) as a safety net
5. The old primary is **detached** from the global cluster. When it recovers, it is a standalone cluster with stale data
6. **The global cluster topology is broken** -- you must manually clean up the old primary and re-add it as a secondary (see failback procedure below)

**CLI command:**
```bash
# Managed unplanned failover (RPO in seconds, potential data loss)
# The --allow-data-loss flag is what makes this an unplanned failover
# Omitting it causes the command to default to switchover behavior
aws rds failover-global-cluster \
  --global-cluster-identifier ecommerce-global \
  --target-db-cluster-identifier arn:aws:rds:us-west-2:111122223333:cluster:ecommerce-dr \
  --allow-data-loss

# ⚠️  In exam questions, --allow-data-loss is the definitive indicator of unplanned failover
# The switchover-global-cluster command has no --allow-data-loss flag (RPO=0 guaranteed)
```

**Write fencing -- why it matters:**
```
WRITE FENCING VIA EPOCH-BASED MECHANISM
════════════════════════════════════════════════════════════════════════════

  Before failover:
  ┌─────────────────────────┐    ┌─────────────────────────┐
  │  us-east-1 (PRIMARY)    │    │  us-west-2 (SECONDARY)  │
  │  Epoch: 1               │───▶│  Epoch: 1               │
  │  Writes: ACCEPTED       │    │  Writes: READ-ONLY      │
  └─────────────────────────┘    └─────────────────────────┘

  After unplanned failover:
  ┌─────────────────────────┐    ┌─────────────────────────┐
  │  us-east-1 (DETACHED)   │    │  us-west-2 (NEW PRIMARY)│
  │  Epoch: 1               │    │  Epoch: 2               │
  │  Writes: FENCED         │    │  Writes: ACCEPTED       │
  │  (old epoch rejected)   │    │  (new epoch wins)       │
  └─────────────────────────┘    └─────────────────────────┘

  If old primary somehow recovers and tries to write:
  Storage layer checks epoch → 1 < 2 → REJECT
  This prevents split-brain: only one writer can exist
```

### Switchover vs Failover Comparison

| Dimension | Managed Planned Switchover | Managed Unplanned Failover |
|-----------|---------------------------|---------------------------|
| **CLI command** | `switchover-global-cluster` | `failover-global-cluster --allow-data-loss` |
| **RPO** | Zero (waits for sync) | Seconds (replication lag at failure time) |
| **Downtime** | ~1-2 minutes | ~1-2 minutes |
| **Waits for replication?** | Yes -- pauses writes, drains replication lag to zero | No -- promotes immediately |
| **Topology preserved?** | **Yes** -- both clusters stay in global database with swapped roles | **No** -- old primary detached, must be manually cleaned up |
| **Write fencing?** | Not needed (orderly handoff) | **Yes** -- epoch-based fencing prevents old primary from writing |
| **Automatic snapshot?** | No (not needed -- RPO=0, no data divergence) | **Yes** (`rds:unplanned-global-failover-*`) |
| **Failback complexity** | Simple (just switchover again) | Complex (delete old primary, re-add as secondary) |
| **Primary healthy?** | Must be healthy and reachable | Can be down or degraded |

### Global Writer Endpoint -- The Single Dispatch Number

Before the global writer endpoint (launched October 2024), every Aurora Global Database failover required applications to update their connection strings to point to the new primary cluster's endpoint. This was operationally painful -- a control-plane action (updating DNS or app config) in the critical path of disaster recovery.

The **global writer endpoint** eliminates this. It is a global-level endpoint that automatically resolves to the current writer cluster, regardless of which region is primary.

```
GLOBAL WRITER ENDPOINT -- BEFORE AND AFTER
════════════════════════════════════════════════════════════════════════════

  BEFORE (without global writer endpoint):
  ──────────────────────────────────────────
  Application → ecommerce-primary.cluster-abc.us-east-1.rds.amazonaws.com
                                               ▲
  After failover: application MUST be updated  │
  to:            ecommerce-dr.cluster-xyz.us-west-2.rds.amazonaws.com
                 (manual reconfiguration or Route53/RDS Proxy update required)

  AFTER (with global writer endpoint):
  ─────────────────────────────────────
  Application → ecommerce-global.cluster-abc.global.rds.amazonaws.com
                                               ▲
  After failover: endpoint auto-updates        │
  Application code: NO CHANGES NEEDED
  DNS resolution: automatically points to new primary
```

**Connection to the data-plane principle**: As covered in the [Multi-Region Architecture doc](../../march/aws/2026-03-31-multi-region-architecture-patterns.md), the #1 multi-region principle is that failover should depend on the data plane (which has ~100% SLA), not the control plane (which can be degraded during an outage). The global writer endpoint moves database failover closer to data-plane behavior because the endpoint "just works" without requiring manual DNS updates or application reconfiguration -- both of which are control-plane-dependent operations.

### Failback Procedure After Unplanned Failover

This is operationally complex because unplanned failover breaks the global cluster topology:

```bash
# Step 1: Identify the old primary (now detached, stale data)
# It shows as a standalone cluster, no longer part of the global database

# Step 2: Delete the old stale cluster in us-east-1
# It CANNOT be reconnected -- it has divergent data from the split-brain window
aws rds delete-db-cluster \
  --db-cluster-identifier ecommerce-primary \
  --skip-final-snapshot  # Or create a final snapshot for forensics
  --region us-east-1

# Also delete the instances in the old cluster
aws rds delete-db-instance \
  --db-instance-identifier ecommerce-primary-writer \
  --region us-east-1

# Step 3: Add us-east-1 as a NEW secondary to the global database
# (us-west-2 is now the primary)
aws rds create-db-cluster \
  --db-cluster-identifier ecommerce-east-secondary \
  --global-cluster-identifier ecommerce-global \
  --engine aurora-postgresql \
  --engine-version 16.8 \
  --db-subnet-group-name private-subnets \
  --vpc-security-group-ids sg-abc123 \
  --region us-east-1

# Step 4: Add instances to the new secondary cluster
aws rds create-db-instance \
  --db-instance-identifier ecommerce-east-secondary-reader \
  --db-cluster-identifier ecommerce-east-secondary \
  --instance-class db.r6g.xlarge \
  --engine aurora-postgresql \
  --region us-east-1

# Step 5: Wait for replication lag to reach near-zero
# Monitor AuroraGlobalDBReplicationLag in CloudWatch

# Step 6: When ready to restore us-east-1 as primary,
# use MANAGED SWITCHOVER (not unplanned failover!)
aws rds switchover-global-cluster \
  --global-cluster-identifier ecommerce-global \
  --target-db-cluster-identifier arn:aws:rds:us-east-1:111122223333:cluster:ecommerce-east-secondary
```

**Contrast with DynamoDB Global Tables failback**: As noted in the [DR Strategies doc](../../march/aws/2026-03-30-disaster-recovery-strategies.md), DynamoDB Global Tables failback is effortless because all replicas are read-write. When the original Region recovers, replication simply resumes. No manual steps.

---

## Cross-Region Failover vs In-Region Failover

These are **completely different mechanisms** and a common source of confusion:

```
TWO DIFFERENT FAILOVER MECHANISMS -- DO NOT CONFLATE
════════════════════════════════════════════════════════════════════════════

  IN-REGION FAILOVER (High Availability)
  ────────────────────────────────────────
  Trigger: Writer instance failure, AZ outage, maintenance
  Mechanism: Aurora promotes a read replica to writer using the
             shared storage volume (no data copying needed)
  Time: ~30 seconds
  Data loss: Zero (shared storage, no replication lag)
  Endpoint: Cluster writer endpoint auto-updates (same region)
  Scope: Within the Aurora cluster in one Region

  ┌──── us-east-1 ──────────────────────────────────────────┐
  │  Writer (AZ-a) ──FAILS──▶ Reader (AZ-b) promoted       │
  │       │                         ▲                        │
  │       └── shared storage ───────┘                        │
  │           (6 copies, 3 AZs -- same volume)               │
  └──────────────────────────────────────────────────────────┘

  CROSS-REGION FAILOVER (Disaster Recovery)
  ──────────────────────────────────────────
  Trigger: Region-level failure, planned migration
  Mechanism: Aurora promotes a secondary CLUSTER in another
             Region using storage-level replication
  Time: ~1-2 minutes
  Data loss: Zero (switchover) or seconds (unplanned failover)
  Endpoint: Global writer endpoint auto-updates (cross-region)
  Scope: Across the Aurora Global Database

  ┌── us-east-1 ──┐         ┌── us-west-2 ──┐
  │  Primary       │──FAILS──▶  Secondary     │
  │  Cluster       │         │  Cluster       │
  │  (writer)      │         │  (promoted to  │
  │                │         │   writer)      │
  └────────────────┘         └────────────────┘
       Async storage-level replication (<1s lag)
```

---

## RDS Cross-Region Read Replicas -- The Promotion Path

For databases that cannot use Aurora (Oracle, SQL Server, MariaDB, MySQL with specific requirements), cross-region read replicas provide the DR path.

### Engine Support Differences

| Engine | Cross-Region Read Replicas | Notes |
|--------|---------------------------|-------|
| **MySQL** | Yes | Engine-level binlog replication |
| **MariaDB** | Yes | Engine-level binlog replication |
| **PostgreSQL** | Yes (for RDS PostgreSQL) | Engine-level WAL replication |
| **Oracle** | Yes (with Active Data Guard license) | License cost |
| **SQL Server** | No | Use AWS Backup + snapshot copy for DR |
| **Aurora MySQL** | Cross-region replicas via Global Database | Storage-level replication (preferred over engine-level) |
| **Aurora PostgreSQL** | **Global Database ONLY** | No engine-level cross-region replicas; Global Database is the sole cross-region DR option |

**The PostgreSQL gotcha**: RDS PostgreSQL supports cross-region read replicas. But **Aurora PostgreSQL does not support engine-level cross-region read replicas** -- it ONLY supports Global Database for cross-region replication. If you need Aurora PostgreSQL in multiple regions, Global Database is your only option.

### Cross-Region Read Replica Promotion

Promoting a cross-region read replica is a **one-way, destructive operation**:

```bash
# Promote cross-region read replica to standalone (IRREVERSIBLE)
aws rds promote-read-replica \
  --db-instance-identifier orders-db-replica-eu \
  --region eu-west-1

# After promotion:
# - Replication link is PERMANENTLY broken
# - Replica becomes an independent, writable DB instance
# - Instance endpoint hostname stays the same, but it transitions from read-only to read-write
# - Application must be reconfigured: if pointing at the source's endpoint, switch to promoted instance
# - The old primary (if still alive) has no idea this happened
```

**RPO implications**: The RPO equals the replication lag at the moment of promotion. For cross-region replicas, this is typically seconds to minutes, but can spike under heavy write load or network issues. Monitor `ReplicaLag` CloudWatch metric continuously.

**Comparison with Aurora Global Database:**

| Dimension | RDS Cross-Region Read Replica | Aurora Global Database |
|-----------|-------------------------------|----------------------|
| **Replication layer** | Engine-level (binlog/WAL) | Storage-level (redo log) |
| **Typical replication lag** | Seconds to minutes | Sub-1 second |
| **Promotion type** | Manual, irreversible | Managed switchover or failover |
| **Topology after promotion** | Broken (standalone instance) | Preserved (switchover) or broken (failover) |
| **Endpoint after promotion** | New endpoint, app must reconfigure | Global writer endpoint auto-updates |
| **Failback** | Manual (create new replica from promoted) | Add old region as new secondary |
| **Supported engines** | MySQL, MariaDB, PostgreSQL, Oracle | Aurora MySQL, Aurora PostgreSQL |

---

## RDS Proxy Deep Dive -- The Emergency Switchboard Operator

You covered RDS Proxy basics in the [RDS deep dive](../../march/aws/2026-03-25-rds-aurora-deep-dive.md). Here we focus on its **DR-specific role**: failover acceleration.

### How RDS Proxy Accelerates Failover

**The analogy**: During a hospital room transfer (Multi-AZ failover), imagine a switchboard operator who handles all incoming phone calls. Without the operator, every caller gets disconnected during the 60-120 second transfer and must redial the new room number. With the operator, callers are put on a brief hold (under 10 seconds), the operator connects to the new room, and then patches all the held calls through. No caller ever dials a new number.

**Without RDS Proxy (60-120 second disruption):**
1. Primary fails → RDS initiates failover
2. DNS CNAME record updates to point to standby
3. Applications detect connection failure, throw exceptions
4. DNS propagation takes 30-60 seconds (TTL + resolver caching)
5. Applications re-resolve DNS, establish new connections
6. Total disruption: 60-120 seconds of connection errors

**With RDS Proxy (under 10 seconds):**
1. Primary fails → RDS initiates failover
2. RDS Proxy detects primary is down
3. Proxy **holds all client connections open** (clients see brief latency increase, not errors)
4. Proxy connects to the new primary (it tracks the failover internally)
5. Proxy **drains** connections to the old primary and redirects queries to the new primary
6. Total disruption: under 10 seconds (no DNS propagation needed -- Proxy absorbs it)

```
RDS PROXY FAILOVER ACCELERATION
════════════════════════════════════════════════════════════════════════════

  WITHOUT PROXY:
  ┌────────┐    ┌──────────────┐         ┌──────────────┐
  │  App   │───▶│  DNS CNAME   │───X──▶  │  Primary     │ ← FAILED
  │        │    │  (stale TTL) │         │  (AZ-a)      │
  │        │    └──────────────┘         └──────────────┘
  │        │         ... 60-120s of DNS propagation + reconnection ...
  │        │    ┌──────────────┐         ┌──────────────┐
  │        │───▶│  DNS CNAME   │────────▶│  Standby     │ ← NEW PRIMARY
  │        │    │  (updated)   │         │  (AZ-b)      │
  └────────┘    └──────────────┘         └──────────────┘

  WITH PROXY:
  ┌────────┐    ┌──────────────┐    X    ┌──────────────┐
  │  App   │───▶│  RDS Proxy   │───X──▶  │  Primary     │ ← FAILED
  │        │    │  (holds conn)│         │  (AZ-a)      │
  │        │    │              │         └──────────────┘
  │        │    │  ... <10s ...│
  │        │    │              │─────────▶┌──────────────┐
  │        │    │  (redirects) │         │  Standby     │ ← NEW PRIMARY
  └────────┘    └──────────────┘         │  (AZ-b)      │
                 ▲                        └──────────────┘
                 │
                 Proxy IP stays the same
                 Client connections never break
```

### Connection Pinning -- The DR Disruptor

Connection pinning occurs when session-level state prevents the proxy from safely multiplexing a connection. A pinned connection is tied to a specific database connection and **cannot be transparently redirected during failover**.

**What causes pinning:**
- **Session-level variables**: `SET` statements (`SET search_path`, `SET timezone`, etc.)
- **Prepared statements**: Server-side prepared statements hold state on the database connection
- **Temporary tables**: `CREATE TEMPORARY TABLE` creates per-connection state
- **Advisory locks**: `pg_advisory_lock()` in PostgreSQL
- **Cursors**: Explicit cursor declarations with `DECLARE`

**Why pinning matters for DR:**
- Pinned connections cannot be multiplexed → reduces proxy efficiency
- During failover, pinned connections **must be broken** because their session state cannot be transferred to the new primary
- High pinning rates mean more connections experience disruption during failover

**Monitoring pinning:**
```bash
# CloudWatch metric: DatabaseConnectionsCurrentlySessionPinned
# If this is a high percentage of total connections, investigate
# and refactor application code to minimize session state
```

**Mitigation strategies:**
- Use **connection-level** initialization instead of per-query SET statements where possible
- Use **parameterized queries** instead of server-side prepared statements
- Set `init_query` in the RDS Proxy configuration to apply session settings at connection time rather than per-session
- For PostgreSQL, set session parameters in the parameter group instead of with SET statements

### RDS Proxy IAM Authentication and Secrets Manager

```
RDS PROXY AUTHENTICATION FLOW
════════════════════════════════════════════════════════════════════════════

  ┌──────────┐  IAM token    ┌────────────┐  DB creds from   ┌──────────┐
  │  Lambda  │──────────────▶│  RDS Proxy │──────────────────▶│  Aurora  │
  │  Function│  (short-lived │              │  Secrets Manager │  Cluster │
  │          │   IAM auth)   │              │  (long-lived     │          │
  └──────────┘               │              │   DB password)   └──────────┘
                             └──────┬───────┘
                                    │
                                    ▼
                             ┌──────────────┐
                             │   Secrets    │
                             │   Manager   │
                             │  (stores DB │
                             │   password) │
                             └──────────────┘

  Benefits:
  - Lambda never sees the DB password (only IAM token)
  - Secrets Manager handles password rotation
  - IAM policies control which functions can access which databases
  - No credentials stored in Lambda environment variables
```

### The Critical Gotcha: RDS Proxy is Single-Region Only

**RDS Proxy does NOT work cross-region.** It is deployed within a VPC in a single region and can only route to databases in that same region. This means:

- RDS Proxy accelerates **in-region** failover (Multi-AZ) beautifully
- RDS Proxy provides **zero benefit** for cross-region DR failover
- For cross-region DR with Aurora Global Database, the global writer endpoint handles endpoint management, not RDS Proxy
- You need an RDS Proxy **in each region** -- one for the primary region and one for the DR region, each pointing to the local Aurora cluster

```
RDS PROXY -- SINGLE-REGION SCOPE
════════════════════════════════════════════════════════════════════════════

  ┌──── us-east-1 ─────────────────┐    ┌──── us-west-2 ─────────────────┐
  │  App → RDS Proxy → Aurora      │    │  App → RDS Proxy → Aurora      │
  │  (primary writer)               │    │  (secondary reader)            │
  │                                 │    │                                 │
  │  Proxy handles in-region        │    │  Proxy handles in-region       │
  │  Multi-AZ failover (~10s)       │    │  Multi-AZ failover (~10s)      │
  └─────────────────────────────────┘    └─────────────────────────────────┘
           │                                       ▲
           │    Aurora Global Database              │
           └── storage-level replication ──────────┘
               (Proxy is NOT involved in this path)

  Cross-region failover uses the GLOBAL WRITER ENDPOINT,
  not RDS Proxy. Proxy is purely a within-region accelerator.
```

---

## Database DR Decision Framework

This is the synthesis of everything -- mapping your RPO/RTO requirements to the right database DR pattern.

```
DATABASE DR DECISION TREE
════════════════════════════════════════════════════════════════════════════

  What is your database engine?
  │
  ├── Oracle, SQL Server, or Db2 (RDS only)
  │   │
  │   ├── RPO hours, RTO hours?
  │   │   └── AWS Backup + cross-region snapshot copy
  │   │       Cost: $ (storage only in DR region)
  │   │
  │   ├── RPO minutes, RTO tens of minutes?
  │   │   └── Cross-region read replica (Oracle w/ADG, not SQL Server)
  │   │       + RDS Proxy in each region for in-region HA
  │   │       Cost: $$ (replica instance + storage + data transfer)
  │   │
  │   └── RPO near-zero? → Not achievable with these engines on RDS
  │       Consider: DRS for compute + native backup for data tier
  │
  ├── MySQL or MariaDB (RDS)
  │   │
  │   ├── RPO hours?
  │   │   └── RDS cross-region backup replication (enables cross-region PITR)
  │   │       Cost: $ (storage in DR region)
  │   │
  │   ├── RPO minutes?
  │   │   └── Cross-region read replica + manual promotion
  │   │       Cost: $$ (replica instance running 24/7)
  │   │
  │   └── RPO seconds? → Consider migrating to Aurora MySQL
  │       (Aurora Global Database provides managed failover)
  │
  ├── PostgreSQL (RDS)
  │   │
  │   ├── RPO hours?
  │   │   └── RDS cross-region backup replication
  │   │
  │   ├── RPO minutes?
  │   │   └── Cross-region read replica + manual promotion
  │   │
  │   └── RPO seconds? → Migrate to Aurora PostgreSQL + Global Database
  │
  └── Aurora (MySQL or PostgreSQL)
      │
      ├── RPO hours? (Backup & Restore tier)
      │   └── AWS Backup cross-region snapshot copy
      │       Cost: $ (snapshot storage only)
      │
      ├── RPO seconds? (Pilot Light / Warm Standby tier)
      │   └── Aurora Global Database
      │       + Serverless v2 at 0.5 ACU in DR (Pilot Light)
      │       + Or provisioned instances in DR (Warm Standby)
      │       + RDS Proxy in each region for in-region HA
      │       + Global writer endpoint for seamless failover
      │       Cost: $$ (Pilot Light) to $$$ (Warm Standby)
      │
      └── RPO near-zero? (Active-Active tier)
          └── Aurora Global Database with write forwarding
              (single-writer, writes forwarded from secondary)
              + Full compute in both regions
              + Route53 latency routing or Global Accelerator
              + Global writer endpoint
              Cost: $$$$ (full prod in both regions)
```

### Pattern Comparison Table

| DR Pattern | RPO | RTO | Failover Action | Endpoint Change? | Cost Multiplier |
|-----------|-----|-----|-----------------|-----------------|-----------------|
| **AWS Backup cross-region copy** | Hours | Hours | PITR to new instance in DR + app reconfiguration | Yes (new instance) | +5-10% |
| **RDS cross-region backup replication** | 5 min | Hours | Cross-region PITR to new instance + app reconfiguration | Yes (new instance) | +10-15% |
| **RDS cross-region read replica** | Min | 10s of min | Promote replica (manual, irreversible) + app reconfiguration | Yes (new endpoint) | +20-30% |
| **Aurora Global DB (Pilot Light)** | Seconds | Minutes | Managed switchover or unplanned failover | No (global writer endpoint) | +15-25% |
| **Aurora Global DB (Warm Standby)** | Seconds | Minutes | Same + already serving reads | No (global writer endpoint) | +25-40% |
| **Aurora Global DB (Active-Active)** | Near-zero | Near-zero | Route53/GA shifts traffic | No (global writer endpoint) | +80-120% |

### AWS Backup vs Native Replication -- When to Use Which

As covered in the [AWS Backup doc](../../april/aws/2026-04-01-aws-backup-centralized-protection.md), AWS Backup provides centralized governance but operates differently from native database replication:

| Dimension | AWS Backup | Native Database Replication |
|-----------|-----------|---------------------------|
| **Mechanism** | Scheduled snapshot copy | Continuous data streaming |
| **RPO** | Hours (depends on backup frequency) | Seconds (Aurora Global DB) to minutes (RDS cross-region replica) |
| **Cross-region PITR?** | No (snapshot-level only) | Yes (RDS cross-region backup replication for RDS; Global DB for Aurora) |
| **Multi-service** | Yes (20+ resource types in one plan) | No (per-service configuration) |
| **Governance** | Organization policies, Vault Lock, compliance | Per-database settings |
| **Use together?** | Yes -- AWS Backup as governance layer + native replication for tight RPO | |

**The rule**: Use **native replication** (Aurora Global Database, RDS cross-region replicas) for operational DR with tight RPO/RTO. Use **AWS Backup** as the foundational governance layer under every strategy -- it protects against data corruption that replication would faithfully propagate across regions.

---

## Production Terraform Example: Aurora Global Database with RDS Proxy

This example builds a complete database DR stack: Aurora Global Database across two regions with RDS Proxy in each region for in-region failover acceleration and cross-region backup replication for an additional RDS PostgreSQL instance.

```hcl
# ═══════════════════════════════════════════════════════════════════════
# PROVIDERS
# ═══════════════════════════════════════════════════════════════════════

provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

# ═══════════════════════════════════════════════════════════════════════
# AURORA GLOBAL DATABASE
# ═══════════════════════════════════════════════════════════════════════

resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "orders-global"
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"
  database_name             = "orders"
  storage_encrypted         = true
}

# --- Primary Region Cluster ---

resource "aws_rds_cluster" "primary" {
  provider = aws.primary

  cluster_identifier        = "orders-primary"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"
  master_username           = "admin"
  manage_master_user_password = true  # Secrets Manager integration

  db_subnet_group_name   = aws_db_subnet_group.primary.name
  vpc_security_group_ids = [aws_security_group.aurora_primary.id]

  # Maximum backup retention for PITR -- protects against data corruption
  backup_retention_period = 35
  preferred_backup_window = "03:00-04:00"
  copy_tags_to_snapshot   = true

  deletion_protection = true
}

resource "aws_rds_cluster_instance" "primary_writer" {
  provider = aws.primary

  identifier         = "orders-primary-writer"
  cluster_identifier = aws_rds_cluster.primary.id
  instance_class     = "db.r6g.2xlarge"
  engine             = "aurora-postgresql"
  promotion_tier     = 0
}

resource "aws_rds_cluster_instance" "primary_reader" {
  provider = aws.primary

  identifier         = "orders-primary-reader"
  cluster_identifier = aws_rds_cluster.primary.id
  instance_class     = "db.r6g.xlarge"
  engine             = "aurora-postgresql"
  promotion_tier     = 1  # Failover candidate, scales with writer
}

# --- DR Region Cluster (Warm Standby) ---

resource "aws_rds_cluster" "dr" {
  provider = aws.dr

  cluster_identifier        = "orders-dr"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"

  db_subnet_group_name   = aws_db_subnet_group.dr.name
  vpc_security_group_ids = [aws_security_group.aurora_dr.id]

  # Independent PITR in DR region -- critical for data corruption recovery
  backup_retention_period = 35

  # Enable write forwarding so DR region apps can write via secondary
  enable_global_write_forwarding = true

  deletion_protection = true

  depends_on = [aws_rds_cluster.primary]
}

resource "aws_rds_cluster_instance" "dr_reader" {
  provider = aws.dr

  identifier         = "orders-dr-reader"
  cluster_identifier = aws_rds_cluster.dr.id
  instance_class     = "db.r6g.large"  # Smaller for warm standby cost optimization
  engine             = "aurora-postgresql"
  promotion_tier     = 0  # First to promote on in-region failover
}

# ═══════════════════════════════════════════════════════════════════════
# RDS PROXY -- PRIMARY REGION (in-region failover acceleration)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_db_proxy" "primary" {
  provider = aws.primary

  name          = "orders-proxy-primary"
  engine_family = "POSTGRESQL"
  require_tls   = true

  vpc_security_group_ids = [aws_security_group.proxy_primary.id]
  vpc_subnet_ids         = var.primary_private_subnet_ids

  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"
    secret_arn  = aws_rds_cluster.primary.master_user_secret[0].secret_arn
  }

  role_arn            = aws_iam_role.proxy_primary.arn
  idle_client_timeout = 1800

  tags = {
    Name   = "orders-proxy-primary"
    Region = "us-east-1"
    Role   = "primary"
  }
}

resource "aws_db_proxy_default_target_group" "primary" {
  provider = aws.primary

  db_proxy_name = aws_db_proxy.primary.name

  connection_pool_config {
    connection_borrow_timeout    = 120
    max_connections_percent      = 100
    max_idle_connections_percent = 50
  }
}

resource "aws_db_proxy_target" "primary" {
  provider = aws.primary

  db_proxy_name          = aws_db_proxy.primary.name
  target_group_name      = aws_db_proxy_default_target_group.primary.name
  db_cluster_identifier  = aws_rds_cluster.primary.id
}

# ═══════════════════════════════════════════════════════════════════════
# RDS PROXY -- DR REGION (in-region failover acceleration)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_db_proxy" "dr" {
  provider = aws.dr

  name          = "orders-proxy-dr"
  engine_family = "POSTGRESQL"
  require_tls   = true

  vpc_security_group_ids = [aws_security_group.proxy_dr.id]
  vpc_subnet_ids         = var.dr_private_subnet_ids

  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"
    # DR cluster gets its own Secrets Manager secret when promoted to writer
    secret_arn = aws_secretsmanager_secret.dr_db_credentials.arn
  }

  role_arn            = aws_iam_role.proxy_dr.arn
  idle_client_timeout = 1800

  tags = {
    Name   = "orders-proxy-dr"
    Region = "us-west-2"
    Role   = "dr"
  }
}

resource "aws_db_proxy_default_target_group" "dr" {
  provider = aws.dr

  db_proxy_name = aws_db_proxy.dr.name

  connection_pool_config {
    connection_borrow_timeout    = 120
    max_connections_percent      = 100
    max_idle_connections_percent = 50
  }
}

resource "aws_db_proxy_target" "dr" {
  provider = aws.dr

  db_proxy_name          = aws_db_proxy.dr.name
  target_group_name      = aws_db_proxy_default_target_group.dr.name
  db_cluster_identifier  = aws_rds_cluster.dr.id
}

# ═══════════════════════════════════════════════════════════════════════
# RDS CROSS-REGION BACKUP REPLICATION (for a separate RDS PostgreSQL instance)
# Demonstrates the feature for non-Aurora databases
# ═══════════════════════════════════════════════════════════════════════

resource "aws_db_instance" "legacy_rds" {
  provider = aws.primary

  identifier     = "legacy-orders-rds"
  engine         = "postgres"
  engine_version = "16.4"
  instance_class = "db.r7g.xlarge"

  allocated_storage     = 200
  max_allocated_storage = 1000
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.primary.arn

  multi_az                = true
  db_subnet_group_name    = aws_db_subnet_group.primary.name
  vpc_security_group_ids  = [aws_security_group.rds_primary.id]

  backup_retention_period = 35
  backup_window           = "03:00-04:00"

  manage_master_user_password = true
  deletion_protection         = true
}

# Enable cross-region automated backup replication
# This replicates BOTH snapshots AND transaction logs → enables cross-region PITR
resource "aws_db_instance_automated_backups_replication" "legacy_rds" {
  provider = aws.dr

  source_db_instance_arn = aws_db_instance.legacy_rds.arn
  kms_key_id             = aws_kms_key.dr.arn  # Multi-region KMS key in DR region
  retention_period       = 35
}
```

---

## Critical Gotchas and Interview Traps

**1. "PITR always creates a NEW instance -- it never restores in-place."**
Both RDS and Aurora PITR create a new DB instance/cluster with a new endpoint. Your DR runbook must include the step "update application connection strings" or "repoint RDS Proxy target group." The only in-place recovery is Aurora Backtrack (MySQL only, 72-hour max window). This restore-to-new-instance constraint is why Pilot Light and above (with stable, pre-existing endpoints) provide dramatically faster RTO than Backup & Restore.

**2. "RDS cross-region backup replication gives cross-region PITR. AWS Backup cross-region copy does not."**
AWS Backup copies snapshots only. RDS cross-region backup replication copies snapshots AND transaction logs. If the question says "cross-region PITR for an RDS MySQL instance," the answer is RDS cross-region automated backup replication, not AWS Backup.

**3. "`--allow-data-loss` = unplanned failover. Always."**
The `failover-global-cluster --allow-data-loss` CLI flag is the definitive indicator of unplanned failover. The `switchover-global-cluster` command has no `--allow-data-loss` flag because it guarantees zero data loss. If an exam question mentions `--allow-data-loss`, the scenario is an emergency, not a planned migration.

**4. "RDS Proxy is single-region only."**
This is the gotcha that catches architects who assume RDS Proxy can help with cross-region failover. It cannot. Proxy lives in one VPC in one region. For cross-region DR, use the Aurora Global Database global writer endpoint. Deploy a separate RDS Proxy in each region for in-region HA acceleration.

**5. "Aurora PostgreSQL has no cross-region read replicas -- only Global Database."**
Aurora MySQL supports both engine-level cross-region read replicas (via binlog) and Global Database (via storage-level). Aurora PostgreSQL supports ONLY Global Database. If someone asks "how do I get Aurora PostgreSQL in eu-west-1 for DR?" the only answer is Global Database.

**6. "Unplanned failover breaks topology. Switchover preserves it."**
After a managed switchover, both clusters remain in the global database with roles swapped -- you can switchover back trivially. After an unplanned failover, the old primary is detached and must be deleted and re-created as a new secondary. This asymmetry makes failback after unplanned failover significantly more complex and time-consuming.

**7. "Aurora backtrack is MySQL only and must be enabled at cluster creation."**
You cannot add backtrack to an existing cluster. It is Aurora MySQL only (not PostgreSQL). Maximum rewind window is 72 hours (not 35 days like PITR). Backtrack rewinds the entire cluster, not individual tables. And it has no cross-region capability.

**8. "Connection pinning defeats RDS Proxy multiplexing."**
If your application uses prepared statements, temp tables, or session variables heavily, a high percentage of connections will be pinned. Pinned connections are disrupted during failover and reduce the efficiency of connection pooling. Monitor `DatabaseConnectionsCurrentlySessionPinned` in CloudWatch. Refactor session state out of the application where possible.

**9. "Multi-AZ is HA. Cross-region is DR. Both are needed."**
This connects to the same principle from the [DR Strategies doc](../../march/aws/2026-03-30-disaster-recovery-strategies.md). Multi-AZ handles AZ failures within a region (automatic, seconds). Cross-region replication handles Region failures (manual or managed, minutes). An Aurora cluster should be Multi-AZ (multiple instances across AZs) AND in a Global Database (cross-region replication). They are complementary layers.

**10. "Backup retention of 35 days is the max for both RDS and Aurora -- but manual snapshots have no expiration."**
If compliance requires 7-year data retention, automated backups (max 35 days) are insufficient. You need manual snapshots with a lifecycle management process (or AWS Backup Vault Lock with a compliance retention period, as covered in the [AWS Backup doc](../../april/aws/2026-04-01-aws-backup-centralized-protection.md)).

---

## Key Takeaways

- **RDS PITR works by restoring the nearest snapshot plus replaying transaction logs.** The more hours of logs to replay, the longer the restore. This is why RTO for Backup & Restore tier DR can be hours -- it is not just the snapshot restore time, it is the log replay time too. Aurora's continuous backup architecture means PITR does not require replaying hours of transaction logs from a base snapshot -- the storage layer handles reconstruction differently, making Aurora PITR generally faster and more predictable than RDS PITR for the same data volume (though it still creates a new cluster and restore time depends on data size).

- **RDS cross-region backup replication provides cross-region PITR. AWS Backup does not.** This is the critical distinction for database Backup & Restore tier DR. If you need point-in-time recovery in a DR region for an RDS instance, use the native cross-region backup replication feature, not (or in addition to) AWS Backup.

- **Aurora Global Database has two distinct failover operations with different trade-offs.** Managed switchover (`switchover-global-cluster`): RPO=0, preserves topology, requires healthy primary. Managed unplanned failover (`failover-global-cluster --allow-data-loss`): RPO in seconds, breaks topology, works when primary is down. Know which to use when.

- **The global writer endpoint eliminates post-failover endpoint reconfiguration.** This is a data-plane-like mechanism that makes database DR significantly more operational -- applications never need to change connection strings after a failover.

- **RDS Proxy accelerates in-region failover from 60-120s to under 10s but does not help with cross-region DR.** Deploy one proxy per region. For cross-region failover, rely on the global writer endpoint.

- **Connection pinning reduces RDS Proxy effectiveness during failover.** Minimize session state in your application to reduce pinning and maximize the benefit of connection pooling and failover acceleration.

- **Aurora PostgreSQL's only cross-region option is Global Database.** No engine-level cross-region read replicas. This is a hard constraint that determines your DR architecture for PostgreSQL workloads.

- **Failback after unplanned failover is harder than the failover itself.** The old primary must be deleted and re-created as a new secondary before you can switchover back. Contrast with DynamoDB Global Tables where failback is automatic -- all replicas are read-write and replication simply resumes.

- **Every DR strategy needs backups underneath.** Even with Aurora Global Database providing sub-second replication for DR, enable 35-day backup retention in both regions. Replication faithfully copies data corruption across regions in seconds. Only PITR lets you roll back to before the corruption.

---

## Further Reading

- [Aurora Global Database -- Switchover and Failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html) -- Definitive reference for both failover operations
- [Aurora Global Database Writer Endpoint Deep Dive](https://aws.amazon.com/blogs/database/diving-deep-into-the-new-amazon-aurora-global-database-writer-endpoint/) -- October 2024 launch blog with architecture details
- [RDS Snapshot, Restore, and Recovery Demystified](https://aws.amazon.com/blogs/database/amazon-rds-snapshot-restore-and-recovery-demystified/) -- Internal backup mechanics
- [RDS Cross-Region Automated Backup Replication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReplicateBackups.html) -- Cross-region PITR setup
- [RDS Proxy -- How It Works](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html) -- Connection pooling, multiplexing, pinning deep dive
- [Comparing Aurora Replication Solutions](https://docs.aws.amazon.com/prescriptive-guidance/latest/aurora-replication-options/introduction.html) -- Decision framework for Aurora cross-region options
- [RDS & Aurora Deep Dive (Mar 25)](../../march/aws/2026-03-25-rds-aurora-deep-dive.md) -- Multi-AZ, Read Replicas, Aurora architecture, RDS Proxy basics
- [Disaster Recovery Strategies (Mar 30)](../../march/aws/2026-03-30-disaster-recovery-strategies.md) -- Four DR strategies, RPO/RTO, failover decision tree
- [Multi-Region Architecture (Mar 31)](../../march/aws/2026-03-31-multi-region-architecture-patterns.md) -- Data plane vs control plane, routing patterns, static stability
- [AWS Backup (Apr 1)](../../april/aws/2026-04-01-aws-backup-centralized-protection.md) -- Centralized backup governance, cross-region copy, Vault Lock
