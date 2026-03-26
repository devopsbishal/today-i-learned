# S3 Advanced Deep Dive -- The City Archive System Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), and established the global Anycast expressway (Global Accelerator). But where does all the actual *stuff* go? Every application produces data -- documents, images, logs, backups, compliance records, machine learning datasets -- and that data needs to live somewhere that is durable, scalable, and cost-optimized for how it will be accessed over its lifetime. A medical image accessed daily for the first month, weekly for the next year, and perhaps never again afterward should not cost the same to store at each stage. A financial record required by regulators to be immutable for seven years needs stronger protection than a temporary upload. A dataset replicated across continents for disaster recovery demands different guarantees than a developer's test bucket. That is the domain of S3's advanced features -- storage classes, lifecycle policies, replication, access points, Object Lock, versioning, event notifications, and transfer acceleration -- the operational machinery that transforms S3 from a simple object store into an enterprise data management platform.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| S3 Storage Classes | Different floors in a city archive building -- ground floor is instant access, basement levels are cheaper but require longer retrieval | Eight storage tiers with different cost, retrieval speed, and durability trade-offs; objects move down as access frequency decreases |
| S3 Intelligent-Tiering | A smart archive clerk who watches which files you request and automatically moves them between floors | Automatic cost optimization that monitors access patterns and moves objects between tiers with zero retrieval fees but a per-object monitoring charge |
| S3 Express One Zone | A desk drawer in your office -- single-digit millisecond access but only one copy in one location | Ultra-low-latency storage class using directory buckets and session-based auth, co-located with compute for ML/analytics workloads |
| Lifecycle Policies | A filing schedule that says "after 30 days move to the back office, after 90 days send to the warehouse, after 7 years shred" | Automated rules that transition objects between storage classes and expire them based on age, with transition constraints (30/90/180-day minimums) |
| Cross-Region Replication (CRR) | Making a photocopy of every document and mailing it to the branch office in another city for safekeeping | Asynchronous object replication to a bucket in a different AWS region for DR, compliance, or latency reduction |
| Same-Region Replication (SRR) | Making a copy and filing it in a different cabinet in the same building | Replication within the same region for log aggregation, cross-account copies, or data sovereignty |
| Replication Time Control (RTC) | Paying for express courier with a guaranteed 15-minute delivery SLA instead of standard mail | SLA-backed replication ensuring 99.99% of objects replicate within 15 minutes; required for compliance workloads |
| S3 Batch Replication | Hiring a moving crew to retroactively copy all existing files to the new branch office | One-time job to replicate objects that existed before a replication rule was enabled, or to re-replicate failed objects |
| Access Points | Dedicated service windows at the archive -- one for the legal team, one for analytics, one for auditors -- each with its own entry rules | Named network endpoints attached to a bucket, each with its own IAM policy, simplifying access management for shared datasets |
| Multi-Region Access Points (MRAPs) | A single phone number that automatically connects you to the nearest branch office | Global endpoint backed by Global Accelerator that routes requests to the closest bucket based on network latency |
| Object Lock (Compliance Mode) | A time-locked vault -- once sealed, not even the building owner can open it before the timer expires | WORM protection where no one (including root) can delete or shorten the retention period; assessed for SEC 17a-4 |
| Object Lock (Governance Mode) | A locked cabinet where the building manager can override the lock with a special master key | WORM protection that most users cannot bypass, but users with s3:BypassGovernanceRetention can override; useful for testing policies |
| Legal Hold | A "DO NOT DESTROY" sticky note placed on a file during an investigation -- no expiration date | Binary hold flag independent of retention periods; toggled on/off as needed for litigation or regulatory investigations |
| MFA Delete | Requiring both a key card and a fingerprint scan to access the document shredder | Forces the use of MFA (plus root credentials) to disable versioning or permanently delete object versions |
| Event Notifications | An alarm bell that rings whenever someone adds, removes, or modifies a file in the archive | Push notifications to SNS, SQS, Lambda, or EventBridge when S3 events occur (object created, deleted, restored, etc.) |
| Transfer Acceleration | A network of express drop-off points across the city -- drop your package at the nearest one and it takes the express highway to the archive | Uploads route through CloudFront edge locations and travel over the AWS backbone to the bucket region, accelerating long-distance transfers |
| Server Access Logging | The archive's handwritten visitor log -- free but occasionally misses an entry | Best-effort access logging delivered to an S3 bucket; free (storage cost only), captures auth failures, but no delivery guarantee |
| CloudTrail Data Events | A professional audit trail with tamper-proof records and guaranteed delivery | Paid per-event logging with integrity validation, cross-account delivery, and 5-15 minute latency; does not capture auth failures |
| Versioning | The archive keeping every revision of every document instead of overwriting | Every PUT creates a new version with a unique version ID; DELETE adds a delete marker without destroying previous versions |

---

## The Big Picture: S3 as a City Archive System

Think of Amazon S3 as a **city archive system** -- a massive, purpose-built facility for storing every type of document the city produces. The archive has multiple floors, each optimized for a different access pattern and cost profile.

**The ground floor** (S3 Standard) has open shelves -- any clerk can walk over and grab a file in seconds. It is the most expensive floor to maintain because of the lighting, climate control, and staffing, but retrieval is instant.

**The back office** (Standard-IA, One Zone-IA) is down a hallway. Files are just as accessible, but you pay a small fee every time you retrieve one because a clerk must walk further to get it. The rent is cheaper, so you store files here once the initial flurry of access dies down.

**The basement levels** (Glacier tiers) are progressively deeper underground. Glacier Instant Retrieval is a well-organized basement with an elevator -- still millisecond access, but significantly cheaper rent with higher retrieval fees. Glacier Flexible Retrieval requires submitting a retrieval request form and waiting minutes to hours. Glacier Deep Archive is an off-site vault that takes 12-48 hours to retrieve from -- but storage costs less than a tenth of a penny per gigabyte per month.

**The desk drawer** (S3 Express One Zone) breaks the analogy slightly -- it is a small, ultra-fast cache right at your workstation. Single-digit millisecond latency, but it only exists in one location and uses a different addressing system (directory buckets instead of general purpose buckets).

