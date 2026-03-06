# AWS Security Services -- Threat Detection -- The Hospital Security Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), and wired up the airport security checkpoints (IAM). Now it is time to answer the question those systems were designed to support: **what happens when something goes wrong?** When credentials are stolen, when a resource drifts out of compliance, when a container image ships with a known vulnerability -- who detects it, who reports it, and who fixes it? Four AWS services form the detection-to-remediation pipeline, and together they work like the security apparatus of a large hospital complex.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Amazon GuardDuty | The 24/7 surveillance camera system with AI-powered behavior analysis | Continuously monitors CloudTrail, VPC Flow Logs, and DNS logs (plus optional data sources) using machine learning and threat intelligence to detect suspicious activity -- you never configure the cameras, they just watch |
| GuardDuty finding types | Incident reports filed by the camera system | Structured alerts categorized by threat type (Recon, Backdoor, CryptoCurrency, UnauthorizedAccess, etc.) and severity (Low/Medium/High/Critical), each naming the resource type and detection method |
| GuardDuty protection plans | Specialized surveillance upgrades for specific hospital wings | Optional add-ons that extend monitoring to S3 data events, EKS audit logs, RDS login activity, Lambda network activity, runtime OS events, and malware scanning -- each wing requires its own camera type |
| AWS Security Hub | The central security operations center (SOC) with a wall of monitors | Aggregates findings from GuardDuty, Inspector, Config, Macie, and third-party tools into one dashboard using a normalized format (ASFF); evaluates your posture against security standards (CIS, FSBP, PCI DSS); routes findings to EventBridge for automated response |
| ASFF (AWS Security Finding Format) | The standardized incident report template every department must use | A JSON schema that normalizes findings from dozens of different sources into a single format so the SOC can correlate, prioritize, and act on them without translating between formats |
| Security standards/controls | Hospital accreditation checklists (Joint Commission, HIPAA) | Predefined sets of security best-practice checks (AWS FSBP, CIS Benchmarks, PCI DSS, NIST) that Security Hub continuously evaluates your accounts against, generating compliance scores and control findings |
| AWS Config | The building inspector who photographs every room, every day | Records a point-in-time snapshot (configuration item) of every resource whenever it changes; evaluates those snapshots against rules (managed or custom Lambda); flags non-compliant resources and can trigger automated remediation |
| Config Rules | The building code checklist the inspector uses | Managed (pre-built by AWS) or custom (your Lambda function or Guard policy) rules that evaluate whether a resource's configuration is compliant -- triggered by configuration changes or on a periodic schedule |
| Conformance packs | A bundled building code booklet for a specific regulation | A YAML template containing multiple Config Rules and remediation actions deployed as a single unit -- like handing the inspector an entire HIPAA or PCI compliance checklist instead of individual rules |
| Config aggregator | The regional inspector supervisor who collects reports from every hospital branch | A read-only view that collects Config data (compliance status, configuration items) from multiple accounts and regions into a single aggregator account |
| Amazon Inspector | The automated medical equipment safety auditor | Continuously scans EC2 instances, ECR container images, and Lambda functions for known software vulnerabilities (CVEs) and unintended network exposure; adjusts severity scores based on your actual network topology |
| Inspector risk score | The auditor's risk assessment adjusted for your hospital layout | An environment-aware CVSS score that raises or lowers the base NVD severity based on whether the vulnerable resource is actually reachable from the internet |
| Delegated administrator | The chief security officer appointed by the hospital board | A member account (Security Tooling) designated via Organizations to centrally manage each security service on behalf of all accounts, keeping administrative workload off the management account |
| Detection-to-remediation pipeline | The full incident response chain: camera detects, SOC triages, responder acts | GuardDuty/Config/Inspector generate findings, Security Hub normalizes and aggregates them, EventBridge routes them, Lambda/SSM/SNS executes the response -- detection to remediation in seconds |

---

## The Big Picture

In the Organizations doc, you learned how to structure accounts into OUs. In the Control Tower doc, you saw detective controls (Config Rules) deployed automatically. In the IAM doc, you mastered the eight-layer permission evaluation that governs every API call. But all of that is **prevention** -- setting up the rules so bad things are harder to do.

This doc is about what happens when prevention is not enough. When an IAM credential is exfiltrated. When someone opens port 22 to 0.0.0.0/0. When a container image has a critical CVE. When a resource drifts away from its compliant state.

Think of your AWS multi-account environment as a **large hospital complex** -- a sprawling campus with dozens of buildings (accounts), specialized wings (workloads), thousands of staff (IAM principals), and millions of daily interactions (API calls). The hospital board (Organizations) set the policies. The building management system (Control Tower) enforced the baseline wiring. The badge system (IAM) controls who enters which room. But a hospital also needs:

1. **Surveillance cameras** that watch everything and flag anomalies (GuardDuty)
2. **A building inspector** who photographs every room's configuration and checks it against code (AWS Config)
3. **A medical equipment auditor** who scans every device for known defects (Inspector)
4. **A central security operations center (SOC)** that collects all reports, normalizes them, and dispatches responders (Security Hub)

These four services are not alternatives -- they are **complementary layers**, each watching a different dimension of your environment. GuardDuty watches behavior (what is happening right now). Config watches state (is this resource configured correctly). Inspector watches vulnerabilities (does this software have known holes). Security Hub aggregates all three into a single pane of glass and enables automated response.

