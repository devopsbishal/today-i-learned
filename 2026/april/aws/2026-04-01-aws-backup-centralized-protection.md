# AWS Backup & Centralized Protection -- The Corporate Records Management Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock), equipped your instances with personal workbenches, shared libraries, and specialty studios (EBS, EFS, FSx), built the managed hotel chain with its revolutionary shared vault system (RDS and Aurora), organized the giant filing cabinet with drawer labels and folder tabs (DynamoDB with DAX and Global Tables), installed the deli counter for lightning-fast repeat orders (ElastiCache with Global Datastore), mastered the hospital emergency preparedness playbook (DR Strategies), and composed all those services into a coherent multinational corporation operating across countries (Multi-Region Architecture). Every one of those services has its own backup mechanism -- RDS has automated snapshots, DynamoDB has PITR, EBS has Data Lifecycle Manager, S3 has versioning and replication. But here is the problem: **each department keeps its own records in its own way, on its own schedule, with its own retention rules, and nobody has a unified view of what is backed up, what is not, and whether any of it actually works.** This is where AWS Backup enters -- not as another backup mechanism, but as the **centralized records management department** that orchestrates, governs, and audits all backups across your entire organization from a single control plane. The jump from "each service backs itself up" to "a central authority governs all backups" is the jump from every employee keeping their own notes in their desk drawer to a corporate records management department with standardized policies, fireproof vaults, offsite storage, and annual audits.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| AWS Backup Service | A corporate records management department that does not create the original documents but orchestrates when copies are made, where they are stored, how long they are kept, and whether the system actually works | Centralized backup orchestration across 20+ AWS services with unified scheduling, retention, cross-region/cross-account copies, and compliance auditing |
| Backup Plan | The records retention policy document that specifies "financial records: copy daily, move to cold storage after 90 days, destroy after 7 years" with different rules for different document types | A set of backup rules defining schedule (cron), lifecycle (cold storage transition, deletion), and copy actions (cross-region) applied to selected resources |
| Backup Rule | A single line item in the retention policy -- "daily at 3 AM, keep for 35 days, send a copy to the offsite vault" | One scheduling entry within a backup plan specifying frequency, retention, and optional cross-region copy |
| Backup Selection | The list of departments (or tag criteria) that a particular retention policy applies to -- "this policy covers all departments tagged Finance" | Tag-based or ARN-based resource assignment that determines which AWS resources a backup plan protects |
| Backup Vault (Standard) | The records room in your building -- locked, access-controlled, but the building manager can still open it and shred documents if authorized | Default storage location for recovery points, encrypted with KMS, access-controlled via vault access policies |
| Vault Lock (Governance) | A lock on the records room that most employees cannot open, but the Chief Records Officer can override with proper authorization | WORM protection that prevents most users from deleting backups, but privileged IAM principals can remove the lock |
| Vault Lock (Compliance) | A time-lock safe where even the CEO and the board of directors cannot open it before the set date -- once sealed, nobody can delete or modify the contents, period | Irreversible WORM protection: after a mandatory 3-day cooling-off period, no one (including root, including AWS Support) can delete recovery points or remove the lock |
| Logically Air-Gapped Vault | An offsite records storage facility owned and operated by a third-party company -- even if burglars take over your entire building, they cannot touch the records at the offsite facility because it is a completely separate operation with its own security | Backups stored in an AWS service-owned account, isolated from your account; always has Compliance-mode Vault Lock; shared via AWS RAM for restore |
| Cross-Region Copy | Sending a copy of every important document to the branch office in another city, so a fire at headquarters does not destroy all copies | Copy rules within backup plans that automatically replicate recovery points to a vault in another AWS Region |
| Cross-Account Backup | Sending copies to a separate legal entity's vault -- even if your company is sued and assets are frozen, the copies at the separate entity are protected | Copying recovery points to a backup vault in a different AWS account, typically a dedicated backup account in your Organization |
| Organization Backup Policies | Corporate headquarters issuing a company-wide records policy that automatically applies to every subsidiary and regional office through the organizational hierarchy | Backup policies defined at the Organization or OU level that automatically deploy backup plans to member accounts without per-account configuration |
| Restore Testing | The annual fire drill for your records -- actually retrieving documents from the vault, verifying they are readable, and confirming the recovery process works before a real emergency | Automated restore validation that creates resources from recovery points on a schedule, optionally runs validation checks, then cleans up |
| Backup Audit Manager | The internal audit team that continuously checks whether every department is following the records retention policy and produces compliance reports for regulators | Compliance frameworks with control templates that continuously evaluate backup coverage, encryption, frequency, and vault lock status, producing daily reports |

---

## The Big Picture: Why Centralized Backup Orchestration Matters

You already know how to back up individual services. From the RDS/Aurora deep dive (Mar 25), you know Aurora has continuous backups with 5-minute PITR granularity and 35-day retention. From the DynamoDB deep dive (Mar 26), you know DynamoDB offers PITR with 35-day continuous backup. From the block storage deep dive (Mar 24), you know EBS snapshots are incremental and can be managed by Data Lifecycle Manager. From the S3 deep dive (Mar 23), you know S3 has versioning, lifecycle policies, and cross-region replication.

So why do you need AWS Backup at all?

Consider a real organization with 15 AWS accounts, each running dozens of services. Without centralized backup:

- The payments team manually configured RDS snapshots with 7-day retention, but nobody verified it meets the 90-day regulatory requirement
- The analytics team forgot to enable DynamoDB PITR on their new table -- it has been running for 6 months with zero backup protection
- Three teams independently set up cross-region snapshot copies, each using different KMS keys with no consistent naming or lifecycle
- Nobody has ever tested whether any of these backups actually restore successfully
- When the auditor asks "show me proof that all production databases are backed up with 35-day retention," the engineering team spends two weeks running manual queries across 15 accounts

This is the **desk drawer problem**: every employee has notes, but there is no corporate records policy, no central vault, no audit trail, and no proof the system works. AWS Backup solves this by providing a single control plane that sits above all service-native backup mechanisms and adds orchestration, governance, and compliance.

