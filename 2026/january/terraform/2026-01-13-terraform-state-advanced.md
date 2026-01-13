# Terraform State Advanced - Registry Office Operations

> Mastering state manipulation through the lens of a property registry office handling transfers, name changes, demolitions, and vault migrations.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| `terraform state mv` | Update registry: rename "Building A" → "Main Office" (same building, new name) |
| `moved` block | Official name change form - filed paperwork declaring old name = new name |
| `terraform state rm` | Sell building - remove from your ledger, building still exists |
| `removed` block | Transfer deed form - official paperwork to stop managing a property |
| `terraform import` (CLI) | Acquire existing building - register it in your ledger manually |
| `import` block | Acquisition form - declarative paperwork to adopt a building |
| `terraform taint` (deprecated) | "Condemned" sticker - mark building for demolition next work order |
| `-replace` flag | Demolish & rebuild order - single work order: tear down and rebuild |
| `terraform refresh` (deprecated) | Send inspector to all buildings, update ledger immediately |
| `-refresh-only` | Inspector with approval - see inspection report before updating ledger |
| State migration | Move ledger from desk drawer to central registry vault |
| `-migrate-state` | Automated moving truck for your ledger |

---

## The Big Picture

Yesterday (Day 1), we learned that Terraform state is like a **property registry** - the official record of what buildings you own and where they're located.

Today, we're learning **registry office operations** - the advanced procedures for:
- **Renaming buildings** without demolishing them
- **Acquiring buildings** that already exist
- **Transferring ownership** to stop managing a building
- **Condemning buildings** for rebuild
- **Moving your entire registry** to a new vault

Think of it this way:

| Day 1 | Day 2 |
|-------|-------|
| What is the registry? | How to update the registry? |
| Where to store it? | How to manipulate entries? |
| Why is it important? | How to handle edge cases? |

---

## Renaming Resources - The Name Change Process

### The Problem: Terraform Sees Names, Not Buildings

When you rename a resource in your `.tf` file, Terraform doesn't understand it's the same building:

```hcl
# BEFORE: Your original blueprint
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
}

# AFTER: You renamed it for clarity
resource "aws_instance" "web_server" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
}
```

**What Terraform sees:**

```
terraform plan

Terraform will perform the following actions:

  # aws_instance.web will be destroyed
  - resource "aws_instance" "web" {
      - id            = "i-abc123"
      - instance_type = "t2.micro"
    }

  # aws_instance.web_server will be created
  + resource "aws_instance" "web_server" {
      + ami           = "ami-12345"
      + instance_type = "t2.micro"
    }

Plan: 1 to add, 0 to change, 1 to destroy.

😱 It's going to DESTROY and RECREATE!
```

**Analogy:**

```
You renamed "Building A" to "Main Office" in your blueprints.

Without telling the registry:
  Registry clerk: "You want to demolish Building A? OK."
  Registry clerk: "And build a new Main Office? OK."
  
  Result: Building demolished, new one built. 
          Tenants evicted! Data lost! Downtime! 💥
  
With proper name change form:
  Registry clerk: "Ah, same building, new name."
  Registry clerk: "I'll update the record."
  
  Result: Just paperwork. Building untouched. ✅
```

### Solution 1: terraform state mv (Imperative)

The `terraform state mv` command directly updates the registry:

```bash
# Syntax: terraform state mv <old_address> <new_address>
terraform state mv aws_instance.web aws_instance.web_server
```

**What happens in the registry:**

```
┌─────────────────────────────────────────────────────┐
│               PROPERTY REGISTRY                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  BEFORE:                                             │
│  ┌─────────────────────────────────────────────┐    │
│  │ Entry: aws_instance.web                      │    │
│  │ Deed #: i-abc123                             │    │
│  │ Address: 52.45.67.89                         │    │
│  │ Type: t2.micro                               │    │
│  └─────────────────────────────────────────────┘    │
│                                                      │
│  AFTER state mv:                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │ Entry: aws_instance.web_server  ← NEW NAME   │    │
│  │ Deed #: i-abc123                ← SAME!      │    │
│  │ Address: 52.45.67.89            ← SAME!      │    │
│  │ Type: t2.micro                  ← SAME!      │    │
│  └─────────────────────────────────────────────┘    │
│                                                      │
└─────────────────────────────────────────────────────┘

Building unchanged. Only registry entry renamed.
```

**Verification:**

