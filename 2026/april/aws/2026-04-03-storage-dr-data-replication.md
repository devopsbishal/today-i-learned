# Storage DR & Data Replication -- The Museum Conservation Analogy

> You have built the managed hotel chain with its revolutionary shared vault system (RDS and Aurora), organized the giant filing cabinet with drawer labels and folder tabs (DynamoDB), deployed the centralized records department (AWS Backup with Vault Lock and cross-region copy), mapped the four-tier hospital emergency preparedness spectrum (DR strategies), designed multi-region architectures with data-plane-first routing, and just yesterday dissected the internal machinery of how databases back up, replicate, fail over, and recover. But databases are only half of the data story. The other half -- often the larger half by volume -- lives in **object storage (S3)** and **key-value tables (DynamoDB)**. These are the storage foundations of serverless architectures, data lakes, application state, user uploads, configuration, and analytics. They replicate differently than databases, they fail differently, and they recover differently. Here is the uncomfortable truth that catches architects in production: **S3 Cross-Region Replication does not replicate existing objects.** You enable CRR on a bucket with 50 million objects, and zero of them appear in your DR region. Only new writes flow. If your DR plan assumed otherwise, your RPO just went from "near-zero" to "infinite for historical data." DynamoDB Global Tables replicate writes across regions in under a second -- but they also replicate corrupt writes in under a second. Your brilliant application bug that zeroed out every user's balance? It is now zeroed out in every region, and replication made sure of it. This is the domain of storage-layer DR: understanding not just which replication knobs to turn, but which **do not** turn, which fail silently, and which require separate backup mechanisms underneath. Think of it as museum conservation -- replication is the traveling exhibition that shows copies of masterpieces in branch museums around the world, but conservation is the climate-controlled vault, the acid-free archival storage, and the restoration workshop that protects the originals from fire, flood, and the slow decay of time. You need both, and confusing one for the other is how collections are lost.

---

**Date**: 2026-04-03
**Topic Area**: aws
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| S3 Cross-Region Replication (CRR) | Sending high-quality reproductions of every new painting to branch museums in other cities -- but only paintings acquired after the agreement was signed | Async replication of new objects to a bucket in another Region; existing objects require a separate Batch Replication job; versioning required on both buckets |
| S3 Same-Region Replication (SRR) | Copying paintings between galleries in the same city -- different buildings, different owners, same postal district | Same mechanics as CRR but within one Region; used for cross-account compliance copies, log aggregation, and live test environments |
| S3 Replication Time Control (RTC) | A contractual guarantee with the courier service that every painting arrives at the branch museum within 15 minutes or you get a refund | SLA-backed 99.9% replication within 15 minutes; enables CloudWatch replication metrics; required for compliance-grade RPO commitments |
| S3 Batch Replication | Hiring a moving company to transport the entire existing collection to the new branch museum -- the courier only handles new acquisitions | Retroactively replicates existing objects, previously failed replications, or objects to newly added destinations using S3 Batch Operations |
| S3 Versioning as DR | The museum keeping every version of a painting -- including the one before a restorer accidentally damaged it -- so you can always go back to the last good version | Every overwrite/delete creates a new version; delete markers are soft deletes; permanent deletion requires specifying the version ID; foundation for all S3 DR |
| Replication-is-not-backup | The branch museum faithfully reproduces even the vandalized painting -- replication copies damage as fast as it copies beauty | Replication protects against infrastructure failure (AZ/Region outage) but NOT logical errors (bad code, accidental deletes, ransomware); separate backup/versioning required |
| DynamoDB Global Tables (MREC) | Branch museums that each have their own restorers working independently -- if two restorers touch the same painting, the last one to finish wins | Multi-Region eventual consistency with last-writer-wins conflict resolution; any replica accepts writes; sub-second async replication; RPO near-zero for infrastructure failures |
| DynamoDB Global Tables (MRSC) | Branch museums connected by pneumatic tubes that require a signed receipt from a second museum before any restoration is considered complete | Multi-Region strong consistency with synchronous replication; RPO=0; exactly 3 regions from predefined sets; significant feature restrictions (no LSI, no TTL, no transactions) |
| DynamoDB PITR | The museum's time-lapse camera that records every change to every painting -- you can rewind to any second in the last 35 days and see exactly what the collection looked like | Continuous backups with per-second granularity; configurable 1-35 day retention; restore creates a NEW table; protects against corruption that replication would spread |
| DynamoDB Export to S3 | Photographing the entire collection and storing the photos in a fireproof archive across town -- even if the museum burns down, the archive survives independently | Full or incremental table export to S3; zero RCU consumption (reads from PITR backups); cross-account/cross-region capable; the cold backup tier for corruption defense |
| Delete marker replication | Telling the branch museum "we removed painting #47 from display" so they remove it too -- but never telling them to permanently destroy it | Optional in CRR V2 configs; replicates soft deletes (delete markers) but NEVER replicates permanent version deletions -- intentional protection against cross-region malicious deletion |

---

## The Museum Conservation Framework

Every concept in storage-layer DR maps to a role in museum conservation. The museum is your data. The conservation program has two distinct functions:

1. **Traveling exhibitions (replication)**: Getting copies of the collection to branch museums around the world so visitors in any city can see the art. If the main museum floods, the branches still have complete copies. This is S3 CRR, DynamoDB Global Tables -- protection against infrastructure failure.

2. **Archival conservation (backup)**: Climate-controlled vaults, acid-free storage, time-stamped photography of every piece, and the ability to restore a painting to its pre-damage state. This is versioning, PITR, Export to S3, AWS Backup -- protection against logical errors.

The critical insight: **a traveling exhibition faithfully reproduces vandalism.** If someone slashes a Monet at the main museum, the branch museums receive a perfect reproduction of the slashed Monet. Only the conservation archives -- which captured the painting before the vandalism -- can restore it.