```
THE CENTRALIZED BACKUP ARCHITECTURE
════════════════════════════════════════════════════════════════════════════

  WITHOUT AWS BACKUP (Desk Drawer Chaos):

  ┌── Account A ──┐   ┌── Account B ──┐   ┌── Account C ──┐
  │ RDS: 7-day    │   │ RDS: 14-day   │   │ RDS: ???      │
  │ DynamoDB: no  │   │ DynamoDB: yes │   │ DynamoDB: yes │
  │   PITR        │   │ EBS: manual   │   │ EBS: DLM      │
  │ EBS: DLM      │   │   snapshots   │   │ EFS: no backup│
  │ No cross-     │   │ Cross-region  │   │ No cross-     │
  │   region      │   │   yes         │   │   region      │
  │ No audit      │   │ No audit      │   │ No audit      │
  └───────────────┘   └───────────────┘   └───────────────┘
      ↑ Each team manages backups independently -- inconsistent,
        unaudited, untested, no central visibility

  WITH AWS BACKUP (Corporate Records Department):

                    ┌─────────────────────────────────────┐
                    │   AWS BACKUP (Management Account    │
                    │   or Delegated Admin Account)       │
                    │                                     │
                    │   Organization Backup Policies      │
                    │     ├── Production OU: daily,       │
                    │     │   35-day retention, cross-    │
                    │     │   region, vault lock          │
                    │     └── Dev OU: weekly, 7-day       │
                    │         retention, no cross-region  │
                    │                                     │
                    │   Audit Manager: continuous         │
                    │     compliance monitoring           │
                    │                                     │
                    │   Restore Testing: monthly          │
                    │     automated validation            │
                    └────────┬────────┬────────┬──────────┘
                             │        │        │
                    ┌────────▼──┐ ┌───▼─────┐ ┌▼──────────┐
                    │ Account A │ │Account B│ │ Account C │
                    │ Backup    │ │ Backup  │ │ Backup    │
                    │ plan auto-│ │ plan    │ │ plan auto-│
                    │ deployed  │ │ auto-   │ │ deployed  │
                    │ via org   │ │ deployed│ │ via org   │
                    │ policy    │ │         │ │ policy    │
                    │           │ │         │ │           │
                    │ All       │ │ All     │ │ All       │
                    │ resources │ │resources│ │ resources │
                    │ tagged    │ │ tagged  │ │ tagged    │
                    │ "backup:  │ │ "backup:│ │ "backup:  │
                    │  prod"    │ │  prod"  │ │  prod"    │
                    │ auto-     │ │ auto-   │ │ auto-     │
                    │ protected │ │protected│ │ protected │
                    └───────────┘ └─────────┘ └───────────┘
```

---

## Core Concepts

### Backup Plans -- The Records Retention Policy

**The Analogy**: A records retention policy at a corporation specifies exactly what gets copied, when, where copies go, how long they are kept, and when to move old records from the premium filing cabinet to the cheaper warehouse. Different document types have different rules -- financial records might be kept for 7 years, HR records for the employee's tenure plus 5 years, and marketing materials for 1 year.

**The Technical Reality**: A backup plan is a collection of **backup rules**, each defining:

| Rule Component | Records Analogy | Technical Detail |
|----------------|-----------------|------------------|
| Schedule | "Copy financial records every business day at 5 PM" | Cron expression: `cron(0 5 ? * * *)` for daily at 5 AM UTC |
| Start Window | "The copy clerk must start copying within 1 hour of the scheduled time, or skip this cycle" | Maximum time (minutes) after the schedule before the backup job is canceled if it has not started (default: 8 hours) |
| Completion Window | "Copying must finish within 6 hours of starting" | Maximum time (minutes) a backup job can run before being canceled (must be > start window) |
| Lifecycle | "After 90 days, move copies from the filing cabinet to the warehouse; after 7 years, shred them" | `cold_storage_after` (days) transitions to cold storage (~75% cheaper); `delete_after` (days) permanently deletes |
| Copy Action | "Send a duplicate to the branch office in Chicago" | Cross-region copy to a destination vault ARN with its own lifecycle rules |
| Continuous Backup | "Record every change in real time, not just daily snapshots" | Enables PITR for supported resources (S3, RDS, Aurora, SAP HANA on EC2). Note: DynamoDB has native PITR but AWS Backup creates on-demand snapshots only |
| Recovery Point Tags | "Stamp every copy with 'Department: Finance' for tracking" | Tags applied to recovery points for organizational and cost-tracking purposes |

**Critical detail about cold storage**: Not all resource types support cold storage transitions. Resource types that support cold storage include: EBS, EFS, DynamoDB, FSx (Lustre and Windows File Server), Timestream, SAP HANA on EC2, CloudFormation, and VMware. Notably, Aurora, RDS, and S3 recovery points **cannot** be moved to cold storage -- they remain in warm storage for their entire lifecycle. Minimum cold storage retention is 90 days (you pay for 90 days even if you delete earlier).

### Backup Selections -- Who Gets Covered

**The Analogy**: The retention policy is useless without knowing which departments it applies to. The records manager can assign the policy by either listing specific departments by name (ARN-based) or by saying "every department with a 'compliance-required' badge on their door" (tag-based).

**The Technical Reality**: A backup selection links a backup plan to resources using three methods:

1. **Tag-based** (recommended for scale): All resources with tag `Environment = prod` are automatically protected. New resources tagged correctly are picked up without any backup plan changes.
2. **Resource ARN** (explicit): List specific resource ARNs. Precise but requires manual updates.
3. **Wildcard ARN**: Use `arn:aws:ec2:*:*:volume/*` to select all EBS volumes. Broad but simple.

Tag-based selection is the enterprise pattern because it shifts backup responsibility to resource creators: "Tag your resource `Backup = daily-prod`, and the backup plan automatically protects it." Any untagged resource is visibly unprotected in Audit Manager reports.

### Backup Vaults -- The Three-Tier Vault System

This is the architectural centerpiece. AWS Backup offers three vault tiers, each adding a layer of protection. Think of them as three levels of records storage security.

