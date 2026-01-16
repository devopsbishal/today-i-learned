# Terraform Modules Part 2 - The Architectural Blueprint Library (Advanced)

> Continuing the architecture firm analogy: versioning blueprints, refactoring buildings, publishing to city records, and quality inspections.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Version constraint `~> 2.0` | "Use any 2.x building code, but not 3.0" |
| Version constraint `= 2.0.0` | "Use EXACTLY the 2019 building code, no exceptions" |
| `moved` block | Change of address form - same building, new name |
| Publishing to registry | Submitting your floor plan to the city's public library |
| Private registry | Your firm's internal blueprint vault (employee access only) |
| Semantic versioning | v1.0.0 = Major.Minor.Patch (Blueprint edition system) |
| `terraform init -upgrade` | "Get the latest approved version from the library" |
| Dependency injection | Pre-wired connection points vs hardcoded wiring |
| Module testing | Building inspection before occupancy permit |
| Sensitive outputs | Sealed envelope with security codes |
| Ephemeral values | Self-destructing documents (never stored!) |
| `ephemeral = true` | "This message will self-destruct after reading" |
| Write-only arguments | One-time PIN that's used but never recorded |
| Provider configuration | Which construction crew builds in which city |
| `count` vs `for_each` | Numbered units vs named units (Apt 1 vs Apt "Penthouse") |

---

## The Story Continues...

Remember our architecture firm with the Blueprint Library? Business is booming! But now we face new challenges:

```
GROWING PAINS AT THE ARCHITECTURE FIRM
──────────────────────────────────────

❓ "Which version of the 3-bedroom floor plan should I use?"
   → Version Constraints

❓ "We renamed 'Building A' to 'Sunrise Tower' - will it be demolished and rebuilt?"
   → The moved block

❓ "Other firms want to use our amazing floor plans!"
   → Publishing to Registry

❓ "Our floor plans are proprietary - employees only!"
   → Private Registry

❓ "How do we know our floor plans actually work?"
   → Module Testing

❓ "The safe combination shouldn't be visible on the spec sheet!"
   → Sensitive Outputs
```

---

## Version Constraints - Building Code Editions

### The Problem: Which Blueprint Version?

Your firm has been improving the 3-bedroom floor plan for years:

```
3-BEDROOM FLOOR PLAN VERSION HISTORY
────────────────────────────────────
v1.0.0 (2020) - Original design
v1.1.0 (2020) - Added optional balcony
v1.2.0 (2021) - Energy-efficient windows
v1.2.1 (2021) - Fixed bathroom vent issue
v2.0.0 (2022) - MAJOR REDESIGN: Open concept kitchen
v2.1.0 (2023) - Smart home wiring
v3.0.0 (2024) - MAJOR REDESIGN: Split-level option

⚠️ Problem: Project started with v1.x can't suddenly use v3.0!
   The foundation is different!
```

### The Solution: Version Constraints

Just like building codes have edition years, you specify which versions are acceptable:

```
┌─────────────────────────────────────────────────────────────────┐
│              BLUEPRINT REQUEST FORM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Floor Plan: 3-Bedroom Apartment                                 │
│                                                                  │
│  Version Requirement (choose one):                               │
│                                                                  │
│  ○ "= 2.0.0"    Use EXACTLY v2.0.0, nothing else                │
│                 "I need the 2022 edition specifically"           │
│                                                                  │
│  ○ "~> 2.0"     Use any 2.x version (2.0.0, 2.1.0, 2.9.0)       │
│                 but NOT 3.0.0                                    │
│                 "Any 2.x edition is fine, they're compatible"    │
│                                                                  │
│  ○ ">= 2.0.0"   Use 2.0.0 or newer (including 3.0, 4.0...)      │
│                 "2022 or newer, I'll adapt"                      │
│                                                                  │
│  ○ "~> 2.1.0"   Use 2.1.x only (2.1.0, 2.1.1, 2.1.5)            │
│                 but NOT 2.2.0                                    │
│                 "Only patch updates, no new features"            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**In Terraform:**

```hcl
# Pessimistic constraint (RECOMMENDED)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"    # Any 5.x, but not 6.0
}

