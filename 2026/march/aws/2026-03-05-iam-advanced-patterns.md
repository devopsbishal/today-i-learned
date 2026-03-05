# IAM Advanced Patterns -- The Airport Security Analogy

> You have set up the airline alliance (Organizations) and established the centralized airport authority (Control Tower). Now it is time to understand the **security checkpoints every traveler must pass through** -- the layered system that determines exactly what any person, contractor, or automated system is allowed to do inside the terminal. IAM advanced patterns are the airport security of AWS: multiple independent checkpoints where every checkpoint can only restrict -- never expand -- what the previous checkpoint allowed.

---

## TL;DR

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| 8 IAM policy types | Eight independent security checkpoints at an airport | Every API request is evaluated against up to eight policy types; any explicit Deny wins instantly, and the effective permission is the intersection of all applicable layers |
| Policy evaluation logic | The checkpoint flowchart posted at every security station | A deterministic decision tree: explicit Deny wins, then each policy type is checked in order; same-account and cross-account evaluation follow different flowcharts |
| Permission Boundary | The restricted-access zone printed on your badge | A guardrail attached to an IAM user or role that sets the *maximum* permissions it can ever have; the effective permission is the intersection of the identity policy and the boundary |
| Session Policy | The day pass issued at the visitor desk | An optional inline policy passed during AssumeRole or federation that further restricts the session beyond what the role's identity policy and boundary allow |
| Cross-account role (AssumeRole) | A visitor badge issued by Building B so an employee from Building A can enter | Account B creates a role with a trust policy naming Account A as a trusted principal; Account A's user calls sts:AssumeRole to receive temporary credentials scoped to that role |
| Trust policy | The visitor approval form at Building B's front desk | A resource-based policy on the IAM role that specifies *who* can assume it; the trust policy lives in the target account, not the source account |
| External ID | The pre-shared confirmation code for third-party visitors | A condition in the trust policy that prevents the confused deputy problem; the assuming principal must present the correct ExternalId or the AssumeRole call fails |
| SAML 2.0 Federation | A corporate badge system linked to a government-issued ID | Your corporate IdP (Okta, Azure AD) authenticates users and sends a SAML assertion to AWS STS, which exchanges it for temporary credentials mapped to a specific IAM role |
| OIDC Federation | A digital ID card verified by a trusted third party | A web identity provider (GitHub, Google, your EKS cluster) issues a JWT token that STS validates and exchanges for temporary AWS credentials -- no long-lived keys needed |
| IAM Identity Center (SSO) | A single reception desk that manages badges for every building in the complex | A centralized service that replaces manual cross-account role creation; you define permission sets (badge templates), assign them to users/groups, and Identity Center auto-creates the roles in each target account |
| Permission Set | A badge template at the reception desk | A collection of IAM policies defined once in Identity Center that gets stamped into an IAM role in every account where it is assigned |

---

## The Big Picture

In the Organizations and SCPs doc, you learned that SCPs and RCPs set the **account-level ceiling** -- the maximum permissions any IAM principal or resource in the account can have. In the Control Tower doc, you saw how preventive controls (SCPs) are deployed automatically across OUs. But neither doc answered the deeper question: *once a request passes the SCP/RCP check, what happens next?*

The answer is the IAM policy evaluation engine -- a deterministic flowchart that evaluates up to **eight distinct policy types** for every single API request. Understanding this engine is the difference between writing IAM policies that happen to work and writing IAM policies that you *know* are correct.

Think of it this way. The airport authority (Organizations) maintains **eight independent security checkpoints**. They are not a sequential pipeline where you pass through checkpoint 1, then 2, then 3. Instead, the system checks ALL of them and combines the results:

1. **SCPs** (approved airline list) -- does this account's OU allow this action at all? Works as a deny-list (block specific actions) or allow-list (only permit specific actions). You covered both strategies on Mar 2.
2. **RCPs** (Resource Control Policies) -- the complement to SCPs you learned about on Mar 2. SCPs restrict *principals*, RCPs restrict *resources*. Does this resource allow this action from this type of principal?
3. **Resource-based policies** (the gate's own boarding list) -- the specific gate (S3 bucket, KMS key, SQS queue) may have its own rules about who boards. **Unique among restrictive layers: resource-based policies can actively grant access**, and in same-account scenarios, they can grant access even without an identity policy.
4. **Identity-based policies** (your boarding pass) -- do you have a valid ticket (permission grant) for this flight (API action)?
5. **Permission boundaries** (restricted-access badge) -- even if you have a ticket, your badge limits which terminals you can access.
6. **Session policies** (day pass restrictions) -- if you are a visitor on a temporary badge, the day pass narrows your access further.
7. **VPC endpoint policies** (toll booth on a private road) -- if your request travels through a VPC endpoint (private road) instead of the public internet, the toll booth can restrict which principals perform which actions on which services. Traffic on the public internet never hits this checkpoint.
8. **ACLs** (legacy physical keys) -- an older, coarser access mechanism for S3 objects, left over from before the modern policy system. AWS recommends disabling them (BucketOwnerEnforced).

The critical insight: **most of these layers can only restrict, never expand, what identity-based policies grant.** A generous identity policy cannot override a restrictive permission boundary. A permissive boundary cannot override an SCP Deny. The effective permission is the intersection of all applicable restrictive layers. The one exception is **resource-based policies**, which can actively grant access in same-account scenarios (covered in the flowcharts below).

> **Important**: The numbered list above is a conceptual overview, NOT the evaluation order. AWS does not evaluate policies as a sequential pipeline. The actual evaluation logic is: (1) check ALL policy types for any explicit Deny, (2) then check each policy type for Allow in a specific order. The flowcharts in Part 1 show the real evaluation logic.

```
THE EIGHT POLICY TYPES -- CONCEPTUAL OVERVIEW (NOT evaluation order)
=============================================================================

  ⚠️  This diagram shows what each policy type does. It is NOT the
      evaluation sequence. See the flowcharts in Part 1 for actual
      evaluation logic. The real flow is:
      1. Check ALL types for explicit Deny (Deny always wins)
      2. Check each type for Allow in the order shown in Part 1

  ┌─────────────────────────────────────────────────────────────┐
  │  1. SCPs (Organization-level ceiling for PRINCIPALS)       │
  │     "Is this action permitted for principals in this acct?"│
  │     Checked for: All principals in member accounts         │
  │     Grants permissions? NO -- only restricts               │
  │     You learned these on Mar 2.                            │
  └─────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────┐
  │  2. RCPs (Organization-level ceiling for RESOURCES)        │
  │     "Is this action permitted on resources in this acct?"  │
  │     Checked for: Resources in member accounts              │
  │     Grants permissions? NO -- only restricts               │
  │     Complement to SCPs. You learned these on Mar 2.        │
  └─────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────┐
  │  3. Resource-based policies (on the target resource)       │
  │     "Does this S3 bucket / KMS key / SQS queue explicitly  │
  │      grant access to this principal?"                      │
  │     Checked for: Resources with attached policies          │
  │     Grants permissions? YES -- can grant same-account      │
  │     access even without identity policy (see Part 1)       │
  └─────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────┐
  │  4. Identity-based policies (on the calling principal)     │
  │     "Does this user/role have an attached policy that       │
  │      allows this action on this resource?"                 │
  │     Checked for: Every request from IAM users/roles        │
  │     Grants permissions? YES -- this is the primary grant   │
  └─────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────┐
  │  5. Permission boundaries (on the calling principal)       │
  │     "Is this action within the boundary attached to this   │
  │      user/role?"                                           │
  │     Checked for: Principals that have a boundary attached  │
  │     Grants permissions? NO -- only restricts               │
  │     Effective = identity policy ∩ boundary                 │
  └─────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────┐
  │  6. Session policies (inline during AssumeRole/Federation) │
  │     "Was this session further restricted when it was        │
  │      created?"                                             │
  │     Checked for: Assumed role sessions, federated sessions │
  │     Grants permissions? NO -- only restricts               │
  └─────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────┐
  │  7. VPC endpoint policies                                  │
  │     "Does the VPC endpoint through which this request       │
  │      arrived allow this action?"                           │
  │     Checked for: Requests routed through a VPC endpoint    │
  │     Grants permissions? NO -- only restricts               │
  └─────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────┐
  │  8. ACLs (Access Control Lists)                            │
  │     "Does the S3 ACL grant access?"                        │
  │     Checked for: S3 objects (legacy)                       │
  │     Grants permissions? YES -- but legacy, avoid using     │
  │     AWS recommends disabling ACLs (BucketOwnerEnforced)    │
  └─────────────────────────────────────────────────────────────┘

  EFFECTIVE PERMISSION =
    At least one policy must grant Allow
    AND no policy has an explicit Deny
    AND all applicable restrictive layers (SCP, RCP, boundary,
        session policy, VPC endpoint policy) must Allow
```

---

## Part 1: Policy Evaluation Logic -- The Flowchart You Must Memorize

### Same-Account Evaluation

When a principal makes a request within its own account, the evaluation follows this decision tree:

```
SAME-ACCOUNT POLICY EVALUATION FLOWCHART
=============================================================================

  API Request arrives
       │
       ▼
  ┌──────────────────────────┐
  │ Is there an EXPLICIT     │──── YES ───▶ ❌ DENIED
  │ DENY in ANY policy type? │              (Deny always wins.
  └──────────┬───────────────┘               No override possible.)
             │ NO
             ▼
  ┌──────────────────────────┐
  │ Is there an SCP?         │──── YES ───▶ Does SCP Allow? ─── NO ──▶ ❌ DENIED
  │ (account in an org)      │                    │                   (implicit deny)
  └──────────┬───────────────┘                YES │
             │ NO (standalone account)            │
             ▼◄───────────────────────────────────┘
  ┌──────────────────────────┐
  │ Is there an RCP?         │──── YES ───▶ Does RCP Allow? ─── NO ──▶ ❌ DENIED
  │ (account in an org)      │                    │                   (implicit deny)
  └──────────┬───────────────┘                YES │
             │ NO (standalone account)            │
             ▼◄───────────────────────────────────┘
  ┌──────────────────────────┐
  │ Is there a RESOURCE-     │──── YES ───▶ Does it Allow    ─── YES ──▶ ✅ ALLOWED *
  │ BASED policy on the      │              this principal?
  │ target resource?         │                    │
  └──────────┬───────────────┘                 NO │    * IMPORTANT NUANCE:
             │ NO                                 │      Resource-based policy
             ▼◄───────────────────────────────────┘      can grant same-account
  ┌──────────────────────────┐                           access WITHOUT identity
  │ Does the principal have  │──── NO ────▶ ❌ DENIED    policy IF the policy
  │ an IDENTITY-BASED policy │             (implicit      names an IAM user ARN
  │ that Allows the action?  │              deny)         or role SESSION ARN.
  └──────────┬───────────────┘                           If it names a role ARN
             │ YES                                        (not session), identity
             ▼                                            policy is still needed.
  ┌──────────────────────────┐
  │ Does the principal have  │──── YES ───▶ Does boundary     ─── NO ──▶ ❌ DENIED
  │ a PERMISSION BOUNDARY?   │              Allow the action?
  └──────────┬───────────────┘                    │
             │ NO                              YES │
             ▼◄───────────────────────────────────┘
  ┌──────────────────────────┐
  │ Is this a SESSION        │──── YES ───▶ Does session      ─── NO ──▶ ❌ DENIED
  │ (assumed role /          │              policy Allow?
  │  federated user)?        │                    │
  └──────────┬───────────────┘                 YES │
             │ NO                                  │
             ▼◄────────────────────────────────────┘
         ✅ ALLOWED
```

### Cross-Account Evaluation -- The Critical Difference

Cross-account access is evaluated in **both** accounts. The calling account checks whether the principal is allowed to call `sts:AssumeRole` (or the equivalent). The target account checks whether the role's trust policy allows the principal. The key difference from same-account:

> **In cross-account requests, both sides must independently grant access.** The source account's identity policy must Allow the action, AND the target account's resource-based policy (or trust policy for AssumeRole) must also Allow. Unlike same-account evaluation, a resource-based policy alone is NOT sufficient for cross-account -- the source account must also grant permission. This applies universally across services, including S3.

```
CROSS-ACCOUNT EVALUATION -- BOTH SIDES MUST AGREE
=============================================================================

  ACCOUNT A (Source)                    ACCOUNT B (Target)
  ┌─────────────────────────┐          ┌─────────────────────────┐
  │                         │          │                         │
  │  IAM User/Role "Alice"  │          │  IAM Role "CrossRole"   │
  │                         │          │                         │
  │  Identity Policy:       │  ─STS──▶ │  Trust Policy:          │
  │  {                      │ Assume   │  {                      │
  │    Allow:               │  Role    │    Allow:               │
  │    sts:AssumeRole on    │          │    sts:AssumeRole       │
  │    arn:...:CrossRole    │          │    Principal:           │
  │  }                      │          │    Account A or Alice   │
  │                         │          │  }                      │
  │  SCP: Must Allow sts:*  │          │  SCP: Must Allow the    │
  │                         │          │       actions CrossRole │
  │                         │          │       will perform      │
  └─────────────────────────┘          └─────────────────────────┘
           │                                     │
           │  BOTH must say YES                  │
           │  for access to work                 │
           ▼                                     ▼
       SOURCE CHECK:                        TARGET CHECK:
       1. SCP allows sts:AssumeRole?        1. Trust policy allows principal?
       2. Identity policy allows            2. SCP allows the actions?
          sts:AssumeRole on this ARN?       3. Identity policy on role
       3. Permission boundary allows           allows the actions?
          sts:AssumeRole? (if set)
       4. Session policy allows?
          (if already in a session)
```

---

## Part 2: Permission Boundaries -- The Restricted-Access Badge

### The Analogy

Imagine an airport where every employee has a **security clearance badge**. The badge defines the *maximum areas* the employee can ever access, regardless of what additional access passes they are given. A ground crew member's badge might allow access to the tarmac and baggage areas. Even if their manager gives them a day pass to the executive lounge (identity policy), the badge does not include the lounge -- so the pass is useless. The effective access is always the **intersection** of the badge (boundary) and the pass (identity policy).

Now, why do badges exist separately from passes? Because the **airport authority** (central security team) prints the badges, while **individual airlines** (development teams) distribute the passes. The airport authority does not want to be involved in every pass decision -- they just want to guarantee that no pass can ever exceed the badge. This is the delegation use case.

### The Technical Reality

A permission boundary is a **managed IAM policy** that you attach to an IAM user or role. It does not grant any permissions. It sets the ceiling for what identity-based policies can grant.

```
PERMISSION BOUNDARY -- EFFECTIVE PERMISSIONS
=============================================================================

  IDENTITY-BASED POLICY              PERMISSION BOUNDARY
  (what you are granted)             (maximum you CAN be granted)
  ┌───────────────────────┐          ┌───────────────────────┐
  │                       │          │                       │
  │  s3:*                 │          │  s3:GetObject         │
  │  ec2:*                │          │  s3:PutObject         │
  │  iam:*                │          │  s3:ListBucket        │
  │  lambda:*             │          │  ec2:Describe*        │
  │                       │          │  lambda:*             │
  │                       │          │  logs:*               │
  │                       │          │  cloudwatch:*         │
  └───────────────────────┘          └───────────────────────┘

  EFFECTIVE PERMISSIONS = INTERSECTION
  ┌───────────────────────┐
  │  s3:GetObject         │  ← s3:* ∩ s3:{Get,Put,List} = only these three
  │  s3:PutObject         │
  │  s3:ListBucket        │
  │  ec2:Describe*        │  ← ec2:* ∩ ec2:Describe* = Describe only
  │  lambda:*             │  ← lambda:* ∩ lambda:* = full lambda
  │                       │
  │  iam:* -- BLOCKED     │  ← not in boundary, silently denied
  │  ec2:Run* -- BLOCKED  │  ← ec2:* requested but boundary only has Describe
  └───────────────────────┘
```

### The Delegation Use Case

This is the primary reason permission boundaries exist. The scenario: your central IAM team is a bottleneck. Every time a developer needs a new Lambda execution role, they file a ticket and wait. You want to let developers create their own IAM roles, but you cannot let them create roles more powerful than what the security team approves.

The solution: permission boundaries + a delegation policy.

```json
// STEP 1: The security team creates the boundary policy
// This defines the MAXIMUM permissions any developer-created role can have
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServicesForDevRoles",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords",
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage"
      ],
      "Resource": "*"
    }
  ]
}
```

```json
// STEP 2: The security team gives developers a policy that lets them
// create roles -- BUT ONLY if they attach the boundary
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCreateRoleWithBoundary",
      "Effect": "Allow",
      "Action": "iam:CreateRole",
      "Resource": "arn:aws:iam::123456789012:role/dev-*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DevBoundary"
        }
      }
    },
    {
      "Sid": "AllowAttachPolicyToDevRoles",
      "Effect": "Allow",
      "Action": [
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy"
      ],
      "Resource": "arn:aws:iam::123456789012:role/dev-*"
    },
    {
      "Sid": "DenyBoundaryModification",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRolePermissionsBoundary",
        "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "arn:aws:iam::123456789012:role/dev-*"
    },
    {
      "Sid": "DenyBoundaryPolicyModification",
      "Effect": "Deny",
      "Action": [
        "iam:CreatePolicyVersion",
        "iam:DeletePolicy",
        "iam:DeletePolicyVersion",
        "iam:SetDefaultPolicyVersion"
      ],
      "Resource": "arn:aws:iam::123456789012:policy/DevBoundary"
    }
  ]
}
```

**Why the Deny statements matter**: Without them, a developer could:
1. Remove the boundary from a role they created (`DeleteRolePermissionsBoundary`) -- now the role has no ceiling
2. Modify the boundary policy itself (`CreatePolicyVersion` on the boundary) -- now the ceiling is whatever they want
3. Replace the boundary with a more permissive one (`PutRolePermissionsBoundary`) -- same result

These are the **privilege escalation paths** that the delegation pattern must block.

**But three Deny statements are not enough.** A thorough delegation policy must also address:

- **`iam:CreateUser`** -- users are not roles; the boundary condition on `CreateRole` does not apply to `CreateUser`. A developer could create an IAM user with `AdministratorAccess` and no boundary.
- **`iam:PassRole`** -- if unrestricted, a developer can pass a bounded role to Lambda, and if the boundary allows any IAM write actions, the Lambda function can chain those for escalation.
- **`iam:UpdateAssumeRolePolicy`** -- modifying a `dev-*` role's trust policy to let a more privileged principal assume it.

The principle: audit every IAM API action the developer can invoke and ask "can this be chained to reach permissions outside the boundary's ceiling?"

> **Resource-based policy exception**: Whether a permission boundary restricts resource-based policy access depends on **what type of principal** the resource-based policy names:
>
> | Resource-based policy names... | Boundary restricts? | Why |
> |-------------------------------|---------------------|-----|
> | IAM user ARN (`arn:aws:iam::123:user/Alice`) | **No** | User ARN grants bypass boundary |
> | IAM role session ARN (`arn:aws:sts::123:assumed-role/Role/Session`) | **No** | Session ARN grants bypass boundary |
> | IAM role ARN (`arn:aws:iam::123:role/MyRole`) | **Yes** | Role ARN does NOT bypass boundary |
>
> The key distinction: naming the **session** (`sts:assumed-role/...`) is different from naming the **role** (`iam:role/...`). This trips people up on the exam.

---

## Part 3: Session Policies -- The Day Pass

### The Analogy

You are a visitor at an airport. The airline (role's identity policy) has a broad range of things its staff can do: board passengers, access gates, handle luggage. But when you -- a visitor -- assume a temporary role by checking in at the visitor desk, you are given a **day pass** that restricts your access to just the specific gates and areas relevant to your visit. You cannot do everything a full airline employee can do, even though you are wearing a badge derived from their role.

### The Technical Reality

Session policies are inline policies passed as a parameter during `AssumeRole`, `AssumeRoleWithSAML`, `AssumeRoleWithWebIdentity`, or `GetFederationToken`. They further restrict the session below the role's identity policy and permission boundary.

```
SESSION POLICY -- FURTHER NARROWING THE EFFECTIVE PERMISSIONS
=============================================================================

  ROLE'S IDENTITY POLICY        PERMISSION BOUNDARY         SESSION POLICY
  (what the role grants)        (role's ceiling)            (session ceiling)
  ┌──────────────────────┐     ┌──────────────────────┐    ┌────────────────────┐
  │  s3:*                │     │  s3:GetObject        │    │  s3:GetObject      │
  │  dynamodb:*          │     │  s3:PutObject        │    │  Resource:         │
  │  sqs:*               │     │  dynamodb:*          │    │  arn:...:bucket/   │
  │                      │     │  sqs:SendMessage     │    │  vendor-A/*       │
  └──────────────────────┘     └──────────────────────┘    └────────────────────┘

  EFFECTIVE SESSION PERMISSIONS = identity ∩ boundary ∩ session
  ┌────────────────────┐
  │  s3:GetObject      │  ← Only GetObject (session restricts)
  │  Resource:         │     Only on vendor-A prefix (session restricts)
  │  bucket/vendor-A/* │
  │                    │
  │  dynamodb -- NO    │  ← Session policy did not include dynamodb
  │  sqs -- NO         │  ← Session policy did not include sqs
  └────────────────────┘
```

**When session policies matter**:

```bash
# Passing a session policy during AssumeRole
aws sts assume-role \
  --role-arn arn:aws:iam::111111111111:role/VendorAccessRole \
  --role-session-name vendor-a-session \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::shared-bucket/vendor-a/*"
    }]
  }'
```

This pattern is powerful for **programmatic access brokers** -- an application that vends temporary credentials to different vendors, each scoped to their own S3 prefix, all using the same underlying IAM role. Without session policies, you would need a separate role per vendor.

> **Key distinction**: If you do NOT pass a session policy, the session inherits the full permissions of the role (intersected with any boundary). Session policies only matter when explicitly provided. They are optional, not automatic.

> **Implicit deny vs explicit Deny**: This distinction is critical for understanding resource-based policy bypass. When an identity policy simply does not mention an action, that is an **implicit deny** (absence of Allow). When a policy has `"Effect": "Deny"` for an action, that is an **explicit Deny**. Resource-based policies naming user ARNs or session ARNs can bypass implicit denies but NEVER bypass explicit Denies. This is why a resource-based policy can grant access when the identity policy is silent, but cannot override a Deny statement in an SCP, boundary, or any other policy.

---

## Part 4: Cross-Account Access -- The Visitor Badge System

### The Analogy

Building A (Account A) and Building B (Account B) are separate buildings in the corporate complex. An employee in Building A needs to access a conference room in Building B. The process:

1. **Building B creates a visitor badge** (IAM role with trust policy) that says "employees from Building A are welcome."
2. **Building A's security approves the visit** (identity policy grants `sts:AssumeRole` on Building B's role).
3. **The employee walks to Building B's reception**, presents their Building A ID, and receives the visitor badge (STS returns temporary credentials).
4. **While wearing the visitor badge**, the employee operates under Building B's rules and permissions -- their Building A privileges are irrelevant.

### The Technical Reality: AssumeRole Flow

```
CROSS-ACCOUNT AssumeRole -- STEP BY STEP
=============================================================================

  ACCOUNT A (111111111111)              ACCOUNT B (222222222222)
  ════════════════════════              ════════════════════════

  1. Account B creates an IAM role with a TRUST POLICY:

     Role: arn:aws:iam::222222222222:role/CrossAccountAuditRole
     ┌─────────────────────────────────────────────────────────┐
     │  TRUST POLICY (who can assume this role):               │
     │  {                                                      │
     │    "Version": "2012-10-17",                             │
     │    "Statement": [{                                      │
     │      "Effect": "Allow",                                 │
     │      "Principal": {                                     │
     │        "AWS": "arn:aws:iam::111111111111:root"          │
     │      },                                                 │
     │      "Action": "sts:AssumeRole",                        │
     │      "Condition": {                                     │
     │        "StringEquals": {                                │
     │          "sts:ExternalId": "audit-2026-xK9mP"           │
     │        }                                                │
     │      }                                                  │
     │    }]                                                   │
     │  }                                                      │
     │                                                         │
     │  PERMISSIONS POLICY (what the role can do):             │
     │  {                                                      │
     │    "Effect": "Allow",                                   │
     │    "Action": [                                          │
     │      "s3:GetObject", "s3:ListBucket",                   │
     │      "cloudtrail:LookupEvents",                         │
     │      "config:GetComplianceDetailsByResource"            │
     │    ],                                                   │
     │    "Resource": "*"                                       │
     │  }                                                      │
     └─────────────────────────────────────────────────────────┘

  2. Account A grants its user permission to assume that role:

     ┌─────────────────────────────────────────────────────────┐
     │  User: Alice                                            │
     │  Identity Policy:                                       │
     │  {                                                      │
     │    "Effect": "Allow",                                   │
     │    "Action": "sts:AssumeRole",                          │
     │    "Resource":                                           │
     │      "arn:aws:iam::222222222222:role/CrossAccountAudit*"│
     │  }                                                      │
     └─────────────────────────────────────────────────────────┘

  3. Alice calls STS:

     aws sts assume-role \
       --role-arn arn:aws:iam::222222222222:role/CrossAccountAuditRole \
       --role-session-name alice-audit-session \
       --external-id audit-2026-xK9mP

  4. STS validates:
     ├── Is Alice's identity policy allowing sts:AssumeRole? ✅
     ├── Does the trust policy on CrossAccountAuditRole trust Alice
     │   (or Account A)? ✅
     ├── Does the ExternalId match? ✅
     ├── Does Account A's SCP allow sts:AssumeRole? ✅
     └── Does Account B's SCP allow the role's actions? ✅

  5. STS returns temporary credentials:
     {
       "AccessKeyId": "ASIA...",
       "SecretAccessKey": "...",
       "SessionToken": "...",
       "Expiration": "2026-03-05T14:30:00Z"
     }

  6. Alice uses these credentials to operate IN Account B,
     with the permissions of CrossAccountAuditRole.
     Her original Account A permissions are IRRELEVANT now.

     CRITICAL: Any API calls Alice makes with these credentials
     are evaluated as SAME-ACCOUNT requests in Account B --
     not cross-account. The cross-account evaluation only
     applies to the AssumeRole call itself. Once assumed,
     Alice IS Account B from IAM's perspective.
```

### The Confused Deputy Problem and External IDs

**The Analogy**: Imagine Building B trusts "anyone from Building A" to enter. A malicious person in Building C discovers this and tricks a Building A employee into visiting Building B on their behalf -- the employee unknowingly uses their trusted status to give Building C access to Building B's resources. This is the **confused deputy**.

**The Technical Reality**: The confused deputy occurs with third-party services. Suppose you hire a SaaS vendor (Account C) to analyze your S3 data in your account (Account B). You create a role in Account B that trusts Account C. But Account C serves many customers. A malicious customer of Account C could tell the vendor "use my account's role" and provide YOUR role ARN. The vendor's service (the "deputy") is "confused" -- it assumes your role on behalf of the attacker.

The fix: **External IDs**.

```json
// Trust policy WITH ExternalId -- prevents confused deputy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::333333333333:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "customer-12345-unique-secret"
      }
    }
  }]
}
```

The ExternalId is a shared secret between you and the vendor. The vendor's service must present it during AssumeRole. A malicious customer cannot guess your ExternalId, so they cannot trick the vendor into assuming your role.

> **When you do NOT need ExternalId**: When both accounts are within your own Organization. The confused deputy problem only applies to **third-party cross-account access**. For internal cross-account roles, use Organization-aware condition keys instead:
>
> | Condition Key | Restricts To | Example |
> |--------------|-------------|---------|
> | `aws:PrincipalOrgID` | Any principal in your Organization | `"o-abc123def4"` |
> | `aws:PrincipalOrgPaths` | Principals in a specific OU path | `"o-abc123def4/r-root/ou-prod/*"` |
>
> Use `PrincipalOrgID` for broad "anyone in our org" trust. Use `PrincipalOrgPaths` when you need to restrict to a specific OU (e.g., only production accounts can assume this role).

```json
// Internal cross-account: use aws:PrincipalOrgID instead of ExternalId
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "*"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "aws:PrincipalOrgID": "o-abc123def4"
      }
    }
  }]
}
```

---

## Part 5: SAML 2.0 Federation -- Corporate Badge Meets Government ID

### The Analogy

Your company uses a badge system (Okta, Azure AD) that identifies every employee. AWS is a government facility that requires its own credentials. Instead of issuing every employee a separate government ID (IAM user), the government agrees to **recognize your corporate badge** -- if a trusted corporate badge office (IdP) vouches for an employee, the government facility will issue a temporary visitor pass (STS credentials). The employee never gets permanent government credentials. They flash their corporate badge, the badge office sends a signed letter (SAML assertion) to the government, and the government hands back a day pass.

### The Technical Reality

```
SAML 2.0 FEDERATION FLOW
=============================================================================

  ┌──────────┐     ┌──────────────────┐     ┌──────────────────┐
  │          │     │                  │     │                  │
  │  USER    │     │  CORPORATE IdP   │     │  AWS STS         │
  │ (Browser)│     │  (Okta/Azure AD) │     │                  │
  │          │     │                  │     │                  │
  └────┬─────┘     └────────┬─────────┘     └────────┬─────────┘
       │                    │                         │
       │  1. User navigates │                         │
       │  to corporate      │                         │
       │  SSO portal        │                         │
       │───────────────────▶│                         │
       │                    │                         │
       │  2. IdP            │                         │
       │  authenticates     │                         │
       │  user (MFA, etc.)  │                         │
       │◀───────────────────│                         │
       │                    │                         │
       │  3. IdP generates  │                         │
       │  SAML Assertion    │                         │
       │  containing:       │                         │
       │  - User identity   │                         │
       │  - Group membership│                         │
       │  - Role ARN to     │                         │
       │    assume           │                         │
       │  - Signed by IdP   │                         │
       │◀───────────────────│                         │
       │                    │                         │
       │  4. Browser POSTs SAML assertion             │
       │  to AWS sign-in endpoint                     │
       │  (https://signin.aws.amazon.com/saml)        │
       │─────────────────────────────────────────────▶│
       │                                              │
       │  5. STS calls AssumeRoleWithSAML:            │
       │  - Validates SAML assertion signature        │
       │  - Checks IAM SAML provider entity           │
       │  - Checks role's trust policy                │
       │  - Returns temporary credentials             │
       │◀─────────────────────────────────────────────│
       │                                              │
       │  6. User is now authenticated in AWS         │
       │  Console with the assumed role's permissions  │
       │                                              │
```

**Setting up SAML federation requires three components**:

1. **IAM SAML Identity Provider** -- an entity in IAM that holds the IdP's metadata XML (contains the IdP's public certificate for validating SAML assertions)