```
THE HOSPITAL SECURITY APPARATUS -- FOUR COMPLEMENTARY LAYERS
=============================================================================

  ┌──────────────────────────────────────────────────────────────────────┐
  │  HOSPITAL COMPLEX (AWS Multi-Account Environment)                   │
  │                                                                     │
  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │
  │  │ Building A      │  │ Building B      │  │ Building C      │       │
  │  │ (Account 1)     │  │ (Account 2)     │  │ (Account 3)     │       │
  │  │                 │  │                 │  │                 │       │
  │  │ Cameras ◉       │  │ Cameras ◉       │  │ Cameras ◉       │       │
  │  │ (GuardDuty)     │  │ (GuardDuty)     │  │ (GuardDuty)     │       │
  │  │                 │  │                 │  │                 │       │
  │  │ Inspector ◈     │  │ Inspector ◈     │  │ Inspector ◈     │       │
  │  │ (Bldg Inspector)│  │ (Bldg Inspector)│  │ (Bldg Inspector)│       │
  │  │                 │  │                 │  │                 │       │
  │  │ Auditor ◇       │  │ Auditor ◇       │  │ Auditor ◇       │       │
  │  │ (Amazon Insp.)  │  │ (Amazon Insp.)  │  │ (Amazon Insp.)  │       │
  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘       │
  │           │                    │                    │                │
  │           └────────────────────┼────────────────────┘                │
  │                                │                                    │
  │                    ALL FINDINGS FLOW DOWN                           │
  │                                │                                    │
  │           ┌────────────────────▼─────────────────────┐              │
  │           │  SECURITY OPERATIONS CENTER (SOC)        │              │
  │           │  (AWS Security Hub)                      │              │
  │           │                                          │              │
  │           │  ┌──────────┐ ┌──────────┐ ┌──────────┐ │              │
  │           │  │GuardDuty │ │  Config   │ │Inspector │ │              │
  │           │  │ Findings │ │ Findings  │ │ Findings │ │              │
  │           │  └────┬─────┘ └────┬─────┘ └────┬─────┘ │              │
  │           │       │            │            │        │              │
  │           │       └──── ASFF ──┼──── ASFF ──┘        │              │
  │           │            (normalized format)           │              │
  │           │                    │                     │              │
  │           │        ┌───────────▼──────────┐          │              │
  │           │        │ Security Standards   │          │              │
  │           │        │ CIS | FSBP | PCI DSS │          │              │
  │           │        └───────────┬──────────┘          │              │
  │           │                    │                     │              │
  │           │              EventBridge                 │              │
  │           └────────────────────┬─────────────────────┘              │
  │                                │                                    │
  │              ┌─────────────────┼─────────────────┐                  │
  │              ▼                 ▼                 ▼                   │
  │         ┌─────────┐     ┌──────────┐     ┌───────────┐             │
  │         │ Lambda  │     │   SSM    │     │   SNS     │             │
  │         │ (auto-  │     │ (auto-   │     │ (notify   │             │
  │         │ respond)│     │ remediate│     │  humans)  │             │
  │         └─────────┘     └──────────┘     └───────────┘             │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Amazon GuardDuty -- The AI-Powered Surveillance System

### The Analogy

GuardDuty is the **24/7 surveillance camera system with AI-powered behavior analysis**. Unlike a simple motion detector that trips on every bird, these cameras use machine learning, anomaly detection, and threat intelligence feeds to distinguish a nurse walking to the pharmacy (normal API call) from a stranger trying every door handle at 3 AM (credential stuffing). You never install the cameras yourself -- when you flip the switch (enable GuardDuty), the cameras are already positioned throughout the hospital, watching three things by default:

1. **Badge reader logs** (CloudTrail management events) -- who badged into which room and when
2. **Hallway traffic patterns** (VPC Flow Logs) -- who walked where, how much traffic, what direction
3. **Phone directory lookups** (DNS logs) -- what addresses people are looking up (are they querying known malicious domains?)

The key insight: **GuardDuty reads these logs directly from AWS infrastructure through an independent, internal data stream.** It does not pull from your account's CloudTrail S3 bucket or your VPC Flow Logs configuration. Even if you have not enabled VPC Flow Logs in your account, GuardDuty still sees the flow data through its own separate stream. If you also enable VPC Flow Logs, the two streams are independent -- no duplicates, no interference. This means you cannot tamper with the data sources -- the cameras are hardwired into the building, not plugged into a wall socket you can unplug.

> **Interview tip**: "Does GuardDuty require you to enable CloudTrail or VPC Flow Logs?" The answer is NO. GuardDuty consumes its own independent copy of these data streams directly from AWS infrastructure. This is one of the most commonly tested GuardDuty concepts.

### Foundational Data Sources (Always On)

These three data sources are enabled automatically when you turn on GuardDuty. There is no additional cost for them beyond the base GuardDuty charge:

| Data Source | What It Sees | Hospital Analogy |
|-------------|-------------|-----------------|
| CloudTrail management events | Every API call (who did what, from where, when) | Badge reader logs at every door |
| VPC Flow Logs | Network traffic metadata (source/dest IP, port, protocol, bytes) | Hallway traffic cameras tracking movement patterns |
| DNS logs | DNS query activity from EC2 instances via the **Amazon-provided DNS resolver** (the .2 address in each VPC) | Phone directory lookups -- detecting calls to known bad numbers |

**Edge case**: GuardDuty's DNS log monitoring specifically watches queries to the Amazon-provided DNS resolver. If an instance is configured to use a **custom DNS resolver** (common in enterprises with on-prem DNS infrastructure), GuardDuty's DNS data source will not see those queries. The mitigation is **Runtime Monitoring** -- it captures DNS queries at the OS level regardless of which resolver the instance uses.

### Protection Plans (Specialized Camera Upgrades)

Each protection plan extends GuardDuty into a specific domain. Think of them as **specialized surveillance equipment for specific hospital wings** -- the pharmacy needs different monitoring than the emergency room.

| Protection Plan | Data Source | What It Detects | Analogy |
|-----------------|-----------|----------------|---------|
| **S3 Protection** | CloudTrail S3 data events | Data exfiltration, destruction, public access grants | Cameras in the records vault watching who accesses patient files |
| **EKS Protection** | EKS audit logs | Suspicious Kubernetes API calls, privilege escalation, anonymous access | Cameras in the biotech lab watching who touches the equipment |
| **Runtime Monitoring** | OS-level events (EKS, ECS, EC2) | File system access, process execution, network connections at runtime | Vital sign monitors on every patient -- tracking heartbeat (process execution), blood pressure (network connections), and breathing (file access) continuously, alerting when patterns deviate from baseline |
| **RDS Protection** | RDS login activity | Anomalous login attempts, successful logins from suspicious IPs | Badge reader on the database server room door |
| **Lambda Protection** | Lambda network activity logs | Cryptomining, communication with known malicious IPs | Cameras on the automated dispensing machines |
| **Malware Protection for EC2** | EBS volume scanning | Malware, trojans, rootkits on EC2 instances | Scanning every piece of equipment for contamination |
| **Malware Protection for S3** | Newly uploaded S3 objects | Malware in uploaded files | X-raying every incoming package at the loading dock |

### Finding Types and Severity

GuardDuty findings follow a naming convention: `[ThreatPurpose]:[ResourceType]/[ThreatFamilyName].[DetectionMechanism]`

For example: `CryptoCurrency:EC2/BitcoinTool.B!DNS` means:
- **ThreatPurpose**: CryptoCurrency (what the attacker is trying to do)
- **ResourceType**: EC2 (what is being targeted)
- **ThreatFamilyName**: BitcoinTool.B (specific threat variant)
- **DetectionMechanism**: DNS (detected via DNS query analysis)

Major threat purpose categories:

| Category | What It Means | Example |
|----------|-------------|---------|
| **Recon** | Attacker is probing your environment | `Recon:EC2/Portscan` |
| **UnauthorizedAccess** | Someone is accessing resources without permission | `UnauthorizedAccess:EC2/SSHBruteForce` |
| **CryptoCurrency** | Resource is mining cryptocurrency | `CryptoCurrency:EC2/BitcoinTool.B!DNS` |
| **Trojan** | Malware is communicating with command-and-control | `Trojan:EC2/DropPoint` |
| **Backdoor** | Backdoor installed for persistent access | `Backdoor:EC2/C&CActivity.B!DNS` |
| **Stealth** | Attacker is covering their tracks | `Stealth:IAMUser/CloudTrailLoggingDisabled` |
| **CredentialAccess** | Credential theft attempt | `CredentialAccess:IAMUser/AnomalousBehavior` |
| **Exfiltration** | Data is being moved out | `Exfiltration:S3/AnomalousBehavior` |
| **Impact** | Attacker is degrading/destroying resources | `Impact:EC2/PortSweep` |
| **Policy** | Security policy violation | `Policy:S3/BucketPublicAccessGranted` |
| **Discovery** | Internal reconnaissance of your resources | `Discovery:S3/AnomalousBehavior` |
| **Persistence** | Attacker maintaining long-term access | `Persistence:IAMUser/AnomalousBehavior` |
| **PrivilegeEscalation** | Attacker trying to gain higher permissions | `PrivilegeEscalation:IAMUser/AnomalousBehavior` |
| **DefenseEvasion** | Attacker disabling security controls | `DefenseEvasion:IAMUser/AnomalousBehavior` |
| **AttackSequence** | Multi-stage attack detected by Extended Threat Detection | `AttackSequence:EC2/CompromisedInstanceGroup` |

Severity levels:

| Severity | Score Range | Meaning | Action Required |
|----------|-----------|---------|-----------------|
| **Critical** | 9.0 - 10.0 | Primarily multi-stage attack sequences (Extended Threat Detection), though other finding types can also reach Critical | Immediate investigation and response |
| **High** | 7.0 - 8.9 | Resource is actively compromised | Investigate and remediate immediately |
| **Medium** | 4.0 - 6.9 | Suspicious activity deviating from normal | Investigate when possible |
| **Low** | 1.0 - 3.9 | Attempted suspicious activity that did not succeed | Review periodically |

### Extended Threat Detection

Extended Threat Detection is automatically enabled at no additional cost. It correlates multiple individual findings across time and resources to detect **multi-stage attacks** -- situations where individual events look benign but the sequence reveals a coordinated attack. Think of it as the AI system watching the camera feeds and realizing that the person who entered through a side door, visited three medicine cabinets, and photocopied a badge is not a new employee -- they are executing a planned theft.

### Multi-Account Management

GuardDuty supports centralized management via Organizations using the **delegated administrator** pattern you learned in the Organizations doc:

1. The management account designates the Security Tooling account as the GuardDuty delegated administrator
2. The delegated admin can enable GuardDuty in all member accounts with a single action
3. All findings from all accounts flow to the delegated admin
4. Member accounts can view their own findings but cannot disable GuardDuty (the delegated admin controls that)

```json
{
  "AutoEnableOrganizationMembers": "ALL",
  "Features": [
    { "Name": "S3_DATA_EVENTS",      "AutoEnable": "ALL" },
    { "Name": "EKS_AUDIT_LOGS",      "AutoEnable": "ALL" },
    { "Name": "RDS_LOGIN_EVENTS",    "AutoEnable": "ALL" },
    { "Name": "LAMBDA_NETWORK_LOGS", "AutoEnable": "ALL" },
    { "Name": "RUNTIME_MONITORING",  "AutoEnable": "ALL" }
  ]
}
```

### Suppression Rules, Trusted IPs, and Threat Lists

In production, GuardDuty generates a lot of findings. Without filtering, teams drown in false positives. Three mechanisms help manage noise:

| Mechanism | What It Does | Hospital Analogy |
|-----------|-------------|-----------------|
| **Suppression rules** | Auto-archive findings matching criteria (e.g., specific finding types from dev accounts) | Telling cameras to stop alerting on the cleaning crew who walk the halls every night |
| **Trusted IP lists** | IP addresses that GuardDuty will never generate findings for | The staff badge registry -- cameras do not alert on known employees |
| **Threat IP lists** | Custom IP addresses that GuardDuty should flag even if its threat intel feeds do not include them | Your own "banned visitors" list beyond what the police database provides |

Suppression rules do NOT delete findings -- they automatically set the finding status to **Archived**. You can still view and analyze suppressed findings. This is important for audit trails.

### GuardDuty Findings Export

Active findings are retained in GuardDuty for only **90 days**. For long-term retention, configure findings export to an S3 bucket in the Log Archive account, encrypted with a KMS customer-managed key. This is the same Log Archive account you learned about in the Control Tower doc -- it serves as the central, immutable log repository.

---

## Part 2: AWS Config -- The Building Inspector

### The Analogy

AWS Config is the **building inspector who photographs every room, every day**. Every time a room changes -- a wall is moved, an outlet is added, a door is removed -- the inspector takes a new photograph (configuration item) and files it. Then the inspector checks the photograph against the building code (Config Rules). If the room violates code, the inspector files a non-compliance report. If you want, the inspector can also call in a repair crew automatically (remediation action via SSM).

The inspector does not prevent changes (that is the SCP's job). The inspector **detects and reports** changes after they happen. This is why Control Tower uses Config Rules as detective controls -- they catch drift.

> **Interview tip**: "What is the difference between GuardDuty and AWS Config?" GuardDuty watches **behavior** (is someone doing something suspicious?). Config watches **state** (is this resource configured correctly?). GuardDuty detects an attacker; Config detects misconfiguration. This distinction is heavily tested.

### Core Concepts

**Configuration Recorder**: The mechanism that records configuration changes. When enabled, it continuously monitors your supported resources and creates configuration items for each change. You can record all resource types (recommended) or a subset.

**Configuration Item (CI)**: A point-in-time snapshot of a resource's attributes, relationships, and current configuration. Every time an EC2 instance's security group changes, a new CI is created. CIs are the raw data that Config Rules evaluate against.

**Configuration History**: The collection of CIs for a single resource over time. It answers "what did this security group look like last Tuesday?" -- invaluable for forensic investigation after an incident.

**Delivery Channel**: Config can deliver CIs to an S3 bucket (for archival) and an SNS topic (for real-time notification). In the AWS SRA, configuration snapshots are delivered to a centralized S3 bucket in the Log Archive account.

### Config Rules

Config Rules are the building code checklists. They come in three flavors:

| Rule Type | Description | Example |
|-----------|-----------|---------|
| **Managed Rules** | Pre-built by AWS, covers 300+ common compliance checks | `s3-bucket-server-side-encryption-enabled`, `ec2-instance-no-public-ip`, `root-account-mfa-enabled` |
| **Custom Rules (Lambda)** | You write a Lambda function that evaluates the CI and returns COMPLIANT or NON_COMPLIANT | "All EC2 instances must have the tag `CostCenter` with a value matching pattern `CC-\d{4}`" |
| **Custom Rules (Guard)** | You write a declarative Guard policy (same language as CloudFormation Guard) | Simpler syntax than Lambda for many configuration checks |

Rules can be triggered in two ways:

| Trigger Type | When It Fires | Use Case |
|-------------|-------------|----------|
| **Configuration change** | When a matching resource is created, changed, or deleted | Real-time compliance: "Tell me immediately if someone opens port 22 to the world" |
| **Periodic** | On a schedule (1h, 3h, 6h, 12h, 24h) | Recurring checks: "Every 24 hours, verify all S3 buckets have encryption enabled" |

### Evaluation Modes: Detective vs Proactive

| Mode | When It Evaluates | What It Does | Key Limitation |
|------|------------------|-------------|----------------|
| **Detective** | After resource is deployed | Evaluates existing resource configurations | Finds problems after the fact |
| **Proactive** | Before resource is deployed (via API call) | Evaluates proposed resource properties against rules | Does NOT prevent deployment -- only reports whether the proposed config would be compliant |

Proactive evaluation is useful in CI/CD pipelines: before deploying a CloudFormation stack, call the Config API to check if the proposed resources would pass your rules. If they fail, abort the deployment. But note: proactive rules are advisory, not enforcement. They do not block the API call. For actual enforcement at deploy time, see **proactive controls in the Control Tower doc** -- those use CloudFormation Hooks to block non-compliant resources from being created.

### Conformance Packs

A conformance pack is a **bundled building code booklet** -- a YAML template containing multiple Config Rules and their remediation actions, deployed as a single unit. AWS provides sample conformance packs for common frameworks:

- **Operational Best Practices for CIS AWS Foundations Benchmark**
- **Operational Best Practices for PCI DSS**
- **Operational Best Practices for HIPAA Security**
- **Operational Best Practices for NIST 800-53**

You can also create custom conformance packs for your organization's internal standards. When deployed via Organizations, a conformance pack applies the same set of rules to every member account.

### Remediation Actions

When a Config Rule flags a resource as NON_COMPLIANT, you can configure automatic or manual remediation:

| Remediation Type | How It Works | Example |
|-----------------|-------------|---------|
| **Automatic** | Config triggers an SSM Automation document immediately when non-compliance is detected | Rule detects public S3 bucket, SSM doc automatically enables `BlockPublicAccess` |
| **Manual** | Config creates a remediation action that an operator must approve before execution | Rule detects unencrypted EBS volume, operator reviews and clicks "Remediate" |

Both types use **SSM Automation documents** as the remediation mechanism. AWS provides pre-built remediation documents for common scenarios, or you can write custom ones.

```yaml
# Example: AWS Config Rule with automatic remediation in Terraform
resource "aws_config_config_rule" "s3_encryption" {
  name = "s3-bucket-server-side-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }

  scope {
    compliance_resource_types = ["AWS::S3::Bucket"]
  }
}