# Exact version (very strict)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "= 5.0.0"   # ONLY 5.0.0
}

# Range constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 4.0.0, < 6.0.0"  # 4.x or 5.x
}
```

### Semantic Versioning - The Blueprint Edition System

```
VERSION: 2.1.3
         │ │ │
         │ │ └── PATCH (bug fix)
         │ │     "Fixed typo in measurements"
         │ │     "Corrected vent placement"
         │ │     ✓ Always safe to upgrade
         │ │
         │ └──── MINOR (new feature, backwards compatible)
         │       "Added optional skylight"
         │       "New balcony option"
         │       ✓ Usually safe, new features available
         │
         └────── MAJOR (breaking changes)
                 "Complete redesign - open concept"
                 "Different foundation required"
                 ⚠️ Existing buildings may need modifications!
```

**Choosing the right constraint:**

| Environment | Recommended | Why |
|-------------|-------------|-----|
| Production | `= 5.0.0` or `~> 5.0.0` | Stability, predictability |
| Development | `~> 5.0` | Get latest features in major version |
| Testing | `>= 5.0.0` | Test compatibility with newer versions |
| Never | No constraint | Chaos! Could get v99.0.0 tomorrow |

---

## The `moved` Block - Change of Address Form

### The Problem: Renaming Causes Demolition!

Your firm built "Building A" last year. The client now wants to rename it to "Sunrise Tower":

```
WITHOUT PROPER PAPERWORK:
─────────────────────────

City Records:
  "Building A" exists at 123 Main St → KNOWN

Rename Request:
  "Delete Building A, Create Sunrise Tower"

City Response:
  1. DEMOLISH Building A (destroy)
  2. BUILD Sunrise Tower (create)
  
Client: "WAIT! I just wanted to change the name, 
        not demolish my $10M building!"
```

### The Solution: Change of Address Form

```
┌─────────────────────────────────────────────────────────────────┐
│              CHANGE OF ADDRESS / RENAME FORM                     │
│              (No Demolition Required)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  This form certifies that:                                       │
│                                                                  │
│  PREVIOUS NAME/ADDRESS:  Building A                              │
│                          ─────────────────                       │
│                                                                  │
│  NEW NAME/ADDRESS:       Sunrise Tower                           │
│                          ─────────────────                       │
│                                                                  │
│  ☑ This is the SAME building, just renamed                       │
│  ☑ No demolition required                                        │
│  ☑ Update city records only                                      │
│                                                                  │
│  City Clerk: "Understood! I'll update the records."              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**In Terraform:**

```hcl
# BEFORE: Module was named "server"
# AFTER: Renamed to "web_server"

module "web_server" {
  source = "./modules/ec2-instance"
  # ... configuration
}

# The "change of address form"
moved {
  from = module.server
  to   = module.web_server
}
```

**What Terraform sees:**

```bash
$ terraform plan

Terraform will perform the following actions:

  # module.server has moved to module.web_server
    resource "aws_instance" "this" {
        id = "i-abc123"
        # (no changes to infrastructure)
    }

Plan: 0 to add, 0 to change, 0 to destroy.
# ✅ Just a rename, no demolition!
```

### Common `moved` Scenarios

**1. Renaming a module:**
```hcl
moved {
  from = module.old_name
  to   = module.new_name
}
```

**2. Moving a resource into a module:**
```hcl
# Resource was in root, now inside a module
moved {
  from = aws_instance.web
  to   = module.web_server.aws_instance.this
}
```

**3. Renaming a resource inside a module:**
```hcl
# Inside the module
moved {
  from = aws_instance.main
  to   = aws_instance.this
}
```

**4. Changing from count to for_each:**
```hcl
# Before: module.servers[0], module.servers[1]
# After: module.servers["web"], module.servers["api"]

moved {
  from = module.servers[0]
  to   = module.servers["web"]
}

moved {
  from = module.servers[1]
  to   = module.servers["api"]
}
```

