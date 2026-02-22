# Terraform Cloud/Enterprise - The Central Architecture Firm

> Understanding Terraform Cloud and Enterprise through a real-world analogy of a central architecture firm that manages blueprints, enforces building codes, runs construction remotely, and maintains a private library of approved designs.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Terraform Cloud (TFC) | Architecture firm as a service (they own the building) |
| Terraform Enterprise (TFE) | Your own in-house architecture department |
| Workspace | Project folder for a specific building (state, vars, secrets) |
| Remote Run | Firm's construction crew builds for you (you don't lift a hammer) |
| Run Workflow | Plan → Cost Estimate → Code Review → Approval → Build |
| Speculative Plan | Architect's "what-if" sketch for PR review (can't build from it) |
| Sentinel | Building code inspector (must pass before construction) |
| Advisory policy | Inspector's suggestion ("consider better insulation") |
| Soft Mandatory policy | Inspector's warning ("fix this, or get manager approval") |
| Hard Mandatory policy | Inspector's stop order ("no construction until fixed") |
| Policy Set | Collection of building codes for a project type |
| Private Registry | Firm's private library of approved blueprints |
| Private Module | Pre-approved floor plan only your firm can use |
| VCS Integration | Blueprints stored in secure vault, auto-synced |
| Run Triggers | "When Building A is done, start Building B" |
| Cost Estimation | Budget preview before construction starts |
| Teams | Departments with different access levels |
| Agent Pools | Your own construction crews in private job sites |

---

## The Big Picture

> **Note:** In April 2024, HashiCorp rebranded **Terraform Cloud** to **HCP Terraform**. You may see both names in documentation and tooling. This doc uses "Terraform Cloud" / "TFC" as the concepts and features remain the same.

Imagine you're a **real estate developer** who builds infrastructure across multiple cities. Instead of hiring individual contractors and managing everything yourself, you partner with a **central architecture firm** that handles everything:

```
THE CENTRAL ARCHITECTURE FIRM MODEL
───────────────────────────────────────────────────────────────

WITHOUT TERRAFORM CLOUD (DIY Approach):
───────────────────────────────────────────────────────────────

  Developer Office
  │
  ├── Store blueprints locally
  ├── Manage your own vault combinations (secrets)
  ├── Run construction from your laptop
  ├── No building code enforcement
  ├── No central oversight
  └── Hope everyone follows the rules 🤞

  PROBLEMS:
  ├── "Who has the latest blueprint?"
  ├── "Did anyone review this before building?"
  ├── "Our credentials are on everyone's laptop!"
  └── "The intern just demolished production!"


WITH TERRAFORM CLOUD (Central Firm):
───────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────┐
  │              TERRAFORM CLOUD - Architecture Firm            │
  │                                                             │
  │   📁 WORKSPACES (Project Folders)                          │
  │   ├── NYC-Office-Prod     (state, secrets, variables)      │
  │   ├── NYC-Office-Staging                                   │
  │   └── Chicago-Warehouse                                    │
  │                                                             │
  │   🏗️ REMOTE RUNS (Construction Crews)                      │
  │   └── Builds happen HERE, not on developer laptops         │
  │                                                             │
  │   📋 SENTINEL (Building Code Inspector)                    │
  │   └── Every plan reviewed against company policies         │
  │                                                             │
  │   📚 PRIVATE REGISTRY (Approved Blueprint Library)         │
  │   └── Pre-approved designs for teams to reuse              │
  │                                                             │
  │   🔐 SECRETS VAULT                                          │
  │   └── Vault combinations stored securely, not on laptops   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
          │
          │  All developers connect to central firm
          │
    ┌─────┴─────┬─────────────┐
    ▼           ▼             ▼
  Alice       Bob          CI/CD
  (Developer) (Developer)  (Automation)

  BENEFITS:
  ├── Single source of truth for blueprints
  ├── All builds reviewed by inspector (Sentinel)
  ├── Credentials never leave the firm
  ├── Full audit trail of who built what
  └── Cost estimates before construction
```

---

## Core Components

### Terraform Cloud vs Enterprise - Renting vs Owning the Firm