```
THE MUSEUM CONSERVATION FRAMEWORK
════════════════════════════════════════════════════════════════════════════

  INFRASTRUCTURE FAILURE               LOGICAL ERROR / CORRUPTION
  (Region outage, AZ failure)          (Bad code, accidental delete, ransomware)
         │                                       │
         ▼                                       ▼
  ┌──────────────────┐               ┌──────────────────────┐
  │   REPLICATION     │               │   BACKUP / VERSIONING │
  │                  │               │                      │
  │  S3 CRR/SRR     │               │  S3 Versioning       │
  │  DynamoDB GT     │               │  S3 Object Lock      │
  │  (MREC/MRSC)     │               │  DynamoDB PITR       │
  │                  │               │  DynamoDB Export→S3   │
  │  RPO: seconds    │               │  AWS Backup           │
  │  Copies current  │               │                      │
  │  state instantly │               │  RPO: point-in-time  │
  └──────────────────┘               │  Preserves history   │
         │                           └──────────────────────┘
         │                                       │
         ▼                                       ▼
  "Branch museum has                  "Conservation archive has
   the same collection"               the painting before damage"

  ⚠️  YOU NEED BOTH.  Replication without backup = fast propagation of corruption.
                       Backup without replication = hours/days of data loss on Region failure.
```

---

## Part 1: S3 Replication -- The Traveling Exhibition Program

You covered CRR and SRR conceptually in the [S3 Advanced deep dive (Mar 23)](../../march/aws/2026-03-23-s3-advanced-features.md). Today goes into the operational mechanics that determine whether your DR actually works when you need it.

### What Replicates vs What Does Not -- The Exhibition Contract Fine Print

**The analogy**: When a museum signs an agreement with a branch museum to share its collection, the contract has pages of fine print. Not everything in the main museum goes to the branches. Sculptures in the basement storage? Not included. Pieces on loan from other museums? Not included. Items acquired before the agreement date? Not included unless you pay extra for a one-time bulk transfer.

S3 replication has the same fine print, and failing to read it is how DR plans fail silently.

#### What DOES Replicate

| Object Type | Replicates? | Notes |
|------------|------------|-------|
| New objects created after rule is enabled | Yes | The core use case |
| Unencrypted objects | Yes | Default behavior |
| SSE-S3 encrypted objects | Yes | Default behavior |
| SSE-KMS encrypted objects | Yes, with opt-in | Must specify destination KMS key ARN in replication config |
| Object metadata | Yes | Including user-defined metadata |
| Object tags | Yes | Tag changes trigger replication |
| Object ACL changes | Yes | Only if owner override is not enabled |
| S3 Object Lock retention | Yes | Legal holds and retention settings replicate; **note: Object Lock can only be enabled at bucket creation** -- you cannot add it retroactively to a destination bucket |

#### What Does NOT Replicate

This list is where production surprises and exam questions live:

| Object Type | Replicates? | Why It Matters |
|------------|------------|----------------|
| **Existing objects** (before rule was enabled) | **No** | The #1 gotcha -- 50M existing objects stay only in source; requires Batch Replication |
| **SSE-C encrypted objects** (customer-provided keys) | **No** | AWS cannot access the key to decrypt and re-encrypt for the destination |
| **Objects in Glacier/Deep Archive** | **No** | Must be restored to a retrievable class first |
| **Replicas from another rule** (A->B->C chaining) | **No** | If bucket A replicates to B, and B replicates to C, objects from A do NOT cascade to C |
| **Lifecycle actions** | **No** | Transitions and expirations do NOT replicate; you need independent lifecycle policies on each bucket |
| **Delete markers** (in V2 Filter configs) | **No** (default) | Can be enabled via `DeleteMarkerReplication`; disabled by default in V2 replication configs |
| **Permanent version deletions** (delete of a specific version ID) | **NEVER** | Intentional security protection -- prevents cross-region malicious deletion propagation |
| **Bucket-level configurations** | **No** | Lifecycle rules, notification configs, bucket policies are never replicated |

```
S3 REPLICATION: WHAT FLOWS AND WHAT DOESN'T
════════════════════════════════════════════════════════════════════════════

  Source Bucket (us-east-1)                 Destination Bucket (us-west-2)
  ─────────────────────────                 ───────────────────────────────

  ┌─────────────────────────┐     CRR      ┌─────────────────────────────┐
  │ New objects (post-rule)  │────────────▶│ New objects replicated       │
  │ SSE-S3 / SSE-KMS objects │────────────▶│ Re-encrypted with dest key  │
  │ Tag/metadata changes     │────────────▶│ Tags/metadata synced        │
  │ Object Lock settings     │────────────▶│ Lock replicated             │
  ├─────────────────────────┤             ├─────────────────────────────┤
  │ Delete markers (opt-in)  │─ ─ ─ ─ ─ ▶│ Only if DeleteMarkerRepl ON │
  ├─────────────────────────┤     ╳       ├─────────────────────────────┤
  │ Existing objects (pre)   │─────╳──────│ NOT replicated              │
  │ SSE-C encrypted objects  │─────╳──────│ NOT replicated              │
  │ Glacier/Deep Archive     │─────╳──────│ NOT replicated              │
  │ Lifecycle transitions    │─────╳──────│ NOT replicated              │
  │ Version ID deletes       │─────╳──────│ NEVER replicated (security) │
  │ Replicas from other rules│─────╳──────│ No cascading (A→B→C fails)  │
  │ Bucket configs           │─────╳──────│ NOT replicated              │
  └─────────────────────────┘             └─────────────────────────────┘

  ⚠️  "Existing objects" = anything written BEFORE the replication rule was enabled
       Use Batch Replication to backfill these to the destination
```

### Bidirectional Replication for Active-Active

For Active-Active architectures (discussed in your [Multi-Region Architecture doc (Mar 31)](../../march/aws/2026-03-31-multi-region-architecture-patterns.md)), you need CRR flowing in **both directions**. Configure CRR rules on both buckets pointing at each other, and enable the `ReplicaModifications` setting in `source_selection_criteria` to prevent replication loops -- this tells S3 to replicate changes made to replica objects (metadata updates on the destination) back to the source. Without `ReplicaModifications`, bidirectional CRR creates an infinite replication loop.

### Delete Marker Replication -- The Security Asymmetry

This deserves special attention because it is a common exam trap and a critical DR design decision.

**The analogy**: When the main museum removes a painting from display (a soft delete -- the painting goes to storage, not the incinerator), should the branch museums also remove it? That is configurable. But if the main museum permanently destroys a specific painting (a hard delete -- shredding version v3 specifically), the branch museums are NEVER told to destroy their copy. This is intentional: it prevents a rogue employee at the main museum from destroying art across the entire museum network.

**Technical reality:**
- **Delete markers** (soft deletes when you `DELETE` an object without specifying a version ID): NOT replicated by default in V2 configs (those using `Filter`). Enable explicitly with `DeleteMarkerReplication { Status: Enabled }`. In V1 configs (using `Prefix`), delete markers replicate by default.
- **Version ID deletes** (permanent deletion of a specific version: `DELETE object?versionId=xyz`): NEVER replicated, regardless of configuration. The destination retains that version forever.

