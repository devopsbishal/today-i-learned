# AWS Organizations & Service Control Policies -- The Corporate Headquarters Analogy

> Understanding AWS Organizations and SCPs through the analogy of a corporate headquarters governing a family of subsidiary companies -- where the parent company sets non-negotiable policies that apply across all subsidiaries, regional offices group subsidiaries by function, and the corporate charter defines what any subsidiary is *allowed to do* regardless of how much authority an individual employee has within their own office.

---

## TL;DR

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| AWS Organization | A corporate conglomerate | A single parent entity that owns and governs multiple subsidiary companies (accounts), providing centralized billing, policy enforcement, and resource sharing |
| Management Account | Corporate headquarters (the parent company) | The one account that creates the Organization, manages OUs, attaches SCPs, and controls billing; it is exempt from SCPs -- which makes it a high-security target |
| Member Account | A subsidiary company | An individual AWS account that operates under the governance of the Organization; subject to all SCPs attached above it |
| Root | The corporate charter (articles of incorporation) | The top-level container of the Organization; every OU and account descends from it; SCPs attached here apply to every member account |
| Organizational Unit (OU) | A regional office or division | A logical grouping of accounts by function (Security, Infrastructure, Workloads, Sandbox); policies attached to an OU apply to every account inside it |
| Service Control Policy (SCP) | A corporate policy manual | A JSON policy that sets the *maximum permissions ceiling* for every account in the OU; it never grants permissions -- it only restricts what IAM policies can do |
| FullAWSAccess SCP | The default "do anything" charter | The managed SCP attached by default at every level when you enable SCPs; removing it without a replacement implicitly denies everything |
| Deny-list strategy | A corporate policy that says "you can do anything EXCEPT these banned activities" | Keep FullAWSAccess and layer on explicit Deny statements for what you want to block; recommended for most organizations |
| Allow-list strategy | A corporate policy that says "you can ONLY do these pre-approved activities" | Remove FullAWSAccess and explicitly Allow only the services you approve; used in highly regulated environments |
| SCP inheritance | Policy cascading from HQ down through regional offices | Allow must exist at every level (intersection); Deny at any level blocks the entire subtree below (union); a Deny at root overrides any Allow at the OU or account level |
| Effective permissions | What an employee can actually do after all policies are evaluated | The intersection of SCP ceiling, IAM policy grant, and (optionally) permission boundary; if any layer says no, the answer is no |
| Consolidated billing | One corporate finance department paying all subsidiary bills | All member accounts' usage rolls up to the management account's single invoice; enables volume discounts and Reserved Instance sharing |
| Delegated administrator | Appointing a subsidiary as the regional compliance office | Designating a member account to manage a specific AWS service (GuardDuty, Config, Security Hub) on behalf of the Organization, keeping workload off the management account |
| Trusted access | Giving a corporate auditor keys to every subsidiary's office | Enabling an AWS service to operate across all accounts in the Organization (CloudTrail creating an org trail, Config deploying org rules) |

---

## The Big Picture

In the Week 1 networking deep dives, you built the physical infrastructure: VPCs, Transit Gateway, Direct Connect, VPN. But who *owns* those VPCs? Who decides which teams get which accounts? Who enforces that nobody disables CloudTrail or launches resources in unauthorized regions? That is the domain of **AWS Organizations and Service Control Policies** -- the governance layer that sits above all the networking and compute infrastructure you have built so far.

Think of it this way. In Week 1, you were the **highway engineer** designing roads, bridges, and toll plazas. This week, you step into the role of **corporate headquarters** -- the parent company that owns every subsidiary, organizes them into divisions, and publishes the policy manual that every subsidiary must follow. The highways you built still exist, but now you are deciding which divisions get to use them, what they are allowed to ship on them, and what is categorically forbidden across the entire corporation.

The analogy works because AWS Organizations is fundamentally a **governance hierarchy**. A corporate conglomerate has a parent company (management account), divisions (OUs), and subsidiaries (member accounts). The parent company publishes policies (SCPs) that cascade down through the hierarchy. A subsidiary can give its employees (IAM users/roles) whatever permissions it wants internally, but those permissions can never exceed what the corporate charter permits. If headquarters says "no operations in Asia-Pacific," then no subsidiary employee can launch an EC2 instance in ap-southeast-1 -- even if their local manager (IAM policy) explicitly grants it.