```
CHOOSING YOUR ARCHITECTURE FIRM MODEL
───────────────────────────────────────────────────────────────

TERRAFORM CLOUD (TFC)
"Rent office space in HashiCorp's building"
───────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────┐
  │   HASHICORP BUILDING                                        │
  │                                                             │
  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
  │   │ Your Firm   │  │ Other Firm  │  │ Another     │        │
  │   │ (Your Org)  │  │ (Isolated)  │  │ (Isolated)  │        │
  │   │             │  │             │  │             │        │
  │   │ Workspaces  │  │ Workspaces  │  │ Workspaces  │        │
  │   │ Teams       │  │ Teams       │  │ Teams       │        │
  │   │ Policies    │  │ Policies    │  │ Policies    │        │
  │   └─────────────┘  └─────────────┘  └─────────────┘        │
  │                                                             │
  │   🏢 HashiCorp maintains the building                       │
  │   🔒 SOC2 certified, they handle security                   │
  │   💰 Free tier available, pay per user                      │
  │   🌐 Internet accessible                                    │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  ✅ Best for: Startups, small teams, standard compliance
  ✅ No infrastructure to manage
  ✅ Free tier for small teams
  ❌ Requires internet connectivity
  ❌ Data lives on HashiCorp's servers


TERRAFORM ENTERPRISE (TFE)
"Build your own architecture firm building"
───────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────┐
  │   YOUR COMPANY'S PRIVATE BUILDING                           │
  │   (On-prem data center or private cloud)                    │
  │                                                             │
  │   ┌─────────────────────────────────────────────────────┐   │
  │   │              YOUR TERRAFORM ENTERPRISE              │   │
  │   │                                                     │   │
  │   │   Same features as TFC, but:                        │   │
  │   │   • Runs in YOUR infrastructure                     │   │
  │   │   • YOUR security controls                          │   │
  │   │   • YOUR network (air-gapped OK)                    │   │
  │   │   • YOUR data residency                             │   │
  │   │                                                     │   │
  │   └─────────────────────────────────────────────────────┘   │
  │                                                             │
  │   🏗️ You maintain the building                              │
  │   🔒 Full control for FedRAMP, HIPAA, etc.                  │
  │   💰 License-based (expensive)                              │
  │   🔌 Can run completely offline                             │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  ✅ Best for: Regulated industries, government, large enterprise
  ✅ Air-gapped environments
  ✅ Full data control
  ❌ You manage infrastructure & updates
  ❌ Significant cost
```

**When to choose each:**

| Scenario | Choose |
|----------|--------|
| Startup, small team | TFC (Free tier) |
| Standard SaaS is acceptable | TFC |
| Must be air-gapped (no internet) | TFE |
| FedRAMP, HIPAA strict compliance | TFE |
| Government contracts | TFE |
| Want minimal ops overhead | TFC |
| Large enterprise, dedicated platform team | TFE |

---

### Remote Runs - The Firm Builds For You

Instead of running `terraform apply` on your laptop, Terraform Cloud runs it on their servers.

```
LOCAL EXECUTION vs REMOTE EXECUTION
───────────────────────────────────────────────────────────────

LOCAL (Traditional):
───────────────────────────────────────────────────────────────

  Developer Laptop
  │
  ├── Has AWS credentials (security risk!)
  ├── Runs: terraform plan
  │         terraform apply
  ├── Manages state file
  └── What if laptop is compromised?

  PROBLEMS:
  ├── Credentials scattered across laptops
  ├── No approval workflow
  ├── No audit trail
  ├── Different versions, different results
  └── "Works on my machine!"


REMOTE (Terraform Cloud):
───────────────────────────────────────────────────────────────

  Developer Laptop              Terraform Cloud
  │                             │
  ├── terraform plan ──────────►│
  │   (uploads config only)     │  1. Queue the run
  │                             │  2. Download config
  │                             │  3. Run plan (in TFC)
  │                             │  4. Cost estimation
  │◄─────────── Plan output ────│  5. Sentinel policies
  │                             │
  │   "Looks good!"             │
  │                             │
  ├── Approve (in UI) ─────────►│  6. Wait for approval
  │                             │  7. Run apply (in TFC)
  │                             │  8. Update state
  │◄─────────── Apply done ─────│
  │                             │
  └── No credentials on laptop! │

  BENEFITS:
  ├── Credentials only in TFC (not on laptops)
  ├── Consistent environment (same TF version)
  ├── Audit trail (who approved what)
  ├── Policy enforcement
  └── Cost estimate before you spend money
```