This asymmetry is an intentional security feature. If an attacker gains write access to your source bucket and permanently deletes versions, the destination preserves them. Combined with S3 Object Lock on the destination (from your [Mar 23 study](../../march/aws/2026-03-23-s3-advanced-features.md)), this creates a resilient defense against ransomware.

### S3 Replication Time Control (RTC) -- The 15-Minute Courier Guarantee

**The analogy**: Standard replication is like sending paintings to the branch museum via regular post -- they usually arrive within hours but there is no contractual guarantee. RTC is upgrading to a premium courier service with a contractual SLA: 99.9% of shipments arrive within 15 minutes, with GPS tracking on every package and automatic alerts if a delivery is running late.

**The technical reality:**

RTC provides an SLA-backed guarantee that **99.9% of objects replicate within 15 minutes**. Both `ReplicationTime` and `EventThreshold` are fixed at 15 minutes. These are not configurable -- there is no option for tighter or looser thresholds.

**What RTC enables:**
1. **CloudWatch replication metrics** (automatically enabled with RTC):
   - `ReplicationLatency` -- maximum seconds the destination is behind the source
   - `BytesPendingReplication` -- total bytes awaiting replication
   - `OperationsPendingReplication` -- count of operations in the replication queue
   - `OperationsFailedReplication` -- count of failed replications
2. **S3 event notifications** for replication status:
   - `s3:Replication:OperationMissedThreshold` -- object failed to replicate within 15 minutes
   - `s3:Replication:OperationReplicatedAfterThreshold` -- late object finally replicated
3. **Compliance-grade RPO documentation** -- the 15-minute SLA gives auditors a number to write in the RPO column

**Cost and capacity considerations:**
- Default data transfer quota: **1 Gbps** (increase via Support ticket)
- SLA does NOT apply when you exceed request rate guidelines or transfer quotas
- Additional charges: CloudWatch custom metrics + RTC premium ($0.015/GB on top of standard CRR data transfer)
- RTC charges apply on top of standard CRR data transfer costs

### S3 Batch Replication -- The Moving Company for Existing Collections

**The analogy**: When a museum opens a new branch, the regular courier handles all future acquisitions. But the existing collection -- thousands of paintings already on the walls -- needs a moving company. The moving company inventories the collection, packs everything into crates, and ships them to the new branch over days or weeks. During this transit period, the branch has an incomplete collection.

**The technical reality:**

S3 Batch Replication uses S3 Batch Operations under the hood to retroactively replicate objects that live replication missed. Three scenarios require it:

1. **Objects created before the replication rule** -- the most common case when enabling CRR on an existing bucket
2. **Previously failed replications** (status: `FAILED`) -- objects that hit transient errors during live replication
3. **Objects already in an existing destination that need re-replication to a new destination** (status: `REPLICA`) -- when you add a second DR region, objects that are already replicas in the first destination do not cascade to the new destination

**How it works:**
- Source bucket must have an existing replication configuration
- Batch Replication auto-generates a manifest from bucket inventory
- Filter by object creation date, replication status (`NONE`, `FAILED`, `REPLICA`)
- The IAM role needs both replication permissions AND Batch Operations permissions

**The DR implication**: When you add a new DR region, live replication only handles new writes. You MUST run Batch Replication to bring existing data into the new region. During this backfill window, your RPO for existing objects is effectively **unbounded**. For a bucket with 100TB of existing data, this backfill can take hours to days.

### S3 Versioning as a Recovery Mechanism

**The analogy**: The museum keeps every version of every painting in its archive -- the original, the one after the 1987 restoration, the one after the 2015 cleaning. If a restorer makes a mistake today, the curator can retrieve yesterday's version from the archive. Deleting a painting from the gallery just puts a "removed from display" tag on it (delete marker) -- the painting itself remains in the archive. Only someone with archive vault keys who specifies the exact catalog number of a specific version can permanently destroy it.

**Technical reality for DR:**

Versioning is the foundation of all S3 DR because it enables recovery from the most common failure: human error.

| Failure Scenario | Recovery with Versioning |
|-----------------|------------------------|
| Accidental overwrite | Previous version still exists; restore by deleting the current version or copying the old version |
| Accidental `DELETE` (no version ID) | Delete marker created; object appears gone but all versions preserved; remove the delete marker to "undelete" |
| Bad deployment pushes corrupt data | List versions, identify last known good version by timestamp, restore |
| Ransomware encrypts objects | Encrypted data is a new version; original versions untouched if attacker lacks `s3:DeleteObjectVersion` permission |

**The cost trade-off**: Every overwrite stores a complete new copy (not a delta). A 1GB object overwritten 100 times consumes 100GB of version storage. Use lifecycle rules to expire old versions:

```hcl
# Lifecycle rule to manage version accumulation costs
resource "aws_s3_bucket_lifecycle_configuration" "versioning_cleanup" {
  bucket = aws_s3_bucket.primary.id

  rule {
    id     = "expire-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days          = 90   # Keep versions for 90 days
      newer_noncurrent_versions = 5   # Always keep at least 5 recent versions
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER_IR"  # Move old versions to cheaper storage after 30 days
    }
  }
}
```

---

## Part 2: DynamoDB DR Mechanisms -- The Living Archive

You covered DynamoDB Global Tables basics (MREC and MRSC) in the [DynamoDB deep dive (Mar 26)](../../march/aws/2026-03-26-dynamodb-deep-dive.md) and their multi-region context in the [multi-region architecture doc (Mar 31)](../../march/aws/2026-03-31-multi-region-architecture-patterns.md). Today focuses on the DR-specific operational mechanics and the backup tier that protects against replication-propagated corruption.

### DynamoDB Global Tables: MREC vs MRSC -- The DR Perspective

From a DR standpoint, the choice between MREC and MRSC determines your RPO, write latency, and the complexity of your failure recovery:

| Dimension | MREC (Last-Writer-Wins) | MRSC (Strong Consistency) |
|-----------|------------------------|--------------------------|
| **Replication** | Asynchronous (<1 second typical) | Synchronous (write completes only after reaching another region) |
| **RPO for infra failure** | Near-zero (sub-second lag) | Zero (data confirmed in multiple regions before ack) |
| **Write latency** | Local (~5 ms single-digit) | Cross-region (~50-100 ms, waits for remote ack) |
| **Conflict handling** | Last-writer-wins (automatic, silent) | `ReplicatedWriteConflictException` (application must retry) |
| **Region constraints** | Any AWS regions | Predefined sets only (US/EU/AP), exactly 3 regions |
| **Failover** | Application redirects writes to surviving region; no promotion needed; all replicas are read-write | Same -- all replicas accept writes |
| **Failback** | Automatic -- resume writing to recovered region; replication catches up | Same |
| **Feature restrictions** | Standard DynamoDB (GSI, LSI, TTL, Streams, Transactions all work) | **No LSI, no TTL, no transactions; Streams not enabled by default (can be enabled manually)** |
| **CAP behavior** | AP (Available + Partition-tolerant; eventually consistent) | CP (Consistent + Partition-tolerant; rejects writes during partition) |
| **Best for DR** | Most workloads -- tolerates brief inconsistency for full feature set and any-region deployment | Financial, inventory, or coordination systems requiring cross-region strong consistency |

**MRSC limitations deep dive** -- these are hard constraints that eliminate MRSC for many workloads:

- **No Local Secondary Indexes**: If your access patterns require LSIs, MRSC is off the table. GSIs work.
- **No TTL**: Items cannot auto-expire. You must implement application-level expiration.
- **No Transactions**: `TransactWriteItems` and `TransactGetItems` return errors. If your application uses DynamoDB transactions, you must refactor or choose MREC.
- **DynamoDB Streams not enabled by default**: MRSC uses a different internal replication mechanism, so Streams are not automatically enabled. You can enable Streams manually on MRSC replicas for CDC use cases (Lambda triggers, Kinesis Data Streams integration), but be aware the Stream is a separate feed -- it is not the replication channel.
- **Cannot change mode after creation**: You choose MREC or MRSC at table creation time. There is no migration path.
- **Cannot add replicas after creation**: All three regions must be specified upfront.
- **Source table must be empty at creation time**: You cannot convert a populated single-region table to MRSC. The table must have zero items when the MRSC global table is created.

**Partition behavior -- MRSC vs MREC:** During a network partition where Region A is isolated from Regions B and C, the systems diverge sharply. MRSC: Region A's writes **fail** (cannot reach quorum -- only 1 of 3), while Regions B+C continue normally (quorum met with 2 of 3). After the partition heals, Region A syncs up cleanly with zero conflicts because it never accepted divergent data. MREC: all three regions continue accepting writes independently. After healing, last-writer-wins silently discards conflicting writes. This is the CAP theorem in practice -- MRSC chooses consistency (CP), MREC chooses availability (AP).

**The witness architecture**: MRSC requires exactly three regions. You can deploy three full replicas, or two replicas plus one **witness**. The witness is a lightweight component in the third region that participates in the quorum (stores data for consensus) but does not support read/write operations. **Critically, the witness does not appear in your AWS account** -- it is an AWS-managed resource that you will not see in the DynamoDB console or API responses. This reduces cost when you only need two active data regions but need a third for consensus.

### DynamoDB Point-in-Time Recovery (PITR) -- The Time-Lapse Camera

**The analogy**: The museum installs a time-lapse camera system that photographs every painting, every second of every day. If a restorer makes a catastrophic mistake on Tuesday at 3:47 PM, the curator can pull up the photograph from 3:46 PM and see exactly what the painting looked like before the damage. This camera system works independently of the traveling exhibition program -- even if the damage was replicated to all branch museums, the camera archive has the original.

**The technical reality:**

PITR provides continuous, automatic backups with **per-second granularity**. You can restore to any point between `EarliestRestorableDateTime` and `LatestRestorableDateTime` (typically 5 minutes before current time).

**January 2025 update -- configurable recovery periods:**
- Previously fixed at 35 days
- Now configurable from **1 to 35 days** per table
- Critical caveat: **pricing is based on table size** (including LSIs), NOT the recovery period length -- setting 1 day costs the same as 35 days
- When you decrease the recovery period, **older backup data is immediately and irrevocably truncated**

**Operational details that matter for DR:**

1. **Restore ALWAYS creates a new table** -- same constraint as RDS PITR. Your DR runbook must include "create new table, reconfigure application endpoints, then delete old table."

2. **Restored tables do NOT inherit:**
   - DynamoDB Streams settings
   - Auto-scaling policies
   - IAM policies
   - CloudWatch alarms
   - Tags
   - Global Table replica configuration
   - You must reconfigure all of these post-restore via IaC or runbook

3. **Safety net on table deletion**: When a PITR-enabled table is deleted, DynamoDB automatically creates a system backup retained for **35 days at no cost**. This protects against accidental table deletion even if someone bypasses deletion protection.

4. **PITR is the corruption defense**: Global Tables replicate corrupt writes to all regions in under a second. PITR is the only DynamoDB-native mechanism that lets you roll back to before the corruption. This is the "replication-is-not-backup" principle in action.

```
DYNAMODB PITR: THE CORRUPTION RECOVERY TIMELINE
════════════════════════════════════════════════════════════════════════════

  Day 1        Day 2        Day 3  3:46PM  3:47PM       Day 4
    │            │            │       │       │            │
    ▼            ▼            ▼       ▼       ▼            ▼
  ──────────────────────────────────────────────────────────────▶ time
  │◄──── continuous PITR backups (per-second granularity) ────▶│

                                      │
                               Bad deployment
                               corrupts data
                                      │
                              ┌───────┴────────┐
                              │ Global Tables   │
                              │ replicates the  │
                              │ corruption to   │
                              │ ALL regions in  │
                              │ < 1 second      │
                              └────────────────┘

  Recovery: PITR restore to 3:46 PM (one minute before corruption)
            → Creates a NEW table with clean data
            → Reconfigure app to use new table
            → Delete corrupted table
            → Re-enable Global Tables on new table if needed

  ⚠️  PITR is the ONLY defense against corruption in Global Tables.
      Replication faithfully spreads the damage.
```

### DynamoDB Export to S3 -- The Fireproof Archive Across Town

**The analogy**: Beyond the time-lapse camera (PITR), the museum also periodically photographs the entire collection and stores the photos in a fireproof archive building across town. Even if the museum and all its branches burn down simultaneously (an event so catastrophic that even the replication and PITR systems are lost), the archive across town has a complete record.