The **lifecycle policy** is the filing schedule posted on the wall: "After 30 days, move from the ground floor to the back office. After 90 days, send to the basement. After 7 years, shred." The **replication system** is the photocopier that sends duplicates to branch offices in other cities. The **Object Lock** is a time-locked vault for regulatory documents. And the **access points** are dedicated service windows -- one for the legal team, one for analytics -- each with its own rules about who can access what.

```
S3 ADVANCED FEATURES -- THE CITY ARCHIVE SYSTEM
════════════════════════════════════════════════════════════════════════════

                    ┌──────────────────────────────────────────────┐
                    │           BUCKET (The Archive Building)       │
                    │                                              │
  ACCESS POINTS ──▶ │  ┌─ Ground Floor (Standard) ──────────────┐  │
  (Service Windows) │  │  Instant access, highest storage cost   │  │
                    │  └─────────────────────────────────────────┘  │
                    │         │ Lifecycle: 30+ days                 │
                    │         ▼                                     │
                    │  ┌─ Back Office (Standard-IA / One Zone-IA)┐  │
                    │  │  Same speed, retrieval fee, cheaper rent │  │
                    │  └─────────────────────────────────────────┘  │
                    │         │ Lifecycle: 90+ days                 │
                    │         ▼                                     │
                    │  ┌─ Basement L1 (Glacier Instant Retrieval) ┐ │
                    │  │  Millisecond access, higher retrieval fee│ │
                    │  └─────────────────────────────────────────┘  │
                    │         │ Lifecycle: 90+ days                 │
                    │         ▼                                     │
                    │  ┌─ Basement L2 (Glacier Flexible Retrieval)┐ │
                    │  │  Minutes-to-hours retrieval              │  │
                    │  └─────────────────────────────────────────┘  │
                    │         │ Lifecycle: 180+ days                │
                    │         ▼                                     │
                    │  ┌─ Off-Site Vault (Glacier Deep Archive) ──┐ │
                    │  │  12-48 hours retrieval, cheapest storage │  │
                    │  └─────────────────────────────────────────┘  │
                    │                                              │
                    │  ┌─ Object Lock (Time-Locked Vault) ────────┐ │
                    │  │  WORM: Governance / Compliance / Legal   │  │
                    │  └─────────────────────────────────────────┘  │
                    └──────────────┬───────────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
     │  Replication  │    │    Event     │    │   Transfer   │
     │  (CRR / SRR)  │    │ Notifications│    │ Acceleration │
     │  Branch copy   │    │  Alarm bell  │    │  Edge upload │
     └──────────────┘    └──────────────┘    └──────────────┘
```

---

## Part 1: Storage Classes -- The Eight Floors of the Archive

S3 offers eight storage classes. Understanding them is less about memorizing numbers and more about understanding the **trade-off axes**: storage cost, retrieval cost, retrieval speed, minimum storage duration, durability, and availability.

### The Complete Storage Class Comparison

| Storage Class | Durability | Availability | AZs | Min Duration | Min Object Size | Retrieval Time | Retrieval Fee | Storage Cost (us-east-1 approx.) |
|---|---|---|---|---|---|---|---|---|
| **S3 Standard** | 99.999999999% (11 9s) | 99.99% | >= 3 | None | None | Milliseconds | None | ~$0.023/GB |
| **S3 Intelligent-Tiering** | 11 9s | 99.9% | >= 3 | None | None | Milliseconds (Frequent/Infrequent/Instant Archive), minutes-hours (Archive/Deep Archive) | None (monitoring fee: $0.0025/1K objects/month) | Varies by tier (same as the tier the object is in) |
| **S3 Standard-IA** | 11 9s | 99.9% | >= 3 | 30 days | 128 KB | Milliseconds | Per-GB retrieval | ~$0.0125/GB |
| **S3 One Zone-IA** | 11 9s | 99.5% | 1 | 30 days | 128 KB | Milliseconds | Per-GB retrieval | ~$0.01/GB |
| **Glacier Instant Retrieval** | 11 9s | 99.9% | >= 3 | 90 days | 128 KB | Milliseconds | Per-GB (highest among Glacier) | ~$0.004/GB |
| **Glacier Flexible Retrieval** | 11 9s | 99.99% (after restore) | >= 3 | 90 days | 40 KB | Expedited: 1-5 min; Standard: 3-5 hrs; Bulk: 5-12 hrs | Per-GB + per-request (varies by speed) | ~$0.0036/GB |
| **Glacier Deep Archive** | 11 9s | 99.99% (after restore) | >= 3 | 180 days | 40 KB | Standard: 12 hrs; Bulk: 48 hrs | Per-GB + per-request | ~$0.00099/GB |
| **S3 Express One Zone** | 11 9s (single AZ) | 99.95% | 1 | None (hourly billing) | None | Single-digit milliseconds | None | ~$0.16/GB (request-based pricing differs) |

**Critical interview details:**

- **All classes have 11 9s durability** (99.999999999%). This means S3 is designed to sustain the loss of data in two facilities simultaneously. Durability does not vary by storage class -- even Glacier Deep Archive has the same durability as Standard. What varies is **availability** (how often you can access the data).

- **One Zone-IA is the exception to multi-AZ storage** -- it stores data in a single AZ, which means if that AZ is physically destroyed, the data is lost. This is why its availability is 99.5% instead of 99.9%. Use it only for data that can be recreated (thumbnails, transcoded media, replicated copies).

- **S3 Express One Zone uses directory buckets**, not general purpose buckets. It has a different API namespace, uses session-based authentication (CreateSession), and is designed for co-location with compute (EC2, SageMaker, Athena) in the same AZ for single-digit millisecond first-byte latency.

### Storage Class Decision Framework

