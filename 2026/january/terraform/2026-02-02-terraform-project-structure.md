# Terraform Project Structure - The Filing System for Blueprints

> Understanding Terraform project organization through a real-world analogy of an architecture firm organizing its blueprint library across offices.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Mono-repo | One central filing cabinet for all teams |
| Multi-repo | Each team has their own filing cabinet |
| Root module | The main project folder on your desk |
| Child modules | Reusable templates in the library |
| `environments/` folder | Separate drawers for dev/staging/prod sites |
| `modules/` folder | Shared blueprint library |
| `main.tf` | The project's master blueprint |
| `variables.tf` | "Customize these options" form |
| `outputs.tf` | "Here's what we built" summary |
| `versions.tf` | "Works with these tool versions" |
| `terraform.tfvars` | Filled-in customization form for this site |
| `backend.tf` | "Store records in this vault" |
| Directory-based envs | Separate folders = separate construction sites |
| Workspace-based envs | Same folder, different tabs in the ledger |
| Naming conventions | Standardized label system |
| `locals.tf` | Shorthand notes and calculations |

---

## The Big Picture

Imagine your **architecture firm has grown** from 2 architects to 50 architects across 8 departments. Each department designs infrastructure for different clients. The firm faces a critical question:

> "How do we organize our blueprints so everyone can find what they need, teams don't step on each other's toes, and we maintain consistency?"

```
THE FILING SYSTEM PROBLEM
───────────────────────────────────────────────────────────────

CHAOS (No structure):
───────────────────────────────────────────────────────────────

  Firm Shared Drive
  │
  ├── vpc_thing_v2_FINAL.tf
  ├── vpc_thing_v2_FINAL_FINAL.tf
  ├── johns_vpc_dont_touch.tf
  ├── prod_stuff/
  ├── prod_stuff_backup/
  ├── old_dont_delete/
  └── new_approach_test/

  PROBLEMS:
  ├── "Which one is the real production VPC?"
  ├── "Who owns this file?"
  ├── "Can I delete this?"
  └── "I accidentally modified the wrong environment!"


ORGANIZED (Proper structure):
───────────────────────────────────────────────────────────────

  infrastructure/
  │
  ├── environments/           ← Construction sites
  │   ├── dev/               ← Development site
  │   ├── staging/           ← Testing site
  │   └── prod/              ← Production site
  │
  └── modules/               ← Reusable blueprints
      ├── vpc/
      ├── eks/
      └── rds/

  BENEFITS:
  ├── Clear separation of environments
  ├── Reusable patterns
  ├── Know exactly where to look
  └── Hard to accidentally modify wrong environment
```

---

## Core Components

### Mono-Repo vs Multi-Repo - One Cabinet or Many?

The first big decision: should all teams share one repository, or should each team have their own?

```
MONO-REPO: One Central Filing Cabinet
───────────────────────────────────────────────────────────────

  infrastructure/                    ← Single repo, all teams
  │
  ├── team-networking/
  │   ├── vpc/
  │   └── transit-gateway/
  │
  ├── team-compute/
  │   ├── eks/
  │   └── ec2/
  │
  ├── team-data/
  │   ├── rds/
  │   └── dynamodb/
  │
  └── modules/                       ← Shared by all teams
      ├── vpc/
      └── security-group/


MULTI-REPO: Each Team Has Their Own Cabinet
───────────────────────────────────────────────────────────────

  team-networking-infra/            ← Team A's repo
  ├── vpc/
  └── transit-gateway/

  team-compute-infra/               ← Team B's repo
  ├── eks/
  └── ec2/

  team-data-infra/                  ← Team C's repo
  ├── rds/
  └── dynamodb/

  shared-modules/                   ← Shared module repo
  └── (published to private registry)
```

**Decision Matrix:**

| Factor | Mono-Repo | Multi-Repo |
|--------|-----------|------------|
| Team size | Small-medium (<20) | Large (20+) |
| Ownership | Shared ownership OK | Clear ownership needed |
| Release cycles | Coordinated releases | Independent releases |
| Cross-cutting changes | Easy (one PR) | Hard (multiple PRs) |
| Blast radius | Large (one bad merge affects all) | Small (isolated) |
| Module sharing | Local paths (easy) | Registry/git tags (overhead) |
| CI/CD complexity | Complex (trigger filtering) | Simple (per-repo) |
| Access control | Harder (branch protection) | Easier (repo permissions) |

