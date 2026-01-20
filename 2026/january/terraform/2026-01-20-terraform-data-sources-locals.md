# Terraform Data Sources & Locals - The City Records Office & Architect's Notebook

> Understanding Terraform data sources, locals, and dependency management through a real-world analogy of an architect consulting city records and keeping organized notes on their blueprints.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Data source | Looking up existing city records (buildings, zones, permits) |
| Resource | Constructing a new building (you manage it) |
| Data source (read-only) | Reading records at city hall (you can't modify them) |
| `data "aws_ami"` | "Find me the latest approved building material catalog" |
| `data "aws_vpc"` | "Look up the existing neighborhood layout" |
| `data "aws_availability_zones"` | "Which construction zones are available in this city?" |
| `data "aws_caller_identity"` | "What's my contractor license number?" |
| `terraform_remote_state` | Reading another architect's completed building specs |
| Locals | Architect's shorthand notes on the blueprint margin |
| `local.common_tags` | "Use these standard labels on everything I build" |
| Locals (computed once) | Notes written in pen (can't be changed by others) |
| Variables (input) | Client's requirements form (they fill it out) |
| `depends_on` | "Don't start construction until permits are approved" |
| Implicit dependency | Terraform auto-detects: "Need foundation before walls" |
| Explicit dependency | Manual note: "Wait for power company, even though not obvious" |
| Data source timing (plan) | Looking up records before construction starts |
| Data source timing (apply) | Looking up records after prerequisite is built |

---

## The Big Picture

Imagine you're an **architect designing a new building** in an established city. You don't build everything from scratch—you need to work with what already exists:

```
THE ARCHITECT'S INFORMATION SOURCES
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                     YOUR DESIGN OFFICE                       │
│                                                              │
│   📋 YOUR BLUEPRINTS (resources)                             │
│   │   "Buildings I'm creating and managing"                  │
│   │                                                          │
│   ├── 🏛️ CITY RECORDS OFFICE (data sources)                  │
│   │   "Look up existing infrastructure I don't manage"       │
│   │   • Existing neighborhoods (VPCs)                        │
│   │   • Available construction zones (AZs)                   │
│   │   • Approved material catalogs (AMIs)                    │
│   │   • My contractor license (caller identity)              │
│   │                                                          │
│   ├── 📓 ARCHITECT'S NOTEBOOK (locals)                       │
│   │   "My shorthand notes, computed values, reusable labels" │
│   │   • Common tags for all buildings                        │
│   │   • Calculated values (name prefixes, formatted IDs)     │
│   │   • Combined specifications                              │
│   │                                                          │
│   └── 📁 OTHER ARCHITECTS' FILES (remote state)              │
│       "Read completed specs from infrastructure team"        │
│       • VPC IDs they created                                 │
│       • Subnet configurations                                │
│       • Security group references                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Without these information sources:**
- You'd have to know every existing building's address by heart
- You'd repeat the same calculations on every blueprint page
- You couldn't reference infrastructure built by other teams
- You'd hardcode values that might change (like available zones)

**With these sources:**
- Query city records dynamically: "Give me all available zones"
- Write notes once, reference everywhere
- Share information across projects cleanly
- Your blueprints stay current even when city changes

---

## Core Components

### Data Sources - The City Records Office

A data source is like **visiting the city records office** to look up information about existing infrastructure. You're not building anything—you're just reading what's already there.

```
CITY RECORDS OFFICE VISIT
─────────────────────────

YOU: "I need information about the downtown neighborhood"

CLERK: "Let me look that up..."
       ┌─────────────────────────────────────┐
       │ RECORD: Downtown Neighborhood       │
       │                                     │
       │ ID: vpc-0abc123def456               │
       │ CIDR: 10.0.0.0/16                   │
       │ Subnets: 4                          │
       │ Created: 2024-01-15                 │
       │ Status: Active                      │
       └─────────────────────────────────────┘

YOU: "Perfect, I'll build my new office there!"

KEY POINT: You READ the record. You don't MODIFY it.
```

**Terraform equivalent:**
```hcl
# "Look up the existing VPC named 'production'"
data "aws_vpc" "existing" {
  tags = {
    Name = "production"
  }
}

# Now use that information in your resource
resource "aws_subnet" "my_subnet" {
  vpc_id     = data.aws_vpc.existing.id      # ← From city records!
  cidr_block = "10.0.1.0/24"
}
```

#### Data Source vs Resource - The Key Difference

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   RESOURCE (You Build & Manage)     DATA SOURCE (You Read Only) │
│   ─────────────────────────────     ─────────────────────────── │
│                                                                 │
│   resource "aws_vpc" "new" {        data "aws_vpc" "existing" { │
│     cidr_block = "10.0.0.0/16"        tags = {                  │
│   }                                     Name = "production"     │
│                                       }                         │
│   ↓                                 }                           │
│   Terraform will:                   ↓                           │
│   • CREATE it                       Terraform will:             │
│   • UPDATE it                       • READ it                   │
│   • DELETE it                       • That's it!                │
│   • Track in state                  • Query every plan/apply    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Common Data Sources - Your Frequent City Hall Visits

```hcl
# 1. "Which construction zones are available?"
data "aws_availability_zones" "available" {
  state = "available"
}
# Returns: ["us-east-1a", "us-east-1b", "us-east-1c", ...]

# 2. "Find the latest approved building material (AMI)"
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
# Returns: ami-0abc123... (latest Amazon Linux 2)

# 3. "What's my contractor license number?"
data "aws_caller_identity" "current" {}
# Returns: account_id = "123456789012"

# 4. "What city am I building in?"
data "aws_region" "current" {}
# Returns: name = "us-east-1"

# 5. "Look up the existing production VPC"
data "aws_vpc" "production" {
  tags = {
    Environment = "production"
  }
}
# Returns: id = "vpc-0abc123...", cidr_block = "10.0.0.0/16"
```

**Using them together:**
```hcl
resource "aws_instance" "server" {
  ami               = data.aws_ami.amazon_linux.id
  instance_type     = "t3.micro"
  availability_zone = data.aws_availability_zones.available.names[0]
  subnet_id         = data.aws_subnet.existing.id
  
  tags = {
    Account = data.aws_caller_identity.current.account_id
    Region  = data.aws_region.current.name
  }
}
```

---

### Locals - The Architect's Notebook

Locals are like **shorthand notes in the margin of your blueprint**. You calculate or define something once, then reference it everywhere.

```
ARCHITECT'S NOTEBOOK
────────────────────

Page 1 of Blueprint:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   MY SHORTHAND NOTES:                                       │
│   ───────────────────                                       │
│                                                             │
│   • Project prefix: "acme-prod"                             │
│   • Common tags: {Env=prod, Team=platform, ManagedBy=TF}    │
│   • Full name format: "{prefix}-{component}-{env}"          │
│                                                             │
│   Now I can write:                                          │
│   "Build server using [common tags] named [prefix]-web"     │
│                                                             │
│   Instead of:                                               │
│   "Build server with Env=prod, Team=platform, ManagedBy=TF  │
│    named acme-prod-web"                                     │
│                                                             │
│   ✅ Shorter                                                │
│   ✅ Change once, updates everywhere                        │
│   ✅ Reduces copy-paste errors                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Terraform equivalent:**
```hcl
locals {
  # Simple value
  environment = "production"
  
  # Computed value
  name_prefix = "${var.project}-${local.environment}"
  
  # Complex structure
  common_tags = {
    Environment = local.environment
    Project     = var.project
    ManagedBy   = "terraform"
    Owner       = "platform-team"
  }
  
  # Combining data source output
  first_az = data.aws_availability_zones.available.names[0]
}

# Now use everywhere!
resource "aws_instance" "web" {
  ami               = data.aws_ami.amazon_linux.id
  instance_type     = "t3.micro"
  availability_zone = local.first_az
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}

resource "aws_instance" "api" {
  ami               = data.aws_ami.amazon_linux.id
  instance_type     = "t3.micro"
  availability_zone = local.first_az
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-api"
  })
}
```

#### Locals vs Variables - Notes vs Client Requirements

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   VARIABLES (Input)                 LOCALS (Internal)           │
│   ─────────────────                 ───────────────             │
│                                                                 │
│   📝 Client fills out form          📓 Architect's own notes    │
│                                                                 │
│   variable "environment" {          locals {                    │
│     type    = string                  name_prefix = "acme-${    │
│     default = "dev"                     var.environment}"       │
│   }                                   upper_env = upper(        │
│                                         var.environment)        │
│   • Provided by USER                }                           │
│   • Can change per run                                          │
│   • terraform.tfvars               • Computed INTERNALLY        │
│   • -var flag                      • Fixed after evaluation     │
│   • Environment variables          • Not exposed to users       │
│                                    • local.name_prefix          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### When to Use Locals

```
USE LOCALS WHEN:
────────────────

1. AVOID REPETITION
   ❌ tags = { Environment = var.env, Project = var.project }  # 10 times!
   ✅ tags = local.common_tags                                  # once defined

2. SIMPLIFY COMPLEX EXPRESSIONS
   ❌ cidr = "${var.base_cidr}.${count.index * 16}.0/20"       # hard to read
   ✅ cidr = local.subnet_cidrs[count.index]                   # precomputed

3. COMBINE DATA SOURCE OUTPUTS
   ❌ subnet_id = data.aws_subnets.private.ids[0]              # repeated
   ✅ subnet_id = local.primary_subnet                         # clear name

4. CREATE DERIVED VALUES
   ✅ locals {
        is_production = var.environment == "prod"
        instance_type = local.is_production ? "t3.large" : "t3.micro"
      }
```

---

### Remote State - Reading Another Architect's Completed Specs

When teams split infrastructure across multiple Terraform projects, you need a way to share information.

```
MULTI-TEAM ARCHITECTURE
───────────────────────

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   INFRASTRUCTURE TEAM              APPLICATION TEAM         │
│   ────────────────────            ──────────────────        │
│                                                             │
│   📋 network/                     📋 app/                   │
│   ├── main.tf                     ├── main.tf               │
│   │   • VPC                       │   • EC2 instances       │
│   │   • Subnets                   │   • Load balancer       │
│   │   • NAT Gateway               │   • Auto scaling        │
│   │                               │                         │
│   └── outputs.tf                  └── data.tf               │
│       output "vpc_id"                 data "terraform_      │
│       output "private_subnets"           remote_state"      │
│       output "public_subnets"                ↓              │
│              │                        "Give me the VPC ID   │
│              │                         from infra team"     │
│              │                               │              │
│              └───────────────────────────────┘              │
│                     STATE FILE SHARING                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Infrastructure team outputs (network project):**
```hcl
# network/outputs.tf
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "The VPC ID for application deployments"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "Private subnet IDs for internal resources"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "Public subnet IDs for load balancers"
}
```

**Application team consumes (app project):**
```hcl
# app/data.tf
data "terraform_remote_state" "network" {
  backend = "s3"
  
  config = {
    bucket = "company-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# app/main.tf
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  # Use network team's subnet!
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
  
  tags = {
    Name = "app-server"
  }
}
```

#### Remote State Security Concern ⚠️

```
⚠️  IMPORTANT SECURITY NOTE
────────────────────────────

terraform_remote_state requires READ ACCESS to the entire state file,
even though you only access outputs.

State files may contain:
• Database passwords
• API keys
• Private IP addresses
• Sensitive resource attributes

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ALTERNATIVES TO REMOTE STATE:                             │
│                                                             │
│   1. SSM Parameter Store                                    │
│      • Infra team writes: aws_ssm_parameter                 │
│      • App team reads: data "aws_ssm_parameter"             │
│                                                             │
│   2. Secrets Manager                                        │
│      • For sensitive values                                 │
│                                                             │
│   3. Data Sources                                           │
│      • Look up resources by tags/name instead of state      │
│      • data "aws_vpc" { tags = { Name = "production" } }    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### depends_on - The Construction Permit System

Terraform automatically builds a **dependency graph** based on references. But sometimes dependencies are hidden.

```
AUTOMATIC vs EXPLICIT DEPENDENCIES
──────────────────────────────────

IMPLICIT (Terraform auto-detects):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   resource "aws_subnet" "main" {                            │
│     vpc_id = aws_vpc.main.id    ← Terraform sees this!      │
│   }                                                         │
│                                                             │
│   Terraform: "Ah, subnet needs VPC. I'll create VPC first." │
│                                                             │
└─────────────────────────────────────────────────────────────┘

EXPLICIT (Hidden dependency):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   resource "aws_instance" "app" {                           │
│     ami           = "ami-12345"                             │
│     instance_type = "t3.micro"                              │
│                                                             │
│     # No reference to IAM role, but app NEEDS S3 access!    │
│     # The role attachment is a side effect Terraform        │
│     # can't see from this resource.                         │
│                                                             │
│     depends_on = [aws_iam_role_policy_attachment.s3_access] │
│   }                                                         │
│                                                             │
│   resource "aws_iam_role_policy_attachment" "s3_access" {   │
│     role       = aws_iam_role.app.name                      │
│     policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnly" │
│   }                                                         │
│                                                             │
│   Terraform: "I don't see a connection... oh wait,          │
│               you told me explicitly. Got it!"              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**When to use `depends_on`:**
```hcl
# 1. IAM role must be attached before EC2 instance starts
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy_attachment.app_policy]
}