```
THE THREE-TIER VAULT HIERARCHY
════════════════════════════════════════════════════════════════════════════

  TIER 1: STANDARD VAULT                    TIER 2: VAULT LOCK              TIER 3: LOGICALLY
  ──────────────────────                     ──────────────────              AIR-GAPPED VAULT
                                                                            ───────────────
  ┌───────────────────────┐    ┌───────────────────────┐    ┌───────────────────────────────┐
  │  Records Room         │    │  Time-Lock Safe       │    │  Offsite Facility             │
  │                       │    │                       │    │  (Third-Party Operated)       │
  │  ─ Locked door        │    │  ─ Everything in      │    │                               │
  │  ─ Access-controlled  │    │    Tier 1, plus:      │    │  ─ Everything in Tier 2, plus:│
  │  ─ KMS encrypted      │    │                       │    │                               │
  │  ─ Vault access       │    │  ─ GOVERNANCE mode:   │    │  ─ Stored in AWS service-     │
  │    policy (resource-  │    │    Most users cannot   │    │    owned account (logically   │
  │    based)             │    │    delete; privileged  │    │    isolated from YOUR account)│
  │                       │    │    admins CAN override │    │                               │
  │  ─ IAM users with     │    │    No cooling-off     │    │  ─ Always Compliance mode     │
  │    permission CAN     │    │    period              │    │    (cannot be removed)        │
  │    delete recovery    │    │                       │    │                               │
  │    points             │    │  ─ COMPLIANCE mode:    │    │  ─ Even if attacker owns      │
  │                       │    │    3-day cooling-off   │    │    your entire account, they  │
  │                       │    │    period, then NO ONE │    │    CANNOT access or delete    │
  │                       │    │    can delete -- not   │    │    backups in this vault      │
  │                       │    │    root, not AWS       │    │                               │
  │                       │    │    Support             │    │  ─ Shared via AWS RAM for     │
  │                       │    │                       │    │    restore operations          │
  │                       │    │  ─ Min/max retention   │    │                               │
  │                       │    │    enforcement         │    │  ─ AWS-owned KMS key by       │
  │                       │    │                       │    │    default (or customer CMK)   │
  └───────────────────────┘    └───────────────────────┘    └───────────────────────────────┘

  Use case:                    Use case:                    Use case:
  Dev/staging backups,         Regulatory compliance        Ransomware defense,
  non-critical workloads       (HIPAA, SOC2, SEC 17a-4),   critical production
                               protection against insider   data, environments
                               deletion                     with high blast-radius
                                                            compromise risk
```

**Vault Lock -- Governance vs Compliance**: If you studied S3 Object Lock in the S3 deep dive (Mar 23), this is the exact same two-mode pattern. Governance mode is the "break glass" option -- most users cannot delete, but privileged admins with `s3:BypassGovernanceRetention` (for S3) or equivalent Backup permissions can override. Compliance mode is the "throw away the key" option -- after the mandatory 3-day grace period expires, the lock is permanent and irreversible. AWS cannot remove it even if you file a support ticket.

**The 3-day grace period**: When you enable Compliance mode, you have a cooling-off period of at least 3 days (configurable via `changeable_for_days`) during which you can still remove or modify the lock. This prevents accidental permanent lockout. Once the grace period expires, the lock is permanent for the lifetime of the vault -- you will pay for storage until every recovery point naturally expires based on the retention period.

**Logically Air-Gapped Vaults -- The Ransomware Defense**: This is the newest vault type and represents the strongest protection AWS offers. The key insight: in a traditional vault (even with Compliance Lock), the vault exists in **your account**. If an attacker gains root access to your account, they cannot delete locked recovery points, but they own the account. With a logically air-gapped vault, the recovery points are stored in an **AWS service-owned account** -- a separate account that you do not own and cannot access directly. Even with full account compromise, the attacker has no path to the backups.

As of late 2025, you can **back up directly** to a logically air-gapped vault, eliminating the previous workflow of first backing up to a standard vault and then copying. This reduces both cost (no duplicate storage) and complexity.

To restore from an air-gapped vault, you share it with your account (or another account) via **AWS RAM** -- the same service used for sharing Transit Gateways, subnets, and other resources across accounts. The restore happens from the shared vault into your target account.

### Legal Hold -- Litigation Preservation

**The Analogy**: A corporate legal hold is a directive from the legal department that says "preserve all documents related to Project X from January through March -- nobody can shred them regardless of the normal retention schedule, because we are in litigation." It targets specific documents, not the entire vault.

**The Technical Reality**: Legal Hold is distinct from Vault Lock. While Vault Lock protects the entire vault from deletion, Legal Hold targets **specific recovery points** filtered by resource type, resource ARN, and creation date range. When a legal hold is placed, matching recovery points cannot be deleted even if their lifecycle retention period expires -- they are preserved until the hold is explicitly removed.

Key distinctions:
- **Vault Lock**: Vault-wide protection, prevents deletion of ALL recovery points in the vault
- **Legal Hold**: Targeted protection, prevents deletion of SPECIFIC recovery points matching filter criteria
- Both can coexist: a vault can have Vault Lock AND individual recovery points under Legal Hold
- Legal Hold is the mechanism for SEC 17a-4, FINRA 4511, and litigation preservation requirements

### S3 Backup Nuances

S3 backup through AWS Backup has important operational constraints worth noting (building on your S3 deep dive from Mar 23):

- **Versioning must be enabled** on the bucket before AWS Backup can protect it
- **EventBridge must be enabled** on the bucket -- if EventBridge notifications are disabled, continuous backup stops silently (a common operational gotcha)
- Only buckets with fewer than **3 billion objects** are supported
- **SSE-C encrypted objects** (customer-provided keys) are not supported
- Cross-region/cross-account copies of continuous S3 backups lose PITR capability -- they become point-in-time snapshots
- AWS introduced a **low-cost warm storage tier** for S3 backups (up to 30% savings)

### Cross-Region and Cross-Account -- The Geographic Distribution Strategy

**The Analogy**: A corporation with offices only in New York faces total records loss if a catastrophe destroys the city. Smart records management sends copies to Chicago (cross-region) and to a separate legal entity's vault (cross-account). The Chicago copy survives a New York disaster. The separate entity's copy survives even if the parent corporation's assets are frozen in a lawsuit.

**Cross-Region Copy**: Defined as a `copy_action` within a backup rule. Each copy action specifies a destination vault ARN in another Region and its own lifecycle (cold storage transition, retention). This is the mechanism that powered the "AWS Backup: cross-region copy rules" block in the Warm Standby architecture from the DR strategies doc (Mar 30).

