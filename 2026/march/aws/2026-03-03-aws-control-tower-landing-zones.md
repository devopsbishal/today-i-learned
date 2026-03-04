# AWS Control Tower & Landing Zones -- The Smart Building Analogy

> Yesterday you learned the property management company (Organizations) and its lease agreements (SCPs). Today: the **smart building management platform** that automates the entire setup -- pre-wiring fire alarms (detective controls), locking doors (preventive controls), inspecting blueprints (proactive controls), and onboarding tenants (Account Factory) so every building meets code before anyone moves in.

---

## TL;DR

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| AWS Control Tower | A smart building management platform | An opinionated orchestration layer that automates what you would otherwise build manually with Organizations, SCPs, Config Rules, CloudFormation StackSets, and IAM Identity Center |
| Landing Zone | A move-in-ready building complex | The pre-configured multi-account environment that Control Tower creates: OU structure, shared accounts, baseline controls, centralized logging, and SSO -- all wired on day one |
| Account Factory | The tenant onboarding office | A self-service mechanism for provisioning new AWS accounts with pre-approved configurations, network settings, and baseline controls -- like a leasing office that guarantees every new tenant gets a building built to code |
| Account Factory for Terraform (AFT) | A city planning department that accepts building permits by mail | A GitOps pipeline where you define account requests as Terraform files, push to a repo, and a Step Functions pipeline provisions the account through Control Tower and applies customizations -- like mailing a blueprint instead of walking to the leasing office counter |
| Preventive controls | Locked doors and physical barriers | SCPs that block forbidden actions before they happen -- the door will not open, period; you already know how these work from yesterday |
| Detective controls | Security cameras and fire alarms | AWS Config Rules that monitor resources after creation and alert when something violates policy -- the alarm goes off after the event, but it does not physically stop it |
| Proactive controls | Construction-permit inspections | CloudFormation Hooks that intercept resource creation at deploy time and reject non-compliant infrastructure before it exists -- the building inspector rejects the blueprint before construction begins |
| Mandatory controls | Fire code requirements | Controls that Control Tower enables automatically and you cannot disable -- non-negotiable safety baselines |
| Strongly recommended controls | Building association best practices | Controls that AWS recommends enabling but leaves the decision to you -- sensible defaults you should adopt unless you have a specific reason not to |
| Elective controls | Optional luxury upgrades | Situational controls you enable based on specific compliance needs -- not universally applicable but valuable in the right context |
| Control Tower dashboard | The building management console | A single pane of glass showing compliance status, drift alerts, and account inventory across your entire Organization |
| Drift detection | A building inspector who periodically photographs every room | Control Tower's mechanism for detecting when someone has manually changed the guardrails, account configurations, or OU structure that it deployed -- like an inspector who compares photos to the original blueprints and flags unauthorized changes |
| Log Archive account | The building's black box recorder | A dedicated account in the Security OU that centralizes CloudTrail logs, Config snapshots, and other audit data -- immutable, append-only, accessible only to the security team |
| Audit (Security Tooling) account | The building inspector's office | A dedicated account with cross-account read access for security tooling -- GuardDuty, Security Hub, Config aggregation -- where the inspector can see into every building without living in any of them |
| CloudFormation StackSets | The building code enforcement crew | The mechanism Control Tower uses to push baseline configurations (Config rules, CloudTrail, IAM roles) into every enrolled account across all governed regions |
| IAM Identity Center | The master key card system | Centralized SSO that gives users federated access to all accounts in the Organization via permission sets that map to IAM roles -- one badge, many doors |

---

## The Big Picture

Yesterday you learned the raw building materials: **AWS Organizations** is the property management company that owns a portfolio of buildings (accounts), organizes them into neighborhoods (OUs), and publishes lease agreements (SCPs) that set the rules for every building. You learned how to write SCPs by hand, attach them to OUs, and reason about inheritance. That knowledge is foundational -- and today you are building directly on top of it.

**AWS Control Tower is the smart building management platform that the property management company installs.** It does not replace the property company (Organizations) -- it sits on top of it. It takes the best practices you learned yesterday (Security OU with Log Archive and Audit accounts, deny-list SCPs for region restriction and root user lockdown, centralized CloudTrail, delegated administrators) and automates the entire setup into a single "landing zone" deployment. Where yesterday you would have manually created OUs, written SCP JSON, configured CloudTrail, and set up IAM Identity Center, today Control Tower does all of that with a guided setup process and keeps monitoring for drift after the fact.

The key mental shift: **Organizations gives you the API primitives. Control Tower gives you the opinionated orchestration.**

Think of it like Terraform versus raw AWS API calls. You *could* provision everything with `aws ec2 run-instances`, but Terraform gives you a declarative, repeatable, drift-detecting layer on top. Control Tower does the same thing for multi-account governance -- it is the "Terraform" of AWS Organizations, with one key caveat: it detects drift but does **not** auto-remediate it the way `terraform apply` does.

```
THE RELATIONSHIP -- CONTROL TOWER SITS ON TOP OF ORGANIZATIONS
=============================================================================

                    ┌─────────────────────────────────────────────────┐
                    │           AWS CONTROL TOWER                     │
                    │  (Opinionated orchestration layer)              │
                    │                                                 │
                    │  ┌──────────────┐  ┌──────────────────────────┐│
                    │  │ Landing Zone │  │ Account Factory          ││
                    │  │ Setup        │  │ (console or AFT)         ││
                    │  └──────┬───────┘  └──────────┬───────────────┘│
                    │         │                     │                │
                    │  ┌──────▼─────────────────────▼───────────────┐│
                    │  │           Controls (Guardrails)            ││
                    │  │  Preventive │ Detective │ Proactive        ││
                    │  └──────┬──────────┬────────────┬─────────────┘│
                    │         │          │            │              │
                    └─────────┼──────────┼────────────┼──────────────┘
                              │          │            │
              ┌───────────────▼┐   ┌─────▼──────┐  ┌─▼──────────────────┐
              │  Organizations  │   │ AWS Config │  │ CloudFormation     │
              │  + SCPs         │   │ Rules      │  │ Hooks + StackSets  │
              └───────────────┬┘   └─────┬──────┘  └─┬──────────────────┘
                              │          │           │
              ┌───────────────▼──────────▼───────────▼──────────────────┐
              │                                                         │
              │  IAM Identity Center  │  CloudTrail  │  Service Catalog │
              │                                                         │
              └─────────────────────────────────────────────────────────┘

  WHAT CONTROL TOWER AUTOMATES vs WHAT YOU DID MANUALLY YESTERDAY:
  ═══════════════════════════════════════════════════════════════════

  Manual (Organizations + SCPs)          Control Tower Automation
  ──────────────────────────────         ──────────────────────────
  Create Security OU                     ✅ Created during landing zone setup
  Create Log Archive account             ✅ Created during landing zone setup
  Create Audit account                   ✅ Created during landing zone setup
  Write deny-root-user SCP               ✅ Deployed as mandatory preventive control
  Write restrict-regions SCP             ✅ Available as strongly recommended control
  Configure CloudTrail org trail         ✅ Auto-configured in every enrolled account
  Deploy Config rules per account        ✅ StackSets push rules to all accounts
  Set up IAM Identity Center             ✅ Configured with preconfigured permission sets
  Detect drift when someone changes      ✅ Built-in drift detection and alerting
  something manually
  Provision new accounts consistently    ✅ Account Factory standardizes provisioning
  Manage all of the above in Terraform   ✅ AFT provides GitOps-based account management
```