```
STORAGE CLASS DECISION TREE
════════════════════════════════════════════════════════════════════

  How frequently is the data accessed?
       │
       ├── Frequently (multiple times/month)
       │     │
       │     ├── Need sub-10ms latency for compute co-location?
       │     │     YES ──▶ S3 Express One Zone (directory bucket)
       │     │     NO  ──▶ S3 Standard
       │     │
       │
       ├── Unpredictable (sometimes heavy, sometimes none)
       │     │
       │     └── S3 Intelligent-Tiering
       │         (auto-moves between tiers, no retrieval fees,
       │          but $0.0025/1K objects/month monitoring fee --
       │          not cost-effective for millions of known-cold objects)
       │
       ├── Infrequently (once a month or less, but need instant access)
       │     │
       │     ├── Can the data be recreated if an AZ is destroyed?
       │     │     YES ──▶ S3 One Zone-IA (~20% cheaper than Standard-IA)
       │     │     NO  ──▶ S3 Standard-IA
       │     │
       │     └── Accessed roughly once per quarter?
       │           ──▶ Glacier Instant Retrieval
       │               (cheapest with millisecond access, highest retrieval fee)
       │
       ├── Rarely (a few times/year, can wait minutes to hours)
       │     │
       │     └── Glacier Flexible Retrieval
       │         (Expedited: 1-5 min, Standard: 3-5 hrs, Bulk: 5-12 hrs)
       │
       └── Almost never (compliance/archival, can wait 12-48 hours)
             │
             └── Glacier Deep Archive
                 (cheapest storage: ~$1/TB/month)
```

### The Intelligent-Tiering Deep Dive

Intelligent-Tiering deserves special attention because it is the only storage class that **automatically optimizes cost** without operational overhead. Think of it as hiring a **smart archive clerk** who watches which files you pull from the shelves and silently rearranges them:

- **Frequent Access tier** (default landing zone): Same performance and cost as S3 Standard.
- **Infrequent Access tier** (auto after 30 days without access): Same as Standard-IA pricing, ~40% savings.
- **Archive Instant Access tier** (auto after 90 days without access): Same as Glacier Instant Retrieval pricing, ~68% savings.
- **Archive Access tier** (optional, configurable 90-730 days): Same as Glacier Flexible Retrieval pricing.
- **Deep Archive Access tier** (optional, configurable 180-730 days): Same as Glacier Deep Archive pricing.

**Key trade-off**: No retrieval fees (unlike Standard-IA and Glacier classes), but you pay a monitoring fee of $0.0025 per 1,000 objects per month. For a bucket with 100 million objects, that is $250/month in monitoring alone -- making it cost-ineffective for massive buckets of objects you *know* are cold. Use Glacier directly for those.

---

## Part 2: Lifecycle Policies -- The Automated Filing Schedule

Lifecycle policies are the **filing schedule posted on the archive wall** -- automated rules that move objects between storage class floors and eventually shred them when their retention period expires. Without lifecycle policies, objects sit in whatever class they were uploaded to forever, and you overpay for storage of data that no one accesses.

### The Transition Waterfall

Not every storage class can transition to every other. Transitions flow **downward** only -- you cannot use a lifecycle rule to move objects from Glacier back up to Standard (you would need to restore and copy).

```
S3 LIFECYCLE TRANSITION WATERFALL
═══════════════════════════════════════════════════════════════

  S3 Standard
       │
       ├──▶ Intelligent-Tiering
       ├──▶ Standard-IA ───────────────┐
       ├──▶ One Zone-IA ───────────────┤
       ├──▶ Glacier Instant Retrieval ─┤
       ├──▶ Glacier Flexible Retrieval ┤
       └──▶ Glacier Deep Archive ──────┘
                                       │
  Intelligent-Tiering                  │
       ├──▶ One Zone-IA               │
       ├──▶ Glacier Instant Retrieval  │
       ├──▶ Glacier Flexible Retrieval │
       └──▶ Glacier Deep Archive       │
                                       │
  Standard-IA                          │
       ├──▶ One Zone-IA               │
       ├──▶ Glacier Instant Retrieval  │
       ├──▶ Glacier Flexible Retrieval │
       └──▶ Glacier Deep Archive       │
                                       │
  One Zone-IA                          │
       ├──▶ Glacier Flexible Retrieval │
       └──▶ Glacier Deep Archive       │
                                       │
  Glacier Instant Retrieval            │
       ├──▶ Glacier Flexible Retrieval │
       └──▶ Glacier Deep Archive       │
                                       │
  Glacier Flexible Retrieval           │
       └──▶ Glacier Deep Archive       │
                                       │
  Glacier Deep Archive                 │
       └──▶ (TERMINAL -- no lifecycle  │
             transitions out)          │
                                       │
  NOTE: One Zone-IA CANNOT transition  │
  to Glacier Instant Retrieval (both   │
  require different AZ architectures)  │
```

### Transition Constraints -- The Interview Gotchas

These constraints are among the most frequently tested S3 topics:

**1. Minimum 30 days before transitioning to IA classes**
Objects must remain in S3 Standard for at least 30 days before lifecycle can transition them to Standard-IA or One Zone-IA. This prevents you from immediately shuffling objects to a class with a 30-day minimum storage charge -- you would be charged for 30 days regardless.

**2. Minimum object size: 128 KB for IA and Glacier Instant Retrieval**
Objects smaller than 128 KB are not transitioned to Standard-IA, One Zone-IA, or Glacier Instant Retrieval by default. You can override this with `ObjectSizeGreaterThan` / `ObjectSizeLessThan` filters, but doing so is almost always a mistake because of metadata overhead.

**3. Metadata overhead makes tiny-object archival counterproductive**
Each archived object in Glacier incurs **8 KB of metadata stored at S3 Standard rates** plus **32 KB stored at Glacier rates**. For a 1 KB object, you are storing 40 KB of metadata -- 40x the actual data. Archiving millions of tiny files to Glacier can cost *more* than keeping them in Standard.

**4. Early deletion charges**
If you delete or transition an object before its minimum storage duration expires, you are charged for the full minimum period:
- Standard-IA / One Zone-IA: 30 days
- Glacier Instant Retrieval / Glacier Flexible Retrieval: 90 days
- Glacier Deep Archive: 180 days

**5. When multiple rules conflict, deletion takes precedence over transition.**

### Practical Lifecycle Configuration

```json
{
  "Rules": [
    {
      "ID": "StandardToIAToGlacierToDeepArchive",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    },
    {
      "ID": "CleanupIncompleteMultipartUploads",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    },
    {
      "ID": "CleanupNoncurrentVersions",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90,
        "NewerNoncurrentVersions": 3
      }
    },
    {
      "ID": "CleanupExpiredDeleteMarkers",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "Expiration": {
        "ExpiredObjectDeleteMarker": true
      }
    }
  ]
}
```