```
CROSS-REGION + CROSS-ACCOUNT COPY FLOW
════════════════════════════════════════════════════════════════════════════

  Account A (Production) -- us-east-1
  ┌─────────────────────────────────────────────────────────────────────┐
  │  Backup Plan: "prod-daily"                                         │
  │    Rule 1: Daily at 3 AM → Vault: prod-vault (us-east-1)          │
  │      copy_action → Vault: prod-vault-dr (us-west-2)  ─────┐      │
  │      copy_action → Vault: backup-acct-vault (us-east-1) ──┼──┐   │
  │                                                            │  │   │
  │  [Aurora] [DynamoDB] [EBS] [EFS] [EC2]                     │  │   │
  │      ↑ all tagged "Backup = daily-prod"                    │  │   │
  └────────────────────────────────────────────────────────────┼──┼───┘
                                                               │  │
  Account A (Production) -- us-west-2                          │  │
  ┌────────────────────────────────────────────────────────────┼──┤───┐
  │  Vault: prod-vault-dr  ◀───────────────────────────────────┘  │   │
  │    (cross-region copy, same account)                          │   │
  │    Vault Lock: Compliance mode (immutable after grace period) │   │
  └───────────────────────────────────────────────────────────────┤───┘
                                                                  │
  Account B (Backup Account) -- us-east-1                         │
  ┌───────────────────────────────────────────────────────────────▼───┐
  │  Vault: backup-acct-vault  ◀─────────────────────────────────────│
  │    (cross-account copy, dedicated backup account)                │
  │    Vault Lock: Compliance mode                                   │
  │    Logically Air-Gapped: Yes                                     │
  │                                                                   │
  │  WHY: Even if Account A is fully compromised, this account       │
  │  is a separate blast radius with its own IAM, its own root,      │
  │  and vault lock preventing deletion.                              │
  └───────────────────────────────────────────────────────────────────┘
```

**Cross-Account Backup**: Requires AWS Organizations (which you covered on Mar 2). The management account or a delegated administrator enables cross-account backup, and vault access policies on the destination vault grant the source account permission to copy recovery points. The recommended architecture is a **dedicated backup account** -- a member account in the Organization whose sole purpose is receiving and protecting backup copies, with restrictive SCPs preventing anyone from deleting vault locks or disabling backup.

### Organization Backup Policies -- Centralized Governance at Scale

**The Analogy**: Instead of each office manager writing their own records policy, corporate headquarters writes a company-wide policy and pushes it down through the organizational hierarchy. The New York headquarters policy says "all financial records: daily backup, 7-year retention." The US subsidiary's policy adds "plus weekly full copies to the Chicago office." Each regional office inherits both policies, and they **merge** -- the resulting policy at any office is the combination of all policies from every level above it.

**The Technical Reality**: Organization backup policies are an AWS Organizations policy type (like SCPs, tag policies, or AI services opt-out policies). They are defined in JSON and attached at the Organization root, OU, or individual account level.

Key behaviors:

1. **Inheritance with operators, not simple merging**: Backup policies use **inheritance operators** (`@@assign`, `@@append`, `@@remove`) that determine how parent and child policies combine. A child OU's policy can add new backup rules via `@@append`, override parent values via `@@assign`, or remove inherited entries via `@@remove`. Crucially, the parent can control what children are allowed to change using **child control operators** (`@@operators_allowed_for_child_policies`) -- for example, the parent can allow children to append new rules but prevent them from overriding the daily backup frequency. This is the governance mechanism: the Organization root establishes baseline protection that child OUs can extend but not weaken (when operators are configured correctly).

2. **Inheritance flow**: Policies flow down the OU tree. An account in `Production/US-East` inherits policies from `Root → Production → US-East`, with operators at each level controlling how values combine.

3. **Delegated administrator**: **One account** can be designated as the delegated administrator for AWS Backup. This avoids using the management account for day-to-day backup operations, reducing blast radius (a principle you covered in the Organizations deep dive on Mar 2).

4. **Resource opt-in**: AWS Backup requires you to explicitly opt-in each resource type before it can be managed. Newer resource types (S3, DynamoDB advanced features, SAP HANA, Timestream) are **not opted in by default**. A common operational mistake: someone creates a backup plan for S3, applies it via tag, and nothing happens because S3 is not opted in. Organization-level opt-in settings override member account settings -- if the Organization says "opt in EFS," member accounts cannot opt out.

### Restore Testing -- Proving Backups Actually Work

**The Analogy**: Your records department can file copies all day long, but if nobody ever retrieves a document from the vault to verify it is readable and complete, you have Schrodinger's backup -- it might work, it might not, and you will not know until a real disaster. The annual fire drill for records is the restore test: pull a document, verify it is legible, then put it back.

**The Technical Reality**: As we discussed in the DR strategies doc (Mar 30), "a DR plan that has never been tested is a hope, not a plan." Restore testing automates this validation:

1. **Restore Testing Plan**: Define a schedule (daily, weekly, monthly), select which backup plans or resource types to test, and specify a recovery point selection strategy (latest, random from last 24h, etc.)
2. **Automated Restore**: AWS Backup restores the selected recovery points into actual AWS resources in your account
3. **Validation Window** (optional): 1-168 hours during which you can run custom validation scripts (data integrity checks, application health tests) against the restored resources
4. **Automatic Cleanup**: After the validation window (or immediately if no validation), the restored resources are automatically deleted to avoid cost accumulation
5. **Results Tracking**: Every restore test produces a status (succeeded, failed, timed out) visible in the AWS Backup console and reportable via Audit Manager

**Best practice**: Run restore tests in a dedicated test account with appropriate IAM roles and networking. This avoids restored resources interfering with production workloads and keeps test costs isolated.

```
RESTORE TESTING FLOW
════════════════════════════════════════════════════════════════════════════

  ┌── Restore Testing Plan ───────────────────────────────────────────┐
  │  Schedule: Weekly (every Sunday 2 AM UTC)                         │
  │  Resource types: Aurora, DynamoDB, EBS                            │
  │  Recovery point selection: Latest successful                      │
  │  Validation window: 4 hours                                       │
  └───────────┬───────────────────────────────────────────────────────┘
              │
              ▼
  ┌── Step 1: Select Recovery Points ─────────────────────────────────┐
  │  Aurora cluster: recovery point from Saturday 3 AM                │
  │  DynamoDB table: recovery point from Saturday 3 AM                │
  │  EBS volume: recovery point from Saturday 3 AM                    │
  └───────────┬───────────────────────────────────────────────────────┘
              │
              ▼
  ┌── Step 2: Restore Resources ──────────────────────────────────────┐
  │  Creates: test-restore-aurora-20260401                            │
  │  Creates: test-restore-dynamodb-20260401                          │
  │  Creates: test-restore-ebs-20260401                               │
  └───────────┬───────────────────────────────────────────────────────┘
              │
              ▼
  ┌── Step 3: Validation Window (4 hours) ────────────────────────────┐
  │  Optional: Run custom validation scripts via EventBridge + Lambda │
  │    ─ Query Aurora: SELECT COUNT(*) FROM orders (verify data)      │
  │    ─ Scan DynamoDB: verify item count matches expected            │
  │    ─ Mount EBS: verify filesystem integrity                       │
  └───────────┬───────────────────────────────────────────────────────┘
              │
              ▼
  ┌── Step 4: Automatic Cleanup ──────────────────────────────────────┐
  │  Deletes: test-restore-aurora-20260401                            │
  │  Deletes: test-restore-dynamodb-20260401                          │
  │  Deletes: test-restore-ebs-20260401                               │
  │  Records: Test status (PASSED / FAILED) in Backup console        │
  └───────────────────────────────────────────────────────────────────┘
```