---

## Part 1: The Landing Zone -- What Gets Built

### The Analogy

When you buy a plot of land and want to build a neighborhood of buildings, you do not just start pouring concrete. First, you **prepare the land**: grade the soil, install water mains, run electrical conduit underground, build roads, install fire hydrants, and wire up a security system. Only then do individual buildings go up. The **landing zone** is this site preparation. It creates the foundational infrastructure that every future account (building) will depend on.

### The Technical Reality

When you set up Control Tower, it creates a **landing zone** -- a pre-configured multi-account environment following AWS best practices. Here is exactly what gets deployed:

```
LANDING ZONE -- WHAT CONTROL TOWER CREATES
=============================================================================

  YOUR EXISTING MANAGEMENT ACCOUNT (must already exist)
  │
  │  Control Tower enables Organizations (if not already enabled)
  │  and deploys the following:
  │
  ├── SECURITY OU (Foundational)
  │   │
  │   ├── LOG ARCHIVE ACCOUNT (created by Control Tower)
  │   │   ├── Centralized CloudTrail logs from ALL enrolled accounts
  │   │   ├── AWS Config delivery channel aggregation
  │   │   ├── S3 buckets with lifecycle policies and MFA-delete protection
  │   │   ├── Bucket policies that deny deletion (even from the account root)
  │   │   └── Only the security/audit team should access this account
  │   │
  │   └── AUDIT ACCOUNT (created by Control Tower)
  │       ├── Preconfigured cross-account IAM roles:
  │       │   ├── AWSControlTowerExecution -- full admin (for CT operations)
  │       │   ├── aws-controltower-ReadOnlyExecutionRole -- read-only
  │       │   │   access to ALL enrolled accounts
  │       │   └── aws-controltower-AuditReadOnlyRole -- for auditors
  │       ├── SNS topics for control notifications
  │       ├── This is where you deploy GuardDuty, Security Hub,
  │       │   Config aggregator as delegated admin
  │       └── Can programmatically audit any enrolled account
  │
  ├── ADDITIONAL OUs (you create these yourself after setup)
  │   ├── Sandbox OU -- recommended for developer experimentation
  │   ├── Workloads OU -- for production and non-production accounts
  │   └── Control Tower only auto-creates the Security OU;
  │       all other OUs are created by you afterward
  │
  ├── MANDATORY CONTROLS (auto-enabled, cannot be disabled)
  │   ├── Preventive: Disallow changes to CloudTrail configuration
  │   ├── Preventive: Disallow changes to AWS Config configuration
  │   ├── Preventive: Disallow deletion of log archive
  │   ├── Detective: Detect whether CloudTrail is enabled
  │   ├── Detective: Detect whether Config is enabled
  │   └── Approximately 20+ mandatory controls (AWS adds new ones over time)
  │
  ├── IAM IDENTITY CENTER CONFIGURATION
  │   ├── SSO enabled for the Organization
  │   ├── Preconfigured permission sets:
  │   │   ├── AWSAdministratorAccess
  │   │   ├── AWSPowerUserAccess
  │   │   ├── AWSReadOnlyAccess
  │   │   └── AWSServiceCatalogEndUserAccess (for Account Factory)
  │   └── Groups and user assignments (you configure)
  │
  ├── CLOUDFORMATION STACKSETS (the deployment mechanism)
  │   ├── BP_BASELINE_CLOUDTRAIL -- deploys CloudTrail in every account
  │   ├── BP_BASELINE_CONFIG -- deploys Config recorder in every account
  │   ├── BP_BASELINE_ROLES -- deploys cross-account IAM roles
  │   ├── BP_BASELINE_SERVICE_ROLES -- deploys service-linked roles
  │   └── Pushed to every enrolled account in every governed region
  │
  └── CONTROL TOWER DASHBOARD
      ├── Compliance status across all OUs and accounts
      ├── Non-compliant resources flagged by detective controls
      ├── Drift detection alerts
      └── Account inventory
```

### How the Landing Zone Connects to Yesterday's Knowledge

Every piece of the landing zone maps to something you already understand:

| Landing Zone Component | Yesterday's Equivalent | What Control Tower Adds |
|----------------------|----------------------|------------------------|
| Security OU | You would create with `aws_organizations_organizational_unit` | Auto-created during setup |
| Log Archive account | You would create with `aws_organizations_account` | Pre-configured with CloudTrail S3 buckets, bucket policies, lifecycle rules |
| Audit account | You would create with `aws_organizations_account` | Pre-configured with cross-account audit roles in every enrolled account |
| Mandatory preventive controls | You would write SCP JSON and attach with `aws_organizations_policy` | Pre-written, auto-attached, cannot be disabled |
| CloudTrail in every account | You would configure `aws_cloudtrail` with `is_organization_trail = true` | Deployed via StackSets to every account in every governed region |
| Config recorder in every account | You would deploy `aws_config_configuration_recorder` per account | Deployed via StackSets automatically |
| IAM Identity Center | You would enable and configure manually | Pre-configured with default permission sets |

---

## Part 2: Controls (Guardrails) -- The Three Lines of Defense

Controls are the heart of Control Tower's governance model. They have two independent classification dimensions: **behavior** (what the control does) and **guidance** (how strongly AWS recommends it).

### Dimension 1: Behavior -- When Does the Control Act?

**The Analogy**: Think of a building's safety systems operating at three different points in time.