**Analogy:**
```
moved BLOCK = CHANGE OF ADDRESS FORM

┌────────────────────────────────────────┐
│  "The building at module.server        │
│   is now known as module.web_server"   │
│                                        │
│  Same building. New name.              │
│  Update records. No demolition.        │
└────────────────────────────────────────┘
```

---

## Publishing to Registry - City Blueprint Library

### The Concept: Sharing Your Designs

Your firm created an amazing floor plan. Other architects want to use it:

```
BEFORE (Sharing via USB drives):
───────────────────────────────
Architect A: "Can you email me the VPC floor plan?"
Architect B: "Which version? I have v2.3 from last month..."
Architect C: "Mine says v2.1... is that outdated?"

CHAOS! No single source of truth!
```

```
AFTER (City Blueprint Library):
───────────────────────────────
┌─────────────────────────────────────────────────────────────────┐
│                 CITY BLUEPRINT LIBRARY                           │
│                 registry.terraform.io                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📁 terraform-aws-modules/vpc/aws                                │
│     ├── Version: 5.0.0 (latest)                                  │
│     ├── Downloads: 50 million                                    │
│     ├── Documentation: ✓                                         │
│     └── Verified: ✓ (HashiCorp Partner)                          │
│                                                                  │
│  📁 your-company/webapp-stack/aws                                │
│     ├── Version: 2.1.0 (latest)                                  │
│     ├── Downloads: 5,000                                         │
│     └── Documentation: ✓                                         │
│                                                                  │
│  Anyone can browse, download, and use these floor plans!         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Publishing Requirements

```
TO SUBMIT YOUR FLOOR PLAN TO THE CITY LIBRARY:
──────────────────────────────────────────────

1. NAMING CONVENTION (Required)
   Repository must be named: terraform-<PROVIDER>-<NAME>
   
   ✓ terraform-aws-vpc
   ✓ terraform-google-network
   ✗ my-cool-module (wrong format!)

2. STANDARD STRUCTURE (Required)
   terraform-aws-vpc/
   ├── main.tf           ← The floor plan
   ├── variables.tf      ← Customization options
   ├── outputs.tf        ← Specifications
   ├── versions.tf       ← Compatibility info
   ├── README.md         ← User manual
   └── LICENSE           ← Terms of use

3. VERSION TAGS (Required)
   Use semantic versioning: v1.0.0, v1.1.0, v2.0.0
   
   $ git tag v1.0.0
   $ git push origin v1.0.0

4. DOCUMENTATION (Required)
   - Variable descriptions
   - Output descriptions
   - Usage examples
   - README with overview

5. EXAMPLES FOLDER (Recommended)
   examples/
   ├── simple/           ← Basic usage
   ├── complete/         ← All features
   └── multi-region/     ← Advanced use case
```

**The publishing flow:**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Your Repo     │     │    Sign In      │     │    Published!   │
│   on GitHub     │────►│  to Registry    │────►│   Available to  │
│                 │     │  Connect Repo   │     │   Everyone      │
│ terraform-aws-  │     │                 │     │                 │
│ my-module/      │     │  Click Publish  │     │  source = "     │
│   ├── main.tf   │     │                 │     │  yourorg/       │
│   ├── v1.0.0    │     │                 │     │  my-module/aws" │
│   └── README.md │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## Private Registry - The Firm's Internal Vault

### The Concept: Proprietary Blueprints

Some floor plans are trade secrets - only employees should access them:

```
┌─────────────────────────────────────────────────────────────────┐
│              YOUR FIRM'S PRIVATE BLUEPRINT VAULT                 │
│              app.terraform.io/your-company                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🔒 EMPLOYEE ACCESS ONLY                                         │
│                                                                  │
│  📁 your-company/proprietary-vpc/aws                             │
│     └── Our special VPC design with trade secrets                │
│                                                                  │
│  📁 your-company/compliant-eks/aws                               │
│     └── EKS setup that meets our security requirements           │
│                                                                  │
│  📁 your-company/database-stack/aws                              │
│     └── Standard database deployment pattern                     │
│                                                                  │
│  Access requires:                                                │
│  • Employee credentials                                          │
│  • API token in ~/.terraformrc                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Setting up access:**