```
AWS ORGANIZATIONS -- THE CORPORATE GOVERNANCE MODEL
=============================================================================

  CORPORATE CONGLOMERATE (AWS Organization)
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                                                                          │
  │  HEADQUARTERS (Management Account - 111111111111)                        │
  │  ├── Pays all bills (consolidated billing)                               │
  │  ├── Creates and dissolves subsidiaries (accounts)                       │
  │  ├── Organizes divisions (OUs)                                           │
  │  ├── Publishes policy manuals (SCPs)                                     │
  │  ├── NOT subject to its own policies (SCPs do not affect mgmt account)  │
  │  └── Should run ZERO workloads (keep it pristine for governance only)   │
  │                                                                          │
  │  ROOT (Corporate Charter)                                                │
  │  ├── SCP: FullAWSAccess (default -- allows everything)                  │
  │  ├── SCP: DenyRootUser (you add this -- applies to ALL member accounts) │
  │  ├── SCP: RestrictRegions (you add this -- applies to ALL)              │
  │  │                                                                       │
  │  ├── SECURITY OU (Compliance Division)                                   │
  │  │   ├── Log Archive Account (centralized CloudTrail, Config logs)      │
  │  │   └── Security Tooling Account (GuardDuty, Security Hub delegated)   │
  │  │                                                                       │
  │  ├── INFRASTRUCTURE OU (Shared Services Division)                        │
  │  │   ├── Networking Account (Transit Gateway, Route 53, VPN)            │
  │  │   └── Shared Services Account (CI/CD, artifact repos, DNS)           │
  │  │                                                                       │
  │  ├── WORKLOADS OU (Business Divisions)                                   │
  │  │   ├── Prod OU (sub-OU)                                               │
  │  │   │   ├── App-A Production Account                                   │
  │  │   │   └── App-B Production Account                                   │
  │  │   └── Non-Prod OU (sub-OU)                                           │
  │  │       ├── App-A Staging Account                                      │
  │  │       └── App-B Dev Account                                          │
  │  │                                                                       │
  │  ├── SANDBOX OU (R&D Labs)                                               │
  │  │   ├── SCP: Aggressive cost controls                                  │
  │  │   ├── Developer-A Sandbox Account                                    │
  │  │   └── Developer-B Sandbox Account                                    │
  │  │                                                                       │
  │  └── SUSPENDED OU (Decommissioned)                                       │
  │      ├── SCP: DenyAll (deny every API call)                             │
  │      └── Old-Project Account (pending 90-day closure)                   │
  │                                                                          │
  └──────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: AWS Organizations Structure -- Inside the Corporate Conglomerate

### The Management Account

**The Analogy**: The management account is **corporate headquarters** -- the parent company that incorporated the conglomerate. HQ signs the checks (consolidated billing), creates new subsidiaries (member accounts), organizes them into divisions (OUs), and publishes the policy manual (SCPs). But here is the critical detail: **headquarters is exempt from its own policies**. The CEO can do things that subsidiary employees cannot. This is why the management account is the most sensitive account in your entire Organization -- if it is compromised, the attacker has unrestricted access to governance controls.

**The Technical Reality**:

- The management account is the account that calls `CreateOrganization`. There can be only one per Organization.
- SCPs do **not** apply to the management account. Even if you attach a Deny-all SCP to the root, the management account is unaffected.
- AWS strongly recommends running **zero workloads** in the management account. It should contain only Organization management resources, billing configuration, and a break-glass IAM role.
- **Why zero workloads matters**: A compromised workload in the management account is not just an account compromise -- it is a **governance compromise**. The attacker can call Organization management APIs to dismantle your entire security posture: `DetachPolicy` (remove SCPs), `MoveAccount` (relocate accounts out of protected OUs), `DeleteOrganizationalUnit` (destroy OU structure), `RemoveAccountFromOrganization` (eject accounts, stripping all SCP protection), and `CreateAccount` (spin up new ungoverned accounts).
- The management account's root user has unrestricted access to all Organization APIs. Protect it with MFA and lock away the credentials.
- You can change the management account's email but you **cannot** transfer the management role to another account without re-creating the Organization.

### Member Accounts

**The Analogy**: Each member account is a **subsidiary company**. It has its own employees (IAM users/roles), its own internal policies (IAM policies), and runs its own business (workloads). But it operates under the governance framework set by headquarters. A subsidiary cannot override corporate policy -- if HQ says "no operations in region X," the subsidiary complies even if the local VP wants to expand there.

**The Technical Reality**:

- Member accounts are standard AWS accounts that have been invited into (or created within) the Organization.
- When you create an account via Organizations (`CreateAccount`), AWS automatically creates an `OrganizationAccountAccessRole` IAM role in the new account. This role trusts the management account and has `AdministratorAccess`. It is your initial cross-account access path.
- Member accounts can be moved between OUs at any time. Moving an account immediately changes which SCPs apply to it.
- A member account can leave the Organization, but only if it has a standalone payment method configured and no SCP blocks `organizations:LeaveOrganization`.

### Organizational Units (OUs)

**The Analogy**: OUs are **regional offices or corporate divisions**. You group subsidiaries by function, not by geography or org chart. The Compliance Division (Security OU) houses the auditing subsidiary and the log-keeping subsidiary. The Manufacturing Division (Workloads OU) houses the production and development subsidiaries. Each division can have its own addendum to the corporate policy manual.

**The Technical Reality**:

- OUs are containers within the Organization that hold accounts and/or other OUs (nested OUs).
- AWS supports **up to five levels** of nesting, but recommends **maximum two** (root > OU > sub-OU). Deeper nesting makes SCP inheritance harder to debug.
- An account can belong to **exactly one OU** at a time (including the root, which is itself a container).
- SCPs can be attached to OUs. When attached, they apply to every account within that OU and all child OUs.
- OUs cannot be deleted if they contain accounts. Move accounts out first.

### The Root

**The Analogy**: The root is the **corporate charter** -- the founding document that establishes the conglomerate. Every division and subsidiary traces its authority back to this charter. Policies attached to the charter apply universally, to every entity in the conglomerate.

**The Technical Reality**:

- The root is the top-level container. It is not an OU, but it can have SCPs attached to it.
- Every Organization has exactly one root.
- When you enable SCPs, AWS attaches the `FullAWSAccess` managed SCP at the root. This is the default baseline that allows everything.
- SCPs attached at the root apply to **all member accounts** in the Organization (but not the management account).

---

## Part 2: Multi-Account Strategy -- Designing the Corporate Structure

AWS recommends a specific OU taxonomy. Here is the rationale for each division:

```
RECOMMENDED OU STRUCTURE -- WHY EACH DIVISION EXISTS
=============================================================================

  ROOT
  │
  ├── SECURITY OU
  │   │  PURPOSE: Isolate security and audit functions from workloads.
  │   │  WHY SEPARATE: Security team needs read access everywhere but
  │   │  should not be affected by workload SCPs. Attackers who compromise
  │   │  a workload account cannot tamper with centralized logs.
  │   │
  │   ├── Log Archive Account
  │   │   ├── Centralized CloudTrail organization trail destination
  │   │   ├── AWS Config delivery channel for org-wide config snapshots
  │   │   ├── VPC Flow Logs aggregation
  │   │   ├── S3 bucket policies deny deletion (even from root user)
  │   │   └── Only the security team has access
  │   │
  │   └── Security Tooling Account
  │       ├── GuardDuty delegated administrator
  │       ├── Security Hub delegated administrator
  │       ├── IAM Access Analyzer (organization-level)
  │       ├── AWS Config aggregator
  │       └── Automated remediation Lambda functions
  │
  ├── INFRASTRUCTURE OU
  │   │  PURPOSE: Shared platform services used by all workloads.
  │   │  WHY SEPARATE: Networking and shared services are consumed by
  │   │  every other account but managed by a dedicated platform team.
  │   │
  │   ├── Networking Account
  │   │   ├── Owns Transit Gateway (shared via RAM)
  │   │   ├── Owns Direct Connect / VPN connections
  │   │   ├── Central Route 53 hosted zones
  │   │   ├── VPC IPAM for centralized CIDR management
  │   │   └── Egress VPC with NAT Gateways
  │   │
  │   └── Shared Services Account
  │       ├── CI/CD pipelines (CodePipeline, Jenkins)
  │       ├── Artifact repositories (ECR, CodeArtifact)
  │       ├── Active Directory / IAM Identity Center
  │       └── AMI factory (golden images)
  │
  ├── WORKLOADS OU
  │   │  PURPOSE: Where actual applications run.
  │   │  WHY SUB-OUs: Prod and non-prod need different SCPs.
  │   │  Prod: strict controls, no experimental services.
  │   │  Non-prod: relaxed controls, broader service access.
  │   │
  │   ├── Prod OU (sub-OU)
  │   │   ├── SCP: Only approved services
  │   │   ├── SCP: Enforce encryption on all storage
  │   │   ├── App-A Prod Account
  │   │   └── App-B Prod Account
  │   │
  │   └── Non-Prod OU (sub-OU)
  │       ├── SCP: Relaxed service list
  │       ├── App-A Staging Account
  │       └── App-B Dev Account
  │
  ├── SANDBOX OU
  │   │  PURPOSE: Developer experimentation without risk.
  │   │  WHY SEPARATE: Developers need freedom to try new services
  │   │  but must not accumulate cost or create security exposure.
  │   │
  │   ├── SCP: Deny expensive services (SageMaker, Redshift, large EC2)
  │   ├── SCP: Restrict to specific regions
  │   ├── AWS Budgets: Alert at $50/month (can trigger auto-remediation)
  │   ├── Auto-cleanup: Lambda that nukes resources older than 7 days
  │   ├── Developer-A Sandbox
  │   └── Developer-B Sandbox
  │
  ├── POLICY STAGING OU
  │   │  PURPOSE: Test SCP changes before applying to production OUs.
  │   │  Contains a sacrificial test account where you validate that
  │   │  new SCPs do not break legitimate workloads.
  │   │
  │   └── SCP-Test Account
  │
  └── SUSPENDED OU
      │  PURPOSE: Quarantine for accounts being decommissioned.
      │  WHY NOT DELETE: AWS requires 90 days to close an account.
      │  During that period, the account should be completely locked down.
      │
      ├── SCP: DenyAll (explicit deny on every action)
      └── Old-Project Account