```
THE THREE CONTROL BEHAVIORS -- DEFENSE IN DEPTH (NOT A SEQUENTIAL CHAIN)
=============================================================================

  A developer tries to create a non-compliant S3 bucket via CloudFormation.
  All three control types operate INDEPENDENTLY and CONCURRENTLY:

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                        │
  │  LAYER 1: PROACTIVE CONTROL (Construction Permit Inspection)           │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  CloudFormation Hook                                             │  │
  │  │  WHEN: During template processing, BEFORE the API call           │  │
  │  │  HOW:  Evaluates the resource definition in the template         │  │
  │  │  SCOPE: CloudFormation deployments only (CDK, CFN, CFN provider) │  │
  │  │  "Your blueprint shows no encryption. Construction denied."      │  │
  │  │  Result: ❌ Stack operation fails. Resource never created.        │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  │                                                                        │
  │  LAYER 2: PREVENTIVE CONTROL (Locked Door)                             │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  SCP (Service Control Policy)                                    │  │
  │  │  WHEN: At the IAM/Organizations authorization layer, when the    │  │
  │  │        API call is made                                          │  │
  │  │  HOW:  Evaluates IAM context (action, resource, conditions)      │  │
  │  │  SCOPE: ALL API calls (console, CLI, Terraform, CloudFormation)  │  │
  │  │  "Access denied. Your OU policy forbids this action."            │  │
  │  │  Result: ❌ API returns AccessDenied. Resource never created.     │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  │                                                                        │
  │  LAYER 3: DETECTIVE CONTROL (Security Camera)                          │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  AWS Config Rule                                                 │  │
  │  │  WHEN: After the resource exists (periodic or change-triggered)  │  │
  │  │  HOW:  Evaluates actual resource configuration                   │  │
  │  │  SCOPE: ALL resources regardless of how they were created        │  │
  │  │  "Bucket exists without encryption. Flagged NON-COMPLIANT."      │  │
  │  │  Result: ⚠️  Resource EXISTS but flagged (can auto-remediate)     │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  │                                                                        │
  │  ═══════════════════════════════════════════════════════════════════    │
  │  KEY INSIGHT: These are NOT a fallback chain. They are independent     │
  │  enforcement points that ALL fire during a CloudFormation deployment:  │
  │                                                                        │
  │  1. Proactive hook evaluates the TEMPLATE (catches bad blueprints)     │
  │  2. SCP evaluates the API CALL (catches unauthorized actions)          │
  │  3. Config rule evaluates the RESOURCE (catches anything that slips    │
  │     through layers 1 and 2, plus resources created outside CFN)        │
  │                                                                        │
  │  If a proactive control passes (template looks compliant) but the      │
  │  SCP denies the action (OU policy forbids the service), the SCP        │
  │  still blocks it. They do not skip each other.                         │
  │                                                                        │
  │  IMPORTANT DISTINCTION: "bypassed" vs "passes but still blocked"       │
  │  ─────────────────────────────────────────────────────────────         │
  │  "Bypassed" = the control never ran (e.g., CLI usage skips proactive   │
  │  controls because there is no CloudFormation template to evaluate).    │
  │  "Passes but still blocked" = both controls ran and reached different  │
  │  conclusions (e.g., template is compliant but the SCP denies the       │
  │  action in that region). The latter proves true independence.          │
  └─────────────────────────────────────────────────────────────────────────┘
```

#### Preventive Controls = SCPs (Locked Doors)

You already know these deeply from yesterday. Preventive controls are implemented as **Service Control Policies**. When Control Tower enables a preventive control on an OU, it creates and attaches an SCP to that OU. The SCP blocks forbidden API calls before they execute.

**Key detail**: Because preventive controls are SCPs, they share all SCP characteristics:
- They do **not** affect the management account
- They do **not** affect service-linked roles
- They evaluate at the API call level -- the resource is never created
- They work regardless of the deployment mechanism (console, CLI, Terraform, CloudFormation)
- They restrict **actions** (API calls), not **resource configurations** -- an SCP can deny `s3:CreateBucket` in a region, but it cannot inspect whether a bucket has encryption enabled the way a Config Rule can; SCPs evaluate IAM context (action, resource ARN, conditions), not the properties of the resource being created

**Example**: The mandatory preventive control "Disallow changes to AWS Config configuration" is an SCP that denies `config:DeleteConfigurationRecorder`, `config:StopConfigurationRecorder`, and related actions. This is the exact same SCP pattern you wrote in yesterday's `DenySecurityServiceChanges` Terraform resource.

#### Detective Controls = Config Rules (Security Cameras)

Detective controls are implemented as **AWS Config Rules** deployed via CloudFormation StackSets to every enrolled account and governed region. They continuously evaluate resource configurations and flag non-compliant resources.

**The critical difference from preventive controls**: Detective controls do not *prevent* the bad thing from happening -- they *detect* it after the fact. The non-compliant resource exists. The alarm is ringing. But the building is already standing without sprinklers.

**Example**: The mandatory detective control "Detect whether CloudTrail is enabled" deploys a Config Rule (`cloudtrail-enabled`) to every enrolled account. If someone manages to disable CloudTrail (perhaps through a mechanism that bypasses the preventive SCP, such as a service-linked role), the detective control flags it.

**Why you need both preventive AND detective**: Preventive controls have blind spots (service-linked roles, management account). Detective controls catch what slips through. Think of it as defense in depth: the locked door stops most intruders, but the security camera catches anyone who finds an alternate entrance.

#### Proactive Controls = CloudFormation Hooks (Building Inspectors)

Proactive controls are the newest type, implemented as **CloudFormation Hooks**. They intercept CloudFormation resource creation at deploy time, evaluate the proposed configuration against a policy, and **reject the deployment** if it violates the policy.

**Important limitation**: Proactive controls ONLY work with CloudFormation (and by extension, CDK which compiles to CloudFormation). Resources created via the console, CLI, or direct API calls are **not** evaluated by proactive controls. This is why proactive controls complement but do not replace preventive and detective controls.

> **Critical callout for Terraform users**: If your team uses Terraform with the standard AWS provider (which is the vast majority of Terraform deployments), **proactive controls do nothing for you**. The AWS provider makes direct API calls, which bypass CloudFormation Hooks entirely. Proactive controls only evaluate CloudFormation templates. This is a key reason you still need preventive controls (SCPs) and detective controls (Config Rules) even when proactive controls exist. Only the rarely-used Terraform CloudFormation provider would trigger proactive controls.

**Example**: The proactive control "Require an Amazon S3 bucket to have server-side encryption configured" evaluates the CloudFormation template before creating the bucket. If the template omits encryption configuration, CloudFormation rejects the stack operation. The bucket is never created.

```
CONTROL BEHAVIOR COMPARISON
=============================================================================

  ATTRIBUTE            PREVENTIVE           DETECTIVE            PROACTIVE
  ──────────────────  ──────────────────   ──────────────────  ──────────────────
  Implementation       SCP                  Config Rule          CloudFormation Hook
  When it acts         API call time        After resource       Before resource
                                            exists (periodic     creation
                                            evaluation)          (deploy time)
  Deployment           Attached to OU       StackSets push to    StackSets push to
  mechanism            (Organization        every enrolled       every enrolled
                       policy)              account/region       account/region

  Blocks creation?     Yes (AccessDenied)   No (flags only)      Yes (stack fails)
  Works with           All (console, CLI,   All (evaluates       CloudFormation
  which tools?         Terraform, CFN,      existing resources   only (CDK, CFN,
                       SDK, anything)       regardless of how    Terraform via CFN
                                            they were created)   provider)

  Affects mgmt         No (SCP exempt)      Yes (Config rules    No* (CT applies
  account?                                  run in every         controls to OUs;
                                            enrolled account)    mgmt account is at
                                                                 the root, not in
                                                                 an OU)

  * You could set up CFN Hooks independently in the mgmt account,
    but Control Tower does not deploy proactive controls there.

  Reports to           N/A (blocks)         Config dashboard     CloudFormation
                                            + CT dashboard       events + CT
                                                                 dashboard
```