**The technical reality:**

DynamoDB Export to S3 provides a cold backup tier that operates independently of both Global Tables and PITR:

- **Full export**: Entire table at a point in time
- **Incremental export**: Only items changed since the last export (minimum 10 MB charge per export)
- **Zero RCU consumption**: Reads from PITR continuous backups, not the live table -- no performance impact
- **PITR must be enabled**: Export reads from the PITR backup stream; it is not an independent mechanism
- **Cross-account and cross-region capable**: Export to an S3 bucket in a different account in a different region -- true blast-radius isolation
- **Output formats**: DynamoDB JSON or Amazon Ion
- **Compression**: Optional GZIP or ZSTD

**DynamoDB Import from S3** reverses the process:
- Creates a **new table** (not in-place -- consistent with PITR's constraint)
- Accepts CSV, DynamoDB JSON, or Ion format
- **Zero WCU consumption**: Uses a bulk import path that bypasses the normal write pipeline
- Supports GZIP/ZSTD compressed inputs

**The PITR dependency risk**: Export reads from the PITR backup stream. If someone disables PITR (via API call or IaC change), PITR history is immediately and irrevocably destroyed AND new exports stop working -- one action breaks both recovery tiers. Existing S3 exports survive, but no new ones can be created. Protect PITR enablement with a layered defense: (1) IAM policy denying `dynamodb:UpdateContinuousBackups` for non-emergency roles, (2) Organization-level SCP blocking PITR disablement even for account admins, (3) AWS Config rule (`dynamodb-pitr-enabled`) for continuous compliance detection, (4) CloudTrail + CloudWatch alarm on `UpdateContinuousBackups` API calls for immediate alerting.

**The DR pattern**: Export daily to a cross-region, cross-account S3 bucket as the "last resort" backup tier. This protects against:
- PITR data being lost (if the table AND its PITR backups are somehow compromised)
- Account-level compromise (the export lives in a separate account)
- Corruption that you discover after the PITR retention window has passed

```
DYNAMODB THREE-TIER PROTECTION STACK
════════════════════════════════════════════════════════════════════════════

  Tier 1: REPLICATION (Global Tables)
  ├── RPO: < 1 second (MREC) or 0 (MRSC)
  ├── Protects against: Region/AZ failure
  ├── Does NOT protect: Data corruption (replicates it)
  └── Recovery: Redirect traffic to surviving region (instant)

  Tier 2: POINT-IN-TIME RECOVERY (PITR)
  ├── RPO: Per-second granularity, 1-35 day window
  ├── Protects against: Corruption, accidental writes, bad deployments
  ├── Does NOT protect: Account compromise, PITR data loss
  └── Recovery: Restore to new table + app reconfiguration (minutes-hours)

  Tier 3: EXPORT TO S3 (cross-account, cross-region)
  ├── RPO: Frequency of exports (daily = 24-hour RPO for this tier)
  ├── Protects against: Account compromise, PITR loss, table deletion
  ├── Does NOT protect: Data changed since last export
  └── Recovery: Import from S3 to new table (hours, depending on data size)

  ┌──────────────────────────────────────────────────────────────┐
  │  Production pattern: ALL THREE TIERS simultaneously          │
  │                                                              │
  │  Global Tables (MREC) ──▶ instant Region failover            │
  │       +                                                      │
  │  PITR (35 days)        ──▶ corruption rollback               │
  │       +                                                      │
  │  Daily Export to S3    ──▶ last-resort cold backup            │
  │  (cross-account)           in isolated blast radius           │
  └──────────────────────────────────────────────────────────────┘
```

---

## Part 3: The Replication-is-not-Backup Principle Applied

You first encountered this principle in the [DR Strategies doc (Mar 30)](../../march/aws/2026-03-30-disaster-recovery-strategies.md) and saw it applied to databases [yesterday (Apr 2)](../../april/aws/2026-04-02-database-disaster-recovery.md). Here it is applied specifically to storage services, with the concrete mechanisms mapped:

| Service | Replication Mechanism | What It Protects Against | What It Does NOT Protect Against | Backup Mechanism Needed |
|---------|----------------------|-------------------------|--------------------------------|------------------------|
| **S3** | CRR / SRR | Region/AZ failure, bucket deletion in one region | Corruption (replicated instantly), accidental overwrites (new version replicated) | Versioning (foundational), AWS Backup for S3 (point-in-time with Vault Lock), Object Lock on destination |
| **DynamoDB** | Global Tables (MREC/MRSC) | Region/AZ failure | Corruption (replicated to all regions < 1s), bad application writes | PITR (per-second rollback), Export to S3 (cross-account cold backup) |

**The pattern is universal**: Every replication mechanism in AWS protects against infrastructure failure but NOT against logical errors. Your architecture must layer backup underneath replication for every data service.

For S3, the defense-in-depth stack looks like:

```
S3 DEFENSE-IN-DEPTH STACK
════════════════════════════════════════════════════════════════════════════

  Layer 1: VERSIONING (foundation -- enables everything else)
  ├── Protects against: Accidental overwrite, accidental delete (soft)
  ├── Cost: Storage for every version (manage with lifecycle rules)
  └── Limitation: Does not protect against account compromise or
      malicious permanent version deletion

  Layer 2: CRR WITH RTC (cross-region replication)
  ├── Protects against: Region failure, single-bucket destruction
  ├── RPO: < 15 minutes (with RTC SLA)
  └── Limitation: Replicates corruption; does NOT replicate lifecycle
      actions, existing objects, or permanent version deletes

  Layer 3: AWS BACKUP FOR S3 (from your Apr 1 study)
  ├── Protects against: Corruption (point-in-time snapshots in Vault Lock)
  ├── Cross-region + cross-account copy for blast radius isolation
  └── Limitation: Snapshot-level granularity (not per-second like DB PITR)

  Layer 4: OBJECT LOCK ON DESTINATION (from your Mar 23 study)
  ├── Protects against: Ransomware, malicious deletion, insider threat
  ├── WORM compliance (Governance or Compliance mode)
  └── Limitation: Cannot modify or delete locked objects even if needed

  Layer 5: MFA DELETE (additional protection)
  ├── Protects against: Unauthorized permanent version deletion
  ├── Requires MFA token to delete versions or change versioning state
  └── Limitation: Can only be enabled via root account with CLI (not console)
```

---

## Part 4: Storage-Layer DR Decision Framework

### When to Use Which Combination

```
STORAGE DR DECISION TREE
════════════════════════════════════════════════════════════════════════════

  Is the data in S3 or DynamoDB?

  ┌── S3
  │   │
  │   ├── Need cross-region DR?
  │   │   ├── Yes → CRR + versioning on both buckets
  │   │   │   ├── Need guaranteed RPO < 15 min? → Add RTC
  │   │   │   ├── Have existing objects? → Add Batch Replication job
  │   │   │   ├── Need soft-delete sync? → Enable DeleteMarkerReplication
  │   │   │   └── Need corruption protection? → AWS Backup + Object Lock on dest
  │   │   │
  │   │   └── No (same region) → SRR for cross-account or compliance copies
  │   │
  │   └── Need compliance/immutability?
  │       ├── Yes → Object Lock (Compliance mode) + Vault Lock via AWS Backup
  │       └── No → Versioning + lifecycle rules for version management
  │
  └── DynamoDB
      │
      ├── Need cross-region DR?
      │   ├── Yes, eventual consistency OK → Global Tables MREC
      │   │   ├── Always add PITR (35 days) for corruption defense
      │   │   └── Add daily Export to S3 (cross-account) for cold backup
      │   │
      │   └── Yes, need strong consistency → Global Tables MRSC
      │       ├── Check constraints: no LSI, no TTL, no transactions,
      │       │   Streams not default, predefined region sets only
      │       ├── All constraints acceptable? → MRSC with witness for cost savings
      │       └── Any constraint blocks? → MREC + application-level consistency
      │
      └── Single-region only?
          ├── Enable PITR (configurable 1-35 days)
          ├── Add Export to S3 for cold backup tier
          └── AWS Backup for centralized governance
```

### Cost Comparison

| DR Pattern | S3 Cost Components | Approx. Monthly Addition |
|-----------|-------------------|------------------------|
| **Versioning only** | Storage for all versions | +20-100% storage (depends on churn rate) |
| **CRR without RTC** | Cross-region data transfer ($0.02/GB) + destination storage + PUT requests | +30-50% of source storage cost |
| **CRR with RTC** | Above + CloudWatch metrics + RTC premium | +35-55% of source storage cost |
| **Batch Replication** | One-time: Batch Operations job fee ($0.25/million objects) + data transfer | One-time cost proportional to existing data volume |
| **AWS Backup for S3** | Backup storage (per GB-month) + cross-region transfer | +10-20% depending on frequency |

| DR Pattern | DynamoDB Cost Components | Approx. Monthly Addition |
|-----------|------------------------|------------------------|
| **PITR** | 20% of table + index storage per month | +20% of storage cost (same regardless of retention period) |
| **Global Tables (MREC)** | Replicated write units (same price as local WCU) + storage in each region | +100% per additional region (writes + storage) |
| **Global Tables (MRSC)** | Higher write cost (synchronous) + storage in each region + witness | +120-150% per additional region |
| **Export to S3 (full, daily)** | $0.10/GB per export (us-east-1) + S3 storage | Proportional to table size |
| **Export to S3 (incremental)** | $0.10/GB changed (min 10 MB, us-east-1) + S3 storage | Much lower for low-churn tables |

---

## Production Terraform Example: Multi-Region Storage DR

This example builds a complete storage-layer DR stack: S3 with CRR + RTC + versioning and DynamoDB with Global Tables (MREC) + PITR + Export to S3.

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
# KMS KEYS (Multi-Region for cross-region encryption)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_kms_key" "primary" {
  provider = aws.primary

  description             = "Storage DR primary region key"
  deletion_window_in_days = 30
  multi_region            = true
  enable_key_rotation     = true
}