```bash
# After state mv, plan should show no changes
terraform plan

No changes. Your infrastructure matches the configuration.
✅
```

**When to use:**
- Quick one-time fixes
- Interactive troubleshooting
- Not working in a team (no code review needed)

**Downsides:**
- Not tracked in version control
- Team members don't see the rename history
- Can't be code-reviewed
- Easy to forget you did it

### Solution 2: moved Block (Declarative) ✅ Recommended

The `moved` block is the modern, team-friendly approach:

```hcl
# Step 1: Rename the resource in your config
resource "aws_instance" "web_server" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
}

# Step 2: Add a moved block to declare the rename
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}
```

**What happens:**

```bash
terraform plan

Terraform will perform the following actions:

  # aws_instance.web has moved to aws_instance.web_server
    resource "aws_instance" "web_server" {
        id            = "i-abc123"
        # (all attributes unchanged)
    }

Plan: 0 to add, 0 to change, 0 to destroy.
```

```bash
terraform apply

aws_instance.web_server: Refreshing state... [id=i-abc123]

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
✅ State updated, building untouched!
```

**Analogy:**

```
┌─────────────────────────────────────────────────────┐
│         OFFICIAL NAME CHANGE FORM                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Property: aws_instance                              │
│                                                      │
│  Current Registered Name: web                        │
│                          ────                        │
│  New Requested Name: web_server                      │
│                      ──────────                      │
│                                                      │
│  Reason: Clarity in codebase                         │
│                                                      │
│  ☑ Same building, same location, same deed           │
│  ☑ Only updating registry records                    │
│                                                      │
│  Filed by: alice@company.com                         │
│  Date: 2026-01-13                                    │
│                                                      │
│  [APPROVED ✓]                                        │
│                                                      │
└─────────────────────────────────────────────────────┘

This form is tracked in Git, reviewed by team, 
and applied during terraform apply.
```

**Benefits:**

| Benefit | Explanation |
|---------|-------------|
| **Version controlled** | `moved` block is in your `.tf` file, tracked in Git |
| **Code reviewable** | Team can review the rename in a PR |
| **Self-documenting** | Future developers see the rename history |
| **Safe** | `terraform plan` shows the move before applying |
| **Team-friendly** | Everyone gets the update when they pull |

**Keeping or removing moved blocks:**

```hcl
# Option 1: Keep forever (documentation)
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}
# ↑ Safe to keep. Terraform ignores it after first apply.

# Option 2: Remove after all team members have applied
# (After a few weeks/sprints when everyone is in sync)
```

### Moving Resources Between Modules

The `moved` block also handles module refactoring:

```hcl
# Scenario: Moving resource INTO a module

# Before: Resource at root level
resource "aws_instance" "web" { ... }

# After: Resource inside a module
module "compute" {
  source = "./modules/compute"
}

# In the module: modules/compute/main.tf
resource "aws_instance" "web" { ... }

# The moved block (at root level):
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

```hcl
# Scenario: Moving resource OUT of a module

moved {
  from = module.old_module.aws_instance.web
  to   = aws_instance.web
}
```

```hcl
# Scenario: Renaming a module

module "compute" {  # Was: module "servers"
  source = "./modules/compute"
}

moved {
  from = module.servers
  to   = module.compute
}
```

**Analogy:** Moving buildings between different divisions of your company:

```
Before: web server managed by IT Division directly
After: web server managed by Compute Division (a subsidiary)

The moved block is the transfer paperwork:
"This building now belongs to Compute Division's registry,
 but it's still the same building at the same address."
```

### Comparison: state mv vs moved Block

| Aspect | `terraform state mv` | `moved` block |
|--------|----------------------|---------------|
| **Type** | Imperative (run command) | Declarative (in code) |
| **Version control** | ❌ Not tracked | ✅ Tracked in Git |
| **Code review** | ❌ No | ✅ Yes |
| **Team sync** | ❌ Manual communication | ✅ Automatic on pull |
| **Plan preview** | ❌ No preview | ✅ Shows in plan |
| **Documentation** | ❌ Lost to history | ✅ Self-documenting |
| **Undo** | ❌ Run reverse command | ✅ Remove block |
| **Best for** | Quick fixes, solo work | Team environments |

---

## Removing Resources from State - The Transfer of Ownership

### The Problem: You Want to Stop Managing a Resource

Sometimes you need Terraform to "forget" about a resource without destroying it:

- Handing off a resource to another team's Terraform
- Migrating to a different IaC tool
- Resource was imported by mistake
- Legacy resource managed manually going forward

### Solution 1: terraform state rm (Imperative)

```bash
terraform state rm aws_instance.legacy_server
```

**What happens:**

```
Registry BEFORE:
├── web_server: i-abc123
├── api_server: i-def456
└── legacy_server: i-xyz789