### Dimension 2: Guidance -- How Strongly Does AWS Recommend It?

The guidance dimension is independent of behavior. A preventive control can be mandatory, strongly recommended, or elective. Same for detective and proactive controls.

```
CONTROL GUIDANCE LEVELS
=============================================================================

  MANDATORY
  ├── Enabled automatically when you set up the landing zone
  ├── Cannot be disabled
  ├── Non-negotiable baselines: CloudTrail, Config, log archive protection
  ├── ~20 controls total
  └── If you are using Control Tower, you accept these -- no exceptions

  STRONGLY RECOMMENDED
  ├── NOT auto-enabled -- you must explicitly turn them on
  ├── AWS strongly recommends enabling them across all OUs
  ├── Examples:
  │   ├── Restrict regions (the SCP you wrote yesterday)
  │   ├── Detect public S3 buckets
  │   ├── Detect root user activity
  │   ├── Require encryption on EBS volumes
  │   └── Detect IAM users without MFA
  └── Start here after landing zone setup -- enable these across all OUs

  ELECTIVE
  ├── Situational -- enable based on your specific compliance requirements
  ├── Examples:
  │   ├── Detect S3 buckets without lifecycle policies
  │   ├── Detect VPCs without flow logs
  │   ├── Require specific tag keys on resources
  │   └── Detect Lambda functions not in a VPC
  └── Map these to your compliance framework (CIS, NIST, PCI DSS)
```

### The Two Dimensions Combined

This creates a 3x3 matrix. Every control in Control Tower's catalog sits at one intersection:

```
THE CONTROL CLASSIFICATION MATRIX
=============================================================================

                      MANDATORY         STRONGLY            ELECTIVE
                                        RECOMMENDED
  ──────────────────┬─────────────────┬───────────────────┬─────────────────
  PREVENTIVE        │ Disallow changes│ Disallow public   │ Disallow changes
  (SCPs)            │ to CloudTrail   │ read access to    │ to bucket policy
                    │ config          │ the log archive   │ for S3 buckets
                    │                 │ S3 bucket         │ with tag
                    │ Disallow changes│                   │
                    │ to Config       │ Restrict root     │
                    │ config          │ user access       │
  ──────────────────┼─────────────────┼───────────────────┼─────────────────
  DETECTIVE         │ Detect whether  │ Detect whether    │ Detect whether
  (Config Rules)    │ CloudTrail is   │ MFA is enabled    │ S3 buckets have
                    │ enabled         │ for root user     │ lifecycle
                    │                 │                   │ policies
                    │ Detect whether  │ Detect public     │
                    │ Config is       │ S3 buckets        │ Detect VPCs
                    │ enabled         │                   │ without flow
                    │                 │ Detect root user  │ logs
                    │                 │ activity          │
  ──────────────────┼─────────────────┼───────────────────┼─────────────────
  PROACTIVE         │ (none currently)│ Require S3 bucket │ Require specific
  (CFN Hooks)       │                 │ server-side       │ tags on resources
                    │                 │ encryption        │ at creation time
                    │                 │                   │
                    │                 │ Require RDS       │
                    │                 │ encryption        │
  ──────────────────┴─────────────────┴───────────────────┴─────────────────
```

---

## Part 3: Account Factory -- The Tenant Onboarding Office

### The Analogy

In a building complex, the **leasing office** handles new tenant onboarding. A prospective tenant does not walk up to an empty lot and start building. Instead, they go to the leasing office, fill out a standardized application, and the office builds their unit according to pre-approved blueprints: same electrical wiring, same fire suppression system, same door locks. The tenant gets a move-in-ready unit that meets all building codes. The leasing office ensures consistency -- Building 47 has the same safety standards as Building 1.

### The Technical Reality

Account Factory is Control Tower's mechanism for provisioning new AWS accounts. It ensures every new account is:
1. Created within the Organization
2. Placed in the correct OU
3. Enrolled in Control Tower governance
4. Configured with baseline resources (CloudTrail, Config, IAM roles) via StackSets
5. Assigned an IAM Identity Center user/group with appropriate permissions
6. Optionally configured with a pre-defined VPC (CIDR, subnets, NAT gateways)

```
ACCOUNT FACTORY -- HOW NEW ACCOUNTS ARE PROVISIONED
=============================================================================

  OPTION 1: CONSOLE / SERVICE CATALOG
  ────────────────────────────────────

  Cloud Admin ──▶ Control Tower Console ──▶ Account Factory
       │              (or Service Catalog       │
       │               product launch)          │
       │                                        │
       │         ┌──────────────────────────────▼───────────────┐
       │         │  Account Factory creates:                     │
       │         │                                               │
       │         │  1. New AWS account (via Organizations API)   │
       │         │  2. Places account in specified OU            │
       │         │  3. Deploys StackSets:                        │
       │         │     ├── CloudTrail configuration              │
       │         │     ├── Config recorder + delivery channel    │
       │         │     ├── IAM roles for cross-account access    │
       │         │     └── Baseline security controls            │
       │         │  4. Applies all controls (SCPs + Config rules │
       │         │     + CFN hooks) from the target OU           │
       │         │  5. Configures IAM Identity Center access     │
       │         │  6. (Optional) Creates a VPC with your        │
       │         │     specified CIDR, subnets, NAT config       │
       │         │                                               │
       │         │  OUTPUT: A fully governed, move-in-ready      │
       │         │  account in ~30 minutes                       │
       │         └───────────────────────────────────────────────┘


  OPTION 2: ACCOUNT FACTORY FOR TERRAFORM (AFT)
  ──────────────────────────────────────────────

  Developer ──▶ Git push to ──▶ CodePipeline ──▶ Step Functions ──▶ Account
                account-request    triggers        orchestrates     Factory
                repo               pipeline        provisioning     API
                                                        │
       ┌────────────────────────────────────────────────▼───────────┐
       │  AFT Pipeline:                                              │
       │                                                             │
       │  1. Reads account request (Terraform .tf file)              │
       │  2. Calls Control Tower Account Factory API                 │
       │  3. Waits for account creation and enrollment               │
       │  4. Applies global customizations (all accounts)            │
       │  5. Applies account-specific customizations                 │
       │  6. Stores account metadata in DynamoDB                     │
       │  7. Sends notification via SNS                              │
       │                                                             │
       │  Each customization step runs Terraform in a CodeBuild      │
       │  container, so you can apply ANY Terraform configuration    │
       │  to the new account: VPCs, security groups, IAM roles,     │
       │  CloudWatch alarms -- anything you can write in HCL.       │
       │                                                             │
       └─────────────────────────────────────────────────────────────┘
```