```hcl
# ~/.terraformrc (your employee badge)

credentials "app.terraform.io" {
  token = "your-api-token"
}
```

**Using private modules:**

```hcl
module "vpc" {
  source  = "app.terraform.io/your-company/vpc/aws"
  version = "2.0.0"
  
  # ... configuration
}
```

**Other private sources:**

```hcl
# Private GitHub (via SSH - uses your SSH keys)
module "vpc" {
  source = "git@github.com:your-company/terraform-modules.git//vpc?ref=v1.0.0"
}

# S3 bucket (private, uses AWS credentials)
module "vpc" {
  source = "s3::https://your-modules.s3.amazonaws.com/vpc/v1.0.0.zip"
}
```

---

## Dependency Injection - Pre-Wired Connection Points

### The Problem: Hardcoded Wiring

Imagine a floor plan with hardcoded electrical connections:

```
BAD FLOOR PLAN (Hardcoded):
───────────────────────────
┌─────────────────────────────────────────┐
│  APARTMENT FLOOR PLAN                   │
│                                         │
│  Electrical: Connect to Panel #47       │  ← Hardcoded!
│  Water: Connect to Main Line #12        │  ← Hardcoded!
│  Gas: Connect to Meter #89              │  ← Hardcoded!
│                                         │
│  ⚠️ This floor plan ONLY works in       │
│     Building A, Floor 3!                │
│                                         │
│  ⚠️ Cannot reuse in other buildings!    │
└─────────────────────────────────────────┘
```

```
GOOD FLOOR PLAN (Connection Points):
────────────────────────────────────
┌─────────────────────────────────────────┐
│  APARTMENT FLOOR PLAN                   │
│                                         │
│  Electrical: Connect to [ PANEL_ID ]    │  ← Variable!
│  Water: Connect to [ WATER_LINE_ID ]    │  ← Variable!
│  Gas: Connect to [ GAS_METER_ID ]       │  ← Variable!
│                                         │
│  ✅ Works in ANY building!              │
│  ✅ Just provide the connection IDs     │
└─────────────────────────────────────────┘
```

### In Terraform Terms

**❌ Anti-pattern: Module looks up its own dependencies**

```hcl
# modules/ec2-instance/main.tf - BAD!

# Module finds its own VPC - not flexible!
data "aws_vpc" "lookup" {
  tags = {
    Name = "main-vpc"  # Hardcoded! What if there's no "main-vpc"?
  }
}

resource "aws_instance" "this" {
  subnet_id = data.aws_subnet.lookup.id  # Looked up internally
  # ...
}
```

**✅ Best practice: Dependencies injected via variables**

```hcl
# modules/ec2-instance/variables.tf - GOOD!

variable "subnet_id" {
  type        = string
  description = "Subnet ID where instance will be deployed"
}

variable "vpc_id" {
  type        = string
  description = "VPC ID for security group"
}
```

```hcl
# modules/ec2-instance/main.tf - GOOD!

resource "aws_instance" "this" {
  subnet_id = var.subnet_id  # Injected from outside!
  # ...
}
```

**Root module wires the connections:**

```hcl
# Root module - YOU control the wiring

module "networking" {
  source = "./modules/vpc"
}

module "web_server" {
  source = "./modules/ec2-instance"
  
  # Inject the connections
  vpc_id    = module.networking.vpc_id
  subnet_id = module.networking.private_subnet_ids[0]
}

module "api_server" {
  source = "./modules/ec2-instance"
  
  # Same module, DIFFERENT connections!
  vpc_id    = module.networking.vpc_id
  subnet_id = module.networking.private_subnet_ids[1]
}
```

**Visual comparison:**