# 2. VPC endpoint must exist before Lambda in VPC
resource "aws_lambda_function" "processor" {
  # ...
  depends_on = [aws_vpc_endpoint.s3]
}

# 3. Database must be ready before app server starts
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_db_instance.main]
}
```

#### Why Avoid Overusing depends_on

```
⚠️  DON'T OVERUSE depends_on
─────────────────────────────

Terraform builds an optimized dependency graph for PARALLEL creation.

WITH implicit dependencies (references):
┌─────────────────────────────────────────────────────────────┐
│   VPC ──→ Subnet ──→ EC2                                    │
│     ↘                                                       │
│       Security Group ───────→ (parallel with subnet)        │
│                                                             │
│   Terraform: "I can create SG while waiting for subnet!"    │
└─────────────────────────────────────────────────────────────┘

WITH unnecessary depends_on:
┌─────────────────────────────────────────────────────────────┐
│   VPC ──→ Subnet ──→ SG ──→ EC2                             │
│                                                             │
│   Terraform: "Fine, I'll wait for each one... slowly..."   │
└─────────────────────────────────────────────────────────────┘

RULE: Use depends_on ONLY when there's no reference but
      a real dependency exists (like IAM policy side effects).
```

---

### Data Source Timing - When Does Terraform Read Records?

```
DATA SOURCE EVALUATION TIMING
─────────────────────────────