### Account Factory Inputs

When you provision a new account (either through the console or AFT), you specify:

| Input | Description | Example |
|-------|-------------|---------|
| Account name | Friendly name | `app-a-production` |
| Account email | Unique email (root user) | `aws+app-a-prod@example.com` |
| Target OU | Where to place the account | `Workloads > Prod` |
| IAM Identity Center user/group | Who gets initial access | `ProdAdmins` group |
| VPC configuration (optional) | Pre-built networking | CIDR `10.42.0.0/16`, 3 AZs, NAT |

---

## Part 4: Account Factory for Terraform (AFT) -- GitOps Account Provisioning

### The Analogy

AFT is like a **city planning department that accepts building permits by mail (Git commits) instead of requiring you to walk to the counter (console)**. Mail a blueprint (push a Terraform file to a Git repo), and the department reviews it, pulls permits (calls Account Factory API), dispatches contractors (Step Functions pipeline), and sends you a certificate of occupancy (SNS notification) -- all without you setting foot in the office. Change the blueprint and mail an updated version, and the department modifies the building to match.

### The Technical Reality

AFT deploys its own infrastructure in a **dedicated AFT management account** (separate from the Organization management account). This keeps the pipeline infrastructure isolated.

```
AFT ARCHITECTURE
=============================================================================

  MANAGEMENT ACCOUNT                    AFT MANAGEMENT ACCOUNT
  ┌────────────────────┐               ┌──────────────────────────────────┐
  │                    │               │                                  │
  │  Control Tower     │◀──────────────│  AFT Pipeline Infrastructure     │
  │  Account Factory   │  API calls    │                                  │
  │  API               │               │  ┌──────────────────────────┐   │
  │                    │               │  │  CodeCommit / GitHub /   │   │
  └────────────────────┘               │  │  Bitbucket repos:        │   │
                                       │  │                          │   │
                                       │  │  1. account-request      │   │
                                       │  │  2. global-customizations│   │
                                       │  │  3. account-customizations│  │
                                       │  │  4. account-provisioning-│   │
                                       │  │     customizations       │   │
                                       │  └────────────┬─────────────┘   │
                                       │               │                 │
                                       │  ┌────────────▼─────────────┐   │
                                       │  │  CodePipeline            │   │
                                       │  │  ├── Source: Git repo    │   │
                                       │  │  ├── Build: CodeBuild    │   │
                                       │  │  │   (runs Terraform)    │   │
                                       │  │  └── Orchestration:      │   │
                                       │  │      Step Functions      │   │
                                       │  └────────────┬─────────────┘   │
                                       │               │                 │
                                       │  ┌────────────▼─────────────┐   │
                                       │  │  DynamoDB                │   │
                                       │  │  (account metadata,      │   │
                                       │  │   request tracking)      │   │
                                       │  └──────────────────────────┘   │
                                       │                                  │
                                       └──────────────────────────────────┘
```

### AFT Account Request -- What the Terraform Looks Like

This is where your Terraform knowledge directly applies. An AFT account request is a standard Terraform file:

```hcl
# ──────────────────────────────────────────────────────────
# AFT Account Request -- This file lives in the
# account-request Git repository
# ──────────────────────────────────────────────────────────

module "app_a_production" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail              = "aws+app-a-prod@example.com"
    AccountName               = "App-A-Production"
    ManagedOrganizationalUnit = "Workloads/Prod"    # OU path
    SSOUserEmail              = "cloud-admin@example.com"
    SSOUserFirstName          = "Cloud"
    SSOUserLastName           = "Admin"
  }

  # Custom fields stored in DynamoDB for use in customizations
  account_tags = {
    Environment = "production"
    Application = "app-a"
    CostCenter  = "engineering-42"
    ManagedBy   = "aft"
  }

  # Custom Terraform variables passed to account customizations
  account_customizations_name = "production-baseline"

  # Change management
  change_management_parameters = {
    change_requested_by = "platform-team"
    change_reason       = "New production account for App-A microservices"
  }
}

module "app_b_staging" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail              = "aws+app-b-staging@example.com"
    AccountName               = "App-B-Staging"
    ManagedOrganizationalUnit = "Workloads/Non-Prod"
    SSOUserEmail              = "dev-lead@example.com"
    SSOUserFirstName          = "Dev"
    SSOUserLastName           = "Lead"
  }

  account_tags = {
    Environment = "staging"
    Application = "app-b"
    CostCenter  = "engineering-43"
    ManagedBy   = "aft"
  }

  account_customizations_name = "non-production-baseline"

  change_management_parameters = {
    change_requested_by = "platform-team"
    change_reason       = "Staging environment for App-B"
  }
}
```

### AFT Customization Layers

AFT applies customizations in a specific order -- each layer is a separate Git repository:

```
AFT CUSTOMIZATION PIPELINE ORDER
=============================================================================

  1. ACCOUNT PROVISIONING CUSTOMIZATIONS (runs during initial creation only)
     ├── One-time setup that should not be repeated
     ├── Example: bootstrap IAM roles, initial resource creation
     └── Repo: aft-account-provisioning-customizations/

  2. GLOBAL CUSTOMIZATIONS (runs on ALL accounts)
     ├── Baseline configuration every account needs
     ├── Examples:
     │   ├── CloudWatch alarm for root user login
     │   ├── Default security group that blocks all traffic
     │   ├── VPC flow logs to centralized S3 bucket
     │   ├── SSM Session Manager configuration
     │   └── Cost anomaly detection alerts
     └── Repo: aft-global-customizations/

  3. ACCOUNT-SPECIFIC CUSTOMIZATIONS (runs per account or account group)
     ├── Configuration unique to this account or account type
     ├── Selected by the account_customizations_name field
     ├── Examples:
     │   ├── production-baseline: strict security groups, WAF, backup policies
     │   ├── non-production-baseline: relaxed settings, auto-shutdown
     │   └── sandbox-baseline: budget alarms, auto-cleanup Lambda
     └── Repo: aft-account-customizations/

  EACH REPO STRUCTURE:
  ─────────────────────
  aft-global-customizations/
  ├── api_helpers/
  │   ├── python/            # Custom Python scripts (optional)
  │   └── pre-api-helpers.sh # Run before Terraform
  │       post-api-helpers.sh# Run after Terraform
  └── terraform/
      ├── main.tf            # Your Terraform code
      ├── variables.tf
      └── providers.tf       # AFT injects provider config automatically
```

### Example Global Customization -- Terraform