2. **IAM Role with SAML trust policy** -- the role that federated users will assume

3. **IdP configuration** -- the corporate IdP must be configured to send SAML assertions to AWS with the correct attributes (Role ARN, Principal ARN)

```json
// Trust policy for a SAML-federated role
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::111111111111:saml-provider/OktaProvider"
    },
    "Action": "sts:AssumeRoleWithSAML",
    "Condition": {
      "StringEquals": {
        "SAML:aud": "https://signin.aws.amazon.com/saml"
      }
    }
  }]
}
```

> **SAML vs IAM Identity Center**: For new deployments, AWS strongly recommends IAM Identity Center over direct SAML federation. Identity Center uses SAML under the hood but automates the role creation, trust policy management, and permission set distribution across accounts. Direct SAML federation requires manually creating roles and trust policies in every account -- exactly the toil that Identity Center eliminates. Use direct SAML only for legacy integrations or edge cases where Identity Center does not support your IdP.

---

## Part 6: OIDC Federation -- Digital ID for Workloads

### The Analogy

OIDC federation is like a **digital notary service**. Instead of a person walking to a desk with a badge, a software system (GitHub Actions runner, EKS pod, mobile app) presents a digitally signed certificate (JWT token) from a trusted notary (OIDC provider). The government facility (AWS) verifies the notary's signature, checks the certificate's claims ("this is the main branch of repo X"), and issues a temporary access pass. No permanent credentials are ever created or stored.