### Critical Lifecycle Actions Beyond Transitions

| Action | What It Does | Why It Matters |
|--------|-------------|----------------|
| **AbortIncompleteMultipartUpload** | Deletes incomplete multipart upload parts after N days | Orphaned multipart uploads are invisible in the console but consume storage and cost money silently; always set this (7 days is a common choice) |
| **NoncurrentVersionExpiration** | Deletes noncurrent versions after N days, optionally keeping the newest N versions | Without this, versioned buckets grow unbounded; `NewerNoncurrentVersions: 3` keeps the 3 most recent for rollback |
| **ExpiredObjectDeleteMarker** | Removes delete markers when they are the only remaining version | Prevents accumulation of orphaned delete markers that clutter listings and consume minimal but unnecessary storage |

---

## Part 3: Versioning and MFA Delete -- The Revision Archive

### Versioning

Enabling versioning on a bucket means S3 **keeps every revision of every object** instead of overwriting. Think of it as the archive keeping every draft of a contract, each stamped with a unique revision number (version ID).

**How operations behave with versioning enabled:**

- **PUT**: Creates a new version with a unique version ID. The previous version becomes "noncurrent" but is not deleted.
- **GET** (no version ID): Returns the current (latest) version.
- **GET** (with version ID): Returns that specific version -- even if it is noncurrent.
- **DELETE** (no version ID): Does **not** delete anything. Instead, S3 inserts a **delete marker** -- a zero-byte placeholder that tells S3 "this object appears deleted." A subsequent GET returns a 404, but all previous versions still exist.
- **DELETE** (with version ID): **Permanently deletes** that specific version. This is the only way to truly remove data from a versioned bucket.

```
VERSIONING BEHAVIOR
═══════════════════════════════════════════════════════════════

  PUT report.pdf (v1)     PUT report.pdf (v2)     DELETE report.pdf
        │                       │                       │
        ▼                       ▼                       ▼
  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │ report.pdf   │       │ report.pdf   │       │ report.pdf   │
  │              │       │              │       │              │
  │ v1 (current) │       │ v2 (current) │       │ Delete Marker│◀── GET returns 404
  │              │       │              │       │ v2 (noncurr) │
  │              │       │ v1 (noncurr) │       │ v1 (noncurr) │
  └──────────────┘       └──────────────┘       └──────────────┘
                                                      │
                                          To "undelete": remove
                                          the delete marker by
                                          DELETE with its version ID
```

**Versioning states**: A bucket is in one of three states -- Unversioned (default), Versioning-Enabled, or Versioning-Suspended. Once enabled, versioning can be suspended but **never fully disabled**. Suspended versioning stops generating new version IDs (new objects get version ID "null") but preserves all existing versions.

### MFA Delete -- The Two-Factor Shredder

MFA Delete adds a second authentication factor requirement for the two most destructive versioning operations:

1. **Changing the versioning state** of the bucket (disabling or suspending)
2. **Permanently deleting an object version** (DELETE with a version ID)

**Critical constraints:**
- MFA Delete can **only be enabled or disabled by the root account user** -- not by IAM users or roles, even with full admin permissions
- The root account must provide the MFA device serial number and the current token code
- MFA Delete can only be configured via the **CLI or API** -- the console does not support it
- It provides defense-in-depth against compromised IAM credentials: even if an attacker gains admin access, they cannot permanently destroy versioned data without the root account's MFA device

This pairs with the [IAM](2026-03-05-iam-advanced-patterns.md) and [Organizations/SCPs](2026-03-02-aws-organizations-scps.md) controls you have already studied -- MFA Delete is a last-resort protection that operates below the IAM layer.

---

## Part 4: Replication -- The Branch Office Photocopier

S3 replication asynchronously copies objects from a source bucket to one or more destination buckets. Think of it as a photocopier that automatically duplicates every document received at headquarters and mails the copy to a branch office.

### CRR vs SRR

| Feature | Cross-Region Replication (CRR) | Same-Region Replication (SRR) |
|---------|-------------------------------|------------------------------|
| **Source and destination** | Different AWS regions | Same AWS region |
| **Primary use cases** | DR, compliance (geo-separated copies), latency reduction | Log aggregation, cross-account copies, test/prod sync, data sovereignty |
| **Data transfer cost** | S3 CRR data transfer charges apply | No inter-region transfer; standard request charges |
| **Versioning required** | Yes, on both buckets | Yes, on both buckets |

### What Replicates vs What Does Not

This is one of the most gotcha-heavy areas in S3:

| Replicates | Does NOT Replicate |
|---|---|
| New objects created after the rule is enabled | **Objects that existed before replication was enabled** (use Batch Replication) |
| Object metadata and tags | **Lifecycle actions** (each bucket needs its own lifecycle rules) |
| Object ACLs (if configured) | **SSE-C encrypted objects** (customer-provided keys cannot be shared) |
| SSE-S3 encrypted objects (automatic) | **Objects in the source bucket replicated from another source** (no cascading -- A->B->C does not work) |
| SSE-KMS encrypted objects (if configured with the destination KMS key) | **Bucket-level configurations** (lifecycle, notifications, etc.) |
| Delete markers (optional, off by default in newer rules) | **Permanent deletes** (DELETE with version ID) -- by design, to prevent malicious cross-region deletion |

**The no-cascade rule is critical**: If bucket A replicates to bucket B, and bucket B has its own replication rule to bucket C, objects that arrive at B via replication from A will **not** cascade to C. Only objects written directly to B replicate to C. This prevents infinite replication loops but surprises teams building multi-hop replication topologies.

### Replication Time Control (RTC)

Standard replication is best-effort -- most objects replicate within minutes, but there is no SLA. For compliance workloads that require predictable replication times, RTC provides:

- **99.99% of objects replicated within 15 minutes** (SLA-backed)
- **S3 Replication Metrics** enabled automatically (replication latency, bytes pending, operations pending)
- **S3 event notifications** for replication failures
- **Additional cost** on top of standard replication charges

### S3 Batch Replication

When you enable replication on a bucket that already contains objects, those existing objects are **not replicated**. S3 Batch Replication solves this by creating a one-time job that replicates:

- Objects that existed before the replication rule was created
- Objects that previously failed replication
- Objects that were replicated under a previous rule to a different destination