resource "aws_kms_replica_key" "dr" {
  provider = aws.dr

  primary_key_arn         = aws_kms_key.primary.arn
  description             = "Storage DR replica key in us-west-2"
  deletion_window_in_days = 30
}

# ═══════════════════════════════════════════════════════════════════════
# S3: PRIMARY BUCKET WITH VERSIONING
# ═══════════════════════════════════════════════════════════════════════

resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "acme-data-primary-us-east-1"

  tags = {
    Environment = "production"
    DR          = "source"
  }
}

resource "aws_s3_bucket_versioning" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id

  versioning_configuration {
    status = "Enabled"  # Required for CRR; foundation of all S3 DR
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.primary.arn
    }
    bucket_key_enabled = true  # Reduces KMS API calls by 99%
  }
}

# Lifecycle rule to manage version accumulation cost
resource "aws_s3_bucket_lifecycle_configuration" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id

  rule {
    id     = "manage-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days          = 90
      newer_noncurrent_versions = 5  # Always keep last 5 versions regardless of age
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER_IR"
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# S3: DR BUCKET WITH VERSIONING + INDEPENDENT LIFECYCLE
# ═══════════════════════════════════════════════════════════════════════

resource "aws_s3_bucket" "dr" {
  provider = aws.dr
  bucket   = "acme-data-dr-us-west-2"

  tags = {
    Environment = "production"
    DR          = "destination"
  }
}

resource "aws_s3_bucket_versioning" "dr" {
  provider = aws.dr
  bucket   = aws_s3_bucket.dr.id

  versioning_configuration {
    status = "Enabled"  # Required on destination for CRR
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "dr" {
  provider = aws.dr
  bucket   = aws_s3_bucket.dr.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_replica_key.dr.arn
    }
    bucket_key_enabled = true
  }
}

# CRITICAL: Lifecycle rules do NOT replicate -- must be set independently
resource "aws_s3_bucket_lifecycle_configuration" "dr" {
  provider = aws.dr
  bucket   = aws_s3_bucket.dr.id

  rule {
    id     = "manage-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days          = 90
      newer_noncurrent_versions = 5
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER_IR"
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# S3: CROSS-REGION REPLICATION WITH RTC
# ═══════════════════════════════════════════════════════════════════════

resource "aws_iam_role" "replication" {
  provider = aws.primary
  name     = "s3-crr-replication-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "replication" {
  provider = aws.primary
  name     = "s3-crr-replication-policy"
  role     = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Resource = aws_s3_bucket.primary.arn
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ]
        Resource = "${aws_s3_bucket.primary.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",  # Required for delete marker replication
          "s3:ReplicateTags"
        ]
        Resource = "${aws_s3_bucket.dr.arn}/*"
      },
      {
        # KMS permissions for SSE-KMS encrypted objects
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = [
          aws_kms_key.primary.arn,
          aws_kms_replica_key.dr.arn
        ]
      }
    ]
  })
}