### The Technical Reality

You already know OIDC from [EKS IRSA (Dec 10)](../../../2025/december/eks/2025-12-10-eks-iam-irsa-pod-identity.md). The pattern is the same whether the OIDC provider is EKS, GitHub Actions, or Google:

1. AWS trusts an OIDC provider (registers it as an IAM OIDC Identity Provider)
2. An IAM role has a trust policy that allows `sts:AssumeRoleWithWebIdentity` from that provider
3. The workload obtains a JWT token from the OIDC provider
4. The workload calls STS with the token and receives temporary credentials

### GitHub Actions OIDC -- Eliminating Long-Lived AWS Keys in CI/CD

This is the most impactful practical application. Before OIDC, GitHub Actions workflows stored `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as repository secrets. Those keys were long-lived, had to be rotated manually, and were a prime target for credential theft. With OIDC, GitHub issues a short-lived JWT for each workflow run, and AWS validates it directly.

```json
// IAM OIDC Provider for GitHub Actions
// Created once in the account
// Thumbprint: GitHub's OIDC provider TLS certificate thumbprint
// (AWS validates this to ensure the token came from GitHub, not an impersonator)

// Trust policy for GitHub Actions OIDC role
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::111111111111:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
      }
    }
  }]
}
```

**The condition keys are critical for security**:

| Condition Key | Purpose | Example |
|---------------|---------|---------|
| `:aud` (audience) | Ensures the token was issued for AWS, not another service | `sts.amazonaws.com` |
| `:sub` (subject) | Restricts which repo, branch, or environment can assume the role | `repo:my-org/my-repo:ref:refs/heads/main` |

> **Without the `:sub` condition**, any repository in your GitHub organization (or even any GitHub repository, if you use `*`) could assume this role. Always scope to the specific repository and branch. For production deployments, restrict to `refs/heads/main` or a specific environment.

```yaml
# GitHub Actions workflow using OIDC
name: Deploy to AWS
on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/GitHubActionsDeployRole
          aws-region: us-east-1
          # No access keys needed -- OIDC handles authentication

      - run: aws s3 ls  # Uses temporary OIDC credentials