```
❌ HARDCODED (Module knows too much)
┌─────────────────────────────────┐
│  EC2 Module                     │
│  ┌────────────────────────────┐ │
│  │ data "aws_vpc" (lookup)    │ │  ← "I'll find my own VPC"
│  │ data "aws_subnet" (lookup) │ │  ← "I'll find my own subnet"
│  │                            │ │
│  │ Inflexible! Knows too much!│ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘

✅ DEPENDENCY INJECTION (Module receives connections)
┌──────────────┐         ┌──────────────┐
│  VPC Module  │────────►│  EC2 Module  │
│              │         │              │
│  output:     │         │  input:      │
│   vpc_id  ───┼────────►│   vpc_id     │
│   subnet_id ─┼────────►│   subnet_id  │
│              │         │              │
│  "Here are   │         │  "Just tell  │
│   the IDs"   │         │   me where"  │
└──────────────┘         └──────────────┘
```

---

## count vs for_each - Numbered vs Named Units

### The Problem with count (Numbered Units)

```
APARTMENT BUILDING WITH NUMBERED UNITS:
───────────────────────────────────────
Floor 1: [0] [1] [2]
         Apt Apt Apt

Tenant wants to remove Apt [1]...

BEFORE: [0]=Smith, [1]=Jones, [2]=Wilson
AFTER:  [0]=Smith, [1]=Wilson  ← Wilson shifted from [2] to [1]!

City records say Wilson is at [2], but now they're at [1]!
The city wants to DEMOLISH [2] and REBUILD at [1]!

Wilson: "I just wanted Jones to leave, not rebuild my apartment!"
```

### The Solution: for_each (Named Units)

```
APARTMENT BUILDING WITH NAMED UNITS:
────────────────────────────────────
Floor 1: ["A"] ["B"] ["C"]
          Apt   Apt   Apt

Tenant wants to remove Apt "B"...

BEFORE: ["A"]=Smith, ["B"]=Jones, ["C"]=Wilson
AFTER:  ["A"]=Smith, ["C"]=Wilson  ← Wilson stays at "C"!

Only Apt "B" is affected. No shifting!
```

**In Terraform:**

```hcl
# ❌ count - Index based (problematic)
module "servers" {
  count  = 3
  source = "./modules/ec2"
}
# Creates: module.servers[0], module.servers[1], module.servers[2]
# Remove middle one? Everything shifts!

# ✅ for_each - Key based (stable)
module "servers" {
  for_each = {
    web = { type = "t2.small" }
    api = { type = "t2.medium" }
    db  = { type = "t2.large" }
  }
  source = "./modules/ec2"
  
  instance_type = each.value.type
  name          = each.key
}
# Creates: module.servers["web"], module.servers["api"], module.servers["db"]
# Remove "api"? Only "api" is destroyed!
```

**Key differences:**

| Aspect | `count` | `for_each` |
|--------|---------|------------|
| Identifier | Number [0], [1], [2] | String key ["web"], ["api"] |
| Remove middle | ❌ Shifts all indexes | ✅ Only affects that key |
| Add to middle | ❌ Shifts subsequent | ✅ Just adds new key |
| Input type | Number | Map or Set |
| **Recommendation** | Simple on/off toggles | Almost everything else |

---

## Sensitive Outputs - Sealed Envelopes

### The Concept: Some Specs Are Secret

After building, some specifications shouldn't be publicly visible:

```
REGULAR SPECIFICATION SHEET:
────────────────────────────
┌─────────────────────────────────────────┐
│  BUILDING SPECIFICATIONS                │
├─────────────────────────────────────────┤
│  Address: 123 Main Street               │  ← Public
│  Floors: 10                             │  ← Public
│  Units: 50                              │  ← Public
│  Elevator Access Code: 4829             │  ← WAIT! SECRET!
│  Safe Combination: 36-24-36             │  ← WAIT! SECRET!
└─────────────────────────────────────────┘

Anyone reading this knows the codes! 😱
```

```
WITH SEALED ENVELOPES:
──────────────────────
┌─────────────────────────────────────────┐
│  BUILDING SPECIFICATIONS                │
├─────────────────────────────────────────┤
│  Address: 123 Main Street               │
│  Floors: 10                             │
│  Units: 50                              │
│  Elevator Access Code: [SEALED ENVELOPE]│  ← Hidden
│  Safe Combination: [SEALED ENVELOPE]    │  ← Hidden
└─────────────────────────────────────────┘

Codes are there, but not visible unless you open the envelope.
```

