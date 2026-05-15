# AWS CloudTrail Deep Dive -- The Courthouse Clerk's Ledger

> Yesterday's doc closed with a one-line boundary: **CloudWatch is the operational logbook -- what the factory printed during a shift -- while CloudTrail is the auditor's ledger -- who called what AWS API**. Today we cash that line. CloudTrail in 2026 is no longer "the API log thing"; it is a **four-event-category audit system** (Management, Data, Insights, and the brand-new Network Activity events that went GA February 2025 for VPC endpoints), with **three trail types** (single-region, multi-region, Organization), a **federated SQL query engine** (CloudTrail Lake, Trino under the hood) that can JOIN your CloudTrail history against Config items and Audit Manager evidence -- *and*, as of **May 31, 2026, Lake is closed to new customers**. AWS's official successor is **Amazon CloudWatch with the Lake-import feature** (OpenSearch-powered analytics, Apache Iceberg open access, OCSF/OTel built in) -- announced December 2025 as "simplified import of CloudTrail Lake data in Amazon CloudWatch". For greenfield in 2026, you pick between three forward paths: CloudWatch (the official one), Athena over S3 (cheap with Glacier cold storage), and Amazon Security Lake (OCSF cross-source). Underneath all of that sits a **SHA-256 hash-chain integrity validation** that makes log tampering computationally infeasible -- the forensics story SOC 2 / PCI / ISO 27001 auditors actually ask about. The May 11 doc showed how to fingerprint a GitHub Actions workflow via the OIDC `sub` claim; today the second half of that handshake -- *what did that workflow do once it was inside AWS* -- becomes a Lake/Athena SQL drill.
>
> **The core analogy for today: a courthouse clerk's ledger.** Picture a county courthouse. Every time someone enters the building -- a citizen filing a deed, a lawyer entering an appearance, a sheriff serving a warrant, even an HVAC contractor turning off the air-conditioning -- **the clerk at the front desk records it**. Who. When. What action they took. What documents they touched. Which side door they used. The ledger is **append-only**: pages are bound, sequentially numbered, and end-of-day digests are signed and witnessed so that no one can later go back and erase an entry. That ledger is **CloudTrail**. The **front desk** is the API endpoint -- nothing happens in AWS without going through one. The **clerk writing it down** is automatic; no party has to opt in to be recorded. The **routine entries** -- who arrived, who left, who unlocked the courtroom -- are **Management events** (free for the first copy, every account gets them). The **specific records that auditors care about** -- which file was opened, which exhibit was photocopied, which witness was paged -- are **Data events** (always billed; the famous trap is turning these on for a busy "case file room" like a hot S3 prefix). The **clerk's own raised eyebrow** -- "this person has never asked for a 3 AM filing before; this looks off" -- is the **Insights event** (anomaly detection on top of management activity). The new **courier-desk log** noting which outside parties handed documents to the courthouse without entering the public lobby is **Network Activity events** (VPC endpoint accesses, GA Feb 2025). The **bound, signed end-of-day digest** is **log file integrity validation** (SHA-256 hash chain over hourly digest files). The **central county archive room** that receives copies from every village clerkship is the **Organization trail to a dedicated Log Archive account**. And the **librarian who can SQL-query the entire archive across courthouses, county records, and the assessor's office in one breath** is **CloudTrail Lake with federated event sources** -- before May 31, 2026, anyway.
>
> Three corollaries to bolt on. **(1) Yesterday's CloudWatch / today's CloudTrail boundary is not about "logs vs metrics" -- it is about who wrote the entry.** Your application wrote into CloudWatch Logs (operational telemetry). AWS itself wrote into CloudTrail (audit ledger). One ledger is yours to populate; the other is the API service silently noting every door-swipe. People confuse them because both can be queried with SQL-ish languages and both flow through S3 -- but the bill, the IAM model, the threat model, and the auditor's question they answer are all different. **(2) The bill comes from Data events, not from Management events.** Management is the free-tier of CloudTrail. Data events bill per event, and a hot S3 bucket fronting a CDN origin can generate hundreds of millions of object-level events per day. Corey Quinn's Prime Day analysis put CloudTrail at the **hundreds-of-billions-of-events scale across the Prime Day season** -- the kind of number that turns "enable all data events" into "advanced event selectors with deny-by-default and an explicit allowlist for the 6 buckets you actually care about." **(3) Tomorrow's GenAI doc is the last topic before this audit ledger becomes the most important file in the building.** The reason every CloudTrail design discussion ends with "and the Log Archive account is SCP-locked so even the management account cannot delete from it" is that the moment your team breaches GDPR / SOC 2 / a customer's data, the only artifact that proves *what happened, when, and by whom* is this ledger. Treat it like the original of the deed.

---

**Date**: 2026-05-13
**Topic Area**: aws
**Difficulty**: Intermediate-to-Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| CloudTrail | The courthouse clerk's bound ledger | AWS-side audit log of every API call across your accounts |
| Management event | The clerk noting who entered/exited | Control-plane operations: `RunInstances`, `CreateUser`, `AssumeRole` |
| Data event | The clerk recording every file opened in the case room | Data-plane operations on resources: S3 GetObject, Lambda Invoke, DynamoDB GetItem |
| Insights event | The clerk raising an eyebrow at unusual frequency | ML-detected anomaly on top of management call volume |
| Network Activity event | The courier desk where outsiders hand documents to the courthouse without entering the lobby (GA Feb 2025) | VPC endpoint access records; the VPCE *owner* sees them. `eventName` is the underlying API (e.g. `GetObject`); deny is flagged by `errorCode = VpceAccessDenied` |
| Trail | The clerk's actual ledger book | Persistent delivery configuration writing to S3 + optionally CW Logs + EventBridge |
| Event history | The visitor sign-in sheet at the door | 90-day retention of management events only, free, every account, no config |
| Single-region trail | One clerkship's ledger | Captures events in one region only |
| Multi-region trail | One ledger spanning every clerkship | Default since 2019; captures events in all regions including new ones |
| Organization trail | The central county archive | Created from the management account; logs every member account's events |
| CloudTrail Lake | The county librarian with a SQL terminal | Trino-backed managed query engine over an event data store; **closed to new customers May 31, 2026** |
| Event data store | A bookshelf in the librarian's room | The columnar (ORC) storage backing a Lake query target |
| Federated event source | Connecting another archive (county records, assessor) to the librarian | Glue Data Catalog + Lake Formation registration so you JOIN CloudTrail with Config / Audit Manager |
| Athena over CloudTrail S3 | DIY librarian using rented shelving | Athena partition-projection table over the S3 archive; cheap forensic option with Glacier cold storage |
| CloudWatch with Lake import (Dec 2025) | The librarian's successor desk -- same archives, new tools | AWS's *official* post-Lake path: OpenSearch analytics, Apache Iceberg open access, OCSF/OTel built in |
| Amazon Security Lake | The regional records bureau normalizing every county's books | OCSF-normalized cross-source audit store; correct choice when you want CloudTrail + VPC Flow + GuardDuty in one schema |
| Advanced event selector | A filter the clerk uses to skip routine entries | Field-based filter (`eventCategory`, `resources.type`, `eventName`, `readOnly`) deciding what to capture |
| Basic event selector | The legacy clerk's filter | Pre-2019 syntax, less expressive; deprecated for new trails |
| Log file integrity validation | Bound, sequentially-numbered, witnessed pages | SHA-256 hash chain; hourly digest file signed with SHA-256-RSA per region |
| Digest file | The signed end-of-day witness page | References every log file's SHA-256 + the previous digest's signature |
| Log Archive account | The off-site county vault | Dedicated account that receives org-trail S3 deliveries; SCP-locked |
| Audit account | The auditor's reading room (read-only) | Cross-account read access into Log Archive S3 + Lake event data store |
| EventBridge integration | The clerk's panic button to the sheriff | Real-time fan-out of management events to a rule -> Lambda -> Slack/PagerDuty |
| StopLogging | Someone trying to close the ledger | Sensitive event; ALWAYS alarm on this |
| DeleteTrail | Someone burning the ledger | Sensitive event; ALARM + Cloud Custodian / SCP block |
| Root login (`ConsoleLogin` + `userIdentity.type=Root`) | The county judge personally signing in | Sensitive event; should be near-zero in well-run accounts |
| `AssumeRoleWithWebIdentity` | An out-of-county courier showing federation papers | The OIDC handshake; carries `webIdentityToken` JWT with `sub`/`aud` claims |
| `sub` claim correlation | Looking up the courier's home county | Extracting `repo:org/repo:ref:refs/heads/main` from the JWT in CloudTrail and joining to API actions |
| GuardDuty | The county fraud investigator who reads the ledger 24/7 | Consumes CloudTrail management + S3 data events + VPC Flow + DNS as core sources |
| Security Hub | The DA's consolidated findings dashboard | Aggregates GuardDuty + Inspector + Macie + Config findings, dedup'd and scored |
| CloudTrail Lake retention | How long the librarian keeps the bookshelf | Default 1 year; extended-retention up to 10 years; billed differently |
| S3 trail retention | How long the off-site vault keeps boxes | Your S3 lifecycle policy decides; CloudTrail itself does not delete |
| `readOnly:false` selector | "Only log entries that changed something" | Halves data-event volume but **misses some categorized-readOnly mutations** |
| CloudTrail vs CloudWatch | Auditor's ledger vs factory logbook | "Who called the API" vs "what the app printed" |

---

## The Big Picture in One Diagram

```
THE COURTHOUSE LEDGER SYSTEM -- WHAT EACH PIECE IS
==============================================================================

  THE FACTORY (every AWS account, every region)             THE COUNTY VAULT
  ---------------------------------------------             ---------------------

  +-------------+   ANY AWS API call
  |  IAM user / |---------+
  |  role /     |         |
  |  service    |         v
  +-------------+   +------------+    Management
                    | CloudTrail |    events (free 1st copy)
                    | service    |    Data events (always billed)
                    | (cloudtrail|    Insights events (opt-in)
                    | .amazonaws |    Network Activity (Feb 2025)
                    | .com)      |
                    +-----+------+
                          |
            +-------------+-------------+-------------+
            v             v             v             v
       +---------+   +---------+   +---------+   +---------+
       | Event   |   | Trail   |   | Lake    |   | EventBr |
       | history |   | (-> S3) |   | event   |   | (real-  |
       | (90 d,  |   | (mgmt + |   | data    |   |  time)  |
       | mgmt    |   | data    |   | store   |   |         |
       | only,   |   | sel.)   |   | (ORC)   |   |         |
       | free)   |   +----+----+   +----+----+   +----+----+
       +---------+        |             |             |
                          |             |             |
          KMS encryption  v             v             v
          + integrity     S3 bucket    Trino SQL     Lambda /
          validation      (Log Arch    via CT-Lake   SNS /
          (SHA256-RSA     account,     console or    Slack
          digest chain)   cross-acct   API. CLOSED   alert
                          policy)      TO NEW CUST   on
                          |            MAY 2026.     StopLogging,
                          |                          DeleteTrail,
                          v             |            Root login,
                    +-----------+       |            CreateAccess
                    | Athena    |       |            Key, etc.
                    | partition |<------+
                    | projection|       |
                    | table     |   Federated source
                    +-----------+   (Glue + Lake
                          |        Formation):
                          v        JOIN CloudTrail
                    +-----------+  with Config items,
                    | Glue      |  Audit Manager,
                    | Data      |  S3 inventory, etc.
                    | Catalog   |
                    +-----------+

  THE CONSUMERS                              THE THREAT MODEL
  -------------                              ----------------

  GuardDuty: reads CT + VPC Flow + DNS       1. StopLogging  -> attacker silencing
            -> findings                         the clerk
  Security Hub: aggregates findings          2. DeleteTrail  -> burning the ledger
  Macie: reads CT + S3 data events           3. Tampering    -> integrity validation
  Detective: behavior graph                     hash chain detects
  Athena/QuickSight: ad-hoc SQL              4. Bypassing    -> SCP blocks
                                                CloudTrail:*Logging,*DeleteTrail
                                                in member accounts

  CARDINAL RULES:
    1. Management events are FREE for the first copy.
       Data events ALWAYS bill, can be 20x the underlying API call cost.
    2. CloudTrail Lake CLOSED TO NEW CUSTOMERS May 31, 2026.
       Three forward paths for new builds:
         (a) CloudWatch + Lake-import (AWS's official successor, Dec 2025)
         (b) Athena over S3 (cheap forensic; pair with Glacier)
         (c) Security Lake (OCSF-normalized cross-source)
    3. Org trail to Log Archive account is the production default
       since Control Tower v3.0; SCP-lock the bucket.
    4. EventBridge sees Management events. For Data events to fire
       on EventBridge, the trail must be logging them AND the rule
       must specify the data-event source.
    5. Integrity validation is FREE, on by default for new trails.
       If it's off, you have no auditor story.
```