```

### Key Design Principles

1. **Organize by function and compliance requirements, not by business unit or team structure.** SCPs apply to OUs, not to people. If your security team and your app team are in the same OU, you cannot give them different permission ceilings.

2. **Keep the management account workload-free.** Every resource in the management account is ungoverned by SCPs. Running workloads there creates an ungovernably privileged environment.

3. **One account per workload per environment.** App-A production gets its own account. App-A staging gets its own account. This provides blast radius containment, independent service quotas, and clean billing separation.

4. **Maximum two levels of OU nesting.** Root > OU > sub-OU is the deepest you should go. Deeper nesting makes SCP inheritance exponentially harder to reason about.

---

## Part 3: Service Control Policies -- The Corporate Policy Manual

### What SCPs Are (and Are Not)

**The Analogy**: An SCP is a **corporate policy manual** that defines the *maximum boundaries* of what any employee at any subsidiary can do. The policy manual does not hand out keys or grant access -- it only defines the outer walls. If the corporate policy manual says "no subsidiary may operate heavy machinery," then even if a subsidiary's local manager gives an employee a machinery license (IAM policy), that employee still cannot operate the machinery. The license is valid within the subsidiary, but it exceeds the corporate charter's ceiling.

**The Technical Reality**:

```
WHAT SCPs DO vs WHAT SCPs DO NOT DO
=============================================================================

  SCPs DO:
  ├── Set the MAXIMUM permissions boundary for all IAM entities in an account
  ├── Restrict which AWS services and actions are available
  ├── Use Condition keys to create fine-grained restrictions
  │   (e.g., deny unless encrypted, deny unless in approved region)
  ├── Apply to ALL IAM users, roles, and the ROOT USER in member accounts
  ├── Cascade down from root → OU → child OU → account
  └── Support both Allow and Deny statements

  SCPs DO NOT:
  ├── Grant any permissions (you still need IAM policies for that)
  ├── Affect the management account (even if attached at root)
  ├── Affect service-linked roles (roles created by AWS services)
  ├── Affect resource-based policies evaluated in the target account
  │   (e.g., S3 bucket policy granting cross-account access -- the SCP
  │    is evaluated in the CALLING account, not the resource account)
  ├── Apply to AWS Organizations API calls made by the management account
  └── Affect actions performed by service-linked roles created by
      AWS services (e.g., roles created by ECS, RDS, or Auto Scaling)

  THE CEILING ANALOGY:
  ════════════════════════════════════════════════════════

    SCP CEILING  ──────────────────────────  (max possible)
         │
         │    IAM Policy Grant  ──────────  (what you actually get)
         │         │
         │         │    EFFECTIVE PERMISSIONS = intersection
         │         │         │
         │         ▼         ▼
    ─────┼─────────┼─────────┼──────────────────────
         │         │         │
         │    If IAM grant exceeds SCP ceiling,
         │    the excess is silently clipped.
         │    No error — just denied at evaluation time.
         │
    SCP = "you may do UP TO this much"
    IAM = "you are granted THIS MUCH"
    Effective = the OVERLAP of both
```

### SCP Evaluation Logic -- How the Intersection Works

This is the single most tested Organizations concept in interviews. The evaluation follows two rules:

**Rule 1 -- Allow is an intersection**: For an action to be permitted, an Allow must exist at **every level** from the root, through each OU in the path, down to the account. If any level does not Allow the action, it is implicitly denied.

**Rule 2 -- Deny is a union (inherited)**: A Deny at any level blocks the action for the entire subtree below. A Deny at root blocks the action in every member account. A Deny at an OU blocks it in every child OU and account.

**The Analogy**: Imagine the corporate conglomerate has three levels of approval for employee activities: the corporate charter (root), the division policy (OU), and the subsidiary's internal rules (account). For an employee to do something:
- The charter must allow it **AND**
- The division policy must allow it **AND**
- The subsidiary's rules must allow it **AND**
- No level can have explicitly banned it

If the charter bans firearms, no division and no subsidiary can override that. If the Manufacturing Division bans open flames, subsidiaries in that division comply even if the charter is silent on flames.

```
SCP EVALUATION -- SEVEN SCENARIOS
=============================================================================

  Organization Structure:
  ROOT → OU-A → Account-X

  Scenario 1: DEFAULT STATE (FullAWSAccess everywhere)
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  ROOT    │    │  OU-A    │    │ Account-X │
  │ Allow: * │───▶│ Allow: * │───▶│ Allow: *  │  ✅ ec2:* ALLOWED
  └──────────┘    └──────────┘    └──────────┘
  Explanation: FullAWSAccess at every level. All actions are within the ceiling.

  Scenario 2: DENY AT ROOT
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  ROOT    │    │  OU-A    │    │ Account-X │
  │ Allow: * │───▶│ Allow: * │───▶│ Allow: *  │  ❌ ec2:* DENIED
  │ Deny:ec2 │    │          │    │           │
  └──────────┘    └──────────┘    └──────────┘
  Explanation: Deny at root overrides ALL Allow statements below. No member
  account can use EC2, regardless of OU or account-level SCPs.

  Scenario 3: DENY AT OU
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  ROOT    │    │  OU-A    │    │ Account-X │
  │ Allow: * │───▶│ Allow: * │───▶│ Allow: *  │  ❌ ec2:* DENIED
  │          │    │ Deny:ec2 │    │           │     (for Account-X)
  └──────────┘    └──────────┘    └──────────┘
  Explanation: Deny at OU-A blocks EC2 for all accounts in OU-A.
  Accounts in other OUs are unaffected.

  Scenario 4: REMOVE FullAWSAccess AT OU (no replacement)
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  ROOT    │    │  OU-A    │    │ Account-X │
  │ Allow: * │───▶│ (empty)  │───▶│ Allow: *  │  ❌ ALL ACTIONS DENIED
  └──────────┘    └──────────┘    └──────────┘
  Explanation: Allow must exist at EVERY level. OU-A has no Allow policy,
  so ALL actions are implicitly denied in OU-A and everything below it.
  THIS IS THE MOST COMMON MISTAKE when implementing allow-list strategy.

  Scenario 5: ALLOW-LIST AT OU (explicit Allow for S3 only)
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  ROOT    │    │  OU-A    │    │ Account-X │
  │ Allow: * │───▶│ Allow:s3 │───▶│ Allow: *  │  ✅ s3:* ALLOWED
  └──────────┘    └──────────┘    └──────────┘  ❌ ec2:* DENIED
  Explanation: Root allows everything. OU-A narrows it to S3 only.
  Account-X allows everything, but the intersection with OU-A is S3 only.

  Scenario 6: CONFLICTING ALLOW AND DENY AT SAME LEVEL
  ┌──────────┐    ┌──────────────────────┐    ┌──────────┐
  │  ROOT    │    │  OU-A                │    │ Account-X │
  │ Allow: * │───▶│ Allow: *             │───▶│ Allow: *  │  ❌ ec2:* DENIED
  │          │    │ Deny: ec2:RunInst... │    │           │
  └──────────┘    └──────────────────────┘    └──────────┘
  Explanation: Explicit Deny always wins over Allow at the same level.
  This is identical to standard IAM policy evaluation.

  Scenario 7: FULL CHAIN -- SCP + IAM POLICY
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌───────────────┐
  │  ROOT    │    │  OU-A    │    │ Account-X │    │ IAM Policy    │
  │ Allow: * │───▶│ Allow: * │───▶│ Allow: *  │───▶│ Allow: ec2:*  │
  │ Deny:ec2 │    │          │    │           │    │               │
  │ Term...  │    │          │    │           │    │               │
  └──────────┘    └──────────┘    └──────────┘    └───────────────┘
  Result: ❌ ec2:TerminateInstances DENIED by root SCP
          ✅ ec2:RunInstances ALLOWED (root Deny only covers TerminateInstances)
  The IAM policy grants ec2:*, but the SCP ceiling blocks TerminateInstances.
  All other ec2:* actions pass through. Effective permission = SCP ceiling ∩ IAM grant.