```

### EKS IRSA and Pod Identity Connection

You covered IRSA (IAM Roles for Service Accounts) in December. The pattern is identical:

- EKS runs an OIDC provider per cluster
- Each Kubernetes ServiceAccount can be annotated with an IAM role ARN
- The pod receives a projected JWT token, which the AWS SDK exchanges for STS credentials
- The role's trust policy conditions on `oidc.eks.<region>.amazonaws.com:sub` to restrict which ServiceAccount can assume it

Pod Identity (the newer approach) simplifies this by removing the need to annotate ServiceAccounts and managing the OIDC provider association automatically. Both use the same underlying OIDC federation mechanism.

---

## Part 7: IAM Identity Center (SSO) -- The Single Reception Desk

### The Analogy

Before IAM Identity Center, the corporate complex had a separate reception desk in every building. Each desk maintained its own visitor list, issued its own badges, and enforced its own rules. If you needed access to ten buildings, you filled out ten forms. If your access changed, ten desks had to update their records. This is what manual cross-account IAM roles look like -- it works, but it does not scale.

IAM Identity Center is the **single reception desk in the lobby of the main building**. It connects to the corporate directory (your IdP), maintains a catalog of badge templates (permission sets), and when you need access to Building 7, the reception desk automatically creates the right badge in Building 7's system. One desk, one directory, one set of templates -- replicated across every building.

### The Technical Reality

IAM Identity Center (formerly AWS SSO) is a free service that provides centralized access management across all accounts in your Organization. It replaces the need to manually create cross-account IAM roles, trust policies, and identity providers in each account.

```
IAM IDENTITY CENTER ARCHITECTURE
=============================================================================

  ┌─────────────────────────────────────────────────────────────────────┐
  │                    IAM IDENTITY CENTER                             │
  │                    (lives in management or delegated admin account) │
  │                                                                     │
  │  ┌───────────────────┐    ┌─────────────────────────────────────┐  │
  │  │  IDENTITY SOURCE  │    │  PERMISSION SETS                    │  │
  │  │                   │    │  (badge templates)                  │  │
  │  │  Option 1:        │    │                                     │  │
  │  │  Built-in dir     │    │  ┌─────────────────────────────┐   │  │
  │  │                   │    │  │ AdministratorAccess         │   │  │
  │  │  Option 2:        │    │  │ (maps to AWS managed policy)│   │  │
  │  │  Active Directory │    │  └─────────────────────────────┘   │  │
  │  │  (AD Connector or │    │                                     │  │
  │  │   AWS Managed AD) │    │  ┌─────────────────────────────┐   │  │
  │  │                   │    │  │ ReadOnlyAccess              │   │  │
  │  │  Option 3:        │    │  │ (maps to AWS managed policy)│   │  │
  │  │  External SAML    │    │  └─────────────────────────────┘   │  │
  │  │  IdP (Okta, etc.) │    │                                     │  │
  │  │                   │    │  ┌─────────────────────────────┐   │  │
  │  └───────────────────┘    │  │ CustomDevOps               │   │  │
  │                           │  │ (custom inline + managed    │   │  │
  │                           │  │  policies + boundary)       │   │  │
  │                           │  └─────────────────────────────┘   │  │
  │                           └─────────────────────────────────────┘  │
  │                                                                     │
  │  ┌────────────────────────────────────────────────────────────────┐ │
  │  │  ASSIGNMENTS                                                   │ │
  │  │  (who gets which badge template in which building)             │ │
  │  │                                                                 │ │
  │  │  Group: DevOps-Team ──▶ PermSet: CustomDevOps ──▶ Accounts:   │ │
  │  │                          ├── 111111111111 (Prod)               │ │
  │  │                          ├── 222222222222 (Staging)            │ │
  │  │                          └── 333333333333 (Dev)                │ │
  │  │                                                                 │ │
  │  │  Group: Developers ──▶ PermSet: ReadOnlyAccess ──▶ Accounts:  │ │
  │  │                          └── 111111111111 (Prod)               │ │
  │  │                                                                 │ │
  │  │  Group: Developers ──▶ PermSet: AdministratorAccess ──▶       │ │
  │  │                          └── 333333333333 (Dev)                │ │
  │  └────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────┬──────────────────────────────┘
                                         │
              WHAT HAPPENS BEHIND THE SCENES
              ═══════════════════════════════
                                         │
                                         ▼
  For each assignment, Identity Center AUTO-CREATES:
  ┌──────────────────────────────────────────────────────────────────┐
  │  In Account 111111111111:                                       │
  │  ├── IAM Role: AWSReservedSSO_CustomDevOps_abc123               │
  │  │   ├── Trust Policy: trusts Identity Center SAML provider     │
  │  │   ├── Permissions: matches CustomDevOps permission set       │
  │  │   └── Permission Boundary: if set on the permission set      │
  │  │                                                               │
  │  └── IAM Role: AWSReservedSSO_ReadOnlyAccess_def456             │
  │      ├── Trust Policy: trusts Identity Center SAML provider     │
  │      └── Permissions: ReadOnlyAccess managed policy             │
  │                                                                  │
  │  In Account 222222222222:                                       │
  │  └── IAM Role: AWSReservedSSO_CustomDevOps_abc123               │
  │      └── (same config, auto-created)                            │
  │                                                                  │
  │  In Account 333333333333:                                       │
  │  ├── IAM Role: AWSReservedSSO_CustomDevOps_abc123               │
  │  └── IAM Role: AWSReservedSSO_AdministratorAccess_ghi789        │
  └──────────────────────────────────────────────────────────────────┘