**The Remote Run Workflow:**

```
TERRAFORM CLOUD RUN STAGES
───────────────────────────────────────────────────────────────

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │   PLAN   │ ──► │   COST   │ ──► │ SENTINEL │
  │          │     │ ESTIMATE │     │ POLICIES │
  └──────────┘     └──────────┘     └──────────┘
       │                                  │
       │                                  ▼
       │                           ┌──────────┐
       │                           │ APPROVAL │
       │                           │ (manual) │
       │                           └──────────┘
       │                                  │
       │                                  ▼
       │                           ┌──────────┐
       └───────────────────────────│  APPLY   │
                                   │          │
                                   └──────────┘

  STAGE DETAILS:
  ───────────────────────────────────────────────────────────
  1. PLAN        → What will change?
  2. COST        → How much will it cost? ($$$)
  3. SENTINEL    → Does it pass company policies?
  4. APPROVAL    → Human confirms (for production)
  5. APPLY       → Make the changes
```

---

### Speculative Plans - The "What-If" Sketch

When someone opens a Pull Request, Terraform Cloud automatically runs a "speculative plan" - a preview that **cannot be applied**.

```
SPECULATIVE PLANS IN PR WORKFLOW
───────────────────────────────────────────────────────────────

  Developer                  GitHub                Terraform Cloud
  │                          │                     │
  ├── Create branch          │                     │
  │   (feature/add-bucket)   │                     │
  │                          │                     │
  ├── Push changes ─────────►│                     │
  │                          │                     │
  ├── Open Pull Request ────►│                     │
  │                          ├── Webhook ─────────►│
  │                          │                     │
  │                          │                     ├── 📋 SPECULATIVE
  │                          │                     │      PLAN
  │                          │                     │
  │                          │                     │   "What would
  │                          │                     │    change if we
  │                          │                     │    merged this?"
  │                          │                     │
  │                          │◄── Status Check ────┤
  │                          │    ✅ Plan succeeded │
  │                          │    (link to details) │
  │                          │                     │
  ├── Review plan in TFC ◄───┤                     │
  │   "Adds 1 S3 bucket"     │                     │
  │   "Estimated cost: $5"   │                     │
  │                          │                     │
  ├── Get approval           │                     │
  ├── Merge PR ─────────────►│                     │
  │                          ├── Webhook ─────────►│
  │                          │                     │
  │                          │                     ├── REAL RUN
  │                          │                     │   (Plan + Apply)
  │                          │                     │
  └── Done!                  │                     │


KEY POINT: Speculative Plan = PREVIEW ONLY
───────────────────────────────────────────────────────────────

  ❌ Cannot click "Apply" on a speculative plan
  ✅ Shows what WOULD happen
  ✅ Team reviews impact before approving PR
  ✅ Runs on every PR update automatically
```

---

### Sentinel - The Building Code Inspector

Sentinel is HashiCorp's policy-as-code framework. Think of it as a building code inspector who reviews every construction plan.