```

---

## Part 4: Deny-List vs Allow-List Strategy

### Deny-List Strategy (Recommended)

**The Analogy**: The corporate policy manual says "you may conduct any business activity **EXCEPT** the following banned activities." Subsidiaries have broad freedom. When the conglomerate expands into a new market (AWS launches a new service), subsidiaries can use it immediately without waiting for a policy update.

**The Technical Reality**:

- Keep `FullAWSAccess` attached at all levels (this is the default).
- Layer on explicit `Deny` statements for specific actions you want to block.
- **Advantages**: Low maintenance. New AWS services are automatically available. Each Deny SCP is independent and composable.
- **Disadvantages**: You must proactively identify what to block. If you miss something, it is permitted by default.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

### Allow-List Strategy

**The Analogy**: The corporate policy manual says "you may ONLY conduct the following pre-approved business activities. Everything else is forbidden." Subsidiaries have narrow freedom. When the conglomerate wants to expand into a new market, HQ must update the policy manual first.

**The Technical Reality**:

- Remove `FullAWSAccess` from the OU or account.
- Attach explicit `Allow` policies listing only the services you permit.
- **Advantages**: Maximum control. Only pre-approved services are accessible. Required in some regulated environments (ITAR, government).
- **Disadvantages**: High maintenance. Every new AWS service or feature requires a policy update. Easy to accidentally lock out critical services. The classic landmines:
  - **STS (`sts:*`)** -- forget this and every `AssumeRole` fails. Cross-account access, Lambda execution roles, ECS task roles, and IAM Identity Center all break. Error is a generic "Access Denied" with no hint an SCP is the cause.
  - **CloudWatch Logs (`logs:*`)** -- services keep running but stop emitting logs. You discover the gap days later during a production incident.
  - **Support (`support:*`)** -- nobody can file support tickets during an outage.
  - **Cost Explorer/Budgets (`ce:*`, `budgets:*`)** -- finance team loses billing visibility.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowApprovedServicesOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "rds:*",
        "lambda:*",
        "iam:*",
        "sts:*",
        "cloudwatch:*",
        "logs:*",
        "cloudtrail:*",
        "config:*",
        "kms:*",
        "sns:*",
        "sqs:*",
        "dynamodb:*",
        "elasticloadbalancing:*",
        "autoscaling:*",
        "ecr:*",
        "ecs:*",
        "eks:*",
        "tag:*",
        "support:*",
        "organizations:Describe*",
        "organizations:List*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Why AWS Recommends Deny-List

```
DENY-LIST vs ALLOW-LIST COMPARISON
=============================================================================

  DIMENSION              DENY-LIST                  ALLOW-LIST
  ──────────────────    ──────────────────────     ────────────────────────
  Default posture        Everything allowed          Everything denied
  Maintenance            Low (add Deny as needed)    High (add Allow for new
                                                     services, features)
  New AWS service        Available immediately        Blocked until you update
  Risk of lockout        Low                          High (forget STS or
                                                     CloudWatch and things
                                                     break silently)
  Compliance fit         General enterprise           ITAR, GovCloud, strict
                                                     regulatory environments
  Policy composability   Each Deny SCP is            Allow SCPs must be
                         independent                  coordinated (intersection)
  AWS recommendation     YES -- use this              Only when required by
                                                     regulation
```

---

## Part 5: SCP Inheritance Model -- How Policies Cascade

**The Analogy**: Picture the conglomerate's policy approval chain. The corporate charter sets company-wide rules. Each division adds its own rules. Each subsidiary may have additional local rules. For an employee to perform an activity:

1. The **corporate charter** (root) must not ban it
2. The **division policy** (OU SCP) must not ban it
3. The **subsidiary rules** (account SCP) must not ban it
4. The **employee's authorization** (IAM policy) must explicitly grant it

The key insight: policies *narrow* at each level -- they never widen. A division cannot override a corporate-wide ban. A subsidiary cannot override a division-level ban.

```
SCP INHERITANCE FLOW -- PERMISSIONS NARROW AT EACH LEVEL
=============================================================================

  ROOT SCPs (applies to ALL member accounts)
  ┌──────────────────────────────────────────────────────────────────────┐
  │  FullAWSAccess (Allow *)                                            │
  │  DenyRootUser (Deny * if principal is root)                         │
  │  DenyLeaveOrg (Deny organizations:LeaveOrganization)                │
  │  RestrictRegions (Deny if region not in [us-east-1, us-west-2])     │
  └──────────────────┬───────────────────────────────────┬──────────────┘
                     │                                   │
                     ▼                                   ▼
  WORKLOADS OU SCPs                         SANDBOX OU SCPs
  ┌─────────────────────────────┐          ┌─────────────────────────────┐
  │  FullAWSAccess (inherited   │          │  FullAWSAccess              │
  │  ceiling from root)         │          │  DenyExpensiveInstances     │
  │  DenySecurityChanges        │          │  DenyProductionServices     │
  │  EnforceEncryption          │          │  (more restrictive than     │
  └──────────┬──────────────────┘          │   Workloads OU)            │
             │                              └──────────────────────────────┘
             ▼
  PROD OU SCPs (sub-OU)
  ┌─────────────────────────────┐
  │  FullAWSAccess              │
  │  DenyUnencryptedUploads     │
  │  DenyPublicS3               │
  │  (tightest controls)        │
  └─────────────────────────────┘

  EFFECTIVE PERMISSIONS FOR AN ACCOUNT IN PROD OU:
  ════════════════════════════════════════════════

    Start:  Everything (FullAWSAccess at every level)

    Minus:  Root user actions           (root SCP)
    Minus:  Leave organization          (root SCP)
    Minus:  Non-approved regions        (root SCP)
    Minus:  Disable security services   (Workloads OU SCP)
    Minus:  Unencrypted storage         (Workloads OU SCP)
    Minus:  Unencrypted S3 uploads      (Prod OU SCP)
    Minus:  Public S3 buckets           (Prod OU SCP)

    Result: The intersection of ALL levels = the permission ceiling
            for any IAM identity in this account.
```

---

## Part 6: Real-World SCP Examples

### SCP 1: Deny Root User in Member Accounts

The root user in member accounts should never be used for daily operations. This SCP blocks all actions by the root user except for the few tasks that *only* root can perform.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

**Attach at**: Root (applies to all member accounts).

**Why**: The root user bypasses most IAM controls. If an attacker obtains root credentials for a member account, this SCP prevents them from doing anything -- even though the root user "has" full permissions. Remember, SCPs affect the root user in member accounts.

### SCP 2: Restrict API Calls to Approved Regions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "cur:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "kms:*",
        "mobileanalytics:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "route53-recovery-readiness:*",
        "route53-recovery-control-config:*",
        "s3:GetBucketLocation",
        "s3:ListAllMyBuckets",
        "shield:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*",
        "waf-regional:*",
        "waf:*",
        "wafv2:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        }
      }
    }
  ]
}
```

**Attach at**: Root (applies globally).

**Why `NotAction` instead of `Action`**: The `NotAction` element inverts the match. This SCP says "Deny everything EXCEPT the listed global services if the region is not approved." Global services (IAM, STS, Route 53, CloudFront, Organizations, Support) are excluded because they operate without a specific region or only from us-east-1. Without these exclusions, you would break IAM role assumption, billing access, and DNS management.

> **Why KMS is in the exclusion list**: KMS is technically a regional service, not a global one. It is excluded here because many services (S3, EBS, Lambda) make cross-region KMS calls to encrypt/decrypt data. If KMS were restricted, legitimate operations in approved regions could fail when they need a KMS key from another region. This is a deliberate trade-off — if your compliance requires KMS to be locked to specific regions, you will need a more nuanced SCP or separate KMS-specific controls.

### SCP 3: Prevent Disabling Security Services

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenySecurityServiceChanges",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "guardduty:DisassociateFromAdministratorAccount",
        "guardduty:UpdateDetector",
        "config:DeleteConfigurationRecorder",
        "config:DeleteDeliveryChannel",
        "config:StopConfigurationRecorder",
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail",
        "securityhub:DisableSecurityHub",
        "securityhub:DeleteInvitations",
        "securityhub:DisassociateFromMasterAccount",
        "securityhub:DisassociateFromAdministratorAccount",
        "access-analyzer:DeleteAnalyzer"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/OrganizationAccountAccessRole",
            "arn:aws:iam::*:role/BreakGlassRole"
          ]
        }
      }
    }
  ]
}
```

**Attach at**: Root or Workloads OU.

**The Condition trick**: The `StringNotLike` condition creates an escape hatch. The Deny applies to *everyone except* the `OrganizationAccountAccessRole` (management account cross-account role) and a `BreakGlassRole`. This ensures your security team can still manage these services via a designated role, but rogue IAM users or compromised roles cannot disable security tooling.

> **Note**: The `DisassociateFromMasterAccount` actions are deprecated in favor of `DisassociateFromAdministratorAccount` (AWS renamed "Master" to "Administrator"). Include both variants in production SCPs since either API name can be used to call the action.

### SCP 4: Enforce S3 Encryption

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedS3PutObject",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedS3PutObjectNoHeader",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "Null": {
          "s3:x-amz-server-side-encryption": "true"
        }
      }
    }
  ]
}
```