### Backup Audit Manager -- Continuous Compliance Monitoring

**The Analogy**: The internal audit team does not manage records themselves -- they check whether the records department is following policy. They continuously walk through departments asking: "Are all financial records being backed up daily? Are all copies encrypted? Are backups stored in a locked vault? Does every critical system have a backup plan?" They produce a daily report for the compliance officer.

**The Technical Reality**: Audit Manager creates **compliance frameworks** consisting of **controls** (rules that evaluate backup compliance). AWS provides pre-built control templates:

| Control Template | What It Checks |
|------------------|----------------|
| `BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN` | Are all resources of specified types covered by at least one backup plan? |
| `BACKUP_PLAN_MIN_FREQUENCY_AND_MIN_RETENTION_CHECK` | Do backup plans meet minimum backup frequency and retention period? |
| `BACKUP_RECOVERY_POINT_ENCRYPTED` | Are all recovery points encrypted? |
| `BACKUP_RECOVERY_POINT_MANUAL_DELETION_DISABLED` | Are recovery points protected from manual deletion (via vault lock)? |
| `BACKUP_RESOURCES_PROTECTED_BY_BACKUP_VAULT_LOCK` | Are backups stored in a vault with vault lock enabled? |
| `BACKUP_LAST_RECOVERY_POINT_CREATED` | Has a recovery point been created within the required timeframe? |
| `BACKUP_RECOVERY_POINT_MINIMUM_RETENTION_CHECK` | Do recovery points meet minimum retention requirements? |
| `BACKUP_RESOURCES_PROTECTED_BY_CROSS_REGION` | Are resources protected with cross-region backup copies? |
| `BACKUP_RESOURCES_PROTECTED_BY_CROSS_ACCOUNT` | Are resources protected with cross-account backup copies? |

**Prerequisite**: AWS Config must be enabled in the account. Audit Manager uses Config rules under the hood to evaluate compliance.

**Report delivery**: Daily compliance reports are delivered to an S3 bucket in CSV and/or JSON format, suitable for integration with security dashboards, SIEM tools, or regulatory audit submissions.

---

## Supported Resource Types -- Not Everything Is Equal

AWS Backup supports 20+ resource types, but feature availability varies significantly. This matrix matters for architecture decisions:

```
FEATURE AVAILABILITY MATRIX (KEY RESOURCE TYPES)
════════════════════════════════════════════════════════════════════════════

                       Cross-   Cross-    Cold      Continuous  Restore   Air-
  Resource Type       Region   Account   Storage   Backup      Testing   Gapped
  ──────────────      ──────   ───────   ───────   ──────────  ───────   ──────
  EC2 (AMI)            Yes      Yes       No        No          Yes       Yes
  EBS                  Yes      Yes       Yes       No          Yes       Yes
  RDS                  Yes      Yes       No        Yes (PITR)  Yes       Yes
  Aurora               Yes      Yes       No        Yes (PITR)  Yes       Yes
  DynamoDB             Yes      Yes       Yes       No*         Yes       Yes
  EFS                  Yes      Yes       Yes       No          Yes       Yes
  FSx (Lustre/Win)     Yes*     Yes*      Yes       No          Yes*      Yes*
  S3                   Yes      Yes       No        Yes (PITR)  Yes       Yes
  Neptune              Yes      Yes       No        No          Yes       No
  DocumentDB           Yes      Yes       No        No          Yes       No
  Redshift             Yes      Yes       No        No          No        No
  SAP HANA on EC2      Yes      Yes       Yes       Yes (PITR)  Yes       Yes
  Timestream           Yes      Yes       Yes       No          Yes       Yes
  CloudFormation       Yes      Yes       Yes       No          No        No
    (stack templates)

  * FSx feature availability varies by FSx type (Lustre, Windows, ONTAP, OpenZFS)
  * DynamoDB has native PITR, but AWS Backup creates on-demand snapshots only (not continuous)

  KEY GAPS TO REMEMBER:
  ─ Cold storage: EBS, EFS, DynamoDB, FSx (Lustre/Windows), Timestream, SAP HANA,
    CloudFormation, VMware. Notably NOT RDS, Aurora, or S3.
  ─ Continuous backup (PITR) via AWS Backup: Only S3, RDS, Aurora, SAP HANA
    (DynamoDB PITR is a native feature, not managed through AWS Backup)
  ─ Air-gapped vault support continues to expand -- check the latest docs
```

---

## AWS Backup vs Service-Native Backups -- Decision Framework

This is the most practical question: when should you use AWS Backup versus the service's built-in backup mechanism?

```
DECISION FRAMEWORK: AWS BACKUP vs SERVICE-NATIVE
════════════════════════════════════════════════════════════════════════════

  "Should I use AWS Backup for this service?"

  START
    │
    ├── Do you need CROSS-ACCOUNT backup copies?
    │     YES → AWS Backup (service-native backups cannot copy cross-account)
    │
    ├── Do you need CENTRALIZED compliance auditing across services?
    │     YES → AWS Backup (Audit Manager provides unified compliance view)
    │
    ├── Do you need ORGANIZATION-WIDE backup policy enforcement?
    │     YES → AWS Backup (Organization backup policies via OUs)
    │
    ├── Do you need VAULT LOCK (WORM) or air-gapped vault protection?
    │     YES → AWS Backup (service-native backups have no vault lock
    │            equivalent -- S3 Object Lock is the exception for S3 objects)
    │
    ├── Do you need automated RESTORE TESTING?
    │     YES → AWS Backup (no service-native equivalent)
    │
    ├── Do you need a UNIFIED view of all backup jobs and recovery points?
    │     YES → AWS Backup (single console, single API across all services)
    │
    └── None of the above?
          │
          ├── The service-native backup may be SUFFICIENT for simple cases:
          │     ─ RDS automated snapshots (already on by default, 1-35 day retention)
          │     ─ DynamoDB PITR (one toggle, 35-day continuous backup)
          │     ─ S3 versioning + lifecycle (granular object-level protection)
          │     ─ EBS Data Lifecycle Manager (automated snapshots with lifecycle)
          │
          └── But even then, AWS Backup is RECOMMENDED because:
                ─ It provides the centralized audit trail auditors expect
                ─ It enables cross-region copies consistently across all services
                ─ It is a single Terraform resource to manage, not N different ones
                ─ It costs nothing extra for the orchestration (you only pay for
                  storage and data transfer, same as service-native)
```