**In Terraform:**

```hcl
# outputs.tf

output "endpoint" {
  value       = aws_db_instance.this.endpoint
  description = "Database endpoint"
  # Not sensitive - endpoints are generally okay
}

output "password" {
  value       = random_password.db.result
  description = "Database password"
  sensitive   = true  # ← SEALED ENVELOPE!
}
```

**CLI behavior:**

```bash
$ terraform output

endpoint = "mydb.abc123.us-east-1.rds.amazonaws.com"
password = <sensitive>  # ← Hidden!

# To see the sensitive value:
$ terraform output -raw password
MySecretP@ssw0rd!
```

**⚠️ Important:** Sensitive values are still in the state file (in plaintext). Use:
- Remote state with encryption (S3 + KMS)
- Terraform Cloud/Enterprise
- Careful access controls

---

### Ephemeral Values - Self-Destructing Documents (Terraform 1.10+)

The `sensitive` argument hides values from CLI output, but they're **still stored in state**. What if you want secrets that **never touch the state file at all**?

```
COMPARISON: SEALED ENVELOPE vs SELF-DESTRUCTING DOCUMENT
─────────────────────────────────────────────────────────

SENSITIVE (Sealed Envelope):
┌─────────────────────────────────────────┐
│  Password: [SEALED ENVELOPE]            │  ← Hidden from view
│                                         │
│  But the envelope is STORED in the      │
│  filing cabinet (state file)!           │
│  Anyone with cabinet access can open it │
└─────────────────────────────────────────┘

EPHEMERAL (Self-Destructing Document):
┌─────────────────────────────────────────┐
│  Password: 💨 *poof*                    │  ← NEVER stored!
│                                         │
│  The document self-destructs after      │
│  being read. Nothing in the cabinet!    │
│  🔥 Used during operation, then gone    │
└─────────────────────────────────────────┘
```

**Three ways to use ephemeral values:**

#### 1. Ephemeral Variables (Terraform 1.10+)

```hcl
# Variables that NEVER touch the state file
variable "database_password" {
  description = "Password for the database"
  type        = string
  sensitive   = true   # Hidden from CLI
  ephemeral   = true   # NEVER stored in state!
}
```

#### 2. Ephemeral Outputs (Child Modules Only)

```hcl
# Pass secrets between modules without storing them
output "session_token" {
  value     = ephemeral.auth_provider.main.token
  ephemeral = true   # Not stored in state
  sensitive = true   # Hidden from CLI
}
```

> ⚠️ Note: You cannot use `ephemeral = true` on root module outputs.

#### 3. Ephemeral Resources (Terraform 1.10+)

```hcl
# Generate a temporary password that NEVER touches state
ephemeral "random_password" "db_password" {
  length           = 16
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

# Use with write-only arguments (Terraform 1.11+)
resource "aws_db_instance" "example" {
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  engine              = "postgres"
  username            = "admin"
  skip_final_snapshot = true

  # Write-only: value used during apply, NEVER stored
  password_wo         = ephemeral.random_password.db_password.result
  password_wo_version = 1
}
```

**Analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DOCUMENT SECURITY LEVELS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Level 1: REGULAR OUTPUT                                         │
│  ┌────────────────────────────────────┐                          │
│  │ endpoint = "db.example.com"        │ ← Visible everywhere     │
│  │ Stored in: CLI, State, Plans      │                          │
│  └────────────────────────────────────┘                          │
│                                                                  │
│  Level 2: SENSITIVE (Sealed Envelope)                            │
│  ┌────────────────────────────────────┐                          │
│  │ password = <sensitive>             │ ← Hidden in CLI          │
│  │ Stored in: State, Plans (hidden)   │ ← BUT still stored!     │
│  └────────────────────────────────────┘                          │
│                                                                  │
│  Level 3: EPHEMERAL (Self-Destructing) 🔥                        │
│  ┌────────────────────────────────────┐                          │
│  │ api_token = 💨 *used then gone*    │ ← NEVER stored!         │
│  │ Stored in: NOWHERE                 │                          │
│  │ Only exists during terraform run   │                          │
│  └────────────────────────────────────┘                          │
│                                                                  │
│  Level 4: EPHEMERAL + SENSITIVE (Best of Both)                   │
│  ┌────────────────────────────────────┐                          │
│  │ secret = <sensitive> 💨            │ ← Hidden AND not stored │
│  │ Hidden from: CLI, State, Plans     │                          │
│  └────────────────────────────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Where can ephemeral values be used?**