```
SENTINEL: POLICY ENFORCEMENT
───────────────────────────────────────────────────────────────

THE INSPECTION PROCESS:
───────────────────────────────────────────────────────────────

  Terraform Plan Output
          │
          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    SENTINEL INSPECTOR                       │
  │                                                             │
  │   Checks the plan against company policies:                 │
  │                                                             │
  │   □ "No public S3 buckets"                                  │
  │   □ "All resources must have tags"                          │
  │   □ "EC2 instances must use approved AMIs"                  │
  │   □ "Estimated cost must be under $10,000"                  │
  │   □ "No resources in restricted regions"                    │
  │                                                             │
  │   Results:                                                  │
  │   ├── ✅ All checks passed → Continue to apply              │
  │   └── ❌ Check failed → Block based on enforcement level    │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘


THREE ENFORCEMENT LEVELS:
───────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────┐
  │  ADVISORY                                                   │
  │  "Inspector's Suggestion"                                   │
  │                                                             │
  │  Inspector: "You should add fire sprinklers, but I won't    │
  │              stop you from building."                       │
  │                                                             │
  │  Behavior:  ⚠️ Warning logged, run CONTINUES                │
  │  Use case:  New policies, education, non-critical rules    │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  SOFT MANDATORY                                             │
  │  "Inspector's Warning - Manager Override OK"                │
  │                                                             │
  │  Inspector: "This violates code. I'm stopping construction  │
  │              unless a manager approves an exception."       │
  │                                                             │
  │  Behavior:  ❌ Blocks run, but authorized users can         │
  │                override with a comment                      │
  │  Use case:  Cost limits, general best practices             │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  HARD MANDATORY                                             │
  │  "Inspector's Stop Order"                                   │
  │                                                             │
  │  Inspector: "Absolutely not. This is a safety violation.    │
  │              No construction until you fix it."             │
  │                                                             │
  │  Behavior:  ❌ Blocks run, NO override by default           │
  │             (can enable policy set override if needed)      │
  │  Use case:  Security requirements, compliance, zero         │
  │             tolerance rules                                 │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

**Sentinel Policy Example:**

```sentinel
# policy: require-tags.sentinel

import "tfplan/v2" as tfplan

required_tags = ["Environment", "Owner", "CostCenter"]