resource "aws_config_remediation_configuration" "s3_encryption_remediation" {
  config_rule_name = aws_config_config_rule.s3_encryption.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWSConfigRemediation-ConfigureS3BucketEncryption"
  automatic        = true
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60

  parameter {
    name           = "BucketName"
    resource_value = "RESOURCE_ID"
  }

  parameter {
    name         = "SSEAlgorithm"
    static_value = "aws:kms"
  }
}
```

### Config Advanced Query

Config supports a **SQL-like query language** that lets you query the current configuration state of all recorded resources across accounts and regions. This is powerful for ad-hoc investigations and compliance reporting:

```sql
-- Find all EC2 instances without encryption across all accounts
SELECT resourceId, accountId, awsRegion, configuration
WHERE resourceType = 'AWS::EC2::Volume'
  AND configuration.encrypted = 'false'

-- Find all S3 buckets with public access
SELECT resourceId, accountId, configuration.publicAccessBlockConfiguration
WHERE resourceType = 'AWS::S3::Bucket'
  AND configuration.publicAccessBlockConfiguration.blockPublicAcls = 'false'

-- Count resources by type across the organization
SELECT resourceType, COUNT(*)
WHERE accountId LIKE '%'
GROUP BY resourceType
```

> **Interview tip**: "How would you find all unencrypted EBS volumes across 50 accounts?" Config Advanced Query with an aggregator is the answer. This comes up frequently in exam scenarios.

### Config vs CloudTrail -- A Common Confusion

These two services are often confused. The distinction is simple:

| | **CloudTrail** | **AWS Config** |
|---|---|---|
| **Records** | **Who did what** (API calls / actions) | **What does this resource look like** (state snapshots) |
| **Question it answers** | "Who changed the security group at 3 AM?" | "What does the security group look like right now, and is it compliant?" |
| **Hospital analogy** | Badge reader log (who entered the room) | Room photograph (what the room looks like) |
| **Data source for other services** | GuardDuty reads CloudTrail events | Security Hub uses Config Rules for standards |

Config does NOT read from CloudTrail. Config captures resource state by making its own AWS API calls (Describe/List). They are independent services that complement each other.

### Multi-Account Aggregator

A Config aggregator collects compliance data from multiple accounts and regions into a single aggregator account. In the AWS SRA, the Security Tooling account creates the aggregator.

```
CONFIG AGGREGATOR -- CENTRALIZED COMPLIANCE VIEW
=============================================================================

  AGGREGATOR ACCOUNT (Security Tooling)
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  Config Aggregator (read-only view)              │
  │  ┌────────────────────────────────────────────┐  │
  │  │                                            │  │
  │  │  Account A, us-east-1: 47 rules, 3 NON_C  │  │
  │  │  Account A, eu-west-1: 47 rules, 0 NON_C  │  │
  │  │  Account B, us-east-1: 47 rules, 1 NON_C  │  │
  │  │  Account B, eu-west-1: 47 rules, 0 NON_C  │  │
  │  │  Account C, us-east-1: 47 rules, 5 NON_C  │  │
  │  │  ...                                       │  │
  │  └────────────────────────────────────────────┘  │
  │                                                  │
  │  NOTE: Read-only -- cannot deploy rules or        │
  │     trigger remediation from the aggregator.     │
  │     Rules must be deployed via Organization      │
  │     rules or conformance packs from the          │
  │     delegated admin.                             │
  └──────────────────────────────────────────────────┘
         ▲              ▲              ▲
         │              │              │
    ┌────┴─────┐  ┌────┴─────┐  ┌────┴─────┐
    │Account A │  │Account B │  │Account C │
    │us-east-1 │  │us-east-1 │  │us-east-1 │
    │eu-west-1 │  │eu-west-1 │  │eu-west-1 │
    └──────────┘  └──────────┘  └──────────┘