**The key insight**: AWS Backup does not replace service-native backups. For RDS, it uses the same snapshot mechanism under the hood. For DynamoDB, it creates the same on-demand backups (or enables the same PITR). For EBS, it creates the same snapshots. AWS Backup is an **orchestration layer**, not a separate backup engine. You are not paying twice -- you are paying the same storage cost with better management.

**When service-native is genuinely sufficient**: A single-account, single-region development environment where backup means "RDS automated snapshots with 7-day retention" and nobody needs compliance reports. But the moment you cross the threshold into multi-account, multi-region, or compliance-regulated territory, AWS Backup earns its place.

---

## Cost Considerations

AWS Backup pricing has three components:

| Cost Component | How It Works | Optimization Lever |
|----------------|-------------|-------------------|
| **Backup storage** | Per-GB-month, varies by resource type (EBS snapshots ~$0.05/GB-mo, RDS ~$0.095/GB-mo, DynamoDB ~$0.10/GB-mo warm / ~$0.03/GB-mo cold) | Cold storage lifecycle for eligible resources (~75% savings); incremental backups (EBS, EFS) minimize stored data |
| **Cross-region data transfer** | Standard cross-region transfer rates (~$0.02/GB) for copy actions | Copy only what you need cross-region; use lifecycle rules to keep cross-region copies shorter than primary |
| **Restore** | Per-GB for some resource types (DynamoDB warm restore ~$0.15/GB, cold ~$0.20/GB; S3 restore has retrieval fees) | Restore testing costs real money -- use focused test selections, not "restore everything monthly" |

**The orchestration is free.** Creating backup plans, selections, and vaults costs nothing. You pay only for the storage consumed by recovery points and any data transfer for cross-region copies. This means there is no cost penalty for adopting AWS Backup over service-native backups -- the storage cost is identical.

**Cold storage savings math**: A DynamoDB table with 500 GB of backup data costs ~$50/month in warm storage. After transitioning to cold storage at day 30, the cost drops to ~$15/month -- a 70% reduction. Over a 7-year regulatory retention period, this saves approximately $29,400 per table. Multiply by dozens of tables across an organization and cold storage lifecycle rules become a significant cost optimization.

---

## Production Terraform Example -- Enterprise Backup Architecture

This example implements a production-grade backup architecture with:
- A primary vault with Compliance-mode Vault Lock
- A cross-region DR vault
- Daily and monthly backup plans with lifecycle rules
- Tag-based resource selection
- Cross-region copy rules
- Backup Audit Manager compliance framework