```

### Permission Sets in Detail

A permission set is a **template** that defines:
- Up to 10 managed policies **total** (AWS managed + customer managed combined; can request increase to 20 via Service Quotas)
- One inline policy (max 10,240 characters)
- One permission boundary (optional)
- Session duration (1 hour to 12 hours, default 1 hour)

When you assign a permission set to an account, Identity Center creates an IAM role in that account with a trust policy pointing to Identity Center's SAML provider. The role name follows the pattern `AWSReservedSSO_<PermissionSetName>_<RandomSuffix>`.

> **Connection to Control Tower**: You learned on Mar 3 that Control Tower sets up IAM Identity Center as part of the landing zone. Control Tower creates default permission sets (`AWSAdministratorAccess`, `AWSReadOnlyAccess`, `AWSPowerUserAccess`) and integrates with Account Factory so new accounts are automatically registered with Identity Center. The permission sets you create in Identity Center are the **operational complement** to Control Tower's preventive controls (SCPs) -- SCPs set the ceiling, permission sets define the actual access granted within that ceiling.

### Identity Center vs Manual Cross-Account Roles

| Dimension | IAM Identity Center | Manual Cross-Account Roles |
|-----------|--------------------|-----------------------------|
| Role creation | Automatic -- defined once, deployed everywhere | Manual -- create role + trust policy per account |
| Trust policy management | Managed by Identity Center | You manage each trust policy |
| Adding a new account | Assign existing permission sets | Create new roles in the new account |
| Revoking access | Remove assignment; roles are updated automatically | Delete roles in each account manually |
| User directory | Centralized (built-in, AD, or external IdP) | No directory -- you manage IAM users or federate per account |
| Audit trail | Single CloudTrail event source | Distributed across accounts |
| Cost | Free | Free (but toil is expensive) |
| When to use | Workforce access to AWS Console/CLI | Service-to-service cross-account, programmatic access, third-party vendor access |

---

## Part 8: Putting It All Together -- The Full Restriction Stack

Now you can see the complete picture of how every restriction layer fits together. From broadest (Organization) to narrowest (session), each layer can only restrict what the layer above allows:

```
THE COMPLETE IAM RESTRICTION STACK
=============================================================================

  LAYER 1: SCPs + RCPs (Organization ceiling)
  ┌──────────────────────────────────────────────────────────────────┐
  │  SCPs: "These principals may use these services in these regions│
  │  RCPs: "These resources may be accessed by these principal types│
  │  Set by: Central cloud team at the OU/account level             │
  │  You learned these: Mar 2 (Organizations & SCPs)                │
  │  Affects: All principals/resources in member accounts           │
  ├──────────────────────────────────────────────────────────────────┤
  │                                                                  │
  │  LAYER 2: Resource-based policies (resource-level gate)         │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  "This S3 bucket / KMS key allows these specific         │   │
  │  │   principals to perform these actions"                   │   │
  │  │  Set by: Resource owner                                   │   │
  │  │  Special: Can GRANT same-account access without          │   │
  │  │  identity policy (for user/session ARNs)                 │   │
  │  ├──────────────────────────────────────────────────────────┤   │
  │  │                                                          │   │
  │  │  LAYER 3: Permission boundaries (principal ceiling)      │   │
  │  │  ┌──────────────────────────────────────────────────┐   │   │
  │  │  │  "This role may do AT MOST these actions"         │   │   │
  │  │  │  Set by: Security team (delegation pattern)       │   │   │
  │  │  │  Affects: The specific user/role it is attached to │   │   │
  │  │  ├──────────────────────────────────────────────────┤   │   │
  │  │  │                                                  │   │   │
  │  │  │  LAYER 4: Identity-based policies (grant)        │   │   │
  │  │  │  ┌──────────────────────────────────────────┐   │   │   │
  │  │  │  │  "This user/role is allowed to do X"      │   │   │   │
  │  │  │  │  Set by: Account admin or self-service    │   │   │   │
  │  │  │  │  This is the ACTUAL permission grant      │   │   │   │
  │  │  │  ├──────────────────────────────────────────┤   │   │   │
  │  │  │  │                                          │   │   │   │
  │  │  │  │  LAYER 5: Session policies (session cap) │   │   │   │
  │  │  │  │  ┌──────────────────────────────────┐   │   │   │   │
  │  │  │  │  │  "This specific session may only  │   │   │   │   │
  │  │  │  │  │   do Y (subset of X)"             │   │   │   │   │
  │  │  │  │  │  Set by: Caller during AssumeRole │   │   │   │   │
  │  │  │  │  │  Optional -- only if provided     │   │   │   │   │
  │  │  │  │  └──────────────────────────────────┘   │   │   │   │
  │  │  │  └──────────────────────────────────────────┘   │   │   │
  │  │  └──────────────────────────────────────────────────┘   │   │
  │  └──────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘

  EFFECTIVE PERMISSION = SCP ∩ RCP ∩ boundary ∩ identity policy ∩ session policy
                         (plus resource-based policy grants where applicable)
                         (minus any explicit Deny from any layer)