terraform state rm aws_instance.legacy_server

Registry AFTER:
├── web_server: i-abc123
└── api_server: i-def456

(legacy_server removed from registry)
```

**Important:** The actual EC2 instance still exists in AWS! Only the registry entry is removed.

```bash
# Verify it's gone from state
terraform state list
# aws_instance.web_server
# aws_instance.api_server
# (legacy_server not listed)

# But in AWS Console:
# i-xyz789 is still running! ✅
```

**Next step:** Remove the resource block from your config, or Terraform will try to create it:

```hcl
# BEFORE: Remove this block!
resource "aws_instance" "legacy_server" {
  # ...
}

# AFTER: Block removed from config
# (Nothing here)
```

**Analogy:**

```
Scenario: Selling a building to another company

1. Go to registry office
2. File a "Transfer of Ownership" form
3. Building removed from YOUR company's ledger
4. Building still physically exists
5. New owner registers it in THEIR ledger

Your company no longer manages it - but it's not demolished!
```

### Solution 2: removed Block (Declarative) ✅ Recommended

The `removed` block (Terraform 1.7+) is the declarative way:

```hcl
# Instead of deleting the resource block, replace it with:
removed {
  from = aws_instance.legacy_server

  lifecycle {
    destroy = false  # Keep the resource, just stop managing it
  }
}
```

**What happens:**

```bash
terraform plan

Terraform will perform the following actions:

  # aws_instance.legacy_server will no longer be managed by Terraform
  # (The actual resource will NOT be destroyed)

Plan: 0 to add, 0 to change, 0 to destroy.
```

```bash
terraform apply

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

# Resource removed from state, AWS instance untouched ✅
```

**The lifecycle block options:**

```hcl
removed {
  from = aws_instance.legacy_server

  lifecycle {
    destroy = false  # ← Keep the resource (default: false)
  }
}

# Or if you DO want to destroy:
removed {
  from = aws_instance.old_server

  lifecycle {
    destroy = true  # ← Actually delete the resource
  }
}
```

**Analogy:**

```
┌─────────────────────────────────────────────────────┐
│       TRANSFER OF OWNERSHIP FORM                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Property: aws_instance.legacy_server                │
│  Deed #: i-xyz789                                    │
│                                                      │
│  Action requested:                                   │
│  ☑ Remove from our company's registry                │
│  ☐ Demolish the building                             │
│                                                      │
│  Reason: Transferring to Team B's management         │
│                                                      │
│  lifecycle {                                         │
│    destroy = false  ← "Don't demolish, just forget"  │
│  }                                                   │
│                                                      │
│  Filed by: alice@company.com                         │
│  Date: 2026-01-13                                    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**After applying, you can remove the `removed` block:**

```hcl
# Round 1: Apply the removed block
removed {
  from = aws_instance.legacy_server
  lifecycle { destroy = false }
}

# Round 2: Safe to delete the removed block
# (Resource is already gone from state)
```

### Comparison: state rm vs removed Block

| Aspect | `terraform state rm` | `removed` block |
|--------|----------------------|-----------------|
| **Type** | Imperative | Declarative |
| **Version control** | ❌ Not tracked | ✅ Tracked in Git |
| **Plan preview** | ❌ No | ✅ Shows in plan |
| **Explicit destroy option** | ❌ Only removes | ✅ Can set `destroy = true` |
| **Best for** | Quick removal | Team environments |

---

## Importing Resources - Adopting Existing Buildings

### The Problem: Resources Exist But Aren't in State

Common scenarios:
- Resources created manually in AWS Console
- Migrating from CloudFormation to Terraform
- Adopting another team's infrastructure
- Disaster recovery - rebuilding state from reality

### Solution 1: terraform import (CLI)

The traditional approach:

```bash
# Syntax: terraform import <resource_address> <resource_id>

# Step 1: Write the resource config FIRST
```