Batch Replication uses S3 Inventory reports (or you provide a CSV manifest) to identify the objects, then processes them at scale. This is the only way to backfill a replication destination with pre-existing data.

### Replication Configuration Example (Terraform)

```hcl
# Source bucket replication configuration
resource "aws_s3_bucket_replication_configuration" "source" {
  bucket = aws_s3_bucket.source.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    filter {
      prefix = ""  # Replicate everything
    }

    # Replicate delete markers (optional, explicit opt-in)
    delete_marker_replication {
      status = "Enabled"
    }

    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"  # Save cost on destination

      # Enable Replication Time Control for SLA-backed replication
      replication_time {
        status  = "Enabled"
        time {
          minutes = 15
        }
      }

      # Enable replication metrics (required when RTC is enabled)
      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }

      # If destination uses SSE-KMS, specify the KMS key
      encryption_configuration {
        replica_kms_key_id = aws_kms_key.destination.arn
      }
    }

    # Enable replication of SSE-KMS encrypted objects
    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"
      }
    }
  }

  depends_on = [aws_s3_bucket_versioning.source]
}
```

---

## Part 5: Access Points -- Dedicated Service Windows

### The Problem Access Points Solve

Imagine a single S3 bucket shared by five teams -- analytics, machine learning, legal, marketing, and DevOps. Without access points, you write one enormous bucket policy with five sections of complex conditional logic. Every time a team's requirements change, you edit this monolithic policy, risking a typo that breaks access for everyone. The bucket policy has a **20 KB size limit**, which large organizations can hit.

### The Solution: Named Endpoints with Independent Policies

An access point is a **named network endpoint** attached to a bucket. Each access point has:

- Its own **unique hostname** (e.g., `analytics-ap-123456789012.s3-accesspoint.us-east-1.amazonaws.com`)
- Its own **IAM access point policy** (independent of the bucket policy)
- Optional **VPC restriction** (restricts access to requests from a specific VPC only)

```
ACCESS POINTS -- DEDICATED SERVICE WINDOWS
═══════════════════════════════════════════════════════════════

                     ┌───────────────────────────┐
                     │     Shared S3 Bucket       │
                     │  "The Central Archive"     │
                     └─────┬─────┬─────┬─────────┘
                           │     │     │
              ┌────────────┘     │     └────────────┐
              ▼                  ▼                  ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │  analytics-ap    │ │  ml-training-ap  │ │  legal-review-ap │
    │                  │ │                  │ │                  │
    │  Policy: read    │ │  Policy: read    │ │  Policy: read    │
    │  prefix: data/   │ │  prefix: models/ │ │  all prefixes    │
    │                  │ │                  │ │                  │
    │  VPC: analytics  │ │  VPC: ml-vpc     │ │  VPC: corp-vpc   │
    │  (VPC-restricted)│ │  (VPC-restricted)│ │  (internet OK)   │
    └──────────────────┘ └──────────────────┘ └──────────────────┘
```

**VPC-restricted access points** are powerful: when you set `NetworkOrigin: VPC` on an access point, S3 rejects any request that does not originate from that VPC. This eliminates the need for complex VPC endpoint policies -- the access point itself enforces the network boundary. This connects to your [VPC](2026/february/aws/) knowledge -- the VPC restriction works alongside gateway VPC endpoints for S3 or interface endpoints.

**Delegation model**: The bucket owner can delegate access management to application teams. The bucket policy grants broad permission to the access point (`"Condition": {"StringEquals": {"s3:DataAccessPointArn": "arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-ap"}}`), and the access point policy handles the fine-grained control. This is analogous to SCPs (which you covered on [Mar 2](2026-03-02-aws-organizations-scps.md)) -- the bucket policy sets the ceiling, and the access point policy narrows it.

### Multi-Region Access Points (MRAPs)

MRAPs provide a **single global endpoint** that routes S3 requests to the closest bucket based on network latency, using AWS Global Accelerator infrastructure (which you covered on [Mar 18](2026-03-18-global-accelerator-edge-services.md)).

**How it works:**
1. You create an MRAP backed by S3 buckets in multiple regions (e.g., us-east-1, eu-west-1, ap-southeast-1)
2. Each backing bucket has CRR configured to replicate to the others (bi-directional)
3. The MRAP provides a single ARN and hostname
4. Client requests hit Global Accelerator's Anycast IPs, which route to the nearest region
5. If a region fails, traffic automatically shifts to the next-closest healthy region

**MRAP hostname format:**
```
<mrap-alias>.accesspoint.s3-global.amazonaws.com
```

**Failover controls**: MRAPs support active-active and active-passive configurations. You can set routing controls to shift traffic between regions -- similar to the traffic dials on Global Accelerator endpoint groups that you studied, but applied at the S3 layer.

---

## Part 6: Object Lock -- The Time-Locked Vault

Object Lock provides WORM (Write Once Read Many) protection for S3 objects. Think of it as a **time-locked vault** in the archive -- once a document goes in and the timer is set, the document cannot be removed until the timer expires.

### Three Protection Mechanisms

**1. Retention Period with Governance Mode -- The Manager's Override Lock**

Most users cannot delete or overwrite the object during the retention period. But users with the `s3:BypassGovernanceRetention` permission can override the lock by including the `x-amz-bypass-governance-retention:true` header. Think of this as a locked cabinet where the building manager has a master key.

**Use case**: Testing retention policies before committing to Compliance Mode. You want to verify your lifecycle and retention settings work correctly without being permanently locked in.

**2. Retention Period with Compliance Mode -- The Unbreakable Vault**

**No one** can delete or overwrite the object -- not the IAM admin, not the root account, not even AWS support. The retention period cannot be shortened. The only way to remove the object before expiration is to delete the entire AWS account (which itself has safeguards). Think of this as a bank vault with a time lock -- once sealed, the vault door physically cannot open until the timer hits zero.

**Use case**: Regulatory compliance for financial records (SEC Rule 17a-4, CFTC Rule 1.31, FINRA). These regulations require that broker-dealer records be stored in non-rewritable, non-erasable format. S3 Object Lock in Compliance Mode has been assessed by Cohasset Associates for these requirements.

**3. Legal Hold -- The Indefinite Sticky Note**