**Attach at**: Prod OU.

**Why two statements**: The first denies `PutObject` when encryption is specified but is not KMS. The second denies `PutObject` when the encryption header is entirely missing. Together, they enforce that every S3 object upload in production accounts uses KMS encryption.

> **Context (post-January 2023)**: Since January 2023, all new S3 objects are automatically encrypted with SSE-S3 by default. This SCP is not preventing *unencrypted* objects (that is impossible now) — it enforces that **KMS encryption specifically** is used instead of the default SSE-S3. This matters when your compliance requirements mandate customer-managed keys (CMKs) for audit and rotation control.

### SCP 5: Prevent Leaving the Organization

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeaveOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

**Attach at**: Root.

**Why**: If an attacker compromises a member account, they could call `LeaveOrganization` to escape all SCPs. Once the account leaves, there are no guardrails. This SCP prevents that escape.

### SCP Limits to Know

```
SCP QUOTAS AND LIMITS
=============================================================================

  LIMIT                                    VALUE
  ──────────────────────────────────      ──────────
  Maximum SCP document size                5,120 characters
  Maximum SCPs attached per OU/account     5
  Maximum SCPs per Organization            10,000 (adjustable via Service Quotas)
  Nesting depth for OUs                    5 levels (but 2 recommended)
  SCPs per root                            5

  TIPS FOR STAYING WITHIN 5,120 CHARACTERS:
  ├── Use wildcards: "ec2:*" instead of listing every EC2 action
  ├── Use NotAction instead of listing hundreds of actions to allow
  ├── Split large policies into multiple SCPs (up to 5 per target)
  ├── Minimize whitespace (AWS counts spaces and newlines)
  └── Use short Sid values
```

---

## Part 7: SCPs vs Permission Boundaries vs IAM Policies

This comparison is critical for interviews. The three mechanisms operate at different scopes and for different purposes:

```
THREE LAYERS OF PERMISSION CONTROL
=============================================================================

                    ┌─────────────────────────────────────┐
  ORGANIZATION      │  SERVICE CONTROL POLICIES (SCPs)     │
  LEVEL             │  Scope: entire account               │
                    │  Set by: Organization admin           │
                    │  Applies to: every IAM entity in the │
                    │  account (including root user)        │
                    │  Grants permissions: NO               │
                    │  Purpose: "What can this ACCOUNT do?" │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
  IAM LEVEL         │  PERMISSION BOUNDARIES               │
  (per-role/user)   │  Scope: individual IAM user or role  │
                    │  Set by: IAM admin in the account     │
                    │  Applies to: the specific entity      │
                    │  Grants permissions: NO               │
                    │  Purpose: "What can this ROLE do?"    │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
  IAM LEVEL         │  IAM POLICIES (identity-based)       │
  (per-role/user)   │  Scope: individual IAM user or role  │
                    │  Set by: IAM admin or role itself     │
                    │  Applies to: the specific entity      │
                    │  Grants permissions: YES              │
                    │  Purpose: "What is this ROLE granted?"│
                    └──────────────────────────────────────┘

  EFFECTIVE PERMISSIONS = SCP ∩ Permission Boundary ∩ IAM Policy

  Example:
  ├── SCP allows ec2:*, s3:*, rds:*
  ├── Permission Boundary allows ec2:*, s3:*
  ├── IAM Policy grants ec2:RunInstances, s3:PutObject, rds:CreateDBInstance
  │
  └── Effective:
      ├── ec2:RunInstances      ✅ (in SCP ∩ PB ∩ IAM)
      ├── s3:PutObject          ✅ (in SCP ∩ PB ∩ IAM)
      └── rds:CreateDBInstance  ❌ (in SCP ∩ IAM, but NOT in PB)

  WHEN TO USE EACH:
  ══════════════════════════════════════════════════════

  SCPs:
  ├── Organization-wide guardrails (deny root, restrict regions)
  ├── Coarse-grained: entire services or broad action categories
  ├── Cannot be overridden by anyone in member accounts
  └── Managed centrally by the Organization admin

  Permission Boundaries:
  ├── Delegation safety: let developers create IAM roles that cannot
  │   exceed a defined permission ceiling
  ├── Fine-grained: per-role or per-user
  ├── Managed by the IAM admin in each account
  └── Use case: "Developers can create Lambda execution roles, but
      those roles can never access production databases"

  IAM Policies:
  ├── The actual permission grants
  ├── Fine-grained: specific actions, resources, conditions
  ├── Managed by role owners or automation
  └── The only mechanism that GRANTS permissions
```

### Resource Control Policies (RCPs) -- The Complement to SCPs

> **Launched November 2024.** RCPs are a new Organization policy type that complements SCPs. While SCPs control what **principals** in member accounts can do, RCPs control what **resources** in member accounts can have done to them — even by external principals.

```
SCPs vs RCPs -- TWO SIDES OF THE SAME COIN
=============================================================================

  SCPs (Service Control Policies):
  ├── Control: what PRINCIPALS in member accounts can DO
  ├── Question answered: "Can this IAM role call s3:PutObject?"
  ├── Scope: restricts the calling entity
  └── Cannot stop external principals from accessing your resources

  RCPs (Resource Control Policies):
  ├── Control: what RESOURCES in member accounts can HAVE DONE TO THEM
  ├── Question answered: "Can this S3 bucket be accessed by anyone outside the org?"
  ├── Scope: restricts the resource itself
  └── Can block access even from principals outside the organization

  EXAMPLE: Preventing external access to S3 buckets
  ├── SCP approach: deny s3:PutBucketPolicy if it grants external access
  │   (only works for principals INSIDE the org)
  ├── RCP approach: deny any action on S3 buckets unless aws:PrincipalOrgID
  │   matches your org (blocks ALL external access, including from outside the org)
  └── RCPs close the gap SCPs cannot cover
```

> **Interview tip**: If asked "how do you prevent data exfiltration via S3 bucket policies granting access to external accounts?", RCPs are the modern answer. SCPs alone cannot prevent a resource-based policy from granting access to an external principal.

---

## Part 8: Cross-Account Access Patterns

### Pattern 1: AssumeRole (Identity-Based)

The most common cross-account access pattern. Account A creates a role that trusts Account B. Users/roles in Account B call `sts:AssumeRole` to obtain temporary credentials for Account A.

```
CROSS-ACCOUNT AssumeRole FLOW
=============================================================================

  ACCOUNT B (calling account)           ACCOUNT A (target account)
  ┌──────────────────────────┐         ┌──────────────────────────┐
  │                          │         │                          │
  │  Developer Role          │         │  CrossAccountRole        │
  │  ├── IAM Policy:         │  STS    │  ├── Trust Policy:       │
  │  │   Allow:              │ ──────▶ │  │   Principal:          │
  │  │   sts:AssumeRole on   │ Assume  │  │   arn:aws:iam::       │
  │  │   Account-A role ARN  │ Role    │  │   ACCOUNT-B:root      │
  │  │                       │         │  ├── IAM Policy:         │
  │  │                       │ ◀────── │  │   Allow:              │
  │  │                       │  Temp   │  │   s3:GetObject on     │
  │  │                       │  Creds  │  │   my-bucket/*         │
  │  │                       │         │  │                       │
  └──────────────────────────┘         └──────────────────────────┘

  SCP EVALUATION:
  ├── SCP in Account B: must allow sts:AssumeRole
  ├── SCP in Account A: must allow the actions the role grants
  │   (s3:GetObject in this example)
  ├── IAM policy in Account B: must allow sts:AssumeRole on the target role
  └── IAM policy on the target role in Account A: must grant the actions

  NOTE: The OrganizationAccountAccessRole created by Organizations uses
  this exact pattern. It trusts the management account and has
  AdministratorAccess.
```

### Pattern 2: Resource-Based Policies

Some AWS services support resource-based policies that grant cross-account access directly on the resource. S3 bucket policies, KMS key policies, SNS topic policies, and SQS queue policies all support this.