```hcl
# main.tf - Must exist before importing!
resource "aws_s3_bucket" "existing_bucket" {
  bucket = "my-company-data-bucket"
}
```

```bash
# Step 2: Run import command
terraform import aws_s3_bucket.existing_bucket my-company-data-bucket

# Output:
aws_s3_bucket.existing_bucket: Importing from ID "my-company-data-bucket"...
aws_s3_bucket.existing_bucket: Import prepared!
aws_s3_bucket.existing_bucket: Refreshing state...

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

```bash
# Step 3: Run plan to check for drift
terraform plan

# Often shows differences:
  ~ resource "aws_s3_bucket" "existing_bucket" {
      ~ versioning {
          ~ enabled = true -> false  # Config doesn't match reality!
        }
    }

# Step 4: Update config to match reality
```

```hcl
resource "aws_s3_bucket" "existing_bucket" {
  bucket = "my-company-data-bucket"
}

resource "aws_s3_bucket_versioning" "existing_bucket" {
  bucket = aws_s3_bucket.existing_bucket.id
  versioning_configuration {
    status = "Enabled"  # Match reality
  }
}
```

```bash
# Step 5: Plan again
terraform plan
# No changes. Infrastructure is up-to-date. ✅
```

**Analogy:**

```
Scenario: Your company acquired another company's building

Manual Import Process:
1. Architect creates blueprint for the building (resource block)
2. Go to registry office with the deed number
3. "I want to register this building under my company"
4. Clerk records all building details in your ledger
5. Inspector compares your blueprint to the real building
6. You update blueprint to match reality exactly
7. Building is now in your registry, fully documented ✅
```

### Solution 2: import Block (Declarative) ✅ Recommended

Terraform 1.5+ introduced declarative imports:

```hcl
# Step 1: Add import block and resource block together
import {
  to = aws_s3_bucket.existing_bucket
  id = "my-company-data-bucket"
}

resource "aws_s3_bucket" "existing_bucket" {
  bucket = "my-company-data-bucket"
}
```

**What happens:**

```bash
terraform plan

Terraform will perform the following actions:

  # aws_s3_bucket.existing_bucket will be imported
    resource "aws_s3_bucket" "existing_bucket" {
        bucket = "my-company-data-bucket"
        id     = "my-company-data-bucket"
        # ... (shows all imported attributes)
    }

Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

```bash
terraform apply

aws_s3_bucket.existing_bucket: Importing... [id=my-company-data-bucket]
aws_s3_bucket.existing_bucket: Import complete [id=my-company-data-bucket]

Apply complete! Resources: 1 imported, 0 added, 0 changed, 0 destroyed.
```

**Generating configuration automatically:**

```bash
# Terraform 1.5+ can generate config for you!
terraform plan -generate-config-out=generated.tf

# Creates generated.tf with:
resource "aws_s3_bucket" "existing_bucket" {
  bucket = "my-company-data-bucket"
  # ... all attributes filled in from AWS
}
```

**Analogy:**

```
┌─────────────────────────────────────────────────────┐
│       BUILDING ACQUISITION FORM                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Acquiring: S3 Bucket                                │
│  Existing ID: my-company-data-bucket                 │
│                                                      │
│  import {                                            │
│    to = aws_s3_bucket.existing_bucket                │
│    id = "my-company-data-bucket"                     │
│  }                                                   │
│                                                      │
│  ☑ Register in our company ledger                    │
│  ☑ Document all current specifications               │
│  ☑ Begin managing under our blueprints               │
│                                                      │
│  Filed by: alice@company.com                         │
│  Reviewed by: bob@company.com                        │
│  Date: 2026-01-13                                    │
│                                                      │
└─────────────────────────────────────────────────────┘

This form is version controlled and code reviewed!
```

**After import, remove the import block:**

```hcl
# Round 1: Import block + resource block
import {
  to = aws_s3_bucket.existing_bucket
  id = "my-company-data-bucket"
}

resource "aws_s3_bucket" "existing_bucket" {
  bucket = "my-company-data-bucket"
}

# Round 2: Remove import block (after apply)
resource "aws_s3_bucket" "existing_bucket" {
  bucket = "my-company-data-bucket"
}
```

### Bulk Import with for_each

```hcl
# Import multiple resources at once
locals {
  existing_buckets = {
    "data"   = "company-data-bucket"
    "logs"   = "company-logs-bucket"
    "backup" = "company-backup-bucket"
  }
}

import {
  for_each = local.existing_buckets
  to       = aws_s3_bucket.buckets[each.key]
  id       = each.value
}

resource "aws_s3_bucket" "buckets" {
  for_each = local.existing_buckets
  bucket   = each.value
}
```