A binary on/off flag placed on an object, independent of any retention period. A Legal Hold has **no expiration date** -- it stays until explicitly removed by a user with `s3:PutObjectLegalHold` permission. An object can have both a retention period AND a legal hold simultaneously, and **both must be cleared** before the object can be deleted.

**Use case**: Litigation holds where the duration is unknown. A lawyer says "preserve all documents related to Case X" -- you apply a Legal Hold, and even after the retention period expires, those objects remain protected until the Legal Hold is removed.

### Object Lock Constraints

- Object Lock can be **enabled on new or existing buckets**. Once enabled, it cannot be disabled, and versioning cannot be suspended. Enabling on an existing bucket does not retroactively lock existing objects -- use **S3 Batch Operations** to apply retention settings to existing object versions
- **Versioning is automatically enabled** and cannot be suspended on a bucket with Object Lock
- Object Lock operates on **individual object versions**, not on the bucket as a whole
- A **default retention configuration** can be set at the bucket level (mode + period), which applies to all new objects uploaded without explicit lock settings
- To apply Object Lock to existing objects at scale, use **S3 Batch Operations** with an S3 Inventory manifest

```
OBJECT LOCK COMPARISON
═══════════════════════════════════════════════════════════════

  ┌─────────────────────┬──────────────────┬──────────────────┬────────────────┐
  │                     │ Governance Mode  │ Compliance Mode  │  Legal Hold    │
  ├─────────────────────┼──────────────────┼──────────────────┼────────────────┤
  │ Who can delete?     │ Users with       │ Nobody           │ Nobody (until  │
  │                     │ BypassGovernance │ (not even root)  │ hold removed)  │
  │                     │ Retention perm   │                  │                │
  ├─────────────────────┼──────────────────┼──────────────────┼────────────────┤
  │ Retention period    │ Can be shortened │ Cannot be        │ No retention   │
  │ modifiable?         │ or removed with  │ shortened or     │ period -- it   │
  │                     │ bypass permission│ removed. Can     │ is binary on/  │
  │                     │                  │ only be extended │ off            │
  ├─────────────────────┼──────────────────┼──────────────────┼────────────────┤
  │ Root account        │ Can bypass       │ Cannot bypass    │ Can remove     │
  │ override?           │ (with permission)│                  │ the hold       │
  ├─────────────────────┼──────────────────┼──────────────────┼────────────────┤
  │ Typical use case    │ Testing policies │ SEC 17a-4,       │ Litigation     │
  │                     │ before compliance│ financial        │ holds with     │
  │                     │ mode commitment  │ regulations      │ unknown end    │
  ├─────────────────────┼──────────────────┼──────────────────┼────────────────┤
  │ Can coexist?        │ Yes -- object    │ Yes -- object    │ Yes -- object  │
  │                     │ can have         │ can have         │ can have both  │
  │                     │ retention +      │ retention +      │ retention +    │
  │                     │ legal hold       │ legal hold       │ legal hold     │
  └─────────────────────┴──────────────────┴──────────────────┴────────────────┘
```

---

## Part 7: Event Notifications -- The Archive Alarm Bell

S3 can notify downstream systems when events occur on objects. Think of it as an **alarm bell** wired to the archive's doors -- whenever someone adds, removes, or modifies a file, the bell rings and connected systems can take action.

### Two Delivery Paths

**1. Classic Path (Direct Destinations)**

S3 sends events directly to one of three targets:

| Target | When to Use | Gotcha |
|--------|------------|--------|
| **SNS** | Fan out to multiple subscribers (email, HTTP, SQS, Lambda) | Topic must have a resource policy granting `s3:Publish` |
| **SQS** | Queue for asynchronous processing by workers | Must be a **standard queue** -- SQS FIFO queues are NOT supported as a direct S3 notification destination |
| **Lambda** | Real-time processing (thumbnail generation, metadata extraction) | Lambda must have a resource-based policy granting S3 invoke permission |

**2. EventBridge Path (Recommended for New Architectures)**

S3 sends all events to Amazon EventBridge (must be explicitly enabled per bucket). EventBridge provides significant advantages:

- **Advanced filtering**: Filter by object key prefix, suffix, size, metadata -- far richer than the classic path's prefix/suffix filters
- **Many more targets**: Over 20 EventBridge targets including SQS FIFO queues, Step Functions, Kinesis, ECS tasks, CodePipeline, and cross-account event buses
- **Archive and replay**: Store events and replay them for debugging or reprocessing
- **Schema registry**: Auto-discover event schemas for code generation

```
S3 EVENT NOTIFICATION -- TWO PATHS
═══════════════════════════════════════════════════════════════

  S3 Bucket Event (e.g., s3:ObjectCreated:Put)
       │
       ├───── Classic Path ──────────────────────────────┐
       │      (direct, simple, limited filtering)        │
       │                                                 │
       │      ┌──▶ SNS Topic ──▶ email, HTTP, SQS, etc. │
       │      ├──▶ SQS Standard Queue (NOT FIFO)         │
       │      └──▶ Lambda Function                       │
       │                                                 │
       └───── EventBridge Path ──────────────────────────┐
              (rich filtering, many targets, replay)     │
                                                         │
              S3 ──▶ EventBridge ──▶ Rules ──▶           │
                        ├──▶ Lambda                      │
                        ├──▶ SQS (Standard AND FIFO)     │
                        ├──▶ Step Functions               │
                        ├──▶ SNS                          │
                        ├──▶ Kinesis Data Streams          │
                        ├──▶ ECS Task                      │
                        ├──▶ CodePipeline                  │
                        ├──▶ Cross-account event bus       │
                        └──▶ 15+ other targets             │
```

**Key event types:**
- `s3:ObjectCreated:*` (Put, Post, Copy, CompleteMultipartUpload)
- `s3:ObjectRemoved:*` (Delete, DeleteMarkerCreated)
- `s3:ObjectRestore:*` (Post for initiation, Completed when ready)
- `s3:Replication:*` (OperationFailedReplication, etc.)
- `s3:LifecycleTransition`, `s3:IntelligentTiering`, `s3:ObjectTagging:*`, `s3:ObjectAcl:Put`