```hcl
# ──────────────────────────────────────────────────────────
# Global customization: applied to EVERY AFT-managed account
# File: aft-global-customizations/terraform/main.tf
# ──────────────────────────────────────────────────────────

# CloudWatch alarm for root user console sign-in
resource "aws_cloudwatch_metric_alarm" "root_user_login" {
  alarm_name          = "root-user-console-login"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "RootUserConsoleLogin"
  namespace           = "CloudTrailMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "Alert when root user logs into the console"
  alarm_actions       = [var.security_sns_topic_arn]
}

# Default security group: deny all traffic
# (prevent accidental use of the default SG)
resource "aws_default_security_group" "default" {
  vpc_id = data.aws_vpc.default.id

  # No ingress or egress rules = deny all
  # This forces teams to create explicit security groups
}

# S3 public access block at the account level
resource "aws_s3_account_public_access_block" "block" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# EBS encryption by default
resource "aws_ebs_encryption_by_default" "enabled" {
  enabled = true
}
```

---

## Part 5: Drift Detection -- The Building Inspector's Photo Audit

### The Analogy

After the smart building platform installs fire alarms, security cameras, and locked doors, it does not just walk away. It hires a **building inspector who periodically photographs every room and compares them to the original blueprints**. If someone moved a wall, removed a smoke detector, or changed the locks without filing a permit, the inspector flags it in a report. The inspector does not fix the problem -- they document it and alert the building manager. This is drift detection.

### The Technical Reality

Control Tower monitors for drift in three categories:

```
DRIFT DETECTION -- WHAT CONTROL TOWER MONITORS
=============================================================================

  1. OU DRIFT
     ├── An OU registered with Control Tower is deleted
     ├── An OU is moved outside the Control Tower governance
     └── Detection: Control Tower flags the OU as "In drift"

  2. ACCOUNT DRIFT
     ├── An enrolled account is moved to a different OU
     │   (outside of Control Tower's Account Factory workflow)
     ├── An enrolled account is removed from the Organization
     └── Detection: Control Tower flags the account as "In drift"

  3. CONTROL DRIFT (most common)
     ├── A preventive control SCP is modified directly in Organizations
     │   (bypassing Control Tower's API)
     ├── A preventive control SCP is detached from an OU
     ├── A detective control Config rule is deleted or modified
     ├── A mandatory control is disabled (should not be possible, but
     │   direct API manipulation can cause this)
     └── Detection: Control Tower flags the control as "Drifted"

  WHAT HAPPENS WHEN DRIFT IS DETECTED:
  ─────────────────────────────────────
  ├── Control Tower dashboard shows a "Drift" banner
  ├── SNS notification sent (if configured)
  ├── The drifted resource is flagged
  ├── Resolution: either fix the drift manually or use
  │   "Re-register OU" / "Update landing zone" to reset
  └── Control Tower does NOT auto-remediate drift (unlike Terraform)
      -- it alerts and waits for you to resolve it

  IMPORTANT PARALLEL TO TERRAFORM:
  ═══════════════════════════════════
  Control Tower drift detection ≈ terraform plan detecting drift
  BUT: Terraform can auto-remediate with terraform apply
       Control Tower requires manual resolution (or re-registration)
```

---

## Part 6: Control Tower vs Doing It Yourself -- When to Use Each

This is the comparison that matters for architecture decisions and interviews: when should you use Control Tower, and when should you manage Organizations + SCPs directly?

```
CONTROL TOWER vs MANUAL ORGANIZATIONS + SCPs
=============================================================================

  DIMENSION              CONTROL TOWER                  MANUAL (ORGS + SCPs)
  ──────────────────    ──────────────────────────     ──────────────────────
  Setup time             Hours (guided wizard)          Days to weeks
  Account provisioning   Account Factory (console       aws organizations
                         or AFT)                        create-account + manual
                                                        baseline setup
  Controls               400+ pre-built controls        Write SCPs and Config
                         in a searchable catalog        rules from scratch
  Drift detection        Built-in dashboard and         Build your own with
                         SNS alerts                     Config rules + Lambda
  CloudTrail/Config      Auto-deployed via StackSets    Manually configure per
  baseline               to all accounts/regions        account or write your
                                                        own StackSets
  IAM Identity Center    Pre-configured during          Manual setup and
                         landing zone setup             configuration
  Customization          Limited to what CT supports;   Full control over
  flexibility            AFT adds Terraform power       everything; no
                                                        constraints
  Existing Organization  Can enable CT on existing      N/A -- you are
                         org (careful migration)        already doing this
  Multi-region           Governs selected "home         You deploy to exactly
  governance             region" + additional            the regions you choose
                         governed regions
  Terraform integration  AFT for accounts;              Full Terraform control
                         CT controls via AWS             over every resource
                         provider                        from day one
  Opinionated?           Yes -- CT makes choices        No -- you make every
                         for you (OU names, account     decision
                         structure, mandatory
                         controls)
  Best for               Organizations adopting AWS     Teams with mature
                         multi-account for the first    infrastructure-as-code
                         time; teams that want AWS      practices who want
                         best practices out of the      full control; complex
                         box; compliance-driven         environments that
                         environments                   diverge from CT's
                                                        opinions

  THE HONEST ASSESSMENT:
  ═════════════════════════════════════════════════════════════════
  Control Tower gives you 80% of a well-architected multi-account
  environment with 20% of the effort. The remaining 20% of
  customization (networking, advanced IAM, application-specific
  baselines) is where AFT and manual configuration fill the gap.

  If you are starting from scratch: USE CONTROL TOWER. You can
  always customize later. Starting with manual Organizations
  means building everything Control Tower gives you for free.

  If you have an existing mature Organization: EVALUATE CAREFULLY.
  Enabling Control Tower on an existing org requires enrolling
  existing accounts, which means accepting CT's mandatory controls
  and potentially disrupting existing SCPs.
```

---

## Part 7: Services Under the Hood -- What Control Tower Orchestrates

Control Tower is not a standalone service -- it is an orchestration layer that wires together six AWS services you already know or will learn this week:

```
CONTROL TOWER'S SERVICE DEPENDENCIES
=============================================================================

  ┌──────────────────────────────────────────────────────────────────┐
  │                     CONTROL TOWER                                │
  │                                                                  │
  │  Orchestrates and configures all of the following:              │
  │                                                                  │
  │  ┌────────────────────┐   ┌─────────────────────┐              │
  │  │  AWS Organizations │   │  IAM Identity Center │              │
  │  │                    │   │  (formerly AWS SSO)  │              │
  │  │  - Creates OUs     │   │                      │              │
  │  │  - Creates accounts│   │  - Enables SSO       │              │
  │  │  - Attaches SCPs   │   │  - Creates permission│              │
  │  │    (preventive     │   │    sets               │              │
  │  │     controls)      │   │  - Assigns users/     │              │
  │  │                    │   │    groups to accounts │              │
  │  └────────────────────┘   └─────────────────────┘              │
  │                                                                  │
  │  ┌────────────────────┐   ┌─────────────────────┐              │
  │  │  AWS Config        │   │  CloudFormation      │              │
  │  │                    │   │  StackSets           │              │
  │  │  - Deploys Config  │   │                      │              │
  │  │    rules (detective│   │  - Deploys baselines │              │
  │  │    controls)       │   │    to all enrolled   │              │
  │  │  - Config recorder │   │    accounts          │              │
  │  │    in every account│   │  - Pushes CloudTrail,│              │
  │  │  - Config delivery │   │    Config, IAM roles │              │
  │  │    to Log Archive  │   │  - CFN Hooks for     │              │
  │  │                    │   │    proactive controls │              │
  │  └────────────────────┘   └─────────────────────┘              │
  │                                                                  │
  │  ┌────────────────────┐   ┌─────────────────────┐              │
  │  │  AWS CloudTrail    │   │  AWS Service Catalog │              │
  │  │                    │   │                      │              │
  │  │  - Org trail to    │   │  - Account Factory   │              │
  │  │    Log Archive     │   │    uses SC to expose │              │
  │  │  - Multi-region    │   │    account creation  │              │
  │  │    coverage        │   │    as a "product"    │              │
  │  │  - Log file        │   │  - End users launch  │              │
  │  │    validation      │   │    the product to    │              │
  │  │                    │   │    request accounts  │              │
  │  └────────────────────┘   └─────────────────────┘              │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Best Practices for Rolling Out Controls

AWS recommends an incremental rollout strategy for controls. Here is the playbook:

```
CONTROL ROLLOUT STRATEGY
=============================================================================

  PHASE 1: LANDING ZONE SETUP (Day 1)
  ├── Accept mandatory controls (auto-enabled, no choice)
  ├── Review what mandatory controls enforce
  ├── Verify Log Archive and Audit accounts are functioning
  └── Confirm IAM Identity Center access works

  PHASE 2: ENABLE STRONGLY RECOMMENDED CONTROLS (Week 1)
  ├── Review the strongly recommended control catalog
  ├── Enable on a NON-PRODUCTION OU first (sandbox or dev)
  │   (same pattern as the Policy Staging OU from yesterday)
  ├── Validate no legitimate workloads break
  ├── Then enable on production OUs
  ├── Key controls to enable early:
  │   ├── Restrict root user access (preventive)
  │   ├── Detect public S3 buckets (detective)
  │   ├── Detect root user activity (detective)
  │   ├── Require encryption on EBS volumes (proactive)
  │   └── Restrict regions (preventive -- the SCP you wrote yesterday)
  └── Document which controls are enabled on which OUs

  PHASE 3: ALIGN ELECTIVE CONTROLS TO COMPLIANCE FRAMEWORK (Ongoing)
  ├── Map your compliance requirements (CIS, NIST, PCI, HIPAA) to
  │   Control Tower's control catalog
  ├── Enable relevant elective controls per OU
  ├── Use the "Controls by framework" filter in the console
  └── Review and update quarterly as new controls are released

  PHASE 4: CUSTOM CONTROLS (Advanced)
  ├── If CT's catalog does not cover your needs, create custom SCPs
  │   and attach them via Organizations (outside Control Tower)
  ├── Create custom Config rules for application-specific compliance
  ├── Use CT's dashboard alongside your custom controls
  └── Be aware: custom SCPs outside CT are not drift-detected by CT