```

**Important**: If you are using AWS Organizations, the aggregator does NOT require individual account authorization -- it automatically collects from all member accounts. Without Organizations, each source account must explicitly authorize the aggregator.

---

## Part 3: Amazon Inspector -- The Medical Equipment Safety Auditor

### The Analogy

Amazon Inspector is the **automated medical equipment safety auditor** who continuously walks through every hospital wing, scanning every piece of equipment (EC2, ECR images, Lambda functions) against a database of known defects (the National Vulnerability Database / NVD). When the auditor finds that a ventilator is running firmware with a known critical flaw (CVE), they file a report. But here is what makes this auditor smart: they do not just report the raw defect severity. They check whether the ventilator is in a locked room with no network access (private subnet, no public IP) or sitting in the open lobby with an internet connection (public subnet, open security group). The same defect gets a higher risk score if the equipment is actually reachable.

### What Inspector Scans

| Target | What It Checks | How It Scans |
|--------|---------------|-------------|
| **EC2 instances** | Software vulnerabilities (CVEs) + network reachability | **Hybrid scanning** (default): agent-based via SSM Agent when available, with agentless fallback via EBS snapshots for instances without SSM Agent. The two methods are complementary, not either/or |
| **ECR container images** | Software vulnerabilities in container image layers | Automatically scans images pushed to ECR and rescans when new CVEs are published |
| **Lambda functions** | Software vulnerabilities in function code and layers | Scans application dependencies and runtime packages |

### Two Finding Categories

| Finding Type | What It Detects | Example |
|-------------|----------------|---------|
| **Software vulnerability** | A CVE in an installed package, library, or runtime | "EC2 instance i-abc123 is running OpenSSL 1.1.1t which has CVE-2023-0286 (Critical)" |
| **Network reachability** | Unintended network exposure via security groups, NACLs, route tables | "EC2 instance i-abc123 has port 22 (SSH) reachable from 0.0.0.0/0 via security group sg-xyz" |

### The Inspector Risk Score

This is what differentiates Inspector from a simple CVE scanner. Inspector calculates an **environment-aware risk score** in CVSS format:

```
INSPECTOR RISK SCORE vs RAW NVD SCORE
=============================================================================

  NVD (National Vulnerability Database) says:
    CVE-2023-XXXXX base CVSS score = 9.8 (Critical)
    Attack vector: Network
    Attack complexity: Low
    Privileges required: None

  Inspector checks YOUR environment:
    Is the EC2 instance in a public subnet?  NO ──┐
    Does the security group allow inbound?   NO   │
    Is there a path from the internet?       NO   ├── Score adjusted DOWN
    Is the instance behind a load balancer?  NO   │
    Is the instance in a private subnet?     YES ─┘

  Inspector risk score = 5.2 (Medium)
  Reason: "Network attack vector, but no network path exists to
           this instance from the internet. Reduced from Critical
           to Medium based on actual network topology."