resource "aws_s3_bucket_replication_configuration" "primary_to_dr" {
  provider = aws.primary

  depends_on = [
    aws_s3_bucket_versioning.primary,
    aws_s3_bucket_versioning.dr
  ]

  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.primary.id

  rule {
    id     = "replicate-all-to-dr"
    status = "Enabled"

    # V2 config using filter (required for RTC and delete marker replication control)
    filter {}  # Empty filter = replicate all objects

    # Delete marker replication -- explicitly enabled for DR consistency
    delete_marker_replication {
      status = "Enabled"
    }

    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD"

      # SSE-KMS encryption on destination using the replica KMS key
      encryption_configuration {
        replica_kms_key_id = aws_kms_replica_key.dr.arn
      }

      # Replication Time Control -- 15-minute SLA
      replication_time {
        status = "Enabled"
        time {
          minutes = 15  # Only valid value
        }
      }

      # Replication metrics -- automatically enabled with RTC
      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15  # Only valid value
        }
      }
    }

    # Source object encryption -- opt-in for SSE-KMS objects
    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"
      }
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# CLOUDWATCH ALARM: Replication lag monitoring
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_metric_alarm" "replication_lag" {
  provider = aws.primary

  alarm_name          = "s3-crr-replication-lag-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ReplicationLatency"
  namespace           = "AWS/S3"
  period              = 300
  statistic           = "Maximum"
  threshold           = 900  # 15 minutes in seconds -- matches RTC SLA
  alarm_description   = "S3 CRR replication latency exceeds 15 minutes (RTC SLA threshold)"

  dimensions = {
    SourceBucket     = aws_s3_bucket.primary.id
    DestinationBucket = aws_s3_bucket.dr.id
    RuleId           = "replicate-all-to-dr"
  }

  alarm_actions = [var.sns_topic_arn]
  ok_actions    = [var.sns_topic_arn]
}

resource "aws_cloudwatch_metric_alarm" "replication_failures" {
  provider = aws.primary

  alarm_name          = "s3-crr-replication-failures"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "OperationsFailedReplication"
  namespace           = "AWS/S3"
  period              = 300
  statistic           = "Sum"
  threshold           = 0  # Alert on ANY failure
  alarm_description   = "S3 CRR replication failures detected -- investigate immediately"

  dimensions = {
    SourceBucket     = aws_s3_bucket.primary.id
    DestinationBucket = aws_s3_bucket.dr.id
    RuleId           = "replicate-all-to-dr"
  }

  alarm_actions = [var.sns_topic_arn]
}

# ═══════════════════════════════════════════════════════════════════════
# DYNAMODB: GLOBAL TABLE WITH PITR + EXPORT TO S3
# ═══════════════════════════════════════════════════════════════════════

resource "aws_dynamodb_table" "orders" {
  provider = aws.primary

  name         = "orders"
  billing_mode = "PAY_PER_REQUEST"  # On-demand for unpredictable DR workloads
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "GSI1PK"
    type = "S"
  }

  attribute {
    name = "GSI1SK"
    type = "S"
  }

  # Global Table replica in DR region (MREC -- eventual consistency)
  replica {
    region_name = "us-west-2"
    kms_key_arn = aws_kms_replica_key.dr.arn

    # PITR on the DR replica -- independent from primary
    point_in_time_recovery {
      enabled = true
    }
  }

  # PITR on primary -- 35-day maximum retention for corruption defense
  point_in_time_recovery {
    enabled = true
  }

  # GSI for access pattern flexibility
  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "ALL"
  }

  # TTL for automatic item expiration (works with MREC, NOT with MRSC)
  ttl {
    attribute_name = "ExpiresAt"
    enabled        = true
  }

  # Streams for event-driven architecture (works with MREC, NOT with MRSC)
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.primary.arn
  }

  deletion_protection_enabled = true

  tags = {
    Environment = "production"
    DR          = "global-table-mrec"
  }
}

# ═══════════════════════════════════════════════════════════════════════
# S3 BUCKET FOR DYNAMODB EXPORTS (cross-region cold backup tier)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_s3_bucket" "dynamodb_exports" {
  provider = aws.dr  # Exports go to DR region for blast radius isolation
  bucket   = "acme-dynamodb-exports-us-west-2"

  tags = {
    Environment = "production"
    Purpose     = "dynamodb-cold-backup"
  }
}

resource "aws_s3_bucket_versioning" "dynamodb_exports" {
  provider = aws.dr
  bucket   = aws_s3_bucket.dynamodb_exports.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "dynamodb_exports" {
  provider = aws.dr
  bucket   = aws_s3_bucket.dynamodb_exports.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_replica_key.dr.arn
    }
    bucket_key_enabled = true
  }
}