**Critical gotchas:**
- Events are delivered **at least once** -- design your consumers for idempotent processing
- Events may arrive **out of order**
- If a notification target writes back to the same bucket, you can create an **infinite loop** -- always use prefix/suffix filters or separate buckets to prevent this

---

## Part 8: Transfer Acceleration -- The Express Drop-Off Network

Transfer Acceleration uses CloudFront edge locations as **express drop-off points** for uploads. Instead of sending your file across the public internet from Tokyo to a bucket in us-east-1 (variable latency, packet loss), you drop it at the nearest CloudFront edge location (Tokyo), and it travels the rest of the way over the AWS private backbone (optimized, low-latency).

**How it works:**
1. Enable Transfer Acceleration on the bucket
2. Clients use the accelerated endpoint: `mybucket.s3-accelerate.amazonaws.com`
3. The upload hits the nearest CloudFront edge location
4. The data traverses the AWS backbone to the bucket's region
5. The object lands in the bucket

**When it helps most:**
- Long-distance uploads (intercontinental)
- Large files where the backbone optimization compounds
- Situations with high packet loss on the public internet

**When it does NOT help:**
- Uploads from within the same region as the bucket (the "shortcut" is longer than the direct path)
- Small files where connection overhead dominates
- Use the **S3 Transfer Acceleration Speed Comparison Tool** to test whether acceleration improves speed from your location

**Connection to CloudFront**: Transfer Acceleration and CloudFront both use edge locations, but in opposite directions. CloudFront optimizes **downloads** (origin to viewer). Transfer Acceleration optimizes **uploads** (client to bucket). They share the edge infrastructure but serve complementary purposes -- the same edge locations you studied on [Mar 13](2026-03-13-cloudfront-deep-dive.md) that cache your content for viewers also accept accelerated uploads destined for your buckets.

---

## Part 9: Server Access Logging vs CloudTrail Data Events

Both provide visibility into S3 access patterns, but they serve different purposes and have different trade-offs:

| Feature | Server Access Logging | CloudTrail Data Events |
|---------|----------------------|----------------------|
| **Analogy** | Handwritten visitor log at the archive entrance | Professional audit system with tamper-proof records |
| **Delivery guarantee** | **Best-effort** -- logs may be delayed or incomplete | **Guaranteed** -- every event is delivered |
| **Delivery latency** | Within hours (often minutes, but no SLA) | 5-15 minutes |
| **Cost** | **Free** (you pay only for log storage in the destination bucket) | **Paid** -- charged per data event (~$0.10 per 100,000 events) |
| **Captures auth failures** | **Yes** -- records requests that fail authentication | **No** -- only captures authenticated API calls |
| **Log integrity validation** | No | **Yes** -- digest files for tamper detection |
| **Cross-account delivery** | No (logs go to a bucket in the same account) | **Yes** -- centralize trails in a dedicated logging account |
| **Event format** | Space-delimited text (harder to parse) | JSON (structured, query-friendly) |
| **Integration with Athena** | Requires custom table definitions | Native integration with pre-built queries |
| **Risk of logging loop** | **Yes** -- if the target bucket is the same as the source, logs generate more logs; always use a separate bucket | No -- CloudTrail has its own delivery mechanism |

**When to use which:**
- **Server access logging**: Cost-sensitive environments where you need basic access visibility, or when you need to capture authentication failures (e.g., someone trying to access a bucket without credentials)
- **CloudTrail data events**: Security-critical and compliance workloads where you need guaranteed delivery, tamper detection, cross-account centralization, and structured logs for SIEM integration

**Best practice**: Use both. Server access logging for broad, free, baseline coverage (especially useful for capturing auth failures that CloudTrail misses). CloudTrail data events for critical buckets that require audit-grade logging. Pair CloudTrail with the security services you studied -- [GuardDuty](2026-03-06-aws-security-services-threat-detection.md) consumes CloudTrail data events to detect anomalous S3 access patterns.

---

## Part 10: S3 Select (Deprecated for New Customers)

**Note:** AWS closed S3 Select and Glacier Select to new customers effective July 25, 2024. Existing customers can continue using the service.

S3 Select allowed you to push simple SQL queries (SELECT, WHERE) down to S3 so that only matching rows were returned instead of downloading entire objects. It worked on CSV, JSON, and Parquet files, improving query performance by up to 400% (4x) and reducing data scanned/transferred by up to 80%.

**For new architectures**, use:
- **Amazon Athena** for ad-hoc SQL queries on S3 data (serverless, pay-per-query)
- **S3 Object Lambda** for transforming data on retrieval (e.g., filtering, redacting PII)

You may still encounter S3 Select on certification exams and in legacy architectures. Know what it did and that it is sunset.

---

## Complete Architecture: Production S3 Data Lifecycle