```hcl
# ═══════════════════════════════════════════════════════════════════════
# PROVIDERS -- Primary and DR Regions
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
# KMS KEYS -- Encryption for backup vaults
# As covered in the KMS deep dive (Mar 9), each Region needs its own key.
# Multi-Region KMS keys simplify cross-region encrypted backup restores.
# ═══════════════════════════════════════════════════════════════════════

resource "aws_kms_key" "backup_primary" {
  provider = aws.primary

  description             = "KMS key for backup vault encryption - primary region"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  # Multi-Region key: replica in DR Region shares same key material,
  # so cross-region backup copies can be decrypted without re-encryption
  multi_region = true

  tags = {
    Name        = "backup-vault-key-primary"
    Environment = "prod"
  }
}

resource "aws_kms_alias" "backup_primary" {
  provider = aws.primary

  name          = "alias/backup-vault-primary"
  target_key_id = aws_kms_key.backup_primary.key_id
}

# Replica key in DR Region (shares key material with primary)
resource "aws_kms_replica_key" "backup_dr" {
  provider = aws.dr

  description             = "KMS key replica for backup vault encryption - DR region"
  deletion_window_in_days = 30
  primary_key_arn         = aws_kms_key.backup_primary.arn

  tags = {
    Name        = "backup-vault-key-dr"
    Environment = "prod"
  }
}

resource "aws_kms_alias" "backup_dr" {
  provider = aws.dr

  name          = "alias/backup-vault-dr"
  target_key_id = aws_kms_replica_key.backup_dr.key_id
}

# ═══════════════════════════════════════════════════════════════════════
# BACKUP VAULTS -- Primary (Compliance Lock) + DR (Compliance Lock)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_backup_vault" "primary" {
  provider = aws.primary

  name        = "prod-backup-vault"
  kms_key_arn = aws_kms_key.backup_primary.arn

  tags = {
    Name        = "prod-backup-vault"
    Environment = "prod"
    Region      = "us-east-1"
  }
}

# Vault Lock: Compliance mode on primary vault
# After changeable_for_days expires, this lock is PERMANENT and IRREVERSIBLE.
# Even root and AWS Support cannot remove it.
resource "aws_backup_vault_lock_configuration" "primary" {
  provider = aws.primary

  backup_vault_name   = aws_backup_vault.primary.name
  changeable_for_days = 3     # 3-day cooling-off before lock becomes permanent
  min_retention_days  = 7     # Cannot set retention shorter than 7 days
  max_retention_days  = 2555  # Cannot set retention longer than 7 years (2555 days)
}

resource "aws_backup_vault" "dr" {
  provider = aws.dr

  name        = "prod-backup-vault-dr"
  kms_key_arn = aws_kms_replica_key.backup_dr.arn

  tags = {
    Name        = "prod-backup-vault-dr"
    Environment = "prod"
    Region      = "us-west-2"
  }
}

resource "aws_backup_vault_lock_configuration" "dr" {
  provider = aws.dr

  backup_vault_name   = aws_backup_vault.dr.name
  changeable_for_days = 3
  min_retention_days  = 7
  max_retention_days  = 2555
}

# Vault access policy on a SEPARATE BACKUP ACCOUNT vault (cross-account protection).
# Cross-region copies within the same account (prod us-east-1 → prod us-west-2) do
# NOT need a vault access policy. This policy belongs on a vault in a dedicated backup
# account (Account B), granting the production account (Account A) permission to copy
# recovery points into it. Shown here for reference -- deploy to the backup account.
#
# resource "aws_backup_vault_policy" "backup_account" {
#   # provider = aws.backup_account
#
#   backup_vault_name = aws_backup_vault.backup_account_vault.name
#
#   policy = jsonencode({
#     Version = "2012-10-17"
#     Statement = [
#       {
#         Sid    = "AllowCrossAccountCopy"
#         Effect = "Allow"
#         Principal = {
#           AWS = "arn:aws:iam::${var.production_account_id}:root"
#         }
#         Action   = ["backup:CopyIntoBackupVault"]
#         Resource = "*"
#       }
#     ]
#   })
# }

# ═══════════════════════════════════════════════════════════════════════
# BACKUP PLAN -- Daily + Monthly with Cross-Region Copy
# ═══════════════════════════════════════════════════════════════════════

resource "aws_backup_plan" "production" {
  provider = aws.primary

  name = "prod-backup-plan"

  # Rule 1: Daily backups with 35-day retention + cross-region copy
  rule {
    rule_name         = "daily-with-cross-region"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 3 * * ? *)"  # Daily at 3 AM UTC
    start_window      = 60                    # Must start within 1 hour
    completion_window = 180                   # Must complete within 3 hours

    # Enable continuous backup (PITR) for supported resources
    # S3, RDS, Aurora, SAP HANA get PITR; others get periodic snapshots
    # Note: DynamoDB PITR is a native feature -- AWS Backup creates on-demand snapshots only
    enable_continuous_backup = true

    lifecycle {
      delete_after = 35  # 35-day retention
    }

    # Cross-region copy to DR vault
    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn

      lifecycle {
        delete_after = 35  # Same retention in DR Region
      }
    }

    recovery_point_tags = {
      BackupType  = "daily"
      Environment = "prod"
    }
  }

  # Rule 2: Monthly backups with cold storage + long retention
  rule {
    rule_name         = "monthly-long-retention"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 4 1 * ? *)"  # 1st of each month at 4 AM UTC
    start_window      = 120
    completion_window = 480

    lifecycle {
      cold_storage_after = 90   # Move to cold storage after 90 days
      delete_after       = 2555 # 7-year retention for regulatory compliance
    }

    # Cross-region copy with same cold storage lifecycle
    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn

      lifecycle {
        cold_storage_after = 90
        delete_after       = 2555
      }
    }

    recovery_point_tags = {
      BackupType  = "monthly"
      Environment = "prod"
      Compliance  = "regulatory-7yr"
    }
  }

  tags = {
    Name        = "prod-backup-plan"
    Environment = "prod"
    ManagedBy   = "terraform"
  }
}

# ═══════════════════════════════════════════════════════════════════════
# BACKUP SELECTION -- Tag-Based Resource Assignment
# ═══════════════════════════════════════════════════════════════════════

# IAM role for AWS Backup to assume when performing backup/restore operations
resource "aws_iam_role" "backup" {
  provider = aws.primary

  name = "aws-backup-service-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "backup.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "backup" {
  provider = aws.primary

  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup"
}

resource "aws_iam_role_policy_attachment" "restore" {
  provider = aws.primary

  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores"
}

# Tag-based selection: All resources tagged "Backup = daily-prod"
resource "aws_backup_selection" "production" {
  provider = aws.primary

  name         = "prod-resources"
  plan_id      = aws_backup_plan.production.id
  iam_role_arn = aws_iam_role.backup.arn

  selection_tag {
    type  = "STRINGEQUALS"
    key   = "Backup"
    value = "daily-prod"
  }
}

# ═══════════════════════════════════════════════════════════════════════
# BACKUP AUDIT MANAGER -- Compliance Framework
# Requires AWS Config to be enabled in the account.
# ═══════════════════════════════════════════════════════════════════════

resource "aws_backup_framework" "compliance" {
  provider = aws.primary

  name        = "prod-backup-compliance"
  description = "Compliance framework ensuring all production resources meet backup requirements"

  # Control 1: All resources must be protected by a backup plan
  control {
    name = "BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN"

    input_parameter {
      name  = "requiredFrequencyUnit"
      value = "hours"
    }
    input_parameter {
      name  = "requiredFrequencyValue"
      value = "24"
    }
    input_parameter {
      name  = "requiredRetentionDays"
      value = "35"
    }

    scope {
      tags = {
        Environment = "prod"
      }
    }
  }

  # Control 2: All recovery points must be encrypted
  control {
    name = "BACKUP_RECOVERY_POINT_ENCRYPTED"
  }

  # Control 3: Recovery points must be protected from manual deletion
  control {
    name = "BACKUP_RECOVERY_POINT_MANUAL_DELETION_DISABLED"
  }

  # Control 4: Resources must have cross-region backup protection
  control {
    name = "BACKUP_RESOURCES_PROTECTED_BY_CROSS_REGION"

    input_parameter {
      name  = "crossRegionList"
      value = "us-west-2"
    }

    scope {
      tags = {
        Environment = "prod"
      }
    }
  }

  # Control 5: Recovery points must meet minimum retention
  control {
    name = "BACKUP_RECOVERY_POINT_MINIMUM_RETENTION_CHECK"

    input_parameter {
      name  = "requiredRetentionDays"
      value = "35"
    }
  }

  tags = {
    Name        = "prod-backup-compliance"
    Environment = "prod"
  }
}

# ═══════════════════════════════════════════════════════════════════════
# BACKUP REPORT PLAN -- Daily compliance reports to S3
# ═══════════════════════════════════════════════════════════════════════

resource "aws_backup_report_plan" "compliance_report" {
  provider = aws.primary

  name        = "prod-backup-compliance-report"
  description = "Daily backup compliance report for audit"

  report_delivery_channel {
    s3_bucket_name = var.backup_reports_bucket_name
    s3_key_prefix  = "backup-compliance"
    formats        = ["CSV", "JSON"]
  }

  report_setting {
    report_template = "CONTROL_COMPLIANCE_REPORT"
    framework_arns  = [aws_backup_framework.compliance.arn]

    # Report across all accounts in the Organization
    accounts      = ["ALL_ACCOUNTS"]
    regions       = ["us-east-1", "us-west-2"]
    organization_units = [var.production_ou_id]
  }

  tags = {
    Name        = "prod-compliance-report"
    Environment = "prod"
  }
}

# ═══════════════════════════════════════════════════════════════════════
# RESTORE TESTING PLAN -- Monthly automated validation
# ═══════════════════════════════════════════════════════════════════════

resource "aws_backup_restore_testing_plan" "monthly" {
  provider = aws.primary

  name = "prod-monthly-restore-test"

  recovery_point_selection {
    algorithm             = "LATEST_WITHIN_WINDOW"
    include_vaults        = [aws_backup_vault.primary.name]
    recovery_point_types  = ["SNAPSHOT"]
    selection_window_days = 7  # Select from recovery points in last 7 days
  }

  schedule_expression = "cron(0 2 1 * ? *)"  # 1st of each month at 2 AM UTC

  # Start window: time allowed for the restore job to begin
  start_window_hours = 8

  tags = {
    Name        = "prod-restore-test"
    Environment = "prod"
  }
}

resource "aws_backup_restore_testing_selection" "dynamodb" {
  provider = aws.primary

  name                       = "dynamodb-restore-test"
  restore_testing_plan_name  = aws_backup_restore_testing_plan.monthly.name
  protected_resource_type    = "DynamoDB"
  iam_role_arn               = aws_iam_role.backup.arn

  # Validation window: time for custom checks before automatic cleanup
  validation_window_hours = 4

  protected_resource_conditions {
    string_equals {
      key   = "aws:ResourceTag/Environment"
      value = "prod"
    }
  }
}

# ═══════════════════════════════════════════════════════════════════════
# VARIABLES
# ═══════════════════════════════════════════════════════════════════════

variable "production_account_id" {
  description = "AWS account ID for the production account"
  type        = string
}

variable "backup_reports_bucket_name" {
  description = "S3 bucket for backup compliance reports"
  type        = string
}

variable "production_ou_id" {
  description = "Organizational Unit ID for the production OU"
  type        = string
}
```