| Context | Allowed? |
|---------|----------|
| `locals` block | ✅ Yes |
| `provider` block | ✅ Yes (great for API tokens!) |
| `provisioner` / `connection` blocks | ✅ Yes |
| Write-only arguments (`password_wo`) | ✅ Yes |
| Regular resource arguments | ❌ No |
| Root module outputs | ❌ No |
| Child module outputs | ✅ Yes (with `ephemeral = true`) |

**Quick reference:**

| Feature | `sensitive` | `ephemeral` | Both |
|---------|-------------|-------------|------|
| Hidden from CLI | ✅ | ❌ | ✅ |
| Hidden from HCP Terraform UI | ✅ | ❌ | ✅ |
| Stored in state | ⚠️ Yes | ❌ No | ❌ No |
| Stored in plan | ⚠️ Yes | ❌ No | ❌ No |
| Terraform version | 0.15+ | 1.10+ | 1.10+ |

**When to use which:**

| Scenario | Recommendation |
|----------|----------------|
| Database password (needs to persist) | `sensitive = true` |
| Short-lived API token for provider auth | `ephemeral = true` |
| One-time setup password | `ephemeral = true` + write-only argument |
| Secret that must never be in state | `sensitive = true` + `ephemeral = true` |

---

## Module Testing - Building Inspections

### The Concept: Verify Before Occupancy

Before anyone moves in, buildings need inspection:

```
BUILDING INSPECTION PROCESS:
────────────────────────────

Level 1: BLUEPRINT REVIEW (Static Analysis)
├── Does the floor plan follow building codes?
├── Are all measurements specified?
└── Tools: terraform validate, tflint

Level 2: STRUCTURAL TEST (Plan)
├── Will it actually build according to spec?
├── Any missing materials?
└── Tools: terraform plan

Level 3: TEST CONSTRUCTION (Integration)
├── Build a test unit, verify it works
├── Check plumbing, electrical, etc.
└── Tools: Terratest, terraform test

Level 4: COMPLIANCE CHECK (Policy)
├── Does it meet fire codes?
├── ADA compliance?
└── Tools: Sentinel, OPA
```

**Testing methods:**

```bash
# Level 1: Static analysis
terraform init
terraform validate
tflint

# Level 2: Plan testing
terraform plan

# Level 3: Native testing (Terraform 1.6+)
terraform test
```

**Native test file (`tests/basic.tftest.hcl`):**

```hcl
variables {
  instance_type = "t2.micro"
  environment   = "test"
}

run "create_instance" {
  command = apply

  assert {
    condition     = aws_instance.this.instance_type == "t2.micro"
    error_message = "Instance type mismatch"
  }

  assert {
    condition     = length(aws_instance.this.tags) > 0
    error_message = "Instance must have tags"
  }
}
```

**Example-based testing structure:**

```
modules/vpc/
├── main.tf
├── examples/
│   ├── simple/           ← "Can we build the basic version?"
│   │   ├── main.tf
│   │   └── README.md
│   ├── complete/         ← "Can we build with all options?"
│   │   ├── main.tf
│   │   └── README.md
│   └── multi-az/         ← "Does the HA setup work?"
│       ├── main.tf
│       └── README.md
└── tests/
    └── basic.tftest.hcl  ← Automated checks
```

---

## Provider Configuration - Construction Crews by City

### The Concept: Different Crews for Different Locations

Your firm builds in multiple cities. Each city has its own construction crew:

```
CONSTRUCTION CREWS:
───────────────────
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  NYC Crew       │  │  LA Crew        │  │  Chicago Crew   │
│  (us-east-1)    │  │  (us-west-2)    │  │  (us-central-1) │
│                 │  │                 │  │                 │
│  Knows NYC      │  │  Knows LA       │  │  Knows Chicago  │
│  building codes │  │  earthquake     │  │  wind load      │
│                 │  │  requirements   │  │  requirements   │
└─────────────────┘  └─────────────────┘  └─────────────────┘

When building in NYC, use the NYC crew!
When building in LA, use the LA crew!
```

**In Terraform:**

```hcl
# Root module - Define the crews
provider "aws" {
  region = "us-east-1"
  alias  = "east"
}

provider "aws" {
  region = "us-west-2"
  alias  = "west"
}

# Tell each module which crew to use
module "vpc_east" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.east  # "Use the NYC crew"
  }
}

module "vpc_west" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.west  # "Use the LA crew"
  }
}
```

**Module declaring it needs multiple providers:**

```hcl
# modules/s3-replication/versions.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
      configuration_aliases = [aws.primary, aws.replica]
    }
  }
}
```

```hcl
# modules/s3-replication/main.tf

resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = var.primary_bucket_name
}

resource "aws_s3_bucket" "replica" {
  provider = aws.replica
  bucket   = var.replica_bucket_name
}
```

---

## Documentation - The User Manual

### Required Documentation Elements

```
┌─────────────────────────────────────────────────────────────────┐
│              FLOOR PLAN USER MANUAL                              │
│              (README.md)                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  # 3-Bedroom Apartment Module                                    │
│                                                                  │
│  Creates a standard 3-bedroom apartment unit.                    │
│                                                                  │
│  ## Features                                                     │
│  - Open concept kitchen                                          │
│  - Optional balcony                                              │
│  - Energy-efficient windows                                      │
│                                                                  │
│  ## Usage                                                        │
│  ```hcl                                                          │
│  module "apartment" {                                            │
│    source = "company/apartment/building"                         │
│    sqft   = 1200                                                 │
│  }                                                               │
│  ```                                                             │
│                                                                  │
│  ## Inputs                                                       │
│  | Name | Description | Type | Default | Required |             │
│  |------|-------------|------|---------|----------|             │
│  | sqft | Square feet | number | n/a | yes |                    │
│                                                                  │
│  ## Outputs                                                      │
│  | Name | Description |                                          │
│  |------|-------------|                                          │
│  | unit_id | The apartment unit ID |                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Auto-generate with terraform-docs:**

```bash
# Install
brew install terraform-docs

# Generate
terraform-docs markdown table ./modules/vpc > ./modules/vpc/README.md
```

---

## Key Takeaways

### Versioning
1. **`~> 2.0`** = Any 2.x version (recommended for flexibility)
2. **`= 2.0.0`** = Exact version (production stability)
3. **Semantic versioning** = Major.Minor.Patch

### Refactoring
4. **`moved` block** = Change of address form (no demolition)
5. **Use for** = Renames, restructuring, count→for_each migration

### Publishing
6. **Public registry** = City blueprint library (anyone can use)
7. **Private registry** = Firm's vault (employees only)
8. **Naming** = `terraform-<PROVIDER>-<NAME>`

### Best Practices
9. **Dependency injection** = Pass connections, don't look them up
10. **`for_each` > `count`** = Named units > numbered units
11. **Sensitive outputs** = Sealed envelopes for secrets
12. **Ephemeral values** = Self-destructing docs (never in state!)
13. **Test your modules** = Building inspection before occupancy

### Provider Management
14. **Default inheritance** = Modules use root's providers
15. **Explicit passing** = `providers = { aws = aws.west }`
16. **Multi-region modules** = Use `configuration_aliases`

### Documentation
17. **README.md** = User manual
18. **Variable descriptions** = Rendered in registry
19. **Examples folder** = Working usage samples
20. **terraform-docs** = Auto-generate documentation

---

*Written on January 15, 2026*  
*Day 4/97 - Terraform Modules Part 2*