### Comparison: CLI import vs import Block

| Aspect | `terraform import` | `import` block |
|--------|-------------------|----------------|
| **Type** | Imperative | Declarative |
| **Version control** | ❌ Command not tracked | ✅ Block tracked in Git |
| **Plan preview** | ❌ No | ✅ Shows in plan |
| **Code review** | ❌ No | ✅ Yes |
| **Bulk import** | ❌ Script required | ✅ for_each support |
| **Config generation** | ❌ No | ✅ `-generate-config-out` |
| **Best for** | Quick one-off | Team environments |

---

## Force Recreating Resources - The Demolish & Rebuild Order

### The Problem: Resource is Corrupted or Needs Fresh Start

Sometimes a resource is broken and needs complete recreation:
- EC2 instance has corrupted disk
- Container stuck in bad state
- Resource needs to pick up changes that require recreation
- Testing fresh provisioning

### Old Way: terraform taint (Deprecated ⚠️)

```bash
# DON'T USE THIS - Deprecated!
terraform taint aws_instance.web_server

# What it did:
# 1. Modified state to mark resource as "tainted"
# 2. Next apply would destroy and recreate

# Problems:
# - Modified state immediately (before you could review)
# - If apply failed, resource left in tainted state
# - No preview of what would happen
```

**Analogy (old way):**

```
Condemned Sticker Approach:

1. Go to registry office
2. Put "CONDEMNED" sticker on building record
3. Leave
4. (State now shows building as condemned)
5. Come back later for demolition work order
6. If demolition fails, building still has sticker! 😱

Problems:
- Sticker stays even if you change your mind
- No approval process before stickering
- Multiple steps, easy to forget
```

### New Way: -replace Flag ✅ Recommended

```bash
# Modern approach - single atomic operation
terraform plan -replace="aws_instance.web_server"
```

**What happens:**

```bash
terraform plan -replace="aws_instance.web_server"

Terraform will perform the following actions:

  # aws_instance.web_server will be replaced, as requested
-/+ resource "aws_instance" "web_server" {
      ~ id            = "i-abc123" -> (known after apply)
      ~ public_ip     = "52.45.67.89" -> (known after apply)
        instance_type = "t2.micro"
        # ... (all other attributes)
    }

Plan: 1 to add, 0 to change, 1 to destroy.

# You can REVIEW before applying!
```

```bash
# If you're happy with the plan:
terraform apply -replace="aws_instance.web_server"

aws_instance.web_server: Destroying... [id=i-abc123]
aws_instance.web_server: Destruction complete after 30s
aws_instance.web_server: Creating...
aws_instance.web_server: Creation complete after 45s [id=i-def456]

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
```

**Analogy (new way):**

```
Demolish & Rebuild Work Order:

1. Create work order: "Demolish and rebuild web_server"
2. Review the work order (terraform plan)
   - Shows exactly what will be destroyed
   - Shows exactly what will be created
3. Approve and execute (terraform apply)
4. Single atomic operation
5. If it fails, try again - no stale stickers!

Benefits:
- Preview before action
- Single work order, not multi-step
- No stale "condemned" marks
- Can be code reviewed
```

**Multiple resources:**

```bash
# Replace multiple resources
terraform apply \
  -replace="aws_instance.web_server" \
  -replace="aws_instance.api_server"
```

**In CI/CD pipelines:**

```bash
# Auto-approve for automation
terraform apply -replace="aws_instance.web_server" -auto-approve
```

### Comparison: taint vs -replace

| Aspect | `terraform taint` | `-replace` flag |
|--------|-------------------|-----------------|
| **Status** | ⚠️ Deprecated | ✅ Current |
| **Modifies state** | Yes, immediately | No, only on apply |
| **Preview** | No | Yes, with plan |
| **Atomic** | No (2 steps) | Yes (1 step) |
| **Recovery on failure** | State left tainted | Clean state |
| **Multiple resources** | Multiple commands | Multiple flags |

---

## Syncing State with Reality - The Inspection Process

### The Problem: Reality Drifted from State

Someone made changes outside Terraform:
- Manual changes in AWS Console
- Another tool modified resources
- AWS auto-recovery changed something
- Resource was deleted externally