```

This means you can prioritize findings intelligently: a Critical CVE on a public-facing web server is more urgent than the same Critical CVE on an isolated database instance in a private subnet with no internet path.

> **Interview tip**: "How does Inspector differ from a simple CVE scanner?" Inspector adjusts CVSS scores based on actual network reachability in your environment. A raw Critical CVE becomes Medium if the resource has no internet path. This environment-aware scoring is Inspector's key differentiator.

### Continuous Scanning

Inspector does not require you to schedule scans. It automatically:

1. **Discovers** eligible resources (EC2 with SSM Agent, ECR images, Lambda functions)
2. **Scans** them immediately upon discovery
3. **Rescans** when triggers occur:
   - New package installed on an EC2 instance
   - New CVE published in the NVD that affects an installed package
   - New container image pushed to ECR
   - Lambda function code or layer updated

### Software Bill of Materials (SBOM) Export

Inspector can export a **Software Bill of Materials** in CycloneDX or SPDX format. An SBOM lists every software package and dependency in your environment. This is increasingly important for compliance -- Executive Order 14028 requires SBOM for software sold to the US federal government, and many enterprises are adopting the practice for supply chain security.

### Multi-Account with Organizations

Like GuardDuty, Inspector supports the delegated administrator pattern:
- Security Tooling account is designated as the delegated admin
- Single-click activation across all member accounts
- Aggregated findings visible in the delegated admin account
- Member accounts can view their own findings

---

## Part 4: AWS Security Hub -- The Central Security Operations Center

### The Analogy

Security Hub is the **central security operations center (SOC)** -- the room with a wall of monitors where every camera feed, building inspector report, and equipment audit finding converges. Without the SOC, each department files reports in their own format: the camera system uses one template (GuardDuty finding JSON), the building inspector uses another (Config evaluation result), and the equipment auditor uses a third (Inspector finding). The SOC solves this by requiring every department to translate their reports into a **standardized incident report format** (ASFF) before submission.

But the SOC does more than aggregate. It also runs its own **accreditation checks** -- evaluating your hospital against industry accreditation standards (the Joint Commission, fire codes, infection control protocols), just as Security Hub evaluates your accounts against CIS Benchmarks, AWS Foundational Security Best Practices (FSBP), PCI DSS, and NIST 800-53. And it has a direct line to the response team (EventBridge) so that certain findings trigger automatic responses.

### AWS Security Finding Format (ASFF)

ASFF is the JSON schema that normalizes findings from every integrated service. Key fields:

```json
{
  "SchemaVersion": "2018-10-08",
  "Id": "arn:aws:guardduty:us-east-1:123456789012:detector/abc/finding/xyz",
  "ProductArn": "arn:aws:securityhub:us-east-1::product/aws/guardduty",
  "GeneratorId": "arn:aws:guardduty:us-east-1:123456789012:detector/abc",
  "AwsAccountId": "123456789012",
  "Types": ["TTPs/Initial Access/UnauthorizedAccess:EC2-SSHBruteForce"],
  "CreatedAt": "2026-03-06T10:15:00.000Z",
  "UpdatedAt": "2026-03-06T10:15:00.000Z",
  "Severity": {
    "Label": "HIGH",
    "Normalized": 70
  },
  "Title": "SSH brute force attack detected on i-0abc123def456",
  "Description": "EC2 instance i-0abc123def456 is being targeted by SSH brute force...",
  "Resources": [{
    "Type": "AwsEc2Instance",
    "Id": "arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123def456",
    "Region": "us-east-1"
  }],
  "Workflow": { "Status": "NEW" },
  "RecordState": "ACTIVE"
}
```

Why ASFF matters: when you write an EventBridge rule to trigger a Lambda function, your Lambda receives ASFF-formatted JSON regardless of whether the finding came from GuardDuty, Inspector, Config, or a third-party tool. One Lambda function can process findings from all sources.

### Security Standards and Controls

Security Hub evaluates your accounts against predefined security standards. Each standard consists of multiple controls, and each control maps to one or more Config Rules (Security Hub creates its own service-linked Config Rules for this purpose).

| Standard | What It Checks | Number of Controls |
|----------|---------------|-------------------|
| **AWS Foundational Security Best Practices (FSBP)** | AWS-specific security best practices across 30+ services | 200+ controls |
| **CIS AWS Foundations Benchmark v1.4.0 / v3.0.0** | Industry-standard hardening checks (root MFA, CloudTrail, VPC Flow Logs, etc.) | 50+ controls |
| **PCI DSS v3.2.1 / v4.0** | Payment Card Industry compliance | 100+ controls |
| **NIST SP 800-53 Rev. 5** | US federal government security framework | 200+ controls |

**Important**: Security Hub uses **AWS Config** under the hood to evaluate most controls. This means AWS Config must be enabled and recording resources for Security Hub to generate control findings. The Config Rule *evaluations* that Security Hub creates are service-linked and are not charged additionally as Config Rule evaluations, but the underlying **configuration items** recorded by AWS Config are still charged at standard Config pricing. Do not mistake this for "Security Hub compliance checks are free" -- Config recording costs still apply.

> **Exam tip**: "What is a prerequisite for Security Hub security standards?" AWS Config must be enabled and recording. This is the single most important dependency to remember.

### Central Configuration via Organizations

Security Hub supports **central configuration** from the delegated administrator account. This means:

1. The delegated admin creates **configuration policies** specifying which standards and controls to enable
2. Policies are applied to OUs or specific accounts
3. New accounts added to the OU automatically inherit the policy
4. Member accounts cannot override the central configuration (if managed centrally)

This is the "push model" -- the security team decides what every account evaluates against, and no application team can disable it.

### Cross-Region Aggregation

Security Hub operates per-region. To get a single view across all regions, you designate one region as the **aggregation region** and link other regions to it:

```
SECURITY HUB CROSS-REGION AGGREGATION
=============================================================================

  ┌────────────────────────────┐
  │  AGGREGATION REGION        │
  │  (us-east-1)               │
  │                            │
  │  Findings from:            │
  │  ├── us-east-1 (local)     │◄─────── us-west-2 findings
  │  ├── us-west-2 (linked)    │◄─────── eu-west-1 findings
  │  ├── eu-west-1 (linked)    │◄─────── ap-southeast-1 findings
  │  └── ap-southeast-1        │
  │                            │
  │  Single dashboard for ALL  │
  │  regions                   │
  └────────────────────────────┘