---

## Key Takeaways

- **AWS Backup is an orchestration layer, not a backup engine.** It uses the same underlying snapshot and backup mechanisms as each service. You do not pay extra for orchestration -- only for storage and data transfer. The value is centralized management, cross-account copies, vault lock, and compliance auditing.

- **The three-tier vault hierarchy maps to increasing protection levels.** Standard vault (basic access control) < Vault Lock Governance mode (most users cannot delete, privileged admins can override) < Vault Lock Compliance mode (nobody can delete after grace period, irreversible). Logically Air-Gapped vaults are Compliance-mode vaults with an additional property: storage in an AWS service-owned account, providing isolation that survives full account compromise. Choose the tier based on your blast-radius threat model.

- **Compliance-mode Vault Lock is irreversible after the cooling-off period.** Once the `changeable_for_days` grace period expires (minimum 3 days), the lock is permanent. You will pay for storage until every recovery point naturally expires per its retention period. Test in Governance mode first. This mirrors S3 Object Lock's Compliance mode behavior from your S3 deep dive (Mar 23).

- **Cross-account backup to a dedicated backup account is the enterprise pattern.** A separate AWS account with restrictive SCPs, Compliance-mode Vault Lock, and minimal IAM access provides the strongest protection because it creates a separate blast radius. Even full compromise of the production account cannot delete backups in the backup account.

- **Tag-based backup selection is the scalable approach.** Define the convention (e.g., `Backup = daily-prod`), enforce it with tag policies via Organizations, and every new resource that follows the convention is automatically protected. Untagged resources are visibly unprotected in Audit Manager reports.

- **Not all resource types support all features.** Cold storage is supported by EBS, EFS, DynamoDB, FSx (Lustre/Windows), Timestream, SAP HANA, CloudFormation, and VMware -- but NOT RDS, Aurora, or S3. Continuous backup (PITR) via AWS Backup is limited to S3, RDS, Aurora, and SAP HANA. DynamoDB has its own native PITR, but AWS Backup creates on-demand snapshots only. Check the feature availability matrix before designing your backup architecture.

- **Restore testing is the antidote to Schrodinger's backup.** Automated restore testing on a schedule validates that your backups actually produce functional resources. As covered in the DR strategies doc (Mar 30), untested backups are hopes, not plans. Run restore tests monthly at minimum, in a dedicated test account.

- **Organization backup policies use inheritance operators, not simple merging.** Backup policies support `@@assign` (set/override), `@@append` (add to list), and `@@remove` (delete from list) operators. The parent can control what children are allowed to change via child control operators. This means the Organization root can establish baseline protection that child OUs can extend but not weaken -- when operators are configured correctly. This is fundamentally different from SCPs (which deny at any level).

- **Multi-Region KMS keys simplify cross-region backup encryption.** As covered in the KMS deep dive (Mar 9), a multi-Region key pair shares the same key material across Regions, so cross-region backup copies can be decrypted in the DR Region without re-encryption. Without this, your DR restore will fail at the decryption step.

- **Cold storage saves on storage but costs more on restore.** DynamoDB cold restore is ~$0.20/GB vs ~$0.15/GB warm. For a 2 TB table, that means a cold restore costs ~$400 vs ~$300 warm. This rarely changes the cost optimization decision (the storage savings over months/years dwarf the one-time restore cost), but flag it to cost-conscious stakeholders.

- **Vault Lock does not protect against account closure.** An attacker with root credentials cannot delete Compliance-mode Vault Lock backups directly, but they can close the entire AWS account. After a 90-day suspension period, AWS deletes everything -- including locked backups. The defense: logically air-gapped vaults (stored in AWS service-owned account, outside your blast radius) combined with a pre-built recovery account that has RAM-shared access and tested restore procedures. You cannot build this recovery path during an incident.

- **AWS Backup costs nothing extra for orchestration** -- you pay only for storage (warm and cold), cross-region data transfer, and restore operations. The centralized management, compliance monitoring, and policy enforcement are free. This makes the "AWS Backup vs service-native" decision easy: there is no cost penalty for adopting AWS Backup, and the governance benefits are substantial.

---

## Further Reading

- [AWS Backup Developer Guide](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html) -- Official documentation covering all features
- [Feature Availability by Resource Type](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html) -- Critical matrix for architecture decisions
- [Vault Lock Documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html) -- Governance vs Compliance mode details
- [Logically Air-Gapped Vaults](https://docs.aws.amazon.com/aws-backup/latest/devguide/logicallyairgappedvault.html) -- Service-owned account isolation for ransomware defense
- [Cross-Account Management](https://docs.aws.amazon.com/aws-backup/latest/devguide/manage-cross-account.html) -- Organization backup policies and delegated admin
- [Restore Testing](https://docs.aws.amazon.com/aws-backup/latest/devguide/restore-testing.html) -- Automated restore validation
- [Backup Audit Manager](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-audit-manager.html) -- Compliance frameworks and controls
- Related docs in this repo: [DR Strategies (Mar 30)](../../../march/aws/2026-03-30-disaster-recovery-strategies.md), [Multi-Region Architecture (Mar 31)](../../../march/aws/2026-03-31-multi-region-architecture-patterns.md), [KMS & Encryption (Mar 9)](../../../march/aws/2026-03-09-kms-encryption-deep-dive.md), [S3 Advanced (Mar 23)](../../../march/aws/2026-03-23-s3-advanced-deep-dive.md)