# Lifecycle to transition old exports to cheaper storage
resource "aws_s3_bucket_lifecycle_configuration" "dynamodb_exports" {
  provider = aws.dr
  bucket   = aws_s3_bucket.dynamodb_exports.id

  rule {
    id     = "archive-old-exports"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "GLACIER_IR"  # Millisecond retrieval for recent exports
    }

    transition {
      days          = 90
      storage_class = "DEEP_ARCHIVE"  # Lowest cost for older exports
    }

    expiration {
      days = 365  # Retain exports for 1 year
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# OUTPUTS
# ═══════════════════════════════════════════════════════════════════════

output "s3_primary_bucket" {
  value = aws_s3_bucket.primary.id
}

output "s3_dr_bucket" {
  value = aws_s3_bucket.dr.id
}

output "dynamodb_table_arn" {
  value = aws_dynamodb_table.orders.arn
}

output "dynamodb_table_stream_arn" {
  value = aws_dynamodb_table.orders.stream_arn
}
```

---

## Critical Gotchas and Interview Traps

**1. "S3 CRR does NOT replicate existing objects. You must use Batch Replication."**
This is the #1 S3 replication misconception. Enabling CRR on a bucket with 50 million existing objects replicates zero of them. Only new writes flow. Your DR region is incomplete until you run a Batch Replication job. During the backfill, your RPO for existing objects is unbounded.

**2. "Permanent version deletions are NEVER replicated -- by design."**
Deleting a specific version ID in the source does not propagate to the destination. This is an intentional security feature that prevents cross-region malicious deletion. If someone asks "how do I replicate permanent deletes?" the answer is: you cannot and should not.

**3. "Delete marker replication is OFF by default in V2 configs."**
V2 replication configs (using `Filter` element) do not replicate delete markers by default. You must explicitly enable `DeleteMarkerReplication`. V1 configs (using `Prefix`) replicate delete markers by default. This V1/V2 behavioral difference is a common exam trap.

**4. "Lifecycle actions do NOT replicate. Set them independently on each bucket."**
If your source bucket transitions objects to Glacier after 90 days, the destination does not inherit that rule. You must configure identical (or appropriate) lifecycle policies on every replication destination independently. Forgetting this causes storage cost explosion on the destination.

**5. "S3 replication does not chain. A->B->C does not work."**
If bucket A replicates to B, and B has its own replication rule to C, objects originating from A that land in B as replicas do NOT continue to C. Each replication hop is independent. For A->B->C, you need explicit rules from A to both B and C.

**6. "RTC is always 15 minutes. No other value is configurable."**
Both `ReplicationTime` and `EventThreshold` only accept 15 minutes. You cannot set a tighter SLA. If you need sub-minute RPO for S3, CRR with RTC is insufficient -- consider a write-through pattern where the application writes to both regions simultaneously.

**7. "DynamoDB PITR costs the same whether you set 1 day or 35 days."**
Pricing is 20% of table + index storage per month, regardless of the recovery period. There is no cost incentive to reduce the retention period. Set it to 35 days unless compliance requires shorter retention (some regulations mandate data destruction within specific windows).

**8. "DynamoDB restore always creates a NEW table -- and it inherits almost nothing."**
The new table gets the data. It does NOT get: Streams settings, auto-scaling, IAM policies, CloudWatch alarms, tags, or Global Table replica configuration. Your DR runbook must include reconfiguration steps for all of these, or better yet, your IaC should be able to rebuild the complete table configuration pointing at the restored table.

**9. "MRSC cannot be changed after creation and the table must be empty."**
You cannot convert an existing MREC table to MRSC or vice versa. You cannot add replicas to an existing MRSC table. The source table must be empty when enabling MRSC. These constraints make MRSC a Day 1 architectural decision, not something you can retrofit.

**10. "DynamoDB Export to S3 requires PITR to be enabled."**
Export reads from the PITR continuous backup stream, not the live table. If PITR is disabled, Export to S3 does not work. This is why PITR should always be enabled in production -- it powers both point-in-time recovery AND the cold export backup tier.

---

## Key Takeaways

- **S3 CRR only replicates new objects.** Existing objects require Batch Replication. Every CRR setup on an existing bucket should immediately be followed by a Batch Replication job to backfill historical data to the destination.

- **Permanent version deletions never cross the replication boundary.** This is the intentional security asymmetry that protects your DR copy from malicious deletion at the source. Combined with Object Lock on the destination, it creates a resilient defense against ransomware.

- **Replication Time Control provides the only SLA-backed RPO guarantee for S3.** The 15-minute threshold is fixed and non-negotiable. RTC automatically enables CloudWatch replication metrics that are essential for monitoring replication health and proving RPO compliance to auditors.

- **Lifecycle rules must be configured independently on every bucket.** They do not replicate. Forgetting this is how destination buckets accumulate unlimited versions and objects in expensive storage classes, inflating costs silently.

- **DynamoDB Global Tables (MREC) provide instant cross-region failover with near-zero RPO for infrastructure failures.** But they also provide instant cross-region propagation of corruption. PITR is the mandatory companion mechanism that enables rollback to before the corruption event.

- **MRSC is powerful but narrow.** No LSI, no TTL, no transactions, Streams not enabled by default (though manually enableable), predefined region sets only, cannot change after creation. Use it only when cross-region strong consistency with RPO=0 is a genuine requirement and your workload can live within every restriction.

- **DynamoDB's three-tier protection stack (Global Tables + PITR + Export to S3) mirrors S3's defense-in-depth stack (CRR + Versioning + AWS Backup).** Both follow the same principle: replication for infrastructure failure, backup for logical errors, with each tier addressing a different failure mode at a different cost point.

- **Every storage DR architecture needs both replication AND backup.** Replication without backup is fast propagation of corruption. Backup without replication is hours of data loss on Region failure. The complete architecture layers both with independent monitoring and testing.

---

## Further Reading

- [What Does Amazon S3 Replicate?](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-what-is-isnot-replicated.html) -- Definitive reference for S3 replication edge cases
- [S3 Replication Time Control and Metrics](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-time-control.html) -- RTC SLA, CloudWatch metrics, event notifications
- [S3 Batch Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-batch-replication-batch.html) -- Retroactive replication for existing objects and failed replications
- [DynamoDB PITR How It Works](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/PointInTimeRecovery_Howitworks.html) -- Continuous backup mechanics and configurable recovery periods
- [DynamoDB Multi-Region Strong Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/multi-region-strong-consistency-gt.html) -- MRSC constraints, region sets, witness architecture
- [DynamoDB Export to S3](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataExport.HowItWorks.html) -- Full/incremental export mechanics
- [Designing a Resilient Backup Strategy for Amazon S3](https://aws.amazon.com/blogs/storage/designing-a-resilient-and-cost-effective-backup-strategy-for-amazon-s3/) -- CRR vs AWS Backup as complementary layers
- [S3 Advanced Features (Mar 23)](../../march/aws/2026-03-23-s3-advanced-features.md) -- Storage classes, lifecycle, versioning, Object Lock, CRR/SRR basics
- [DynamoDB Deep Dive (Mar 26)](../../march/aws/2026-03-26-dynamodb-deep-dive.md) -- Partition keys, GSI/LSI, Streams, Global Tables basics
- [Disaster Recovery Strategies (Mar 30)](../../march/aws/2026-03-30-disaster-recovery-strategies.md) -- Four DR strategies, RPO/RTO, replication-is-not-backup principle
- [Multi-Region Architecture (Mar 31)](../../march/aws/2026-03-31-multi-region-architecture-patterns.md) -- Data patterns, CAP/PACELC, routing
- [AWS Backup (Apr 1)](../../april/aws/2026-04-01-aws-backup-centralized-protection.md) -- Centralized backup governance, Vault Lock, cross-region/cross-account
- [Database DR Patterns (Apr 2)](../../april/aws/2026-04-02-database-disaster-recovery.md) -- RDS/Aurora backup internals, failover procedures