```

---

## Part 9: Enrolling Existing Accounts -- The Retrofit Challenge

This is a common interview question: "How would you adopt Control Tower in an organization that already has 50 accounts?"

### The Analogy

Imagine the smart building platform being installed in a neighborhood that already has 50 buildings. Some have their own fire alarms. Some have non-standard wiring. The platform cannot just flip a switch -- it needs to inspect each building, install its sensors alongside (or in place of) the existing ones, and ensure nothing breaks during the retrofit.

### The Technical Reality

Control Tower can be enabled on an **existing** AWS Organization. The process involves two distinct operations:

```
ENROLLING EXISTING ACCOUNTS
=============================================================================

  STEP 1: REGISTER THE OU
  ────────────────────────
  "Register OU" brings an existing OU under Control Tower governance.
  This means:
  ├── Control Tower attaches its mandatory SCPs to the OU
  ├── StackSets deploy baselines (CloudTrail, Config, IAM roles)
  │   to every account in the OU, across all governed regions
  └── All controls enabled on that OU now apply to existing accounts

  STEP 2: ENROLL EACH ACCOUNT
  ────────────────────────────
  "Enroll Account" brings a specific existing account into
  Account Factory's management. This means:
  ├── The account gets the AWSControlTowerExecution IAM role
  ├── Account appears in the Account Factory inventory
  └── Account becomes manageable through CT console/API

  RISKS AND GOTCHAS:
  ══════════════════
  ├── SCP CONFLICTS: If existing accounts have custom SCPs, the
  │   mandatory SCPs from CT may conflict (e.g., if an existing SCP
  │   allows something CT's mandatory SCP denies)
  ├── CONFIG RECORDER CONFLICTS: If accounts already have an AWS
  │   Config recorder enabled, CT's StackSet deployment may fail
  │   because only one Config recorder per region is allowed --
  │   you must delete the existing recorder first
  ├── CLOUDTRAIL CONFLICTS: Similar to Config -- if an org trail
  │   already exists, CT may create a duplicate
  ├── SEQUENTIAL PROCESSING: Accounts are enrolled one at a time;
  │   enrolling 50 accounts takes hours, not minutes
  └── TESTING FIRST: Always register a non-production OU first,
      validate that baselines deploy cleanly, then move to
      production OUs
```

---

## Part 10: Landing Zone Versions, Updates, and Governed Regions

### Landing Zone Versions

Control Tower has a **versioned landing zone** (e.g., LZ 3.x). AWS periodically releases new versions that add features, update baselines, or change default controls.

- You must **manually trigger** landing zone updates from the Control Tower console
- Updates deploy new StackSet templates to all enrolled accounts
- Updates can be disruptive: new mandatory controls may block actions that were previously allowed
- Always review the release notes before updating
- You cannot skip versions -- updates are sequential

### Governed Regions vs Region Deny

These are two complementary but separate concepts that are commonly conflated on exams:

```
GOVERNED REGIONS vs REGION DENY -- TWO SEPARATE MECHANISMS
=============================================================================

  GOVERNED REGIONS (Control Tower concept)
  ────────────────────────────────────────
  ├── The set of AWS regions where Control Tower deploys its baselines
  │   (CloudTrail, Config recorder, IAM roles) via StackSets
  ├── Selected during landing zone setup; can be added later
  ├── More governed regions = higher cost (Config recorder runs
  │   in each governed region = more Config evaluations to pay for)
  └── Does NOT prevent API calls in non-governed regions

  REGION DENY SCP (Organizations/SCP concept)
  ────────────────────────────────────────────
  ├── An SCP that explicitly denies API calls in non-approved regions
  ├── Available as a strongly recommended Control Tower control
  │   ("Deny access to AWS based on the requested AWS Region")
  ├── This IS an SCP -- everything you learned yesterday applies
  └── DOES prevent API calls in denied regions

  THE DISTINCTION THAT MATTERS:
  ═══════════════════════════════
  Governed regions without a region-deny SCP = baselines are deployed
  in selected regions, but users CAN still create resources in
  non-governed regions (those resources just won't have Config/CloudTrail
  coverage).

  Governed regions WITH a region-deny SCP = baselines in selected
  regions AND users are blocked from creating resources elsewhere.

  Best practice: ALWAYS pair governed regions with the region-deny control.
```

---

## Part 11: Customizations for Control Tower (CfCT) vs AFT

The doc has covered AFT in depth, but there is a second customization mechanism that appears on the exam: **Customizations for AWS Control Tower (CfCT)**.

```
CfCT vs AFT -- TWO CUSTOMIZATION APPROACHES
=============================================================================

  CfCT (Customizations for Control Tower)
  ────────────────────────────────────────
  ├── WHAT: A solution that uses a manifest file to define SCPs and
  │   CloudFormation StackSets, deployed via CodePipeline
  ├── PURPOSE: Customize the LANDING ZONE INFRASTRUCTURE itself --
  │   deploy additional SCPs, Config rules, CloudFormation resources
  │   across OUs and accounts
  ├── MECHANISM: You define a manifest.yaml listing which StackSets
  │   and SCPs deploy to which OUs, then push to CodeCommit
  ├── USES: CloudFormation (not Terraform)
  └── BEST FOR: Organizations that want to extend CT's baseline
      with additional governance resources using CloudFormation

  AFT (Account Factory for Terraform)
  ────────────────────────────────────
  ├── WHAT: A GitOps pipeline for ACCOUNT PROVISIONING and
  │   per-account customization
  ├── PURPOSE: Provision new accounts and apply Terraform-based
  │   customizations to individual accounts
  ├── MECHANISM: Terraform files in Git repos, processed by
  │   CodePipeline + Step Functions + CodeBuild
  ├── USES: Terraform
  └── BEST FOR: Teams with Terraform expertise who want GitOps-based
      account vending

  THEY ARE COMPLEMENTARY, NOT COMPETING:
  ═══════════════════════════════════════
  CfCT customizes the governance plane (SCPs, baseline StackSets)
  AFT customizes the account plane (account provisioning, per-account config)
  Many organizations use both.
```

---

## Key Takeaways

### What Control Tower Is
1. **Control Tower is the opinionated orchestration layer on top of Organizations** -- it does not replace Organizations, SCPs, Config, or CloudFormation; it wires them together according to AWS best practices and provides a dashboard, drift detection, and Account Factory on top
2. **The landing zone is the pre-configured foundation** -- Security OU with Log Archive and Audit accounts, mandatory controls, CloudTrail organization trail, Config recorders in every account, and IAM Identity Center; this is what you would spend days building manually
3. **Control Tower is to Organizations what Terraform is to AWS APIs** -- a declarative, opinionated, drift-detecting layer that automates what you could do manually but probably should not

### Controls
4. **Three behaviors provide defense in depth, not a sequential chain** -- preventive controls (SCPs) block API calls at the authorization layer; detective controls (Config Rules) flag non-compliant resources after creation; proactive controls (CloudFormation Hooks) reject non-compliant templates at deploy time; all three operate independently and concurrently during a CloudFormation deployment
5. **Three guidance levels determine urgency** -- mandatory controls are auto-enabled and cannot be disabled; strongly recommended controls should be your first post-setup action; elective controls map to specific compliance requirements
6. **Proactive controls only work with CloudFormation** -- the standard Terraform AWS provider makes direct API calls that bypass proactive controls entirely; this is why you still need preventive and detective controls even when proactive controls exist
7. **Preventive controls are SCPs -- everything you learned yesterday applies** -- they do not affect the management account, do not affect service-linked roles, and evaluate at the API call level; Control Tower just manages the SCP lifecycle for you

### Account Factory and AFT
8. **Account Factory ensures every new account is born compliant** -- it provisions the account, places it in the correct OU, deploys baselines via StackSets, applies all controls, and configures IAM Identity Center access in a single workflow
9. **AFT is the GitOps bridge between Terraform and Control Tower** -- define account requests as Terraform files, push to Git, and a Step Functions pipeline provisions and customizes the account; this connects your 14 days of Terraform knowledge directly to multi-account governance
10. **AFT customizations are layered: provisioning, global, then account-specific** -- global customizations (security baselines, encryption defaults) apply to every account; account-specific customizations (networking, application config) vary by account type

### Operational Wisdom
11. **Drift detection alerts but does not auto-remediate** -- unlike Terraform's `apply`, Control Tower detects drift in OUs, accounts, and controls but requires manual resolution or re-registration to fix
12. **Roll out controls incrementally** -- mandatory controls on day one, strongly recommended controls on non-production OUs first (same as yesterday's Policy Staging OU pattern), then production OUs, then elective controls aligned to compliance frameworks
13. **Control Tower gives you 80% with 20% effort** -- if starting from scratch, use Control Tower; if you have a mature Organization with custom automation, evaluate carefully before enabling CT on existing resources
14. **Enrolling existing accounts is the hardest part of CT adoption** -- watch for Config recorder conflicts, SCP conflicts, and sequential processing limits; always register a non-production OU first
15. **Governed regions and region-deny SCPs are complementary, not the same** -- governed regions control where baselines deploy; the region-deny SCP controls where users can create resources; always pair them together
16. **CfCT and AFT serve different purposes** -- CfCT customizes the governance plane (additional SCPs, baseline StackSets); AFT customizes the account plane (account provisioning, per-account Terraform); many organizations use both

---

## Further Reading

- [What Is AWS Control Tower? -- AWS Docs](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html)
- [How AWS Control Tower Works -- AWS Docs](https://docs.aws.amazon.com/controltower/latest/userguide/how-control-tower-works.html)
- [Control Behavior and Guidance -- AWS Control Reference](https://docs.aws.amazon.com/controltower/latest/controlreference/control-behavior.html)
- [Designing an AWS Control Tower Landing Zone -- AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/designing-control-tower-landing-zone/introduction.html)
- [Account Factory for Terraform (AFT) Overview -- AWS Docs](https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html)
- [Control Tower Guardrails Overview -- Automat-it Blog](https://www.automat-it.com/blog/control-tower-guardrails-overview-preventive-detective-and-proactive/)
- [Best Practices for Applying Controls -- AWS Cloud Operations Blog](https://aws.amazon.com/blogs/mt/best-practices-for-applying-controls-with-aws-control-tower/)
- [AWS Organizations & SCPs (Yesterday's Doc)](./2026-03-02-aws-organizations-scps.md) -- the raw building materials that Control Tower orchestrates

---

*Written on March 3, 2026*