# Find all managed resources being created or updated
all_resources = filter tfplan.resource_changes as _, rc {
    rc.mode is "managed" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

# Check that all required tags are present
tags_present = rule {
    all all_resources as _, resource {
        all required_tags as tag {
            resource.change.after.tags contains tag
        }
    }
}

# Main rule that must pass
main = rule {
    no_public_buckets
}
```

---

### Private Registry - The Approved Blueprint Library

Your organization's private catalog of modules that only your teams can access.

```
PRIVATE REGISTRY: APPROVED BLUEPRINTS
───────────────────────────────────────────────────────────────

PUBLIC REGISTRY (registry.terraform.io)
───────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────┐
  │   PUBLIC LIBRARY - Open to Everyone                         │
  │                                                             │
  │   📚 terraform-aws-modules/vpc/aws                          │
  │   📚 terraform-aws-modules/eks/aws                          │
  │   📚 (thousands of community modules)                       │
  │                                                             │
  │   ✅ Free, lots of options                                  │
  │   ❌ May not meet your compliance requirements              │
  │   ❌ No control over updates                                │
  │   ❌ Could have security issues                             │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘


PRIVATE REGISTRY (Your Terraform Cloud Org)
───────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────┐
  │   YOUR COMPANY'S PRIVATE LIBRARY                            │
  │   (Only your org members can access)                        │
  │                                                             │
  │   📚 acme-corp/vpc/aws (v2.3.0)                             │
  │      └── Your VPC with required encryption, logging,        │
  │          tagging pre-configured                             │
  │                                                             │
  │   📚 acme-corp/eks-cluster/aws (v1.5.0)                     │
  │      └── Approved EKS setup with security policies          │
  │                                                             │
  │   📚 acme-corp/rds-postgres/aws (v3.0.0)                    │
  │      └── RDS with encryption, backups, monitoring           │
  │                                                             │
  │   ✅ Pre-approved by security team                          │
  │   ✅ Consistent patterns across all teams                   │
  │   ✅ Your versioning, your release schedule                 │
  │   ✅ Built-in compliance (encryption, logging, tagging)     │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘


WHY PRIVATE REGISTRY?
───────────────────────────────────────────────────────────────

  1. STANDARDIZATION
     └── "All teams use OUR approved VPC module"
         (No random modules from the internet)

  2. GOVERNANCE
     └── "Security team reviewed and approved v2.3.0"
         (Compliance is built-in)

  3. DISCOVERABILITY
     └── "What modules do we have?"
         (Central catalog, easy to find)

  4. VERSION CONTROL
     └── "Lock to v2.0 until we validate v3.0"
         (Control upgrade timing)

  5. REDUCE DUPLICATION
     └── "Platform team already built this, just use it!"
         (Don't reinvent the wheel)
```

**Using Private Registry modules:**

```hcl
# Using a public registry module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  # ...
}

# Using YOUR private registry module
module "vpc" {
  source  = "app.terraform.io/acme-corp/vpc/aws"
  version = "2.3.0"
  # ...
}
```

---

## How Things Connect

```
THE COMPLETE TERRAFORM CLOUD WORKFLOW
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   DEVELOPER WORKFLOW                                        │
│   │                                                         │
│   │  1. Create feature branch                               │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  2. Write Terraform code                                │
│   │     │  (using private registry modules)                 │
│   │     ▼                                                   │
│   │  3. Push & Open PR                                      │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  ┌─────────────────────────────────────────────────┐    │
│   │  │  SPECULATIVE PLAN (automatic)                   │    │
│   │  │  • Shows what would change                      │    │
│   │  │  • Cost estimate                                │    │
│   │  │  • Sentinel policy check                        │    │
│   │  │  • CANNOT be applied                            │    │
│   │  └─────────────────────────────────────────────────┘    │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  4. Team reviews plan in PR                             │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  5. Merge PR to main                                    │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  ┌─────────────────────────────────────────────────┐    │
│   │  │  REAL RUN (triggered by merge)                  │    │
│   │  │                                                 │    │
│   │  │  ┌──────┐  ┌──────┐  ┌────────┐  ┌─────────┐   │    │
│   │  │  │ PLAN │─►│ COST │─►│SENTINEL│─►│APPROVAL │   │    │
│   │  │  └──────┘  └──────┘  └────────┘  └─────────┘   │    │
│   │  │                                       │         │    │
│   │  │                                       ▼         │    │
│   │  │                                 ┌─────────┐     │    │
│   │  │                                 │  APPLY  │     │    │
│   │  │                                 └─────────┘     │    │
│   │  │                                                 │    │
│   │  └─────────────────────────────────────────────────┘    │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  6. Infrastructure deployed! ✅                         │
│   │                                                         │
│   ▼                                                         │
│   AUDIT TRAIL: Every run logged with who, what, when        │
│                                                             │
└─────────────────────────────────────────────────────────────┘


WORKSPACE ORGANIZATION:
───────────────────────────────────────────────────────────────

  Organization: acme-corp
  │
  ├── 📁 Workspace: vpc-prod
  │   ├── State (vpc-prod.tfstate)
  │   ├── Variables (region, cidr_block)
  │   ├── Secrets (AWS credentials)
  │   └── Runs (history of all plans/applies)
  │
  ├── 📁 Workspace: vpc-staging
  │   └── (same structure, different values)
  │
  ├── 📁 Workspace: eks-prod
  │   └── Run Trigger: Wait for vpc-prod
  │
  └── 📁 Workspace: app-prod
      └── Run Trigger: Wait for eks-prod
```

---

## Key Takeaways

1. **TFC vs TFE: Rent vs Own**
   - TFC: HashiCorp manages everything (SaaS)
   - TFE: You host it yourself (air-gapped, compliance)

2. **Remote Runs = Credentials never leave TFC**
   - Plan/Apply runs in TFC, not on developer laptops
   - Consistent environment, audit trail, approval workflow

3. **Speculative Plans = Safe PR previews**
   - Auto-triggered on PRs
   - Shows impact, cannot be applied
   - Team reviews before merge

4. **Sentinel = Policy-as-Code**
   - Advisory (warn), Soft (block, overridable), Hard (block, strict)
   - Runs after plan, before apply
   - Security, compliance, cost control

5. **Private Registry = Your approved blueprints**
   - Standardization across teams
   - Pre-approved, security-reviewed modules
   - Version control, discoverability

6. **Run Workflow: Plan → Cost → Sentinel → Approval → Apply**
   - Every stage provides a checkpoint
   - No surprises in production

---

*Written on January 30, 2026*