```

### Decision Framework: Which Restriction Layer to Use When

| Scenario | Use This Layer | Why |
|----------|---------------|-----|
| Block all accounts in Sandbox OU from launching GPU instances | SCP | Account-level ceiling, applied at OU |
| Let developers create their own Lambda execution roles safely | Permission Boundary | Delegation without privilege escalation |
| Grant a CI/CD pipeline access to deploy in Account B | Cross-account role (AssumeRole) | Service-to-service cross-account access |
| Restrict a third-party vendor's assumed session to their S3 prefix | Session Policy | Per-session scoping of a shared role |
| Allow an S3 bucket to accept writes from another account | Resource-based Policy | Resource-side grant for cross-account |
| Manage workforce access to 50 accounts via Okta | IAM Identity Center | Centralized SSO with permission sets |
| Legacy on-prem app needs to call AWS APIs | SAML Federation | Direct IdP-to-STS integration |
| GitHub Actions needs to deploy without stored keys | OIDC Federation | Short-lived JWT tokens, no secrets |
| Restrict API access through a VPC endpoint to specific principals | VPC Endpoint Policy | Network-level access control on the endpoint |

---

## Key Takeaways

- **The IAM policy evaluation engine evaluates up to eight policy types for every request** (SCPs, RCPs, resource-based, identity-based, permission boundaries, session policies, VPC endpoint policies, and ACLs). The effective permission is the intersection of all applicable restrictive layers, plus any grants from resource-based policies, minus any explicit Deny from any layer. Explicit Deny always wins.

- **Same-account and cross-account evaluation are fundamentally different.** In same-account, a resource-based policy can grant access without an identity policy (when it names a user ARN or role session ARN). In cross-account, both sides must independently grant access -- no exceptions.

- **Permission boundaries solve the delegation bottleneck.** They let the security team define a ceiling and then step out of the way, allowing developers to self-service IAM role creation without risking privilege escalation. The three Deny statements (deny removing boundary, deny modifying boundary policy, deny changing boundary) are the essential safeguards.

- **Session policies are the most surgical restriction tool.** They scope a single assumed-role session below the role's own permissions. The primary use case is access brokers that vend credentials to different tenants using the same underlying role.

- **External IDs prevent the confused deputy attack on third-party cross-account roles.** For internal cross-account roles within your Organization, use `aws:PrincipalOrgID` instead -- it is simpler and more maintainable.

- **OIDC federation eliminates long-lived credentials for workloads.** GitHub Actions OIDC is the highest-impact quick win for any team still storing AWS keys in CI/CD. Always scope the trust policy's `:sub` condition to the specific repository and branch.

- **IAM Identity Center replaces manual cross-account role management for workforce access.** It uses SAML under the hood but automates role creation, trust policy management, and permission set distribution. Combined with Control Tower (which you learned on Mar 3), it is the standard approach for governing workforce access in a multi-account Organization.

- **The restriction layers stack from broad to narrow**: SCP/RCP (account ceiling) > Permission Boundary (principal ceiling) > Identity Policy (actual grant) > Session Policy (session ceiling). Most layers can only restrict, never expand -- the exception is resource-based policies, which can actively grant same-account access. Understanding this stack is the single most important IAM concept for both production debugging and interviews.

- **AssumeRole sessions have configurable duration.** Default is 1 hour, maximum is 12 hours (configured per role via `MaxSessionDuration`). The caller can request a shorter duration via `--duration-seconds`. Common interview question: "what happens when temporary credentials expire?" -- the application must call AssumeRole again.

- **Service-linked roles are AWS-managed roles for service-to-service access.** Unlike regular service roles (which you create and manage), service-linked roles are created by AWS services (e.g., `AWSServiceRoleForElasticLoadBalancing`) with predefined permissions you cannot modify. They have trust policies that only the specific service can assume. You may need to create them explicitly before using certain services, and you cannot delete them while the service is still using resources.

- **IAM Access Analyzer is your policy validation and least-privilege tool.** It does three things: (1) **external access findings** -- identifies resources (S3 buckets, IAM roles, KMS keys) shared with external entities, (2) **policy validation** -- checks your policies against IAM best practices and flags issues, (3) **policy generation** -- analyzes CloudTrail logs to generate least-privilege policies based on actual usage. It integrates with Security Hub (tomorrow's topic).

---

## Further Reading

- [IAM Policy Evaluation Logic (Official Docs)](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html) -- the authoritative flowcharts for same-account and cross-account evaluation
- [Cross-Account Policy Evaluation (Official Docs)](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html) -- the cross-account specific flowchart
- [Permissions Boundaries for IAM Entities (Official Docs)](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) -- the Maria/Zhang delegation scenario with four policy examples
- [When and Where to Use IAM Permissions Boundaries (AWS Security Blog)](https://aws.amazon.com/blogs/security/when-and-where-to-use-iam-permissions-boundaries/) -- practical guidance on the delegation pattern
- [Configuring OIDC for GitHub Actions (GitHub Docs)](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) -- step-by-step GitHub Actions OIDC setup
- [AWS Organizations & SCPs (Mar 2)](./2026-03-02-aws-organizations-scps.md) -- the SCP evaluation logic this doc builds on
- [AWS Control Tower & Landing Zones (Mar 3)](./2026-03-03-aws-control-tower-landing-zones.md) -- Identity Center setup within Control Tower