```
PRODUCTION S3 ARCHITECTURE -- DATA LIFECYCLE WITH CROSS-REGION DR
═══════════════════════════════════════════════════════════════════════════

  ┌─────────────────── us-east-1 (PRIMARY) ───────────────────┐
  │                                                           │
  │  ┌─ S3 Bucket: prod-data-primary ──────────────────────┐  │
  │  │  Versioning: Enabled                                │  │
  │  │  Object Lock: Compliance Mode (7-year retention)    │  │
  │  │  Encryption: SSE-KMS (key from KMS deep dive)       │  │
  │  │                                                     │  │
  │  │  Access Points:                                     │  │
  │  │  ├── analytics-ap (VPC: analytics-vpc, read-only)   │  │
  │  │  ├── ml-training-ap (VPC: ml-vpc, read-only)        │  │
  │  │  └── app-write-ap (VPC: app-vpc, read+write)        │  │
  │  │                                                     │  │
  │  │  Lifecycle Rules:                                    │  │
  │  │  ├── Current versions:                              │  │
  │  │  │   Standard ──30d──▶ Standard-IA ──90d──▶         │  │
  │  │  │   Glacier IR ──365d──▶ Deep Archive               │  │
  │  │  ├── Noncurrent versions: expire after 90d (keep 3) │  │
  │  │  ├── Abort incomplete multipart: 7 days             │  │
  │  │  └── Cleanup expired delete markers                 │  │
  │  │                                                     │  │
  │  │  Event Notifications (via EventBridge):              │  │
  │  │  ├── ObjectCreated ──▶ Lambda (metadata indexing)   │  │
  │  │  ├── ObjectRemoved ──▶ SQS (audit trail)            │  │
  │  │  └── Replication failure ──▶ SNS (ops alert)        │  │
  │  │                                                     │  │
  │  │  Logging:                                           │  │
  │  │  ├── Server access logs ──▶ s3://logs-bucket        │  │
  │  │  └── CloudTrail data events ──▶ central trail       │  │
  │  └─────────────────────────────────────────────────────┘  │
  │         │                                                 │
  │         │ CRR (with RTC, 15-min SLA)                      │
  │         │ SSE-KMS replication enabled                      │
  │         │ Delete markers replicated                        │
  └─────────┼─────────────────────────────────────────────────┘
            │
            ▼
  ┌─────────────────── eu-west-1 (DR REPLICA) ────────────────┐
  │                                                           │
  │  ┌─ S3 Bucket: prod-data-replica ─────────────────────┐  │
  │  │  Versioning: Enabled                                │  │
  │  │  Encryption: SSE-KMS (eu-west-1 key)                │  │
  │  │                                                     │  │
  │  │  Independent lifecycle rules:                       │  │
  │  │  (Lifecycle does NOT replicate -- must be set        │  │
  │  │   separately on each bucket)                        │  │
  │  │                                                     │  │
  │  │  Storage class on arrival: STANDARD_IA              │  │
  │  │  (DR copy doesn't need frequent-access pricing)     │  │
  │  └─────────────────────────────────────────────────────┘  │
  └───────────────────────────────────────────────────────────┘

  ┌─────────────────── GLOBAL ─────────────────────────────────┐
  │                                                            │
  │  Multi-Region Access Point                                 │
  │  ├── Backed by: prod-data-primary + prod-data-replica      │
  │  ├── Routing: Active-passive (us-east-1 primary)           │
  │  ├── Failover: Shift to eu-west-1 via routing controls     │
  │  └── Uses Global Accelerator Anycast IPs                   │
  │                                                            │
  │  CloudFront Distribution (from Mar 13 deep dive)           │
  │  └── OAC ──▶ prod-data-primary (static content delivery)  │
  │                                                            │
  │  Transfer Acceleration: Enabled on prod-data-primary       │
  │  └── .s3-accelerate endpoint for global upload ingestion   │
  └────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **All eight S3 storage classes have 11 9s durability** -- what varies is availability, retrieval time, and cost. Two classes store data in a single AZ: **One Zone-IA** (cheaper infrequent-access storage) and **S3 Express One Zone** (ultra-low-latency directory buckets co-located with compute). Both are unsuitable for irreplaceable data since an AZ loss destroys the only copy.

- **Intelligent-Tiering is the "set it and forget it" class** but has a monitoring fee that makes it expensive for buckets with millions of known-cold objects. Use Glacier directly when you know the access pattern.

- **The 128 KB minimum object size for transitions is a trap**: Archiving millions of tiny files to Glacier incurs 40 KB of metadata overhead per object (8 KB Standard + 32 KB Glacier), potentially costing more than keeping them in Standard.

- **Lifecycle transitions only flow downward**: Glacier Deep Archive is terminal. You cannot lifecycle-transition an object back to Standard -- you must restore it and copy it.

- **Replication does not replicate existing objects**: New objects only. Use S3 Batch Replication to backfill. Lifecycle actions do not replicate either -- set independent lifecycle rules on each bucket.

- **Permanent deletes (DELETE with version ID) do not replicate by design**: This prevents a compromised account from deleting data in both source and destination. Delete markers can optionally replicate.

- **Replication does not cascade**: A->B replication and B->C replication do not mean A's objects reach C. Only objects written directly to B replicate to C.

- **Object Lock in Compliance Mode is truly immutable**: Not even root can delete the object or shorten the retention period. Test with Governance Mode first. Object Lock can be enabled on new or existing buckets, but once enabled it cannot be disabled.

- **MFA Delete requires root credentials via CLI only**: Even full-admin IAM users cannot enable or use MFA Delete. This is a deliberate last-resort protection layer.

- **EventBridge is the recommended path for S3 event notifications**: It supports richer filtering, more targets (including SQS FIFO), archive/replay, and cross-account routing. Classic notifications to SQS do NOT support FIFO queues.

- **Always set AbortIncompleteMultipartUpload in lifecycle rules**: Orphaned multipart uploads are invisible in the console but silently consume storage. A 7-day cleanup rule is standard practice.

- **Use both server access logging AND CloudTrail data events**: Server access logs are free and capture auth failures. CloudTrail data events provide guaranteed delivery, tamper detection, and structured JSON. They are complementary, not alternatives.

- **Access points simplify multi-team bucket sharing**: Each access point has its own policy and optional VPC restriction, replacing monolithic bucket policies. MRAPs add global routing via Global Accelerator.

- **Transfer Acceleration and CloudFront share edge locations but serve opposite directions**: CloudFront optimizes downloads (origin to viewer). Transfer Acceleration optimizes uploads (client to bucket).

---

## Further Reading

- [S3 Storage Classes Overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html) -- Comparison table for all eight classes
- [Lifecycle Transition Constraints](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html) -- The waterfall diagram and minimum duration rules
- [Replication Overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html) -- CRR, SRR, RTC, and what replicates vs what does not
- [Object Lock Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html) -- Governance, Compliance, Legal Hold, and SEC 17a-4
- [Multi-Region Access Points](https://docs.aws.amazon.com/AmazonS3/latest/userguide/MultiRegionAccessPoints.html) -- Global endpoints with Global Accelerator routing
- [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html) -- Classic and EventBridge delivery paths
- [Transfer Acceleration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transfer-acceleration.html) -- Edge upload acceleration
- Related docs in this repo: [CloudFront & OAC](2026-03-13-cloudfront-deep-dive.md), [KMS & Encryption](2026-03-09-aws-kms-encryption-deep-dive.md), [Global Accelerator](2026-03-18-global-accelerator-edge-services.md), [IAM Advanced Patterns](2026-03-05-iam-advanced-patterns.md), [Organizations & SCPs](2026-03-02-aws-organizations-scps.md), [Security Services](2026-03-06-aws-security-services-threat-detection.md)