---

## Part 1: The Four Event Categories -- The Distinction Everyone Conflates

Every conversation about CloudTrail eventually trips on the same question: *what are the different types of events, and which ones am I paying for?* The official answer is four categories, and most engineers can name only the first two. The four are:

1. **Management events** -- control-plane operations.
2. **Data events** -- data-plane operations on individual resources.
3. **Insights events** -- ML-detected anomalies on top of management activity.
4. **Network Activity events** -- VPC endpoint access records (GA February 2025).

Internalizing the four categories is the entire foundation of the cost story, the security-monitoring story, and the "why doesn't my EventBridge rule fire" story below.

### Management events -- the routine clerk entries

**Management events are control-plane API calls** -- the operations that *configure* your AWS account, as opposed to operations that read or write *data inside* the resources you've configured.

Canonical examples:

- `iam:CreateUser`, `iam:AttachRolePolicy`, `iam:CreateAccessKey`
- `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:AuthorizeSecurityGroupIngress`
- `sts:AssumeRole`, `sts:AssumeRoleWithWebIdentity`
- `s3:CreateBucket`, `s3:PutBucketPolicy`, `s3:DeleteBucket` (the bucket itself, not objects in it)
- `cloudtrail:StartLogging`, `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`
- `signin:ConsoleLogin`