```

### Automation Rules

Security Hub automation rules let you modify findings automatically based on criteria you define, without writing Lambda code:

| Action | Example |
|--------|---------|
| **Suppress findings** | Auto-suppress GuardDuty DNS findings from dev/sandbox accounts |
| **Update severity** | Elevate Inspector findings to Critical if the resource is in a production OU |
| **Update workflow status** | Auto-resolve findings for resources that have been terminated |
| **Add notes** | Append "Known issue, tracked in JIRA-1234" to specific finding patterns |

### Integration with EventBridge

Every finding that flows into Security Hub is also published to EventBridge as a `Security Hub Findings - Imported` event. This is the bridge from detection to remediation:

EventBridge rule pattern to catch high-severity GuardDuty findings:

```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "ProductArn": ["arn:aws:securityhub:us-east-1::product/aws/guardduty"],
      "Severity": {
        "Label": ["HIGH", "CRITICAL"]
      },
      "Workflow": {
        "Status": ["NEW"]
      }
    }
  }
}
```

---

## Part 5: The Complete Architecture -- How All Four Services Work Together

### The Security Tooling Account Pattern

This is the architecture you would deploy in a real enterprise environment. It builds directly on the OU structure from your Organizations doc and the Security Tooling account from your Control Tower doc:

```
SECURITY TOOLING ACCOUNT -- DELEGATED ADMINISTRATOR FOR ALL FOUR SERVICES
=============================================================================

  MANAGEMENT ACCOUNT (governance only, zero workloads)
  │
  │  Designates Security Tooling as delegated admin for:
  │  ├── GuardDuty
  │  ├── Security Hub
  │  ├── AWS Config
  │  └── Inspector
  │
  ▼
  SECURITY TOOLING ACCOUNT (Delegated Administrator)
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
  │  │  GuardDuty   │  │  AWS Config  │  │  Inspector   │          │
  │  │  Admin       │  │  Admin       │  │  Admin       │          │
  │  │              │  │              │  │              │          │
  │  │  - Enable in │  │  - Deploy    │  │  - Enable in │          │
  │  │    all accts │  │    org rules │  │    all accts │          │
  │  │  - View all  │  │  - Deploy    │  │  - View all  │          │
  │  │    findings  │  │    conforman │  │    findings  │          │
  │  │  - Configure │  │    ce packs  │  │  - Configure │          │
  │  │    protect.  │  │  - Aggregate │  │    scan types│          │
  │  │    plans     │  │    compliance│  │              │          │
  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
  │         │                 │                 │                   │
  │         └─────────────────┼─────────────────┘                   │
  │                           │                                     │
  │                  ┌────────▼────────┐                            │
  │                  │  Security Hub   │                            │
  │                  │  (Aggregator)   │                            │
  │                  │                 │                            │
  │                  │  ASFF findings  │                            │
  │                  │  from ALL       │                            │
  │                  │  sources + own  │                            │
  │                  │  standard       │                            │
  │                  │  evaluations    │                            │
  │                  └────────┬────────┘                            │
  │                           │                                     │
  │                  ┌────────▼────────┐                            │
  │                  │  EventBridge    │                            │
  │                  │                 │                            │
  │                  │  Routes:        │                            │
  │                  │  HIGH/CRIT → λ  │                            │
  │                  │  MEDIUM → SNS   │                            │
  │                  │  Config NON_C → │                            │
  │                  │    SSM Auto     │                            │
  │                  └────────┬────────┘                            │
  │                           │                                     │
  │              ┌────────────┼───────────────┐                    │
  │              ▼            ▼               ▼                    │
  │        ┌──────────┐ ┌──────────┐  ┌───────────┐              │
  │        │ Lambda   │ │   SNS    │  │    SSM     │              │
  │        │ Auto-    │ │ (Slack,  │  │ Automation │              │
  │        │ Response │ │  PagerD.)│  │ (fix it)   │              │
  │        └──────────┘ └──────────┘  └───────────┘              │
  └──────────────────────────────────────────────────────────────────┘

  LOG ARCHIVE ACCOUNT (Separate -- immutable storage)
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  ┌─────────────────────┐  ┌──────────────────────┐             │
  │  │ GuardDuty Findings  │  │ Config Snapshots     │             │
  │  │ Export S3 Bucket     │  │ S3 Bucket            │             │
  │  │ (KMS encrypted)     │  │ (delivery channel)   │             │
  │  └─────────────────────┘  └──────────────────────┘             │
  │                                                                  │
  │  ┌─────────────────────┐                                        │
  │  │ CloudTrail Org      │                                        │
  │  │ Trail S3 Bucket     │                                        │
  │  └─────────────────────┘                                        │
  │                                                                  │
  │  Separation of duties: Security Tooling MANAGES the services,  │
  │  Log Archive STORES the evidence. Neither can tamper with the   │
  │  other's domain.                                                │
  └──────────────────────────────────────────────────────────────────┘