```
RESOURCE-BASED POLICY CROSS-ACCOUNT ACCESS
=============================================================================

  ACCOUNT B (calling account)           ACCOUNT A (resource account)
  ┌──────────────────────────┐         ┌──────────────────────────┐
  │                          │         │                          │
  │  Lambda Role             │ Direct  │  S3 Bucket Policy:       │
  │  ├── IAM Policy:         │ ──────▶ │  {                       │
  │  │   Allow:              │ API     │    "Principal": {        │
  │  │   s3:GetObject on     │ Call    │      "AWS": "arn:aws:iam │
  │  │   Account-A bucket    │         │      ::ACCT-B:role/      │
  │  │                       │         │      LambdaRole"         │
  │  │                       │         │    },                    │
  │  │                       │         │    "Action":"s3:Get*",   │
  │  │                       │         │    "Resource":"arn:../*" │
  │  │                       │         │  }                       │
  └──────────────────────────┘         └──────────────────────────┘

  KEY DIFFERENCE FROM AssumeRole:
  ├── No STS call needed -- the caller uses their own credentials directly
  ├── SCP in Account B (calling account) must allow the action
  ├── SCP in Account A does NOT affect this access -- resource-based
  │   policies are evaluated in the resource account, and SCPs only
  │   evaluate calls made BY principals in member accounts
  └── This is a subtle and important distinction for interviews

  IMPORTANT NUANCE:
  When a principal in Account B accesses a resource in Account A
  via a RESOURCE-BASED policy:
  ├── The SCP of Account B IS evaluated (it is the calling account)
  ├── The SCP of Account A is NOT evaluated (the resource-based policy
  │   grants access independently of the calling principal's account SCPs)
  └── EXCEPT: if the call is made using an AssumeRole-obtained credential
      from Account A, then Account A's SCP IS evaluated
```

### The `aws:PrincipalOrgID` Condition Key

A critical companion to SCPs for cross-account access. Use this condition key in **resource-based policies** to restrict access to only principals within your Organization — without listing individual account IDs.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::shared-artifacts/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-abc123def4"
        }
      }
    }
  ]
}
```

**Why this matters**: When you add a new account to the Organization, it automatically gets access — no policy update needed. When an account leaves, access is automatically revoked. This is heavily tested in interviews alongside SCPs and is the standard pattern for org-wide resource sharing.

---

## Part 9: Organizations Service Integrations

### How Organizations Powers Other Services

```
AWS ORGANIZATIONS SERVICE INTEGRATIONS
=============================================================================

  SERVICE                  INTEGRATION TYPE         WHAT IT ENABLES
  ──────────────────────  ────────────────────     ──────────────────────────
  CloudTrail               Trusted access           Organization trail: single
                                                    trail logs ALL accounts to
                                                    the Log Archive account

  AWS Config               Trusted + delegated      Organization rules: deploy
                           admin                    Config rules to all accounts
                                                    from the Security Tooling
                                                    account

  GuardDuty                Trusted + delegated      Centralized threat detection:
                           admin                    Security Tooling account sees
                                                    findings from all accounts

  Security Hub             Trusted + delegated      Aggregated security posture:
                           admin                    single pane of glass for all
                                                    accounts' compliance status

  IAM Access Analyzer      Trusted + delegated      Organization-level analyzer:
                           admin                    detects resources shared
                                                    outside the Organization

  AWS RAM                  Trusted access            Share resources (TGW, subnets,
                                                    License Manager configs)
                                                    across accounts in the org

  Service Catalog          Trusted + delegated      Portfolio sharing: approved
                           admin                    products available to all
                                                    accounts without per-account
                                                    setup

  IAM Identity Center      Trusted access           SSO across all Organization
  (formerly AWS SSO)                                accounts; permission sets
                                                    map to IAM roles per account

  AWS Backup               Trusted + delegated      Organization-wide backup
                           admin                    policies applied via OUs

  Tag Policies             Organization policy      Enforce consistent tagging
                                                    standards across all accounts

  Backup Policies          Organization policy      Centralized backup plans
                                                    applied to OUs

  AI Services Opt-Out      Organization policy      Opt out of AI service data
                                                    usage across all accounts
```

### Consolidated Billing

**The Analogy**: Instead of each subsidiary having its own accountant and paying its own bills, the corporate finance department at headquarters receives one consolidated invoice. This has two benefits: (1) simplified accounting, and (2) **volume discounts** -- the subsidiaries' combined spend qualifies for pricing tiers that no single subsidiary could reach alone.

**The Technical Reality**:

- All member accounts' usage rolls up to the management account's single bill.
- **Volume discounts**: S3 storage, EC2 data transfer, and other services have tiered pricing. Combined usage across all accounts qualifies for lower per-unit pricing.
- **Reserved Instance and Savings Plan sharing**: An RI purchased in Account A can apply to matching usage in Account B (unless you disable RI sharing). This is a significant cost optimization.
- **Cost allocation tags**: Tag resources across accounts, and the consolidated bill breaks down costs by tag.
- **AWS Budgets**: Set budgets per account, per OU, or across the Organization.

### Delegated Administrator Pattern

**The Analogy**: Instead of the CEO (management account) personally running every compliance audit, the CEO appoints the VP of Compliance (Security Tooling account) as the delegated authority for security operations. The VP has the power to manage GuardDuty, Security Hub, and Config across all subsidiaries, without the CEO being involved in day-to-day operations.

**The Technical Reality**:

- The management account designates a member account as the delegated administrator for a specific service.
- The delegated admin account can manage that service organization-wide (e.g., enable GuardDuty in all accounts, view Security Hub findings from all accounts).
- **Why not just use the management account?** Reducing the management account's surface area. Every IAM role, Lambda function, or human login in the management account is ungoverned by SCPs and has privileged access to Organization APIs. Pushing operational workload to a delegated admin keeps the management account pristine.
- A single account can be the delegated admin for multiple services.

---

## Part 10: Terraform Examples

### Creating the Organization and OU Structure

```hcl
# ──────────────────────────────────────────────────────────
# AWS Organization -- The Corporate Conglomerate
# ──────────────────────────────────────────────────────────

resource "aws_organizations_organization" "main" {
  # Enable ALL features (SCPs, tag policies, backup policies, etc.)
  # The alternative is "CONSOLIDATED_BILLING" which only enables billing
  feature_set = "ALL"

  # Enable trusted access for services that need org-wide visibility
  aws_service_access_principals = [
    "cloudtrail.amazonaws.com",
    "config.amazonaws.com",
    "guardduty.amazonaws.com",
    "securityhub.amazonaws.com",
    "ram.amazonaws.com",
    "sso.amazonaws.com",
    "access-analyzer.amazonaws.com",
    "backup.amazonaws.com",
    "tagpolicies.tag.amazonaws.com",
  ]

  # Enable the policy types you need
  enabled_policy_types = [
    "SERVICE_CONTROL_POLICY",
    "TAG_POLICY",
    "BACKUP_POLICY",
  ]
}

# ──────────────────────────────────────────────────────────
# Organizational Units -- The Corporate Divisions
# ──────────────────────────────────────────────────────────

# Security OU (directly under root)
resource "aws_organizations_organizational_unit" "security" {
  name      = "Security"
  parent_id = aws_organizations_organization.main.roots[0].id
}

# Infrastructure OU (directly under root)
resource "aws_organizations_organizational_unit" "infrastructure" {
  name      = "Infrastructure"
  parent_id = aws_organizations_organization.main.roots[0].id
}

# Workloads OU (directly under root)
resource "aws_organizations_organizational_unit" "workloads" {
  name      = "Workloads"
  parent_id = aws_organizations_organization.main.roots[0].id
}