**Pricing model:** *Free for the first copy* (every account gets one free management-event trail's worth of delivery). Additional copies into additional trails or event data stores cost roughly **$2.00 per 100,000 events**.

The "first copy is free" rule is critical. Most accounts have **one trail** capturing management events (often an Organization trail), and that capture costs them nothing. The cost arrives when someone duplicates management events into a *second* trail or into a Lake event data store for SQL queries.

### Data events -- the file-room inspector's notebook

**Data events are data-plane operations** -- read/write operations on the *resources* that management events configured. The two flagship examples are:

- **S3 object-level**: `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on objects (not the bucket).
- **Lambda invocations**: `lambda:Invoke`.
- **DynamoDB item-level**: `dynamodb:GetItem`, `PutItem`, `UpdateItem`, `Query`, `Scan`.

There are ~50 supported data-event sources in 2026 -- KMS key usage (`kms:Decrypt`), SNS publishes, SQS message operations, Step Functions executions, Bedrock model invocations, and so on.

**Pricing model:** *Always billed*. There is no free tier. Roughly **$0.10 per 100,000 events**. That sounds cheap until you do the math on a hot S3 prefix.

#### The canonical Data-events cost trap

This is the gotcha worth tattooing across every cloud-security blog. Suppose you have a bucket fronting a CDN origin -- think static assets, images, video segments. CloudFront pulls maybe 100 million objects/day from it. You "turn on S3 data events for all buckets" because the security team wants object-level audit. The math:

- 100 million GetObject events/day = ~3 billion events/month.
- At $0.10 per 100K events = **$3,000/month** in CloudTrail data event ingestion alone.
- The S3 GET requests themselves cost ~$120/month (at $0.0004/1000).

You are paying **25x the underlying API call cost** just to audit it. Corey Quinn's "Prime Day" analysis put a sharper edge on this: AWS internally processed CloudTrail events at the **hundreds-of-billions-per-Prime-Day-season** scale (the often-quoted 830 billion figure was a season aggregate, not a single day). The lesson holds regardless of the exact number: if a customer enabled all S3 data events on Prime Day traffic, the CloudTrail bill would dwarf the actual S3 costs by an order of magnitude.

> **The rule:** Data events are not "default on." They are surgical. Pick the **2-6 buckets** where object-level audit genuinely matters (PII, regulated data, secrets material, anything with a Macie scan) and turn data events on *only for those*, via advanced event selectors with explicit ARN allowlists.

We'll cover the exact selector syntax in Part 3.

### Insights events -- the clerk's raised eyebrow

**Insights events are ML-detected anomalies** on top of management API call volume. CloudTrail tracks the baseline rate at which a principal calls each API and emits an Insights event when the call rate deviates significantly from baseline. Two insight types in 2026:

- **`ApiCallRateInsight`** -- abnormal call frequency. Example: `RunInstances` called 100x normal rate from a usually-idle principal.
- **`ApiErrorRateInsight`** -- abnormal error rate. Example: `DescribeInstances` returning 50% access-denied errors when normally returns 0%.

Insights events are **opt-in per trail or event data store**. Pricing **splits by event category** -- management events are **$0.35 per 100K analyzed**, data events are **$0.03 per 100K analyzed**. The split matters: data event volume routinely dwarfs management by 10-100x, so doing the math at the management rate against your data-event count will overshoot your real Insights bill by an order of magnitude (in the right direction). Either way, billing is **per event analyzed, not per insight emitted** -- Insights costs money even on days it finds nothing.

**The ~7-day baseline gotcha:** Insights needs roughly **7 days of baseline learning** before it emits any findings. Turn it on Monday, no findings until next Monday. This is the #1 surprise for engineers piloting Insights -- they enable it, see nothing for 6 days, and conclude "it's broken." It isn't; it's still building the call-rate baseline.

Use cases: insider-threat detection (a developer's role suddenly calling APIs it never called before), runaway scripts, compromised credentials being used at machine pace.

### Network Activity events -- the courier-desk log (GA Feb 2025)

The newest category. **Network Activity events capture access to your VPC endpoints (PrivateLink) for select AWS services** -- picture a courier desk at the back of the courthouse where outside parties hand documents to a clerk without entering the public lobby. As of GA in February 2025, five services are supported:

- S3 (via S3 gateway/interface endpoints)
- EC2
- KMS
- Secrets Manager
- CloudTrail (the service itself, accessed via VPCE)

**The critical detail engineers get wrong:** the `eventName` is **the underlying service API itself** -- `GetObject`, `Decrypt`, `ListKeys`, `DescribeInstances`. It is **not** a synthetic event name like `VpceAccessAllowed` or `VpceAccessHandshake` (those don't exist). The deny case is flagged by the **`errorCode` field**: `errorCode = "VpceAccessDenied"` means the VPC endpoint policy blocked the request, unset means traffic was allowed. So **filter the deny case at query time on `errorCode`, NOT by event name** -- you'll see this trip up a lot of blog posts.

**The VPCE-owner scoping detail** that matters for shared-VPC and cross-account stories: Network Activity events are captured in the CloudTrail of the **account that owns the VPC endpoint**, not the account of the calling identity. If a workload account uses a shared-VPC endpoint owned by the networking account, those events land in the networking account's trail. Design your Org-trail aggregation accordingly -- otherwise you'll wonder why your member-account trail shows no Network Activity events when you "know" the workload is hitting the VPCE.

The killer use case: **data-perimeter detection**. If you've built SCPs that say "all production S3 access must go through our VPC endpoint, never the public internet", Network Activity events finally let you *see* who from outside the org is hitting your VPCE -- the missing observability piece for an SCP that previously was deny-only.

The cheap pattern -- **capture all NetworkActivity events for the service you care about, then filter at query time for `errorCode = 'VpceAccessDenied'`**:

```json
{
  "Name": "VPCE access for S3 (capture; query for denies)",
  "FieldSelectors": [
    { "Field": "eventCategory", "Equals": ["NetworkActivity"] },
    { "Field": "eventSource",   "Equals": ["s3.amazonaws.com"] }
  ]
}
```

> **There is no advanced-event-selector field for `errorCode`.** Selectors capture; deny-vs-allow filtering happens in Athena/Lake SQL with `WHERE errorCode = 'VpceAccessDenied'`. The supported selector fields for NetworkActivity are `eventCategory` (required), `eventSource` (required), `eventName`, and `vpcEndpointId` -- not `errorCode`.

Network Activity events bill similar to Data events (~$0.10/100K), so scope by `eventSource` (and optionally `vpcEndpointId`) to keep capture bounded; do the cheap part (deny-filter) at query time.

### The boundary that matters

Everyone conflates Management and Data events. The cleanest way to keep them straight:

| Question | Category |
|----------|----------|
| Who created/modified/deleted the resource itself? | Management |
| Who read or wrote the data inside the resource? | Data |
| Is anyone behaving anomalously across resource configuration? | Insights |
| Who from where hit my VPC endpoint? | Network Activity |

A second mnemonic: **"created the bucket vs. opened a file in it" -- the first is management, the second is data.**

---

## Part 2: Trails -- Single-Region, Multi-Region, and Organization

A **trail** is the persistent delivery configuration that tells CloudTrail where to send events. Three variants exist; in 2026 you almost always want either multi-region or Organization.

### Single-region trail (legacy)

Captures only events in one region. The original 2013-era trail type. Cannot see events in other regions. Almost never the right answer today, because:

- New AWS regions are not auto-included.
- Attackers can hop to an unmonitored region.
- The cost saving over multi-region is negligible for management events (free first copy) and zero for data events (you'd configure resource-by-resource anyway).

### Multi-region trail (default since 2019)

Captures events in **all regions** including newly-enabled regions automatically. Since 2019, the AWS console defaults new trails to multi-region. Almost everyone running CloudTrail in a single account wants this.

### Organization trail (the production default since Control Tower)

Created from the **management account** of an AWS Organization, an Org trail logs events from **every member account** into a single bucket. Members **cannot disable, modify, or even see the trail** -- it appears in their console as managed-by-org.

This is the production-default since Control Tower landing-zone v3.0 (2020+). The canonical pattern:

```
Management account
   |
   v
Org Trail (multi-region, mgmt + selected data events)
   |
   +-----> S3 bucket in **Log Archive account** (a dedicated, SCP-locked member account)
   |
   +-----> CloudWatch Logs in audit account (optional, for EventBridge alerting)
   |
   +-----> CloudTrail Lake event data store (optional, for SQL queries)

Every member account (Account A, B, C, ..., N)
   API calls -> auto-captured by Org Trail -> delivered to Log Archive
```

The gotchas of Org trails:

- **You can't disable from member accounts.** Members see the trail as read-only. Good for tamper-resistance; occasionally confusing for engineers wondering "why can't I delete this trail?"
- **The bucket policy in Log Archive must allow `cloudtrail.amazonaws.com` from every member account ID** -- AWS auto-manages this when you create the Org trail from the management account, but if you bring your own bucket, you write it yourself.
- **KMS encryption on the bucket needs a key policy allowing every member account to encrypt.** This is the one place you almost always need cross-account KMS.
- **Member-account principals cannot natively see "their" events in CloudTrail console.** They see Event History (90-day mgmt-only free tier). Org trail data is in the Log Archive bucket, queried via Athena or Lake -- typically from the Audit account.
- **Enabling Control Tower while you have a pre-existing Org trail forces you to disable trusted access first**, which transiently breaks the trail. Plan the transition carefully.

### Terraform for an Org trail to a Log Archive account

```hcl
# In the management account of the AWS Organization

resource "aws_cloudtrail" "org_trail" {
  name                          = "org-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id  # bucket in log archive account
  s3_key_prefix                 = "AWSLogs"
  include_global_service_events = true
  is_multi_region_trail         = true
  is_organization_trail         = true       # <-- the marquee field
  enable_log_file_validation    = true       # <-- non-negotiable
  kms_key_id                    = aws_kms_key.cloudtrail.arn
  cloud_watch_logs_group_arn    = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn     = aws_iam_role.cloudtrail_to_cwlogs.arn

  # Management events: capture all read AND write
  advanced_event_selector {
    name = "Log all management events"
    field_selector {
      field  = "eventCategory"
      equals = ["Management"]
    }
  }

  # Data events: surgical -- ONLY two specific buckets
  advanced_event_selector {
    name = "Log object-level events on regulated buckets only"
    field_selector {
      field  = "eventCategory"
      equals = ["Data"]
    }
    field_selector {
      field  = "resources.type"
      equals = ["AWS::S3::Object"]
    }
    field_selector {
      field       = "resources.ARN"
      starts_with = [
        "arn:aws:s3:::pii-bucket/",
        "arn:aws:s3:::regulated-bucket/"
      ]
    }
  }

  # Network Activity events: capture S3 VPCE traffic
  # Filter for the deny case (errorCode = 'VpceAccessDenied') AT QUERY TIME,
  # not at the selector level -- there is no advanced-selector field for errorCode.
  advanced_event_selector {
    name = "Capture VPCE network activity for S3"
    field_selector {
      field  = "eventCategory"
      equals = ["NetworkActivity"]
    }
    field_selector {
      field  = "eventSource"
      equals = ["s3.amazonaws.com"]
    }
  }

  # Trail also needs StartLogging to actually emit -- see "trail state" below
  enable_logging = true

  depends_on = [aws_s3_bucket_policy.cloudtrail_logs]
}
```

**On `include_global_service_events = true`:** A handful of global services -- IAM, CloudFront, STS pre-regional endpoints, AWS Support, Route 53 (control plane) -- emit events that are recorded only in **us-east-1** regardless of the trail's home region. The flag tells your trail to also include those us-east-1-only global events. Forgetting this on a multi-region trail homed in, say, `eu-west-1`, means **every `CreateUser`, `AttachRolePolicy`, `CreateRole` disappears from your audit**. Critical Org-trail design point; turn it on, always. AWS's default in the console is `true`; Terraform's default is also `true` for `aws_cloudtrail`, but the explicit declaration is documentation.

### Trail state -- the stealthiest misconfig

A trail can **exist, appear configured in the console, and yet log nothing.** This is because trail **creation** and trail **state (logging on/off)** are separate operations:

- `cloudtrail:CreateTrail` -- registers the trail; logging is **off** by default.
- `cloudtrail:StartLogging` -- separately flips it on.
- `cloudtrail:StopLogging` -- the sensitive event you alarm on; the trail still exists but emits nothing.

In Terraform, the `aws_cloudtrail` resource has a separate argument `enable_logging = true` (defaults to `true`, but explicit is better) that wraps both create + start. If you ever see a trail in the console with the right S3 bucket + right selectors, with no events flowing, **check `GetTrailStatus` and look at `IsLogging`** -- a `false` here is the stealthy misconfig.

```bash
aws cloudtrail get-trail-status --name org-trail --query 'IsLogging'
# true   -- emitting
# false  -- the trail exists but is silently off
```

The Config-rule pattern: `cloudtrail-enabled` checks `IsLogging` across every trail in every account. Required control for SOC 2 / ISO 27001. The auditor's first question is rarely "do you have CloudTrail?" -- it's "**prove it was logging the entire audit period**." `IsLogging` history via Config conformance pack is the answer.

Four properties to memorize on every production trail: `is_multi_region_trail = true`, `is_organization_trail = true` (when applicable), `enable_log_file_validation = true`, `kms_key_id` set to a customer-managed key.

---

## Part 3: Advanced Event Selectors -- The Cost Control Lever

Advanced event selectors are the **single most important page of CloudTrail documentation** for keeping your bill in check. Old "basic" event selectors are coarse (capture-all-or-none-per-resource-type). Advanced selectors let you filter on specific fields.

### The field grammar

Advanced selectors filter on these fields:

| Field | Examples | Used for |
|-------|----------|----------|
| `eventCategory` | `"Management"`, `"Data"`, `"NetworkActivity"` | Top-level category gate (required) |
| `eventName` | `"GetObject"`, `"AssumeRole"`, `"Decrypt"` | The underlying API name -- even on NetworkActivity events |
| `eventSource` | `"s3.amazonaws.com"`, `"dynamodb.amazonaws.com"` | Service that emitted the event (required for NetworkActivity) |
| `errorCode` | `"VpceAccessDenied"` (NetworkActivity only) | NOT a selector field -- filter at query time, NOT in selectors |
| `readOnly` | `"true"`, `"false"` | Read-only vs mutation (note: string values, not bool) |
| `resources.type` | `"AWS::S3::Object"`, `"AWS::Lambda::Function"`, `"AWS::DynamoDB::Table"` | Resource type for Data events |
| `resources.ARN` | `"arn:aws:s3:::my-bucket/"` | Specific resource ARN |
| `userIdentity.arn` | An IAM principal ARN | Narrow to a specific principal |
| `vpcEndpointId` | `"vpce-12345"` | Network Activity narrowing |

> **`Equals` takes an array, not a scalar.** Even when matching a single value, the JSON is `"Equals": ["Management"]`. This bites first-time selector authors writing `"Equals": "Management"` and getting silent capture-of-nothing.

### The operator set

Operators are deliberately **limited**: no wildcards (`*`). Only:

- `Equals` -- exact match (case-sensitive)
- `NotEquals`
- `StartsWith`
- `NotStartsWith`
- `EndsWith`
- `NotEndsWith`

You filter what you want **in**; the absence of a matching selector means the event is **not captured**. This is critical: selectors are *additive* across rules (any matching selector includes the event) but *combined-AND* within a single rule (every field in one selector must match).

> **One `eventCategory` per `advanced_event_selector` block.** You cannot combine `["Management", "Data"]` in a single block's `eventCategory` field selector -- CloudTrail rejects it at apply time. Use **one block per category** (one Management selector, one Data selector, one NetworkActivity selector). Same goes for Data: each Data selector block must declare a single `resources.type` (you can't combine `AWS::S3::Object` and `AWS::Lambda::Function` in one block -- that's two blocks). The Terraform footgun is writing one giant `advanced_event_selector` thinking it captures "everything"; what you get is a parse error or a silent capture-nothing.

### Real-world selector examples

**Capture all management events** (the always-on starter):

```json
{
  "Name": "Log all management events",
  "FieldSelectors": [
    { "Field": "eventCategory", "Equals": ["Management"] }
  ]
}
```

**Capture S3 data events only on a specific bucket prefix**, mutations only:

```json
{
  "Name": "Log writes to pii-bucket only",
  "FieldSelectors": [
    { "Field": "eventCategory", "Equals": ["Data"] },
    { "Field": "resources.type", "Equals": ["AWS::S3::Object"] },
    { "Field": "resources.ARN", "StartsWith": ["arn:aws:s3:::pii-bucket/"] },
    { "Field": "readOnly", "Equals": ["false"] }
  ]
}
```

This four-field selector cuts your S3 data event volume on PII bucket by roughly half (drops every `GetObject`) and to ~0% on every non-PII bucket. Cost savings vs. "all data events all buckets" can be 99%+.

**Capture KMS data events only for the production CMK**, only when used for decryption:

```json
{
  "Name": "Log decrypts of prod-secrets CMK",
  "FieldSelectors": [
    { "Field": "eventCategory", "Equals": ["Data"] },
    { "Field": "resources.type", "Equals": ["AWS::KMS::Key"] },
    { "Field": "resources.ARN", "Equals": ["arn:aws:kms:us-east-1:111122223333:key/abc-123"] },
    { "Field": "eventName", "Equals": ["Decrypt"] }
  ]
}
```

**Capture VPC endpoint accesses for S3** (then filter for denies at query time):

```json
{
  "Name": "S3 VPCE network activity (capture; query for denies)",
  "FieldSelectors": [
    { "Field": "eventCategory", "Equals": ["NetworkActivity"] },
    { "Field": "eventSource",   "Equals": ["s3.amazonaws.com"] }
  ]
}
```

> **Selectors capture; filtering happens at query time.** There is no advanced-event-selector field for `errorCode`, so you cannot pre-filter for denies at the selector level. The supported NetworkActivity selector fields are `eventCategory`, `eventSource`, `eventName`, and `vpcEndpointId`. Do the deny-filter in Athena/Lake with `WHERE errorCode = 'VpceAccessDenied'`.

### The `readOnly:false` trap

A common cost-cutter is "log only mutations" -- `readOnly: false` on management events. This cuts management event volume roughly in half. But **some events that *change* something are mis-categorized as `readOnly:true`** by AWS -- a small list of edge cases where the underlying API technically reads but has side effects (token issuance, certain assume-role variants). Always read AWS's `readOnly` classification docs for the services you care about before relying on this filter.

More importantly: `readOnly:false` on management events means **you lose every `DescribeX` / `ListX` / `GetX` call**, which is exactly what attackers use during reconnaissance. The compromise: capture read AND write management events (free first copy anyway), and apply `readOnly:false` filtering only to Data events.

---

## Part 4: CloudTrail Lake -- The Librarian (and the May 2026 Closure)

CloudTrail Lake is AWS's managed query engine for CloudTrail events. Launched January 2022, it stores events in columnar ORC format in an **event data store** and exposes a Trino-based SQL surface for ad-hoc queries.

### The crucial 2026 caveat -- and the three forward paths

**On May 6, 2026, AWS announced that CloudTrail Lake is closed to new customers effective May 31, 2026.** Existing event data stores continue to function indefinitely; you can extend their retention, run queries, ingest new events. But **new accounts cannot enable Lake** after May 31, 2026.

The strategic read: AWS is **consolidating Lake's feature surface into Amazon CloudWatch** -- the December 2025 launch of "simplified import of CloudTrail Lake data in Amazon CloudWatch" is the explicit signal. CloudWatch now ships native OpenSearch-powered analytics, Apache Iceberg open APIs (so Athena, Redshift, and SageMaker can read CloudWatch's storage directly), and built-in OCSF + OTel format support. That's exactly Lake's feature surface, re-homed inside the CloudWatch product line.

For greenfield in 2026, you have **three forward paths**, each correct for a different shape of problem:

1. **CloudWatch with Lake-import (AWS's official path).** Ship the CloudTrail trail to a CloudWatch log destination; for historical data, use the Dec 2025 simplified-import flow to pull your old Lake event data stores in. You get OpenSearch analytics on hot data, Iceberg open access for cross-tool federation, and the same OCSF normalization Security Lake provides -- but inside the product AWS is investing in. Best when you also want to consolidate operational logs and audit logs into one query surface, and when you're already running CloudWatch heavily. Pricing follows CloudWatch custom-logs ingestion + analytics tiers.

2. **Athena over S3 (the cheap forensic path).** CloudTrail trail to S3 in the Log Archive account, KMS-encrypted, integrity-validated, with an S3 lifecycle policy moving older partitions to Glacier Deep Archive. Glue Data Catalog or partition-projection Athena table on top. Best when your queries are ad-hoc forensics, not continuous analytics -- you pay only for the GB scanned (5/TB), and Glacier Deep Archive storage is roughly $1/GB/year. The right answer for the *cold compliance archive* tier of a tiered audit-data architecture.

3. **Amazon Security Lake (the OCSF cross-source path).** Security Lake ingests CloudTrail + VPC Flow Logs + GuardDuty findings + Route 53 query logs + EKS audit logs + 80+ third-party sources, normalizes them all to **OCSF (Open Cybersecurity Schema Framework)**, and stores them in your S3 in Apache Parquet partitioned by source + region + date. Best when "who did what" needs to JOIN against "what packets flowed where" and "what did the SIEM flag" in a single normalized query plane.

**The combination that production teams pick most often:** CloudWatch for the hot 90-day audit-analytics layer (the operational SecOps query surface), S3 + Athena for the 7-year cold compliance archive (with Glacier Deep Archive lifecycle), and Security Lake when the security team needs cross-source OCSF semantics. The three are complementary, not exclusive.

For accounts that **already have Lake event data stores**: keep them. Lake's Trino dialect is meaningfully nicer than Athena's Presto for some queries, federation is real, and the support cost is identical to before. When ready, use the CloudWatch import flow to migrate historical data without losing it.

For accounts that **don't yet have Lake**: don't try to spin up Lake before the deadline as a "future-proofing" move -- you'll be on a deprecated product. Pick from the three paths above based on workload shape.

### Lake architecture (for the doc completeness)

```
CloudTrail events ---ingest---> Event Data Store (columnar ORC)
                                       |
                                       v
                            Trino SQL query engine
                                       |
                            +----------+----------+
                            v                     v
                  Console SQL editor    StartQuery API
                  (cloudtrail.console)  (boto3 / aws cli)
```

Three properties worth knowing:

- **Retention**: default 1 year, max 10 years (with extended-retention pricing). Ingestion bills at ~$2.50/GB; query bills at ~$0.005/GB scanned (similar to Athena).
- **Federated event sources**: register the event data store with Glue Data Catalog + Lake Formation, and you can JOIN CloudTrail events against AWS Config items, Audit Manager evidence, and external Glue tables in a single query. Killer feature pre-closure.
- **Lake is per-region**: you create an event data store in one region. For multi-region coverage, use the "Copy events from another region" feature.

### Sample Lake/Athena SQL queries

The syntax below is shown in **Lake's Trino dialect** but transposes near-1:1 to Athena's Presto dialect over an S3 partition-projection table.

**Q1. Every root login in the last 30 days:**

```sql
SELECT eventTime, sourceIPAddress, userIdentity.userName, awsRegion
FROM <event_data_store_id>
WHERE eventName = 'ConsoleLogin'
  AND userIdentity.type = 'Root'
  AND eventTime > CURRENT_TIMESTAMP - INTERVAL '30' DAY
ORDER BY eventTime DESC;
```

In a well-run org, this query should return zero or near-zero rows.

**Q2. Every `CreateAccessKey` and which principal initiated it:**

```sql
SELECT eventTime, recipientAccountId,
       userIdentity.arn AS initiator,
       requestParameters.userName AS target_user
FROM <event_data_store_id>
WHERE eventName = 'CreateAccessKey'
  AND eventTime > CURRENT_TIMESTAMP - INTERVAL '7' DAY
ORDER BY eventTime DESC;
```

This is the canonical "is anyone minting long-lived keys" report.

**Q3. Every `StopLogging` or `DeleteTrail` across the org:**

```sql
SELECT eventTime, recipientAccountId, eventName,
       userIdentity.arn,
       requestParameters
FROM <event_data_store_id>
WHERE eventName IN ('StopLogging', 'DeleteTrail', 'UpdateTrail', 'PutEventSelectors')
ORDER BY eventTime DESC;
```

If you ever see a row here from a principal that isn't the management-account audit pipeline, you have a problem.

**Q4. Failed `AssumeRole` attempts -- the brute-force detection:**

```sql
SELECT userIdentity.arn AS attacker,
       sourceIPAddress,
       COUNT(*) AS failed_attempts,
       array_agg(DISTINCT errorCode) AS error_codes
FROM <event_data_store_id>
WHERE eventName = 'AssumeRole'
  AND errorCode IS NOT NULL
  AND eventTime > CURRENT_TIMESTAMP - INTERVAL '24' HOUR
GROUP BY userIdentity.arn, sourceIPAddress
HAVING COUNT(*) > 10
ORDER BY failed_attempts DESC;
```

10+ failures in 24 hours from one principal -> investigate.

**Q5. The federated JOIN -- CloudTrail × Config to answer "which IAM roles were created in the last 30 days and what policies are attached to them now?":**

```sql
SELECT ct.eventTime,
       ct.requestParameters.roleName AS role,
       ct.userIdentity.arn AS creator,
       cfg.configurationItemCaptureTime,
       cfg.configuration.attachedManagedPolicies
FROM <ct_event_data_store_id> ct
JOIN <config_aggregator_glue_table> cfg
  ON cfg.resourceName = ct.requestParameters.roleName
WHERE ct.eventName = 'CreateRole'
  AND ct.eventTime > CURRENT_TIMESTAMP - INTERVAL '30' DAY
  AND cfg.resourceType = 'AWS::IAM::Role'
ORDER BY ct.eventTime DESC;
```

This is Lake's killer feature -- one query spans the **who created** (CloudTrail) and the **what's the current state** (Config). Post-closure, you'd run the same JOIN in Athena with Config aggregator data registered as a separate Glue table.

---

## Part 5: The OIDC `AssumeRoleWithWebIdentity` Correlation -- Closing the May 11 Loop

The May 11 GitHub Actions Security doc covered the OIDC trust-policy story extensively: GitHub mints a JWT with a `sub` claim like `repo:my-org/my-repo:ref:refs/heads/main`, IAM's `sts:AssumeRoleWithWebIdentity` verifies it against a trust policy, and the workflow gets temporary credentials. That doc closed with a forward-looking note: *the second half of the OIDC handshake -- CloudTrail's record of what those credentials did -- gets its full treatment on May 13*. This is that treatment.

### What CloudTrail records about an OIDC handshake

Every `AssumeRoleWithWebIdentity` call produces a CloudTrail **management event** in the trust account. The event payload contains:

- `userIdentity.type = "WebIdentityUser"`
- `userIdentity.principalId = <provider-issuer>:<sub-claim>`
- `requestParameters.roleArn` -- the role being assumed.
- `requestParameters.roleSessionName` -- the session name the caller asked for.
- `requestParameters.webIdentityToken` -- **the full JWT**, base64-encoded.
- `responseElements.assumedRoleUser.assumedRoleId` -- the temporary credential's session ID.
- `responseElements.credentials.accessKeyId` -- the temporary access key.

The `webIdentityToken` is the gold. It contains the OIDC issuer's claims -- `iss`, `aud`, `sub`, `repository`, `repository_owner`, `ref`, `workflow`, `run_id`, `actor`, and (since the immutable-subject-claims rollout in 2026) the immutable numeric IDs `repository_id` and `repository_owner_id`.

### Extracting the `sub` claim from CloudTrail

The JWT comes as three base64**url**-encoded segments separated by `.` -- header, payload, signature. (Base64url uses `-` and `_` instead of `+` and `/` and omits trailing `=` padding -- so a plain `from_base64` won't decode it.) You decode the middle segment (payload), JSON-parse it, and pull the `sub`, `repository`, `workflow`, `run_id`, `actor`, and `repository_id` claims.

> **Two dialect notes before the SQL.** In **CloudTrail Lake** (Trino-backed), `requestParameters` is exposed as a **typed struct** -- you reference fields as `requestParameters.webIdentityToken`. In **Athena** over the standard CloudTrail S3 schema, `requestParameters` is a **STRING column containing JSON** -- you reference fields via `json_extract_scalar(requestparameters, '$.webIdentityToken')`. The two variants below differ only in (a) this struct-vs-string treatment and (b) the base64url-decoding pattern (Trino has `from_base64url(...)`; Athena's Presto build does not, so you do manual url-safe-to-standard substitution before `from_base64`).

**Lake / Trino variant** (requestParameters as struct, `from_base64url` available):

```sql
-- Trino exposes requestParameters as a typed struct; webIdentityToken is a STRING field.
-- Trino built-ins: from_base64url(varchar) -> varbinary, from_utf8(varbinary) -> varchar.

WITH oidc_handshakes AS (
  SELECT
    eventTime,
    recipientAccountId AS aws_account_id,
    requestParameters.roleArn AS assumed_role,
    requestParameters.roleSessionName AS session_name,
    responseElements.credentials.accessKeyId AS issued_access_key,
    responseElements.assumedRoleUser.assumedRoleId AS session_id,
    sourceIPAddress,
    -- second segment of the .-separated token = the JWT payload
    split_part(requestParameters.webIdentityToken, '.', 2) AS jwt_payload_b64
  FROM <event_data_store_id>
  WHERE eventName = 'AssumeRoleWithWebIdentity'
    AND eventTime > CURRENT_TIMESTAMP - INTERVAL '7' DAY
),
decoded AS (
  SELECT
    *,
    from_utf8(from_base64url(jwt_payload_b64)) AS jwt_payload_json
  FROM oidc_handshakes
)
SELECT
  eventTime,
  aws_account_id,
  assumed_role,
  session_id,
  json_extract_scalar(jwt_payload_json, '$.sub')            AS github_sub,
  json_extract_scalar(jwt_payload_json, '$.repository')     AS github_repository,
  json_extract_scalar(jwt_payload_json, '$.workflow')       AS github_workflow,
  json_extract_scalar(jwt_payload_json, '$.run_id')         AS github_run_id,
  json_extract_scalar(jwt_payload_json, '$.actor')          AS github_actor,
  json_extract_scalar(jwt_payload_json, '$.repository_id')  AS github_repository_id_immutable
FROM decoded
WHERE json_extract_scalar(jwt_payload_json, '$.sub') LIKE 'repo:my-org/%'
ORDER BY eventTime DESC;
```

Now you have a row per GitHub Actions workflow run with the AWS-side `session_id` and the GitHub-side `repository`, `workflow`, `run_id`, `actor`.

### The both-sides correlation -- every AWS action taken by a workflow

The session ID returned in `responseElements.assumedRoleUser.assumedRoleId` carries forward into **every subsequent API call's `userIdentity.sessionContext.sessionIssuer.arn` and `userIdentity.principalId`** in CloudTrail. So you can JOIN the handshake table back against the full CloudTrail event stream to enumerate every AWS API action taken by that GitHub workflow:

> **The `accessKeyId` field-path trap.** The same temporary credential ID appears under **different field paths** depending on the event: on the `AssumeRoleWithWebIdentity` event itself, it's `responseElements.credentials.accessKeyId` (the *issued* key); on every subsequent API call made with that session, it's `userIdentity.accessKeyId` (the *using* key). Same value (`ASIA...`), different JSON path. Your JOIN can pivot on either the access key ID or the `principalId`'s role-id prefix (`AROA...`) -- both stay constant across the session. Pick one and use it consistently across all queries.

### Lambda self-invoke vs external-principal -- the forensic discriminator

A related primitive that shows up in incident response: how do you distinguish "this Lambda invoked itself on its EventBridge schedule" from "an external principal assumed this Lambda's execution role and invoked it"? Both end up running the function code; both can mutate the same resources; both would look the same to anyone reading only the data-plane events.

The discriminator is the **role-session name embedded in the assumed-role ARN** in `userIdentity.arn`:

- **Native scheduled self-invoke**: Lambda invokes itself using its execution role natively. The `userIdentity.arn` looks like `arn:aws:sts::111:assumed-role/nightly-cleanup-role/nightly-cleanup` -- the session-name segment after the second `/` is the **function name**, stamped by Lambda's invocation machinery. Crucially, there is **no separate `AssumeRoleWithWebIdentity` or `AssumeRole` event in CloudTrail** for this invocation -- Lambda uses the role natively without minting a fresh session.

- **External `AssumeRole` (CI, user, another service)**: produces a fresh `AssumeRole` (or `AssumeRoleWithWebIdentity`) management event with a `roleSessionName` the caller picked -- something like `GitHubActions-deploy-12345678901` or `terraform-apply-prod`. The subsequent invocation's `userIdentity.arn` shows that caller-chosen session name.

The forensic walk in a Lambda incident: (1) pull the data-plane events on the affected resource (`s3:DeleteObject`, etc.) and read the `userIdentity` -- you'll get an assumed-role ARN with a session name; (2) search the management-event stream for an `AssumeRole*` event minting that session shortly before the data event; (3) if you find one with a non-function-name session, it was external (and the assume event tells you who); if you find none, it was a native self-invoke. Timing alone (02:14 instead of the scheduled 04:00) is a red flag but **not attribution** -- the session-name + presence/absence of an assume event is what proves the chain.

```sql
WITH oidc_handshakes AS (
  -- (same WITH block as above, gives us session_id + github_repository + github_run_id)
  ...
),
session_actions AS (
  SELECT
    eventTime,
    eventName,
    eventSource,
    awsRegion,
    sourceIPAddress,
    -- The session id appears here for subsequent calls
    userIdentity.principalId AS principal_id,
    SPLIT_PART(userIdentity.principalId, ':', 1) AS principal_role_id
  FROM <event_data_store_id>
  WHERE eventTime > CURRENT_TIMESTAMP - INTERVAL '7' DAY
)
SELECT
  h.github_repository,
  h.github_workflow,
  h.github_run_id,
  h.github_actor,
  h.assumed_role,
  s.eventTime,
  s.eventName,
  s.eventSource,
  s.awsRegion
FROM oidc_handshakes h
JOIN session_actions s
  ON s.principal_id LIKE CONCAT(SPLIT_PART(h.session_id, ':', 1), ':%')
WHERE h.github_repository = 'my-org/payments-api'
  AND h.github_run_id = '12345678901'
ORDER BY s.eventTime;
```

That single query answers: **"every AWS API action taken by GitHub Actions run #12345678901 in `my-org/payments-api`."**

Now the two halves of the audit story compose:

- **GitHub audit log** tells you "user @alice approved deployment of commit abc123 to production at 10:34 UTC."
- **CloudTrail Lake** tells you "the workflow assumed `arn:aws:iam::111:role/prod-deploy` at 10:34:17 UTC and called `ecr:PutImage`, `lambda:UpdateFunctionCode`, `s3:PutObject` (×42), and `ecs:UpdateService` between 10:34 and 10:39."

End-to-end attribution from the code review approval to the AWS API call. This is the SOC 2 / SOX / GDPR audit story for CI/CD; it is the whole reason the May 11 doc spent so much effort on getting the `sub` claim right.

### The Athena variant for post-Lake-closure builds

Same query, three structural differences against the standard Athena CloudTrail schema: (a) `requestparameters` is a **STRING** containing JSON (not a struct) -- pull the JWT with `json_extract_scalar(requestparameters, '$.webIdentityToken')`; (b) Athena's Presto build does **not** ship `from_base64url`, so you manually convert url-safe base64 to standard base64 (`-` -> `+`, `_` -> `/`, add `=` padding) and use `from_base64`; (c) bound the query with `partition_year`, `partition_month`, `partition_day` -- partition predicates are non-negotiable for cost.

```sql
WITH oidc_handshakes AS (
  SELECT
    eventtime,
    recipientaccountid,
    json_extract_scalar(requestparameters, '$.roleArn')          AS assumed_role,
    json_extract_scalar(requestparameters, '$.roleSessionName')  AS session_name,
    json_extract_scalar(responseelements, '$.credentials.accessKeyId')          AS issued_access_key,
    json_extract_scalar(responseelements, '$.assumedRoleUser.assumedRoleId')    AS session_id,
    sourceipaddress,
    -- The JWT lives inside the JSON string at $.webIdentityToken
    split_part(json_extract_scalar(requestparameters, '$.webIdentityToken'), '.', 2)
      AS jwt_payload_b64url
  FROM cloudtrail_logs
  WHERE eventname = 'AssumeRoleWithWebIdentity'
    AND partition_year  = '2026'
    AND partition_month = '05'
    AND partition_day BETWEEN '06' AND '12'   -- always bound time on Athena
),
decoded AS (
  SELECT
    *,
    -- Athena (Presto) base64url -> base64 -> bytes -> utf-8 string
    CAST(
      from_base64(
        replace(replace(
          -- right-pad to a multiple of 4 with '='
          rpad(jwt_payload_b64url, ((length(jwt_payload_b64url) + 3) / 4) * 4, '='),
          '-', '+'),
          '_', '/')
      )
      AS VARCHAR
    ) AS jwt_payload_json
  FROM oidc_handshakes
)
SELECT
  eventtime,
  recipientaccountid,
  assumed_role,
  session_id,
  json_extract_scalar(jwt_payload_json, '$.sub')           AS github_sub,
  json_extract_scalar(jwt_payload_json, '$.repository')    AS github_repository,
  json_extract_scalar(jwt_payload_json, '$.workflow')      AS github_workflow,
  json_extract_scalar(jwt_payload_json, '$.run_id')        AS github_run_id,
  json_extract_scalar(jwt_payload_json, '$.actor')         AS github_actor,
  json_extract_scalar(jwt_payload_json, '$.repository_id') AS github_repository_id_immutable
FROM decoded
WHERE json_extract_scalar(jwt_payload_json, '$.sub') LIKE 'repo:my-org/%'
ORDER BY eventtime DESC;
```

The **partition predicate is non-negotiable** in Athena -- it is your scan-cost control. Lake hides partitioning from you; Athena makes it explicit.

---

## Part 6: Log File Integrity Validation -- The Forensics Story

Auditors do not care about your dashboards. They care about whether the log they're reading **is the log AWS originally wrote**. CloudTrail's answer is **log file integrity validation** -- a hash-chain construction that detects tampering computationally.

### How it works

When integrity validation is enabled on a trail:

1. **Every log file** that lands in S3 has its **SHA-256 hash** computed and recorded.
2. **Every hour**, CloudTrail writes a **digest file** to a parallel `CloudTrail-Digest/` prefix in the same S3 bucket. The digest contains: the hashes of all log files delivered in that hour, plus the **hash of the previous hour's digest**, plus a timestamp.
3. The digest is **signed with SHA-256-RSA** using a **per-region private key**. The corresponding public key is available from `cloudtrail:ListPublicKeys` for verification.

The result is a **hash chain**: any single log file's tamper detection requires producing a new file with the same SHA-256 (computationally infeasible), AND falsifying every subsequent digest's chain reference (each digest references the previous), AND minting a valid SHA-256-RSA signature for the falsified digest (requires the AWS-held private key).

### The per-region key gotcha

Each AWS region has its own signing key. When you validate logs that were captured in `us-east-1` from an audit account in `us-west-2`, you must call `ListPublicKeys` with `--region us-east-1` to fetch the right public key. The native CLI handles this automatically:

```bash
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:111122223333:trail/org-trail \
  --start-time 2026-05-06T00:00:00Z \
  --end-time 2026-05-12T23:59:59Z \
  --region us-east-1   # the region of the trail, not your shell
```

The output enumerates every digest file checked and every log file verified, with explicit `Valid`/`Invalid` per file. **A `Gap detected` warning means there is a missing digest** -- which is either (a) a real tampering event, or (b) the validation was *off* for a window and your forensic chain has a hole. The latter is the more common case in practice, which is why **`enable_log_file_validation = true` is non-negotiable on day one**.

### The auditor story

When a SOC 2 / PCI / ISO 27001 auditor asks "how do you prove these logs haven't been tampered with?", the answer is:

1. *Trail integrity validation has been on since trail creation.* (Show CloudFormation/Terraform diff history.)
2. *Bucket Object Lock is enabled in Compliance mode with N-year retention.* (Belt and suspenders -- prevents deletion entirely.)
3. *Bucket policy denies `s3:DeleteObject` and `s3:PutObjectAcl` to every principal except the audit-account role.*
4. *Validation runs nightly via a Lambda that calls `cloudtrail:ValidateLogs` and pages on any `Invalid` result.*

That four-line summary is what auditors are after. Integrity validation is free; turn it on and forget.

---

## Part 7: Multi-Account Centralization -- The Log Archive Pattern

The **Log Archive account** is the AWS-native solution to "where should the audit logs live so an attacker who compromises a workload account cannot reach them?" The pattern, codified by AWS Control Tower since landing-zone v2.0 (2019) and v3.0 (2020), is:

1. **Create a dedicated AWS account** named `Log Archive` (or `audit-logs`, `central-logging` -- naming convention is taste).
2. **No workloads run in this account.** It hosts: the S3 bucket receiving CloudTrail Org trail deliveries, the S3 bucket receiving Config aggregator records, the S3 bucket receiving VPC Flow Logs, and optionally a CloudTrail Lake event data store.
3. **An SCP at the Organization level** denies every principal -- including the management account's principals -- from `s3:DeleteObject`, `s3:DeleteBucket`, `s3:PutBucketPolicy`, `s3:PutObjectAcl`, `cloudtrail:DeleteTrail`, `cloudtrail:StopLogging`, `cloudtrail:UpdateTrail` against resources in the Log Archive account, *except* for a break-glass `LogArchiveAdminRole` that requires SSO + MFA + a manual approval workflow.
4. **A separate `Audit` account** has cross-account read access to the Log Archive S3 bucket, the Lake event data store, and Config aggregator. SecOps does all their querying from here.

The Log Archive bucket policy (cross-account write for org-trail):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::central-cloudtrail-logs"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::central-cloudtrail-logs/AWSLogs/o-1234567890/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:111122223333:trail/org-trail"
        }
      }
    }
  ]
}
```

Two non-obvious things to notice:

- The `aws:SourceArn` condition ties the bucket grant to **one specific trail ARN** in the management account. Without it, any account in the org could potentially write to this bucket (the confused-deputy class).
- The resource ARN includes `o-1234567890` -- the **Organization ID prefix**. CloudTrail-managed Org trails automatically write to this prefix; if your prefix is wrong, deliveries fail silently.

KMS-encryption posture:

```hcl
resource "aws_kms_key" "cloudtrail" {
  description             = "CMK for org CloudTrail"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

data "aws_iam_policy_document" "cloudtrail_kms" {
  statement {
    sid    = "Allow CloudTrail to encrypt"
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
    actions = ["kms:GenerateDataKey*", "kms:DescribeKey"]
    resources = ["*"]
    condition {
      test     = "StringLike"
      variable = "kms:EncryptionContext:aws:cloudtrail:arn"
      values   = ["arn:aws:cloudtrail:*:${var.management_account_id}:trail/*"]
    }
  }

  statement {
    sid    = "Allow member accounts to use the key for decryption when reading their own log files"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = [for id in var.org_account_ids : "arn:aws:iam::${id}:root"]
    }
    actions = ["kms:Decrypt", "kms:DescribeKey"]
    resources = ["*"]
  }

  # ... break-glass admin block ...
}
```

The cross-account KMS dance is the most-failed step in setting up an Org trail. Symptom: trail is delivering to S3 but member accounts can't see their own events. Cause: KMS key policy doesn't grant decrypt to member account principals.

---

## Part 8: EventBridge Integration -- Real-Time Response

CloudTrail is also a **streaming source**. Every event delivered to the trail is *also* (when configured) emitted on the **default EventBridge bus** in the originating account. That makes it a real-time response surface for security-critical events.

### The canonical sensitive-event list

Every production CloudTrail design ships with EventBridge rules for these:

| Event | Why it matters |
|-------|----------------|
| `StopLogging` | Attacker silencing the logger |
| `DeleteTrail` | Attacker burning the ledger |
| `UpdateTrail` | Attacker reducing scope |
| `PutEventSelectors` | Attacker scoping selectors to exclude their activity |
| `ConsoleLogin` with `userIdentity.type=Root` | Root account in use |
| `CreateAccessKey` | Long-lived credentials being minted |
| `AuthorizeSecurityGroupIngress` with `0.0.0.0/0` source | Internet ingress added |
| `PutBucketPolicy` with public principal | S3 bucket being made public |
| `DisableKey` / `ScheduleKeyDeletion` on KMS | Cryptographic material being disrupted |
| `GuardDuty:DeleteDetector` | Disabling threat detection |

### Terraform: EventBridge rule for `StopLogging` and `DeleteTrail` -> Slack

```hcl
resource "aws_cloudwatch_event_rule" "cloudtrail_tamper" {
  name        = "cloudtrail-tamper-detection"
  description = "Alert on attempts to disable or delete CloudTrail"
  event_pattern = jsonencode({
    source      = ["aws.cloudtrail"]
    detail-type = ["AWS API Call via CloudTrail"]
    detail = {
      eventSource = ["cloudtrail.amazonaws.com"]
      eventName   = ["StopLogging", "DeleteTrail", "UpdateTrail", "PutEventSelectors"]
    }
  })
}

resource "aws_cloudwatch_event_target" "to_slack_lambda" {
  rule      = aws_cloudwatch_event_rule.cloudtrail_tamper.name
  target_id = "slack-formatter-lambda"
  arn       = aws_lambda_function.slack_formatter.arn
}

resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.slack_formatter.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.cloudtrail_tamper.arn
}
```

The Lambda formats the event payload (which is a flat JSON with `detail.userIdentity.arn`, `detail.eventName`, `detail.sourceIPAddress`, `detail.requestParameters`) and posts to Slack via a webhook. In an Org trail context, you deploy the rule + Lambda **in every member account** -- or, more cleanly, the rule emits to a **cross-account event bus** in your security account, where one centralized Lambda handles formatting.

### The EventBridge gotcha

**EventBridge does NOT see Data events by default.** EventBridge's CloudTrail event source is **management events only** unless:

1. The trail is configured to log data events, AND
2. The EventBridge rule explicitly matches the `eventCategory: "Data"` pattern, AND
3. (Most importantly) the trail has `cloud_watch_logs_group_arn` set, because data events take a different EventBridge path.

If you build a rule expecting to catch `s3:DeleteObject` on a sensitive bucket without configuring data event capture *and* explicit pattern matching, the rule will sit silently in `OK` forever. Test the rule by manually triggering the event in a non-prod environment.

### Aggregate vs. one-off events: CloudWatch metric filter vs. EventBridge

A design rule that took me a while to internalize:

- **EventBridge is for one-off critical events** -- one `StopLogging` event needs to page someone. Latency target: seconds.
- **CloudWatch metric filters** on the CloudTrail-to-CloudWatch-Logs delivery are for **aggregate anomaly detection** -- "more than 100 failed AssumeRole in 5 minutes" -- and feed into CloudWatch metric alarms.

They are not interchangeable. Use both.

### Closing the loop with yesterday: a CloudWatch metric filter on `StopLogging`

Yesterday's CloudWatch deep dive covered the metric-filter -> metric -> alarm pipeline in detail. Here's the four-line version applied to CloudTrail. With `cloud_watch_logs_group_arn` set on the trail, every event lands in a CloudWatch log group. A metric filter scans for the sensitive pattern and increments a metric; an alarm on that metric pages SNS:

```hcl
# Filter pattern: any CloudTrail event whose eventName is StopLogging or DeleteTrail
resource "aws_cloudwatch_log_metric_filter" "cloudtrail_tamper" {
  name           = "cloudtrail-tamper"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = "{ ($.eventName = StopLogging) || ($.eventName = DeleteTrail) }"

  metric_transformation {
    name          = "CloudTrailTamperCount"
    namespace     = "Security/CloudTrail"
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "cloudtrail_tamper" {
  alarm_name          = "cloudtrail-tamper-detected"
  metric_name         = aws_cloudwatch_log_metric_filter.cloudtrail_tamper.metric_transformation[0].name
  namespace           = "Security/CloudTrail"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.security_pages.arn]
}
```

**When to pick metric filter vs. EventBridge for this same event:** EventBridge is lower latency (event in -> rule fires within seconds) and doesn't require CW Logs delivery, so it's the better choice for "page the on-call" use cases. Metric filters shine when you also want the **count over time** (a CW alarm metric you can graph, set thresholds on for >N events in 5 min, and build dashboards around). Real production deploys both: EventBridge for instant paging, metric filter for "how often is this happening" history. Yesterday's doc covered the filter pattern syntax (`{ $.eventName = ... }`) and the alarm wiring; today's contribution is the realization that the *source* of those log events is exactly the CloudTrail trail you just learned to configure.

---

## Part 9: Retention -- Two Different Timelines

CloudTrail has **two different retention models** depending on where the data lives, and they are not the same model.

### Trail-to-S3 retention

Trails write events to S3. **CloudTrail itself does not delete from S3.** Your S3 lifecycle policy determines retention. The common configuration:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "cloudtrail_logs" {
  bucket = aws_s3_bucket.cloudtrail_logs.id

  rule {
    id     = "tier-and-expire"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 365
      storage_class = "GLACIER_IR"     # or GLACIER for cheaper, slower
    }
    transition {
      days          = 730
      storage_class = "DEEP_ARCHIVE"
    }
    expiration {
      days = 2557                       # 7 years -- typical SOC 2 / SOX retention
    }
  }
}
```

A typical financial / regulated workload: 30-90 days in Standard, 1 year in IA, 7 years in Glacier Deep Archive, then expire. The cost at 7-year retention is dominated by Glacier storage at ~$1/GB/year.

### Lake retention

Event data stores in CloudTrail Lake have **their own retention setting**. Default is 1 year. **Extended retention** can be configured up to 10 years, but with a different price tier (~2x the cost per GB after the first year). Lake is **billed for retention** -- the cost is non-trivial for high-volume orgs.

```hcl
resource "aws_cloudtrail_event_data_store" "org_events" {
  name                          = "org-events-7yr"
  multi_region_enabled          = true
  organization_enabled          = true
  retention_period              = 2557     # days (7 years)
  termination_protection_enabled = true
  billing_mode                  = "EXTENDABLE_RETENTION_PRICING"

  advanced_event_selector {
    name = "Log all management events"
    field_selector {
      field  = "eventCategory"
      equals = ["Management"]
    }
  }
}
```

`termination_protection_enabled = true` is critical -- without it, an accidental `DeleteEventDataStore` API call removes the store after a 7-day grace period.

### The Lake-vs-S3 retention conversation

Lake retention is *expensive* compared to S3 Glacier Deep Archive. For long-term archive (7+ years), the right answer is almost always:

- **Lake**: 1-year retention for hot SQL queries.
- **S3**: 7-year retention for cold compliance archive, queryable via Athena when needed.

Don't pay Lake's retention price for cold data. That's what Athena over S3 is for. With Lake closed to new customers in May 2026, the greenfield tiering becomes: **CloudWatch** for hot (90-day operational analytics, Lake-import flow), **S3 + Athena** for cold (7-year compliance archive on Glacier Deep Archive), and **Security Lake** layered on when OCSF cross-source matters. Same hybrid mental model; just CloudWatch where Lake used to sit.

---

## Part 10: Cost Gotchas -- Where the Five-Figure Surprises Hide

Every CloudTrail bill blowup traces back to one of these:

### 1. Data events on a hot S3 prefix (the canonical trap)

Covered already. The single largest CloudTrail cost surprise. Audit selectors quarterly.

### 2. Lake ingestion on an org with too many member accounts

Org trail to Lake: every member account's events stream into one event data store. At 50 member accounts × 30M management events / month each = 1.5B events/month, Lake ingestion alone is ~$3,000/month before queries. Compare to S3 + Athena: ingestion is free (you pay only for S3 storage), and you pay only for the GB scanned by Athena.

### 3. Insights events on a high-volume trail

Insights bills $0.35 per 100K events *analyzed*. On a trail processing 100M management events/month, that's $350/month just for Insights -- and Insights only emits findings when there's an actual anomaly. For most accounts, Insights is *not free* monitoring.

### 4. Multi-region trail × multi-region resources × data events

A multi-region trail capturing S3 data events sees S3 events from every region the bucket has been accessed in. If you have CloudFront pulling from one bucket via every region's edge, the multi-region capture multiplies your event count.

### 5. CloudWatch Logs cross-delivery

Sending trail to CloudWatch Logs (for EventBridge metric filters) bills as CloudWatch Logs ingestion -- $0.50/GB. Org-trail volume hitting CW Logs in the management account can easily be $200-500/month. Use it surgically.

### 6. Trail + Lake duplicated

Some teams enable both an Org trail to S3 *and* a Lake event data store ingesting the same events. That's 2x management event ingestion for events you're already paying the first-copy-free tier for once. Cheaper alternative: trail to S3 + Athena partition-projection table; skip Lake.

### Cost-first triage when the bill spikes

The diagnostic flow when the CloudTrail line item jumps:

1. **Cost Explorer, grouped by `UsageType`, filtered on service `CloudTrail`.** ~30 seconds to see *which* usage type spiked: `PaidEventsRecorded` (data events), `Insights-EventsAnalyzed`, `LakeIngestion-Bytes`, `DataDelivery-GB-CloudWatch-Logs`. The answer to "is this real or a billing glitch" comes from this graph alone.
2. **Once you've identified the category, localize it.** For data-event spikes, `aws cloudtrail list-trails` then `aws cloudtrail get-event-selectors --trail-name <name>` (and `get-event-data-store` for Lake) on every trail to find which one's selectors got broadened, on what resource ARNs, by whom. Cross-reference the trail-config-change time against the CloudTrail audit of CloudTrail itself: `eventName = 'PutEventSelectors'` in the org trail will tell you the principal who widened the selector and when.
3. **Triage in this rank order** (most common cost cause first):
   1. **Data event count** -- which buckets/Lambdas/tables, which selectors, were ARNs added or `readOnly` removed?
   2. **Lake ingestion GB** -- duplicated capture, too-broad selectors, or member accounts producing more than forecast?
   3. **Insights events analyzed** -- worth the spend or noise? Remember the **split pricing**: $0.35/100K on management events analyzed but **$0.03/100K on data events analyzed** -- if Insights is on for a high-volume data trail, doing the math at the management rate overshoots by 10x in the right direction.
   4. **CloudWatch Logs delivery GB** -- $0.50/GB ingestion on the trail-to-CW-Logs path adds up; needed or removable?
   5. **Cross-region writes** -- consolidating to one region's Lake/CloudWatch destination?

The exec answer pattern when the VP asks "real or glitch?": "Real. Cost Explorer shows `<usage-type>` spiked from $X to $Y. `<root cause one-line>`. Remediation is `<advanced-event-selector-tightening + specific ARNs>`; I'll have a PR open today."

---

## Part 11: GuardDuty and Security Hub -- The Consumers

CloudTrail is *upstream* of multiple AWS security services. Two matter most:

### GuardDuty -- the fraud investigator who reads the ledger 24/7

**GuardDuty consumes CloudTrail management events + S3 data events + VPC Flow Logs + DNS query logs + EKS audit logs** as its core data sources. It applies AWS-managed threat intelligence + ML models to detect:

- **`Recon:IAMUser/*`** -- IAM enumeration patterns (`ListUsers` from unusual location).
- **`Stealth:IAMUser/CloudTrailLoggingDisabled`** -- the `StopLogging` event you already alarm on, with GuardDuty's threat-intel correlation layered on top.
- **`UnauthorizedAccess:IAMUser/MaliciousIPCaller`** -- API calls from known-malicious IPs (AWS's threat intel).
- **`Discovery:S3/AnomalousBehavior`** -- S3 data event-based anomaly.
- **`CredentialAccess:IAMUser/AnomalousBehavior`** -- credential-exfil pattern.

GuardDuty's killer feature for CloudTrail teams: **you don't have to write the detection logic.** AWS does it. Enable GuardDuty in every account (Org-level enablement is one click), pipe findings to Security Hub, and you have working IAM-finding coverage in an afternoon.

### Security Hub -- the consolidated findings dashboard

**Security Hub aggregates findings from GuardDuty, Inspector, Macie, AWS Config, IAM Access Analyzer, and 80+ third-party security partners.** It deduplicates findings using the **AWS Security Finding Format (ASFF)** so a single CloudTrail-detected event (StopLogging) appears as one finding regardless of how many tools detected it.

The standard configuration:

```hcl
# requires aws.audit provider alias defined elsewhere in the module
resource "aws_securityhub_organization_admin_account" "main" {
  admin_account_id = var.audit_account_id
}

resource "aws_securityhub_account" "audit" {
  provider = aws.audit
}

resource "aws_securityhub_organization_configuration" "main" {
  auto_enable = true
  auto_enable_standards = "DEFAULT"
}
```

Security Hub at scale: every finding flows into a central audit account, dedup'd against ASFF, scored. Use EventBridge rules on Security Hub findings (`detail-type: "Security Hub Findings - Imported"`) to route critical findings to PagerDuty/Slack.

The mental model: **CloudTrail emits the raw events. GuardDuty reads them and applies detection logic. Security Hub aggregates GuardDuty's (and others') findings into one dashboard.** Three layers, each adding signal.

### CloudTrail vs. AWS Config -- the boundary senior interviewers ask about

**CloudTrail records *who called the API and when* (event-oriented).** Every entry is "principal P called API A at time T against resource R; here are the request and response parameters." It's a time-ordered audit log of *actions*.

**AWS Config records *what the configuration state of a resource is at any point in time* (state-oriented).** Every entry is a **configuration item (CI)** -- a JSON snapshot of a resource's full configuration at a moment in time, plus the diff against the previous snapshot. Config records the *state*, not the *action that changed it*.

| | CloudTrail | AWS Config |
|-|-----------|-----------|
| Shape | Event stream (one row per API call) | State time-series (one CI per resource per change) |
| Answers | "Who called `PutBucketPolicy` on bucket X?" | "What did bucket X's policy look like at 03:00 yesterday?" |
| Source of truth for | The action and the caller | The current and historical state |
| Cost driver | Event count (esp. data events) | Resource count + change count |
| Native query | Lake / Athena / CloudWatch | Config aggregator + advanced queries (Athena-like) |

The two are **complementary, not redundant**. The federated JOIN example in Part 4 (CloudTrail `CreateRole` events ⨝ Config `AWS::IAM::Role` CIs) is the canonical pattern: CloudTrail tells you *who* created the role and *when*; Config tells you *what policies are attached to it right now*. Roll those together and you have full provenance. Senior interviewers love this question because it separates "I've used CloudTrail" from "I understand the AWS audit-and-state architecture."

### S3 server access logs vs. CloudTrail S3 data events -- the cheaper alternative

If you only need "who touched which object on this bucket" without strict IAM-principal attribution, **S3 server access logs** are dramatically cheaper than CloudTrail S3 data events:

| | CloudTrail S3 data events | S3 server access logs |
|-|---------------------------|------------------------|
| Cost | $0.10/100K events (CDN-origin scale = $$$) | S3 storage of the log objects (~free at any scale) |
| IAM identity | Full `userIdentity.arn`, session, MFA, OIDC | The bucket-owner-translated requester (less precise; sometimes anonymous) |
| Latency | Seconds to minutes | 1-2 *hours* delivery delay |
| Format | CloudTrail JSON (rich, structured) | Apache-style log lines (parseable but flatter) |
| Query | Lake / Athena | Athena over S3 |

**Production pattern:** S3 server access logs on every bucket (high-volume audit at near-zero cost), CloudTrail S3 data events *only* on the 2-6 buckets where per-request IAM principal attribution actually matters for compliance (regulated PII, secrets material, audit destinations). Get the cheap broad coverage from access logs; spend CloudTrail's per-event price only where attribution justifies it.

---

## Part 12: Decision Frameworks

### Trail vs. Lake vs. CloudWatch vs. Athena vs. Security Lake (the high-level)

| Use case | Pick |
|----------|------|
| AWS's *official* successor to Lake for greenfield analytics | **CloudWatch with Lake-import** (OpenSearch analytics + Iceberg open APIs; Dec 2025 launch) |
| Operational SecOps querying for past 12 months on a tight budget | Trail -> S3 -> **Athena** (cheap scan-based; pair with Glacier lifecycle) |
| Existing Lake event data store from before May 2026 | Keep it; use CloudWatch import to migrate historical data when ready |
| Cross-source SQL (CloudTrail + Config + Audit Manager) | Lake federation if existing customer; **Athena + Glue Data Catalog** cross-table otherwise; **CloudWatch + Iceberg** for the new path |
| OCSF-normalized cross-source (CloudTrail + VPC Flow + GuardDuty + 3P) | **Amazon Security Lake** |
| Compliance archive (7+ years) | Trail -> **S3** with lifecycle to Glacier Deep Archive |

### Data events: capture all vs. targeted vs. none

| Posture | When |
|---------|------|
| **None** | Default for non-regulated workloads with no PII; rely on Management events for audit |
| **Targeted (recommended)** | Production posture: 2-6 specific S3 buckets / Lambda functions / DynamoDB tables that handle regulated data, with advanced selectors |
| **All** | Almost never the right answer; cost balloons |
| **All-deny + allowlist** | Advanced selector that denies everything by default and explicitly allows the targeted set; the safest production posture |

### Centralization: Org trail to Log Archive vs. per-account trail

| Scenario | Pick |
|----------|------|
| AWS Organization with 3+ accounts | **Org trail to Log Archive.** Always. Non-negotiable for production. |
| Standalone single account | Account-level multi-region trail to S3 in same account; budget for moving to Org trail when you add accounts |
| Subsidiary/acquired company outside org | Cross-account trail delivery from their account to your Log Archive bucket |

### CloudTrail Lake vs. CloudWatch vs. Athena over S3

| Factor | Lake (legacy) | **CloudWatch (official successor)** | Athena over S3 |
|--------|---------------|--------------------------------------|----------------|
| Availability for new customers | **Closed May 31, 2026** | Open; AWS's official path | Always available |
| Ingestion cost | Yes (~$2.50/GB) | CloudWatch custom-logs pricing (Lake import is free) | Free (you only pay S3 + Athena scan) |
| Query language | Trino | OpenSearch + Iceberg (Athena/Redshift/SageMaker via Iceberg) | Presto |
| Open-format access | No -- proprietary ORC | **Yes -- Apache Iceberg open APIs** | Yes -- S3 + Parquet/JSON |
| OCSF/OTel built in | No | Yes | No (manual) |
| Federated queries (Config, Audit Manager) | Yes, built-in | Yes, via Iceberg | Yes, via Glue Data Catalog |
| Partition management | Hidden, automatic | Hidden, automatic | Manual (use partition projection) |
| Time-to-first-query | Minutes (managed) | Minutes (managed) | Hours (setup Glue + projection) |
| Cost discipline | Built-in scan cost | Built-in (custom logs tier) | Same scan cost; partition-bound predicates required |
| Strategic direction post-2026 | Frozen | **Active investment** | Active investment |

For **new builds in 2026 and beyond**: **CloudWatch** for hot operational/security analytics; **Athena over S3** for cold compliance archive; **Security Lake** when OCSF cross-source semantics matter. For **existing Lake customers**: keep using it; use the Dec 2025 import flow to migrate to CloudWatch when ready.

### Trail granularity

| Setup | When |
|-------|------|
| One Org trail, all regions, mgmt + targeted data events | The default for 99% of orgs |
| Separate trail for high-volume data events | When data event volume on one resource dwarfs management events; route the data trail to its own S3 prefix with shorter retention |
| Per-account trail in addition to Org trail | Only if a member account has compliance requirements the Org trail can't meet (rare) |

---

## Part 13: Production Gotchas -- 20 Items That Have Bitten Engineers

A non-exhaustive list of foot-guns:

1. **Data events are not in the default trail.** First-time CloudTrail users assume "trail captures everything." It doesn't. Data event selectors are explicit and additive.
2. **CloudTrail Lake is closed to new customers as of May 31, 2026.** If you weren't using Lake before then, you can't start. Existing event data stores keep working.
3. **Integrity validation off = no forensics chain.** The trail can be on for years with `enable_log_file_validation = false` and no one notices until the auditor asks. Turn it on at creation.
4. **EventBridge does not see Data events by default.** The rule pattern must explicitly match `eventCategory: "Data"` AND the trail must be configured to log them AND the rule firing requires the data event being logged into CW Logs in some setups.
5. **Org-trail KMS key policy needs cross-account decrypt.** Member accounts can't see "their" events otherwise. The most-missed step in Org trail setup.
6. **`readOnly:false` advanced selector misses some categorized-readOnly mutations.** A small list of AWS APIs technically classified as `readOnly:true` despite side effects -- don't rely on this filter for compliance-grade write capture.
7. **Member accounts cannot disable an Org trail.** Good for tamper-resistance; sometimes confusing. The trail appears as read-only in their console.
8. **The S3 bucket policy on the Log Archive bucket needs to allow `cloudtrail.amazonaws.com` from every member account.** AWS auto-writes this if you let CloudTrail create the bucket; if you bring your own, you write it manually.
9. **Lake retention has a different price tier after year 1.** "Extended retention pricing" kicks in at 1 year and is ~2x the per-GB price.
10. **Lake `termination_protection_enabled = false` is the default.** Without it, an accidental delete + 7-day grace = gone.
11. **`PutEventSelectors` itself is a sensitive event.** Alarm on it -- attackers rewrite selectors to exclude their activity rather than deleting the trail.
12. **Insights events bill per event *analyzed***, not per insight emitted. On a 100M-event/month trail, that's $350/month for Insights, whether or not it finds anything.
13. **Multi-region trails do NOT auto-capture data events in new regions.** Data event selectors are tied to specific resource ARNs, so when a bucket is created in a new region, you must add it to the selector.
14. **CloudTrail to CloudWatch Logs adds CloudWatch Logs ingestion costs.** $0.50/GB on top of the trail. Use surgically.
15. **`s3:GetObject` on a CloudFront origin bucket can balloon CloudTrail bill 25-50x the S3 cost.** Don't log data events on CDN origins.
16. **Org trails capture events from accounts created after the trail too.** New member accounts are auto-included. Good for safety; surprising for cost forecasts.
17. **Trail event delivery is asynchronous and has a delay** -- typically 5-15 minutes, sometimes up to an hour. Real-time security response goes through EventBridge, not S3.
18. **Lake event data stores are per-region.** For multi-region coverage, you copy events from other regions into one Lake; this is a separate ingestion path with its own cost.
19. **CloudTrail does not log every API call.** A small list of operations is excluded by default -- the canonical list lives in each service's "unsupported events" or "events not in CloudTrail" doc page. (Note: `s3:HeadObject` *is* logged as a data event when S3 object-level data events are enabled -- the often-repeated "S3 HEAD isn't logged" claim is wrong; what's excluded is more service-specific. Always verify against the per-service AWS docs before designing detections on what you assume is covered.)
20. **`AssumeRoleWithWebIdentity` events leak the JWT.** The full `webIdentityToken` is in `requestParameters` -- if a Log Archive bucket grants cross-account read to an under-trusted principal, that principal can replay the JWT (until it expires, typically 15-60 minutes). Lock down Log Archive bucket reads.

---

## Part 14: The Athena Partition-Projection Table (The Cold-Forensics Path)

CloudWatch is AWS's official Lake successor, but most teams also keep a long-tail S3 archive in Glacier Deep Archive for compliance retention -- and that archive is queried via Athena when an auditor or incident response team needs to drill into events older than the CloudWatch hot tier. Every greenfield build should know this pattern. The trick that makes Athena over CloudTrail S3 affordable is **partition projection** -- Athena dynamically generates partitions based on a date range you declare, rather than crawling them via Glue.

> **SerDe note:** AWS's currently-documented Athena pattern for CloudTrail uses the **Hive JSON SerDe** (`org.apache.hive.hcatalog.data.JsonSerDe`) -- not the older `com.amazon.emr.hive.serde.CloudTrailSerde`. The CloudTrail-specific `InputFormat` (`com.amazon.emr.cloudtrail.CloudTrailInputFormat`) is still used because it knows how to handle CloudTrail's gzipped JSON-array-of-events files. The JSON SerDe handles the per-record schema mapping. Use this combo; it's the one AWS docs ship today.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS cloudtrail_logs (
  eventVersion STRING,
  userIdentity STRUCT<
    type:STRING,
    principalId:STRING,
    arn:STRING,
    accountId:STRING,
    invokedBy:STRING,
    accessKeyId:STRING,
    userName:STRING,
    sessionContext:STRUCT<
      attributes:STRUCT<
        mfaAuthenticated:STRING,
        creationDate:STRING
      >,
      sessionIssuer:STRUCT<
        type:STRING,
        principalId:STRING,
        arn:STRING,
        accountId:STRING,
        userName:STRING
      >
    >
  >,
  eventTime STRING,
  eventSource STRING,
  eventName STRING,
  awsRegion STRING,
  sourceIPAddress STRING,
  userAgent STRING,
  errorCode STRING,
  errorMessage STRING,
  requestParameters STRING,
  responseElements STRING,
  additionalEventData STRING,
  requestId STRING,
  eventId STRING,
  resources ARRAY<STRUCT<
    arn:STRING,
    accountId:STRING,
    type:STRING
  >>,
  eventType STRING,
  apiVersion STRING,
  readOnly STRING,
  recipientAccountId STRING,
  serviceEventDetails STRING,
  sharedEventID STRING,
  vpcEndpointId STRING,
  eventCategory STRING
)
PARTITIONED BY (
  partition_account_id STRING,
  partition_region STRING,
  partition_year STRING,
  partition_month STRING,
  partition_day STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
          OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://central-cloudtrail-logs/AWSLogs/o-1234567890/'
TBLPROPERTIES (
  'projection.enabled'                      = 'true',
  'projection.partition_account_id.type'    = 'enum',
  'projection.partition_account_id.values'  = '111122223333,222233334444,333344445555',
  'projection.partition_region.type'        = 'enum',
  'projection.partition_region.values'      = 'us-east-1,us-west-2,eu-west-1',
  'projection.partition_year.type'          = 'integer',
  -- NB: range upper-bound is a maintenance item. Bump it forward
  -- before year-end, or use a 'date' type projection on a single
  -- yyyy/MM/dd timestamp column (the AWS-docs-recommended shape).
  'projection.partition_year.range'         = '2024,2035',
  'projection.partition_month.type'         = 'integer',
  'projection.partition_month.range'        = '01,12',
  'projection.partition_month.digits'       = '2',
  'projection.partition_day.type'           = 'integer',
  'projection.partition_day.range'          = '01,31',
  'projection.partition_day.digits'         = '2',
  'storage.location.template' = 's3://central-cloudtrail-logs/AWSLogs/o-1234567890/${partition_account_id}/CloudTrail/${partition_region}/${partition_year}/${partition_month}/${partition_day}'
);
```

With this table, every query *must* include the partition predicates -- otherwise it scans the entire bucket:

```sql
-- GOOD: bounded
SELECT * FROM cloudtrail_logs
WHERE partition_year='2026' AND partition_month='05' AND partition_day='13'
  AND eventname = 'ConsoleLogin';

-- BAD: scans all years, all months, all days. $$$
SELECT * FROM cloudtrail_logs
WHERE eventname = 'ConsoleLogin';
```

Athena bills at $5/TB scanned. A typical month of org-trail data is 5-50 GB compressed. A bounded query on one day costs cents. An unbounded query on a year of org-trail data can be $50+ per query.

---

## Cheat Sheet -- Things to Memorize

### Event categories

| Category | Pricing | Default | Use |
|----------|---------|---------|-----|
| Management | Free first copy, $2/100K thereafter | Captured by default | Control-plane: who did what to the account |
| Data | $0.10/100K, always billed | **NOT** captured by default | Data-plane: who read/wrote which object |
| Insights | **$0.35/100K analyzed on mgmt, $0.03/100K analyzed on data** | Opt-in; ~7-day baseline learning period | ML-detected anomalies on call-rate / error-rate |
| Network Activity | ~$0.10/100K | Opt-in (Feb 2025) | VPCE access for 5 services; deny via `errorCode` filter at query time |

### Advanced selector field grammar

```
eventCategory    : Management | Data | Insights | NetworkActivity   (required)
eventName        : the underlying API -- "GetObject", "AssumeRole", "Decrypt"
eventSource      : "s3.amazonaws.com", etc. (required for NetworkActivity)
readOnly         : "true" | "false"  (STRING values, not bool)
resources.type   : "AWS::S3::Object", "AWS::Lambda::Function", "AWS::DynamoDB::Table"
resources.ARN    : the resource ARN
userIdentity.arn : narrow to a specific IAM principal ARN
vpcEndpointId    : "vpce-12345..." for NetworkActivity

operators: Equals, NotEquals, StartsWith, NotStartsWith, EndsWith, NotEndsWith
values are ARRAYS even for one element: Equals: ["Management"]  -- not "Management"
NO WILDCARDS.
NO errorCode field -- filter `VpceAccessDenied` at QUERY time, not in selectors.
```

### The trail trinity (production minimum)

1. **Org trail**, multi-region, in management account, delivering to Log Archive S3.
2. **`enable_log_file_validation = true`** -- always.
3. **`kms_key_id`** -- always a customer-managed key.

### Sensitive-event EventBridge list

```
StopLogging
DeleteTrail
UpdateTrail
PutEventSelectors
ConsoleLogin (Root)
CreateAccessKey
DisableKey
ScheduleKeyDeletion
AuthorizeSecurityGroupIngress (0.0.0.0/0)
PutBucketPolicy (public)
GuardDuty:DeleteDetector
```

### CloudTrail vs CloudWatch (recap from yesterday)

| | CloudWatch Logs | CloudTrail |
|-|-----------------|------------|
| What it captures | App output | AWS API calls |
| Who writes | Your app + OS | AWS itself, automatically |
| Default | Opt-in per log group | On for mgmt events; trail opt-in for data |
| Retention | Per log group | Per trail (S3 lifecycle) + per Lake event data store |
| Query | Logs Insights | Lake (Trino) or Athena (Presto) |
| Cost driver | Ingest GB + scan GB | Data events |

---

## The 10 Commandments of CloudTrail in Production

1. **One Organization trail, multi-region, in the management account, delivering to a dedicated Log Archive account.** Every other configuration is some attenuation of this baseline.
2. **`enable_log_file_validation = true` on day one.** The integrity hash chain is free. The "I'll turn it on later" trail has no forensic chain for the gap.
3. **Data events are surgical, not blanket.** Pick the 2-6 buckets / Lambda functions / DynamoDB tables that handle regulated data and target *only those* with advanced event selectors. Never `resources.ARN = ["arn:aws:s3:::*"]`.
4. **SCP-lock the Log Archive account.** Deny `s3:DeleteObject`, `s3:PutBucketPolicy`, `cloudtrail:Delete*`, `cloudtrail:Stop*`, `cloudtrail:Update*`, and `cloudtrail:PutEventSelectors` to every principal except a break-glass role gated on MFA + manual approval.
5. **Use a customer-managed KMS key** with a key policy that grants encrypt to `cloudtrail.amazonaws.com` and decrypt to every member account. Cross-account KMS is the most-failed step.
6. **EventBridge rules on `StopLogging`, `DeleteTrail`, `UpdateTrail`, `PutEventSelectors`, root `ConsoleLogin`, and `CreateAccessKey`.** These are the sensitive events that page in seconds, not the events you grep for at 9am Monday.
7. **For new builds post-May 31, 2026: prefer CloudWatch (AWS's official Lake successor) for hot analytics; fall back to Athena over S3 for ad-hoc forensics over cold-stored archives; use Security Lake for OCSF cross-source normalization.** CloudTrail Lake is closed to new customers; CloudWatch (with Dec 2025 Lake-import + OpenSearch + Iceberg) is where AWS is investing. Existing Lake customers keep using it and migrate to CloudWatch when ready.
8. **For Athena queries, partition predicates are non-negotiable.** Every query that doesn't bound `partition_year`/`partition_month`/`partition_day` is a scan-cost blowup waiting to happen.
9. **Correlate OIDC handshakes via the `sub` claim in `requestParameters.webIdentityToken`.** The whole point of pinning the trust policy on `repo:my-org/repo:ref:refs/heads/main` (May 11) is that you can now answer "every AWS action this workflow took" with one Lake/Athena query. Both halves of the OIDC handshake together = SOC 2 / SOX-grade attribution.
10. **CloudTrail is the auditor's ledger; CloudWatch is the operational logbook.** They are not interchangeable. The ledger is append-only, integrity-validated, locked away in a separate account, and queried by a single team. The logbook is mutable, query-friendly, and shared with engineers. Conflating them is the #1 misconfiguration in early CloudTrail setups.

---

## What's Next

Tomorrow (2026-05-14) is **AWS Bedrock + Production GenAI Patterns** -- the Phase 2.8 pivot from observability/audit to *what you're actually running in production in 2026*. Foundation models, Knowledge Bases for RAG, Agents with action groups, Guardrails for content filtering and PII redaction, prompt caching, and the cost model that determines whether Bedrock makes sense vs. SageMaker JumpStart vs. self-hosted vLLM on EKS. The audit-trail thread continues there too: every Bedrock model invocation can be captured as a CloudTrail data event (`bedrock:InvokeModel`) -- so the question "which workflow asked which model what prompt with what data" becomes a Lake/Athena query, just like today.