SCENARIO 1: No dependencies on unbuilt resources
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   data "aws_availability_zones" "available" {               │
│     state = "available"                                     │
│   }                                                         │
│                                                             │
│   → READ DURING: terraform plan ✅                          │
│   → Values available immediately                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘

SCENARIO 2: Depends on resource being created
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   resource "aws_instance" "server" {                        │
│     ami           = "ami-12345"                             │
│     instance_type = "t3.micro"                              │
│   }                                                         │
│                                                             │
│   data "aws_instance" "server_info" {                       │
│     instance_id = aws_instance.server.id  ← Doesn't exist! │
│   }                                                         │
│                                                             │
│   → READ DURING: terraform apply ⏳                         │
│   → Plan shows: (known after apply)                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**What you see in plan output:**
```
# data.aws_instance.server_info will be read during apply
# (config refers to values not yet known)
 <= data "aws_instance" "server_info" {
      + ami                         = (known after apply)
      + instance_type               = (known after apply)
      + private_ip                  = (known after apply)
    }
```

---

## How Things Connect

```
THE COMPLETE INFORMATION FLOW
─────────────────────────────

┌──────────────────────────────────────────────────────────────────────┐
│                         YOUR TERRAFORM PROJECT                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. EXTERNAL LOOKUPS (data sources)                                  │
│     ┌─────────────────────────────────────────┐                      │
│     │ data "aws_vpc" "existing" { ... }       │                      │
│     │ data "aws_ami" "latest" { ... }         │                      │
│     │ data "aws_availability_zones" { ... }   │                      │
│     │ data "terraform_remote_state" { ... }   │                      │
│     └───────────────────┬─────────────────────┘                      │
│                         │                                            │
│                         ▼                                            │
│  2. INTERNAL CALCULATIONS (locals)                                   │
│     ┌─────────────────────────────────────────┐                      │
│     │ locals {                                │                      │
│     │   vpc_id      = data.aws_vpc.existing.id│                      │
│     │   first_az    = data.aws_azs...names[0] │                      │
│     │   name_prefix = "${var.project}-${var.env}" │                  │
│     │   common_tags = { ... }                 │                      │
│     │ }                                       │                      │
│     └───────────────────┬─────────────────────┘                      │
│                         │                                            │
│                         ▼                                            │
│  3. RESOURCE CREATION                                                │
│     ┌─────────────────────────────────────────┐                      │
│     │ resource "aws_instance" "app" {         │                      │
│     │   subnet_id         = local.subnet_id   │  ← Uses locals       │
│     │   availability_zone = local.first_az    │                      │
│     │   ami               = data.aws_ami...id │  ← Uses data source  │
│     │   tags              = local.common_tags │                      │
│     │                                         │                      │
│     │   depends_on = [aws_iam_role_policy...] │  ← Explicit dep      │
│     │ }                                       │                      │
│     └─────────────────────────────────────────┘                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Data sources = read-only queries**
   - Look up existing infrastructure you don't manage
   - Evaluated during `plan` (unless depends on uncreated resource)
   - Common: `aws_ami`, `aws_vpc`, `aws_availability_zones`, `aws_caller_identity`

2. **Locals = internal computed values**
   - Define once, use everywhere (DRY principle)
   - Module-scoped (not visible outside)
   - Great for: common tags, name prefixes, complex expressions

3. **Locals vs Variables**
   - Variables = user input (from outside)
   - Locals = internal calculations (computed inside)

4. **Remote state = cross-project data sharing**
   - Read outputs from another Terraform project
   - ⚠️ Requires read access to entire state file (security concern)
   - Alternatives: SSM Parameter Store, data sources by tags

5. **depends_on = explicit hidden dependencies**
   - Use only when Terraform can't detect the dependency
   - Overusing slows down parallel resource creation

---

*Written on January 20, 2026*