```

### The Detection-to-Remediation Pipeline

Here is the full flow from threat detection to automated response:

```
DETECTION-TO-REMEDIATION PIPELINE
=============================================================================

  DETECTION LAYER (what happened?)
  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │  GuardDuty                Config              Inspector   │
  │  "Someone is SSH        "S3 bucket lost      "EC2 has    │
  │   brute-forcing         encryption"          CVE-2024-   │
  │   instance i-abc"                             XXXX"       │
  │                                                            │
  └──────────┬───────────────────┬───────────────────┬─────────┘
             │                   │                   │
             ▼                   ▼                   ▼
  AGGREGATION LAYER (normalize and correlate)
  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │  Security Hub                                              │
  │  ┌──────────────────────────────────────────────────┐     │
  │  │ Convert all findings to ASFF                     │     │
  │  │ Apply automation rules (suppress, update, route) │     │
  │  │ Evaluate against security standards              │     │
  │  │ Calculate compliance scores                      │     │
  │  └──────────────────────────┬───────────────────────┘     │
  │                              │                             │
  └──────────────────────────────┼─────────────────────────────┘
                                 │
                                 ▼
  ROUTING LAYER (who handles this?)
  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │  EventBridge                                               │
  │  ┌──────────────────────────────────────────────────┐     │
  │  │ Rule 1: CRITICAL GuardDuty → Lambda (isolate)    │     │
  │  │ Rule 2: HIGH GuardDuty → SNS (page on-call)     │     │
  │  │ Rule 3: Config NON_COMPLIANT → SSM (auto-fix)   │     │
  │  │ Rule 4: Inspector CRITICAL → SNS + JIRA         │     │
  │  │ Rule 5: All findings → S3 (archive)             │     │
  │  └──────────────────────────┬───────────────────────┘     │
  │                              │                             │
  └──────────────────────────────┼─────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
  RESPONSE LAYER (do something about it)
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Lambda      │  │  SSM Auto.   │  │  SNS         │
  │              │  │              │  │              │
  │  - Isolate   │  │  - Re-enable │  │  - Slack     │
  │    instance  │  │    encryption│  │  - PagerDuty │
  │  - Revoke    │  │  - Block     │  │  - Email     │
  │    IAM keys  │  │    public    │  │  - JIRA      │
  │  - Snapshot  │  │    access    │  │              │
  │    EBS vol   │  │  - Apply     │  │              │
  │  - Quarantine│  │    missing   │  │              │
  │    SG        │  │    tags      │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘
```

### Example: Automated Incident Response for Compromised EC2

This is a realistic end-to-end example of the pipeline in action:

```python
# Lambda function triggered by EventBridge when GuardDuty detects
# a HIGH/CRITICAL finding on an EC2 instance.
#
# IMPORTANT: This Lambda runs in the Security Tooling account, but the
# compromised instance lives in a member account. We must AssumeRole
# into the member account first. Each member account needs a cross-account
# role (e.g., "SecurityIncidentResponse") that this Lambda can assume.
# See the IAM Advanced Patterns doc for the AssumeRole pattern.
import boto3
import json

sts = boto3.client('sts')
sns = boto3.client('sns')

# Pre-created "quarantine" security group that allows NO inbound/outbound
# (must exist in each member account -- deploy via StackSets or Terraform)
QUARANTINE_SG = 'sg-0quarantine000000'
SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:111111111111:security-alerts'
CROSS_ACCOUNT_ROLE = 'arn:aws:iam::{account_id}:role/SecurityIncidentResponse'

def get_member_ec2_client(account_id):
    """Assume role into the member account to operate on its resources."""
    credentials = sts.assume_role(
        RoleArn=CROSS_ACCOUNT_ROLE.format(account_id=account_id),
        RoleSessionName='guardduty-incident-response'
    )['Credentials']
    return boto3.client('ec2',
        aws_access_key_id=credentials['AccessKeyId'],
        aws_secret_access_key=credentials['SecretAccessKey'],
        aws_session_token=credentials['SessionToken']
    )

def handler(event, context):
    finding = event['detail']['findings'][0]
    severity = finding['Severity']['Label']
    instance_id = finding['Resources'][0]['Id'].split('/')[-1]
    title = finding['Title']
    account_id = finding['AwsAccountId']

    print(f"[{severity}] {title} on {instance_id} in {account_id}")

    # Get an EC2 client for the member account where the instance lives
    ec2 = get_member_ec2_client(account_id)

    # Step 1: Isolate the instance by replacing all security groups
    # with a quarantine SG that blocks all traffic
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=[QUARANTINE_SG]
    )

    # Step 2: Create a snapshot of the instance's EBS volumes
    # for forensic analysis
    volumes = ec2.describe_instances(
        InstanceIds=[instance_id]
    )['Reservations'][0]['Instances'][0]['BlockDeviceMappings']

    for vol in volumes:
        volume_id = vol['Ebs']['VolumeId']
        ec2.create_snapshot(
            VolumeId=volume_id,
            Description=f"Forensic snapshot - GuardDuty finding - {instance_id}",
            TagSpecifications=[{
                'ResourceType': 'snapshot',
                'Tags': [
                    {'Key': 'Purpose', 'Value': 'forensics'},
                    {'Key': 'FindingId', 'Value': finding['Id']},
                    {'Key': 'OriginalInstance', 'Value': instance_id}
                ]
            }]
        )

    # Step 3: Notify the security team
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=f"[{severity}] EC2 Instance Quarantined - {instance_id}",
        Message=json.dumps({
            'action': 'Instance isolated and snapshots created',
            'instance_id': instance_id,
            'account_id': account_id,
            'finding_title': title,
            'finding_id': finding['Id'],
            'severity': severity
        }, indent=2)
    )

    return {'statusCode': 200, 'body': f"Quarantined {instance_id}"}