### Old Way: terraform refresh (Deprecated ⚠️)

```bash
# DON'T USE THIS - Deprecated!
terraform refresh

# What it did:
# 1. Checked all resources against AWS
# 2. Updated state IMMEDIATELY
# 3. No preview, no approval

# Problems:
# - No chance to review before updating
# - Could update state incorrectly
# - Risky in production
```

### New Way: -refresh-only Flag ✅ Recommended

```bash
# Step 1: Preview what would change
terraform plan -refresh-only
```

**What happens:**

```bash
terraform plan -refresh-only

Note: Objects have changed outside of Terraform

Terraform detected the following changes made outside of Terraform
since the last "terraform apply":

  # aws_instance.web_server has changed
  ~ resource "aws_instance" "web_server" {
        id            = "i-abc123"
      ~ instance_type = "t2.micro" -> "t2.large"  # Someone upgraded it!
      ~ tags          = {
          + "Environment" = "production"  # Tag added manually
        }
    }

This is a refresh-only plan, so Terraform will not take any actions
to undo these. If you were expecting these changes then you can apply
this plan to record the updated values in the Terraform state.

───────────────────────────────────────────────────────────────────

Would you like to update the Terraform state to reflect these changes?
```

```bash
# Step 2: If changes are expected, apply
terraform apply -refresh-only

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

# State now matches reality ✅
```

**Scenario: Resource was deleted externally**

```bash
terraform plan -refresh-only

Note: Objects have changed outside of Terraform

Terraform detected the following changes:

  # aws_instance.temp_server has been deleted
  - resource "aws_instance" "temp_server" {
      - id            = "i-xyz789" -> null
      - instance_type = "t2.micro" -> null
    }

# If this deletion was intentional:
terraform apply -refresh-only
# State updated: temp_server removed

# Next regular plan will show:
terraform plan
# + aws_instance.temp_server will be created
# (Because config still has it, but state doesn't)
```

**Analogy:**

```
Property Inspector Process:

OLD WAY (terraform refresh):
Inspector visits all buildings
  → Immediately updates ledger
  → No approval from management
  → "Building 5 is now t2.large... I updated the record"
  → What if the inspector made a mistake? Too late!

NEW WAY (-refresh-only):
Inspector visits all buildings
  → Returns with inspection report
  → "Here's what I found different from our ledger:"
     - Building 5: Was t2.micro, is now t2.large
     - Building 8: Has new Environment tag
  → Management reviews report
  → "Approved - update the ledger" ← terraform apply -refresh-only
  → Only then is ledger updated

Benefits:
- Review before updating records
- Catch unexpected changes
- Audit trail of approvals
```

### Use Cases

| Scenario | Command | What Happens |
|----------|---------|--------------|
| Detect drift | `terraform plan -refresh-only` | Shows differences, no changes |
| Accept manual changes | `terraform apply -refresh-only` | Updates state to match reality |
| Revert manual changes | `terraform apply` | Changes resources to match config |
| Skip refresh (fast plan) | `terraform plan -refresh=false` | Uses cached state, faster |

---

## State Migration - Moving the Registry to a New Vault

### The Problem: Changing Where State is Stored

Common migrations:
- Local state → S3 backend (team collaboration)
- S3 bucket A → S3 bucket B (reorganization)
- Adding DynamoDB locking → Switching to `use_lockfile`
- Terraform Cloud → S3 (or vice versa)

### Migration Process

**Scenario: Local → S3**

```hcl
# BEFORE: No backend (local state)
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# State stored in: ./terraform.tfstate
```

```hcl
# AFTER: Add S3 backend
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "prod/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

```bash
# Run init - Terraform detects backend change
terraform init

Initializing the backend...

Terraform detected that the backend type changed from "local" to "s3".

Do you want to copy existing state to the new backend?
  Enter "yes" to copy and "no" to start fresh.

  Enter a value: yes

Successfully configured the backend "s3"!
```

**What happens during migration:**

```
┌───────────────────┐                    ┌───────────────────┐
│   LOCAL STATE     │                    │    S3 BACKEND     │
│                   │                    │                   │
│ terraform.tfstate │ ───── COPY ─────▶  │ s3://bucket/      │
│                   │                    │   terraform.tfstate│
│ (Your desk drawer)│                    │ (Central vault)   │
└───────────────────┘                    └───────────────────┘
         ↓                                        ↓
   (File emptied or                        (Official copy)
    kept as backup)                              