```
RULE OF THUMB:
───────────────────────────────────────────────────────────────
• Tightly coupled infrastructure, small team → Mono-repo
• Many teams, clear boundaries, independent releases → Multi-repo
```

---

### Directory-Based vs Workspace-Based Environments

Once you've chosen mono or multi-repo, how do you separate dev/staging/prod?

```
APPROACH 1: Directory-Based (RECOMMENDED)
───────────────────────────────────────────────────────────────

"Each environment is a separate folder"

terraform/
├── environments/
│   ├── dev/                         ← cd here for dev
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf              ← State: s3://bucket/dev/
│   │   └── terraform.tfvars        ← Dev-specific values
│   │
│   ├── staging/                     ← cd here for staging
│   │   ├── main.tf
│   │   ├── backend.tf              ← State: s3://bucket/staging/
│   │   └── terraform.tfvars
│   │
│   └── prod/                        ← cd here for prod
│       ├── main.tf
│       ├── backend.tf              ← State: s3://bucket/prod/
│       └── terraform.tfvars
│
└── modules/
    └── (shared modules)


APPROACH 2: Workspace-Based
───────────────────────────────────────────────────────────────

"Same folder, switch workspace"

terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf                       ← State: s3://bucket/${workspace}/
├── dev.tfvars
├── staging.tfvars
└── prod.tfvars

# Usage:
terraform workspace select dev
terraform apply -var-file=dev.tfvars

terraform workspace select prod      ← Dangerous! Same folder!
terraform apply -var-file=prod.tfvars
```

**Why Directory-Based is Preferred:**

| Aspect | Directory-Based | Workspace-Based |
|--------|-----------------|-----------------|
| **Safety** | Must `cd prod/` to affect prod | Same folder, easy mistake |
| **Visibility** | See all files for that env | Files mixed, vars differ |
| **Divergence** | Easy to add prod-only resources | Requires conditionals |
| **State isolation** | Completely separate | Same backend, different key |
| **CI/CD** | Clear paths per env | Workspace switching in scripts |

```
SAFETY COMPARISON:
───────────────────────────────────────────────────────────────

Directory-based:
  Developer: "I want to destroy dev resources"
  $ cd terraform/environments/dev
  $ terraform destroy
  ✅ Only dev affected (you're IN the dev folder)

Workspace-based:
  Developer: "I want to destroy dev resources"
  $ cd terraform
  $ terraform workspace select dev   ← Did I run this?
  $ terraform destroy
  ⚠️ Hope you selected the right workspace!
```

---

### Standard Module Structure

Every Terraform module (including root modules) should follow this file structure:

```
STANDARD MODULE FILES
───────────────────────────────────────────────────────────────

modules/vpc/
│
├── main.tf              ← Resources live here
│   │
│   │  resource "aws_vpc" "this" { ... }
│   │  resource "aws_subnet" "private" { ... }
│   │  resource "aws_subnet" "public" { ... }
│   │
│
├── variables.tf         ← Input parameters
│   │
│   │  variable "cidr_block" {
│   │    description = "VPC CIDR block"
│   │    type        = string
│   │  }
│   │
│
├── outputs.tf           ← Values exposed to callers
│   │
│   │  output "vpc_id" {
│   │    value = aws_vpc.this.id
│   │  }
│   │
│
├── versions.tf          ← Version constraints (IMPORTANT!)
│   │
│   │  terraform {
│   │    required_version = ">= 1.5.0"
│   │    required_providers {
│   │      aws = {
│   │        source  = "hashicorp/aws"
│   │        version = ">= 5.0.0"
│   │      }
│   │    }
│   │  }
│   │
│
├── locals.tf            ← Computed values, naming patterns
│   │
│   │  locals {
│   │    name_prefix = "${var.project}-${var.environment}"
│   │  }
│   │
│
├── data.tf              ← Data sources (optional)
│   │
│   │  data "aws_availability_zones" "available" {}
│   │
│
└── README.md            ← Documentation


ROOT MODULE (Environment folder) adds:
───────────────────────────────────────────────────────────────

environments/prod/
│
├── (all the above, plus...)
│
├── backend.tf           ← State storage configuration
│   │
│   │  terraform {
│   │    backend "s3" {
│   │      bucket = "company-terraform-state"
│   │      key    = "prod/infrastructure.tfstate"
│   │    }
│   │  }
│   │
│
└── terraform.tfvars     ← Environment-specific values
    │
    │  environment   = "prod"
    │  instance_size = "m5.xlarge"
    │  node_count    = 10
```

---

### Naming Conventions