```

---

## Part 6: Service Comparison -- When to Use What

| Dimension | GuardDuty | AWS Config | Inspector | Security Hub |
|-----------|-----------|-----------|-----------|-------------|
| **Primary purpose** | Threat detection (behavioral) | Configuration compliance | Vulnerability assessment | Finding aggregation + posture management |
| **What it watches** | API calls, network flows, DNS queries, runtime events | Resource configuration state | Software packages, network exposure | Findings from all other services |
| **Detection type** | Anomaly + threat intelligence | Rule-based compliance | CVE database + network analysis | Standard-based compliance + aggregation |
| **Data source** | CloudTrail, VPC Flow Logs, DNS, EKS audit logs, etc. | AWS resource configuration (API describe/list) | SSM inventory, ECR image layers, Lambda packages | ASFF findings from integrated services |
| **Output** | Findings (threat alerts) | Compliance evaluations (COMPLIANT / NON_COMPLIANT) | Findings (vulnerabilities + network exposure) | Normalized ASFF findings + compliance scores |
| **Can remediate?** | No (detection only -- routes to EventBridge) | Yes (via SSM Automation documents) | No (detection only -- routes to Security Hub) | No (routes to EventBridge for action) |
| **Pricing model** | Per volume of data analyzed | Per config item recorded + per rule evaluation | Per instance/image/function scanned | Per finding ingested + per security check |
| **Requires Config?** | No | N/A | No | Yes (for security standard checks) |

---

## Cost Considerations

These services can generate surprise bills if not managed carefully:

| Service | Cost Driver | Common Gotcha |
|---------|-----------|---------------|
| **GuardDuty** | Volume of CloudTrail events + VPC Flow Log data + protection plan data | High-traffic accounts with many API calls or network flows can generate significant costs. S3 Protection and Runtime Monitoring add substantially |
| **AWS Config** | Configuration items recorded + rule evaluations | Enabling Config recording in all regions (recommended for Security Hub) means paying for CIs in regions where you have few resources. Can add up fast at scale |
| **Inspector** | Per instance/image/function scanned per month | Inspector v2 uses continuous scanning (no more one-time assessment pricing). Costs scale with fleet size |
| **Security Hub** | Per finding ingested + per security check run | The checks themselves are charged, plus you need Config running (which has its own costs). In a large org with all standards enabled, check volume adds up |

**Tip**: Use the 30-day free trial for each service in a representative account to estimate costs before org-wide rollout. GuardDuty and Security Hub both offer usage statistics pages to project costs.

---

## Key Takeaways

- **GuardDuty is always-on, zero-configuration threat detection.** It reads logs directly from AWS infrastructure -- you do not configure VPC Flow Logs or CloudTrail in your account for GuardDuty to work. The three foundational data sources (CloudTrail management events, VPC Flow Logs, DNS logs) are automatically consumed. Protection plans extend coverage to specific services (S3, EKS, RDS, Lambda, Malware, Runtime).

- **AWS Config is about state, not behavior.** It answers "is this resource configured correctly right now?" not "is someone doing something suspicious?" Config Rules are the detective controls you saw in Control Tower. Conformance packs bundle rules for specific compliance frameworks. Remediation actions use SSM Automation documents to fix non-compliant resources automatically.

- **Inspector is about known vulnerabilities, not unknown threats.** It scans against the NVD/CVE database and adjusts severity based on your actual network topology (the Inspector risk score). It covers EC2, ECR images, and Lambda functions. It continuously rescans when new CVEs are published or when packages change.

- **Security Hub is the aggregation and posture management layer.** It does not detect threats itself (except through security standard checks powered by Config Rules). Its value is normalizing findings from all sources into ASFF, evaluating your posture against industry standards, and routing everything to EventBridge for automated response.

- **All four services support the delegated administrator pattern, but Config's model is more limited.** GuardDuty, Security Hub, and Inspector use the standard `RegisterDelegatedAdministrator` API. Config's delegated admin is registered through the Config service itself and is capped at 3 delegated admins. The delegation scope is also narrower -- it covers organization rules, conformance packs, and aggregation, but not all Config management operations. Despite this difference, the Security Tooling account still serves as delegated admin for all four.

- **The Log Archive account stores the evidence; the Security Tooling account manages the investigation.** This separation of duties prevents the team managing security services from tampering with the logs those services produce. GuardDuty findings export, Config snapshots, and CloudTrail org trails all land in Log Archive S3 buckets encrypted with KMS keys.

- **Security Hub requires AWS Config.** Security Hub's security standard checks are implemented as service-linked Config Rules. If Config is not enabled and recording resources, Security Hub cannot evaluate controls. The Config Rule evaluations created by Security Hub are not charged as additional rule evaluations, but the underlying configuration item recording costs still apply at standard Config pricing.

- **The detection-to-remediation pipeline is: Source Service generates a finding, Security Hub normalizes it to ASFF, EventBridge routes it based on rules, and Lambda/SSM/SNS executes the response.** This pipeline can operate in seconds for automated responses or route to humans for manual investigation.

- **GuardDuty findings are retained for 90 days.** Configure findings export to S3 in the Log Archive account for long-term retention. This is especially important for compliance audits that require historical evidence.

- **Proactive Config Rules evaluate before deployment but do not prevent it.** They are advisory only. For actual prevention, use SCPs (preventive controls) or CloudFormation Hooks (proactive controls from Control Tower).

- **This four-service stack has specific blind spots.** It does not inspect data contents (use **Amazon Macie** for sensitive data discovery), does not analyze permission sprawl or unused access (use **IAM Access Analyzer**), does not correlate findings into attack chain narratives for investigation (use **Amazon Detective**), and does not inspect application-layer HTTP payloads for SQL injection or XSS (use **AWS WAF**). True security coverage requires all these services working together, with Security Hub as the central aggregation point.

---

## Further Reading

- [GuardDuty Finding Types Reference](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html) -- complete list of all finding types with severity levels
- [AWS Config Managed Rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html) -- full list of 300+ pre-built rules
- [Security Hub Security Standards](https://docs.aws.amazon.com/securityhub/latest/userguide/standards-reference.html) -- details on FSBP, CIS, PCI DSS, and NIST standards
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/security-tooling.html) -- how all services integrate in the Security Tooling account
- [AWS Security Services Best Practices -- Security Hub](https://aws.github.io/aws-security-services-best-practices/guides/security-hub/) -- production deployment patterns