```

### Non-Interactive Migration

For CI/CD or scripting:

```bash
# Skip the prompt - automatically migrate
terraform init -migrate-state

# Or force reconfigure (lose state - dangerous!)
terraform init -reconfigure
```

### Migrating Between S3 Buckets

```hcl
# BEFORE
terraform {
  backend "s3" {
    bucket = "old-state-bucket"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}

# AFTER
terraform {
  backend "s3" {
    bucket = "new-state-bucket"  # New bucket
    key    = "prod/terraform.tfstate"  # New path
    region = "us-east-1"
  }
}
```

```bash
terraform init -migrate-state

Initializing the backend...
Backend configuration changed!

Terraform will copy the state from "s3" to "s3".

Do you want to copy existing state?
  Enter "yes": yes

Successfully configured the backend "s3"!
```

### Migrating from DynamoDB Locking to use_lockfile

```hcl
# BEFORE (old locking)
terraform {
  backend "s3" {
    bucket         = "my-state"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # Old method
  }
}

# AFTER (new locking)
terraform {
  backend "s3" {
    bucket       = "my-state"
    key          = "terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true  # New method
    # dynamodb_table removed
  }
}
```

```bash
terraform init -migrate-state

# State stays in same location, just locking mechanism changes
```

**Analogy:**

```
Moving Your Registry to a New Vault:

BEFORE: Ledger in desk drawer (local)
AFTER: Ledger in central bank vault (S3)

Process:
1. Announce: "We're moving to the central vault"
2. Inventory: "Here's everything in current ledger"
3. Question: "Copy everything to new vault?"
4. Confirm: "Yes, copy it"
5. Transport: Ledger copied to new vault
6. Verify: "New vault has all records?"
7. Done: Old location no longer used

terraform init -migrate-state = Automated moving truck
- Inventories your ledger
- Copies to new vault
- Verifies transfer
- All in one command
```

### State Migration Checklist

| Step | Command | Purpose |
|------|---------|---------|
| 1. Backup current state | `terraform state pull > backup.tfstate` | Safety net |
| 2. Update backend config | Edit `.tf` file | Define new location |
| 3. Initialize migration | `terraform init -migrate-state` | Copy state |
| 4. Verify state | `terraform state list` | Confirm resources |
| 5. Test with plan | `terraform plan` | Ensure no changes |

---

## Key Takeaways

### Renaming Resources
1. **Never just rename** - Terraform sees it as destroy + create
2. **Use `moved` block** - Declarative, version-controlled, team-friendly
3. **`state mv` for quick fixes** - Imperative, not tracked, solo use

### Removing from State
4. **`removed` block** - Declarative way to stop managing resources
5. **`destroy = false`** - Keep resource in AWS, just forget about it
6. **`state rm` for quick fixes** - Imperative, not tracked

### Importing Resources
7. **`import` block** - Declarative, code-reviewable, supports for_each
8. **`-generate-config-out`** - Auto-generate config from imports
9. **CLI import for one-offs** - Quick adoption, not tracked

### Force Recreation
10. **Use `-replace` flag** - Modern, atomic, previewable
11. **Avoid `taint`** - Deprecated, modifies state immediately, risky

### Syncing State
12. **Use `-refresh-only`** - Preview before updating state
13. **Avoid `terraform refresh`** - Deprecated, no preview

### State Migration
14. **`terraform init -migrate-state`** - Automated migration
15. **Always backup first** - `terraform state pull > backup.tfstate`
16. **Verify after migration** - `terraform plan` should show no changes

---

## Quick Reference: Imperative vs Declarative

| Task | Imperative (CLI) | Declarative (Block) |
|------|------------------|---------------------|
| Rename resource | `terraform state mv A B` | `moved { from=A, to=B }` |
| Remove from state | `terraform state rm A` | `removed { from=A }` |
| Import resource | `terraform import A id` | `import { to=A, id=id }` |
| Force recreate | `terraform taint A` ⚠️ | `terraform apply -replace=A` |
| Sync with reality | `terraform refresh` ⚠️ | `terraform apply -refresh-only` |

**Rule of thumb:** Use declarative blocks for team environments, CLI for quick solo fixes.

---

*Written on January 13, 2026*  
*Day 2/97 - Terraform State Advanced Operations*