Consistent naming prevents confusion and enables automation.

```
NAMING CONVENTION STANDARDS
───────────────────────────────────────────────────────────────

RESOURCE NAMES (Terraform identifiers):
───────────────────────────────────────────────────────────────

  # In MODULES with single resource of a type → use "this"
  resource "aws_vpc" "this" { }
  resource "aws_s3_bucket" "this" { }

  # In ROOT MODULES or multiple resources → use descriptive names
  resource "aws_s3_bucket" "logs" { }
  resource "aws_s3_bucket" "artifacts" { }
  resource "aws_s3_bucket" "backups" { }


RESOURCE NAMES (AWS/Cloud names):
───────────────────────────────────────────────────────────────

  Pattern: {company}-{project}-{environment}-{resource}-{region}

  locals {
    name_prefix = "${var.company}-${var.project}-${var.environment}"
  }

  resource "aws_s3_bucket" "logs" {
    bucket = "${local.name_prefix}-logs-${var.region}"
    # Result: "acme-payments-prod-logs-us-east-1"
  }


VARIABLE NAMES:
───────────────────────────────────────────────────────────────

  # Use snake_case
  variable "instance_count" { }      ✅
  variable "instanceCount" { }       ❌ (camelCase)
  variable "instance-count" { }      ❌ (kebab-case)


MODULE NAMES:
───────────────────────────────────────────────────────────────

  # Registry format: terraform-{provider}-{name}
  terraform-aws-vpc
  terraform-aws-eks-cluster
  terraform-google-network


FILE NAMES:
───────────────────────────────────────────────────────────────

  # Standard files
  main.tf, variables.tf, outputs.tf, versions.tf

  # Split by resource type (large modules)
  vpc.tf, subnets.tf, route_tables.tf, security_groups.tf


STATE FILE KEYS:
───────────────────────────────────────────────────────────────

  Pattern: {project}/{environment}/{component}.tfstate

  Examples:
  ├── payments/prod/network.tfstate
  ├── payments/prod/compute.tfstate
  └── payments/dev/network.tfstate
```

---

## How Things Connect

```
COMPLETE PROJECT STRUCTURE
───────────────────────────────────────────────────────────────

infrastructure/
│
├── modules/                              ← REUSABLE BLUEPRINTS
│   │
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── README.md
│   │
│   ├── eks/
│   │   └── ...
│   │
│   └── rds/
│       └── ...
│
├── environments/                         ← CONSTRUCTION SITES
│   │
│   ├── dev/
│   │   ├── main.tf           ─────────┐
│   │   ├── variables.tf               │
│   │   ├── outputs.tf                 │  Uses modules
│   │   ├── versions.tf                │  with dev settings
│   │   ├── backend.tf                 │
│   │   └── terraform.tfvars  ◄────────┘
│   │
│   ├── staging/
│   │   └── (same structure, staging values)
│   │
│   └── prod/
│       └── (same structure, prod values)
│
├── .gitignore                            ← Ignore .terraform/, *.tfstate
├── .terraform-version                    ← Pin Terraform version (tfenv)
└── README.md                             ← Project documentation


WORKFLOW:
───────────────────────────────────────────────────────────────

  1. Developer works on module
     $ cd modules/vpc
     $ (make changes)

  2. Test in dev environment
     $ cd environments/dev
     $ terraform plan
     $ terraform apply

  3. Promote to staging
     $ cd ../staging
     $ terraform plan
     $ terraform apply

  4. Promote to production
     $ cd ../prod
     $ terraform plan
     $ (PR review required)
     $ terraform apply
```

---

## Key Takeaways

1. **Mono-repo vs Multi-repo depends on team size and ownership**
   - Small team, shared ownership → Mono-repo
   - Large org, clear boundaries → Multi-repo

2. **Directory-based environment separation is safer**
   - Must `cd` into environment folder
   - Harder to accidentally affect wrong environment
   - Easier to add environment-specific resources

3. **Standard file structure is non-negotiable**
   - `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`
   - Add `backend.tf` and `terraform.tfvars` for root modules

4. **Consistent naming enables automation**
   - Resources: `{company}-{project}-{env}-{type}-{region}`
   - State keys: `{project}/{env}/{component}.tfstate`

5. **Module versioning prevents breaking changes**
   - New optional variable → minor version bump
   - New required variable → major version bump

6. **`random_id` is state-persisted, not regenerated**
   - Safe for bucket names
   - Only changes on destroy/taint/keeper change

---

*Written on February 2, 2026*