# Prod sub-OU (under Workloads)
resource "aws_organizations_organizational_unit" "prod" {
  name      = "Prod"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

# Non-Prod sub-OU (under Workloads)
resource "aws_organizations_organizational_unit" "non_prod" {
  name      = "Non-Prod"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

# Sandbox OU (directly under root)
resource "aws_organizations_organizational_unit" "sandbox" {
  name      = "Sandbox"
  parent_id = aws_organizations_organization.main.roots[0].id
}

# Policy Staging OU (directly under root)
resource "aws_organizations_organizational_unit" "policy_staging" {
  name      = "PolicyStaging"
  parent_id = aws_organizations_organization.main.roots[0].id
}

# Suspended OU (directly under root)
resource "aws_organizations_organizational_unit" "suspended" {
  name      = "Suspended"
  parent_id = aws_organizations_organization.main.roots[0].id
}
```

### Creating and Attaching SCPs

```hcl
# ──────────────────────────────────────────────────────────
# SCP: Deny Root User -- Attached at Root
# ──────────────────────────────────────────────────────────

resource "aws_organizations_policy" "deny_root_user" {
  name        = "DenyRootUserActions"
  description = "Deny all actions by root user in member accounts"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyRootUser"
        Effect    = "Deny"
        Action    = "*"
        Resource  = "*"
        Condition = {
          StringLike = {
            "aws:PrincipalArn" = "arn:aws:iam::*:root"
          }
        }
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "deny_root_user_at_root" {
  policy_id = aws_organizations_policy.deny_root_user.id
  target_id = aws_organizations_organization.main.roots[0].id
}

# ──────────────────────────────────────────────────────────
# SCP: Restrict Regions -- Attached at Root
# ──────────────────────────────────────────────────────────

resource "aws_organizations_policy" "restrict_regions" {
  name        = "RestrictRegions"
  description = "Deny API calls in non-approved regions, excluding global services"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyNonApprovedRegions"
        Effect    = "Deny"
        NotAction = [
          "a4b:*",
          "budgets:*",
          "ce:*",
          "chime:*",
          "cloudfront:*",
          "cur:*",
          "globalaccelerator:*",
          "health:*",
          "iam:*",
          "importexport:*",
          "kms:*",
          "mobileanalytics:*",
          "organizations:*",
          "pricing:*",
          "route53:*",
          "route53domains:*",
          "route53-recovery-readiness:*",
          "route53-recovery-control-config:*",
          "s3:GetBucketLocation",
          "s3:ListAllMyBuckets",
          "shield:*",
          "sts:*",
          "support:*",
          "trustedadvisor:*",
          "waf-regional:*",
          "waf:*",
          "wafv2:*",
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = [
              "us-east-1",
              "us-west-2",
            ]
          }
        }
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "restrict_regions_at_root" {
  policy_id = aws_organizations_policy.restrict_regions.id
  target_id = aws_organizations_organization.main.roots[0].id
}

# ──────────────────────────────────────────────────────────
# SCP: Deny Leaving Organization -- Attached at Root
# ──────────────────────────────────────────────────────────

resource "aws_organizations_policy" "deny_leave_org" {
  name        = "DenyLeaveOrganization"
  description = "Prevent member accounts from leaving the Organization"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "DenyLeaveOrg"
        Effect   = "Deny"
        Action   = "organizations:LeaveOrganization"
        Resource = "*"
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "deny_leave_org_at_root" {
  policy_id = aws_organizations_policy.deny_leave_org.id
  target_id = aws_organizations_organization.main.roots[0].id
}

# ──────────────────────────────────────────────────────────
# SCP: Deny Security Service Changes -- Attached at Workloads OU
# ──────────────────────────────────────────────────────────

resource "aws_organizations_policy" "deny_security_changes" {
  name        = "DenySecurityServiceChanges"
  description = "Prevent disabling GuardDuty, Config, CloudTrail, Security Hub"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenySecurityChanges"
        Effect = "Deny"
        Action = [
          "guardduty:DeleteDetector",
          "guardduty:DisassociateFromMasterAccount",
          "guardduty:UpdateDetector",
          "config:DeleteConfigurationRecorder",
          "config:DeleteDeliveryChannel",
          "config:StopConfigurationRecorder",
          "cloudtrail:DeleteTrail",
          "cloudtrail:StopLogging",
          "cloudtrail:UpdateTrail",
          "securityhub:DisableSecurityHub",
          "securityhub:DeleteInvitations",
          "securityhub:DisassociateFromMasterAccount",
          "access-analyzer:DeleteAnalyzer",
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" = [
              "arn:aws:iam::*:role/OrganizationAccountAccessRole",
              "arn:aws:iam::*:role/BreakGlassRole",
            ]
          }
        }
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "deny_security_changes_workloads" {
  policy_id = aws_organizations_policy.deny_security_changes.id
  target_id = aws_organizations_organizational_unit.workloads.id
}

# ──────────────────────────────────────────────────────────
# SCP: Enforce S3 Encryption -- Attached at Prod OU
# ──────────────────────────────────────────────────────────

resource "aws_organizations_policy" "enforce_s3_encryption" {
  name        = "EnforceS3Encryption"
  description = "Deny S3 PutObject without KMS encryption in production"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "DenyUnencryptedS3Uploads"
        Effect   = "Deny"
        Action   = "s3:PutObject"
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      },
      {
        Sid      = "DenyS3UploadsWithoutEncryptionHeader"
        Effect   = "Deny"
        Action   = "s3:PutObject"
        Resource = "*"
        Condition = {
          Null = {
            "s3:x-amz-server-side-encryption" = "true"
          }
        }
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "enforce_encryption_prod" {
  policy_id = aws_organizations_policy.enforce_s3_encryption.id
  target_id = aws_organizations_organizational_unit.prod.id
}

# ──────────────────────────────────────────────────────────
# SCP: Deny All -- Attached at Suspended OU
# ──────────────────────────────────────────────────────────

resource "aws_organizations_policy" "deny_all" {
  name        = "DenyAllActions"
  description = "Block all API calls for accounts in the Suspended OU"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "DenyEverything"
        Effect   = "Deny"
        Action   = "*"
        Resource = "*"
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "deny_all_suspended" {
  policy_id = aws_organizations_policy.deny_all.id
  target_id = aws_organizations_organizational_unit.suspended.id
}
```

### Creating Member Accounts

```hcl
# ──────────────────────────────────────────────────────────
# Member Accounts -- The Subsidiaries
# ──────────────────────────────────────────────────────────

resource "aws_organizations_account" "log_archive" {
  name      = "Log Archive"
  email     = "aws+log-archive@example.com"
  parent_id = aws_organizations_organizational_unit.security.id

  # Role name for cross-account access from management account
  role_name = "OrganizationAccountAccessRole"

  # Prevent Terraform from trying to close the account on destroy
  close_on_deletion = false

  # Ignore changes to role_name after creation (it cannot be changed)
  lifecycle {
    ignore_changes = [role_name]
  }

  tags = {
    Environment = "security"
    ManagedBy   = "terraform"
  }
}

resource "aws_organizations_account" "security_tooling" {
  name      = "Security Tooling"
  email     = "aws+security-tooling@example.com"
  parent_id = aws_organizations_organizational_unit.security.id
  role_name = "OrganizationAccountAccessRole"

  close_on_deletion = false
  lifecycle { ignore_changes = [role_name] }

  tags = {
    Environment = "security"
    ManagedBy   = "terraform"
  }
}

resource "aws_organizations_account" "networking" {
  name      = "Networking"
  email     = "aws+networking@example.com"
  parent_id = aws_organizations_organizational_unit.infrastructure.id
  role_name = "OrganizationAccountAccessRole"

  close_on_deletion = false
  lifecycle { ignore_changes = [role_name] }

  tags = {
    Environment = "infrastructure"
    ManagedBy   = "terraform"
  }
}

# ──────────────────────────────────────────────────────────
# Delegated Administrator -- Push GuardDuty admin to
# the Security Tooling account
# ──────────────────────────────────────────────────────────

resource "aws_organizations_delegated_administrator" "guardduty" {
  account_id        = aws_organizations_account.security_tooling.id
  service_principal = "guardduty.amazonaws.com"
}

resource "aws_organizations_delegated_administrator" "securityhub" {
  account_id        = aws_organizations_account.security_tooling.id
  service_principal = "securityhub.amazonaws.com"
}

resource "aws_organizations_delegated_administrator" "config" {
  account_id        = aws_organizations_account.security_tooling.id
  service_principal = "config.amazonaws.com"
}
```

### Organization-Wide CloudTrail

```hcl
# ──────────────────────────────────────────────────────────
# Organization Trail -- Single trail logs ALL accounts
# Run this in the management account (or via delegated admin)
# ──────────────────────────────────────────────────────────

resource "aws_cloudtrail" "organization" {
  name                       = "organization-trail"
  s3_bucket_name             = "log-archive-org-cloudtrail"  # Bucket in Log Archive account
  is_organization_trail      = true
  is_multi_region_trail      = true
  enable_log_file_validation = true

  # Log data events for S3 and Lambda (optional, adds cost)
  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3"]
    }
  }

  tags = {
    ManagedBy = "terraform"
    Purpose   = "organization-audit-trail"
  }
}

# The S3 bucket in the Log Archive account needs a bucket policy
# that allows CloudTrail from the Organization to write to it.
# This bucket policy is deployed in the Log Archive account.
```

---

## Part 11: SCP Best Practices -- Operational Wisdom

```
SCP OPERATIONAL BEST PRACTICES
=============================================================================

  1. ALWAYS TEST IN POLICY STAGING OU FIRST
     ├── Move a test account to the Policy Staging OU
     ├── Attach the new SCP
     ├── Run through your critical workflows (deploy, monitor, debug)
     ├── Use IAM Policy Simulator and Access Analyzer to validate
     └── Only after validation, attach to the target OU

  2. USE THE BREAK-GLASS PATTERN
     ├── Every Deny SCP should have a Condition that exempts a
     │   BreakGlassRole (or OrganizationAccountAccessRole)
     ├── Lock the BreakGlassRole credentials in a vault
     ├── Only use it when an SCP accidentally blocks critical operations
     └── Audit every use of the break-glass role

  3. PREFER DENY OVER RESTRICTIVE ALLOW
     ├── Each Deny SCP works independently
     ├── Adding a new Deny does not affect existing Deny SCPs
     ├── Allow SCPs interact via intersection -- adding a new Allow
     │   at one level can break permissions at another level
     └── Deny is composable; Allow is fragile

  4. MONITOR SCP CHANGES VIA CLOUDTRAIL
     ├── CloudTrail logs CreatePolicy, UpdatePolicy, AttachPolicy,
     │   DetachPolicy events
     ├── Set CloudWatch alarms on these events
     └── Any SCP change should trigger a security notification

  5. DOCUMENT SCP INTENT
     ├── Use meaningful Sid values that describe the business purpose
     ├── Use the Description field on aws_organizations_policy
     ├── Maintain a policy registry that maps SCPs to OUs with rationale
     └── SCPs are governance-as-code -- treat them like production code

  6. ACCOUNT FOR SERVICE-LINKED ROLES
     ├── SCPs do NOT affect service-linked roles
     ├── This means: even if you SCP-deny a service, the service-linked
     │   role can still perform its actions
     ├── Example: An SCP denying all EC2 actions does NOT prevent
     │   Auto Scaling's service-linked role from launching instances
     └── Design your SCPs knowing this exception exists

  7. WATCH THE CHARACTER LIMIT
     ├── 5,120 characters per SCP (including whitespace)
     ├── Use Terraform's jsonencode() -- it produces compact JSON
     ├── Split large policies across multiple SCPs (up to 5 per target)
     └── If you need more than 5 SCPs per OU, rethink your OU structure
```

---

## Key Takeaways

### Organization Structure
1. **The management account is exempt from SCPs and should run zero workloads** -- it is the most privileged account in your Organization; protect it like the crown jewels and push all operational work to delegated administrator accounts
2. **Organize OUs by function and compliance requirements, not by team or business unit** -- Security OU, Infrastructure OU, Workloads OU (with Prod/Non-Prod sub-OUs), Sandbox OU, Suspended OU; this structure enables precise SCP targeting
3. **Maximum two levels of OU nesting** -- root > OU > sub-OU; deeper nesting makes SCP inheritance exponentially harder to debug and is explicitly discouraged by AWS

### SCP Mechanics
4. **SCPs are permission ceilings, not permission grants** -- an SCP can never give an IAM user the ability to do something; it can only take away the ability; effective permissions = SCP ceiling intersected with IAM policy grant
5. **Allow is an intersection across all levels; Deny is a union (inherited)** -- an Allow must exist at root, OU, and account for the action to be permitted; a Deny at any level blocks the entire subtree; this is the most tested Organizations concept in interviews
6. **Use the deny-list strategy unless regulation requires allow-list** -- keep FullAWSAccess at all levels and layer on explicit Deny SCPs; this is lower maintenance, composable, and does not break when AWS launches new services

### Real-World SCP Patterns
7. **Four SCPs every Organization should have at root**: deny root user actions, restrict to approved regions (using NotAction to exclude global services), prevent leaving the Organization, and prevent disabling security services
8. **Always include a break-glass escape hatch in Deny SCPs** -- use `StringNotLike` on `aws:PrincipalArn` to exempt a locked-down BreakGlassRole; without this, an overly broad SCP can lock you out with no recovery path
9. **Test SCPs in a Policy Staging OU before attaching to production** -- an incorrect SCP can instantly break every workload in every account under that OU; validate with IAM Policy Simulator and a sacrificial test account

### Cross-Account and Integration
10. **The OrganizationAccountAccessRole is your initial cross-account bridge** -- created automatically in accounts provisioned by Organizations; trusts the management account with AdministratorAccess; replace it with least-privilege roles as you mature
11. **Delegated administrator keeps the management account clean** -- designate the Security Tooling account as delegated admin for GuardDuty, Security Hub, Config, and Access Analyzer; the management account should never run security automation directly
12. **Consolidated billing enables volume discounts and RI sharing** -- all member accounts' usage aggregates under one bill; Reserved Instances and Savings Plans purchased in one account can apply to matching usage in any other account

### SCPs vs Other Mechanisms
13. **SCPs for account-level ceilings, Permission Boundaries for role-level ceilings, IAM Policies for grants** -- use SCPs for broad organizational guardrails (deny regions, deny services), permission boundaries for delegation safety (developers creating roles that cannot exceed a ceiling), and IAM policies for the actual permissions that enable work
14. **SCPs do not affect service-linked roles** -- even if you SCP-deny EC2 in an account, Auto Scaling's service-linked role can still launch instances; design guardrails knowing this exception exists
15. **Resource-based policies bypass the target account's SCP** -- when Account B accesses Account A's S3 bucket via a bucket policy, Account A's SCP is not evaluated on that access (Account B's SCP still is); this matters for cross-account data access patterns

---

## Further Reading

- [What is AWS Organizations? -- AWS Docs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html)
- [Organizing Your AWS Environment Using Multiple Accounts -- AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)
- [SCP Evaluation -- AWS Official Docs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_evaluation.html)
- [How to Use SCPs to Set Permission Guardrails -- AWS Security Blog](https://aws.amazon.com/blogs/security/how-to-use-service-control-policies-to-set-permission-guardrails-across-accounts-in-your-aws-organization/)
- [Best Practices for OUs with AWS Organizations -- AWS Blog](https://aws.amazon.com/blogs/mt/best-practices-for-organizational-units-with-aws-organizations/)
- [SCP Examples Repository -- AWS Samples (GitHub)](https://github.com/aws-samples/service-control-policy-examples)
- [Best Practices for SCPs in a Multi-Account Environment -- AWS Industries Blog](https://aws.amazon.com/blogs/industries/best-practices-for-aws-organizations-service-control-policies-in-a-multi-account-environment/)
- [Transit Gateway Deep Dive (Feb 23)](./2026-02-23-transit-gateway-deep-dive.md) -- RAM sharing and multi-account TGW patterns referenced in this doc

---

*Written on March 2, 2026*
