# Terraform Variables & Validation - The Blueprint Parameters

> Understanding Terraform variables through a real-world analogy of an architect creating customizable blueprints with constraints and specifications that clients must follow.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Variable | Customizable blueprint parameter ("number of floors") |
| Variable type | Parameter format (number, text, list of items) |
| Default value | Standard option if client doesn't specify |
| Description | Label explaining what the parameter controls |
| Validation block | Building code requirement ("floors must be 1-50") |
| `condition` | The actual rule to check |
| `error_message` | What to tell client when they violate the code |
| `sensitive = true` | "Don't read this aloud in meetings" (hidden from logs) |
| `ephemeral = true` | "Shred after reading" (not stored anywhere) |
| `nullable = false` | "You MUST provide a value, no blanks allowed" |
| terraform.tfvars | Client's specification sheet |
| *.auto.tfvars | Auto-loaded spec sheets (no flag needed) |
| `-var` flag | "Actually, change that one thing" (CLI override) |
| Variable precedence | Later specifications override earlier ones |
| `object({})` | Structured form with specific fields |
| `map(string)` | Open-ended key-value list |

---

## The Big Picture

Imagine you're an **architect who creates customizable building blueprints**. Instead of creating a fixed design, you create **templates with adjustable parameters** that clients can modify within certain limits:

```
CUSTOMIZABLE BLUEPRINT SYSTEM
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                     BLUEPRINT TEMPLATE                       │
│                                                              │
│   📋 ADJUSTABLE PARAMETERS (Variables)                       │
│   │                                                          │
│   ├── 🏢 Number of Floors: _____ (1-50)                      │
│   │       Default: 10                                        │
│   │       Rule: Must be between 1 and 50                     │
│   │                                                          │
│   ├── 🎨 Building Color: _____                               │
│   │       Options: ["white", "gray", "beige"]                │
│   │       Default: "white"                                   │
│   │                                                          │
│   ├── 🔐 Security Code: ***** (CONFIDENTIAL)                 │
│   │       Don't display in presentations!                    │
│   │       Don't store in public records!                     │
│   │                                                          │
│   └── 📍 Region: _____                                       │
│           No default - CLIENT MUST SPECIFY                   │
│                                                              │
│   📄 CLIENT SPECIFICATION SHEETS                             │
│   │                                                          │
│   ├── terraform.tfvars (standard form)                       │
│   ├── dev.auto.tfvars (auto-loaded additions)                │
│   └── -var "floors=20" (verbal override at meeting)          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Without variables:**
- Blueprints are rigid and one-size-fits-all
- Every project needs a completely new blueprint
- No way to enforce building codes

**With variables:**
- One blueprint template serves many clients
- Clients customize within safe boundaries
- Building codes (validations) prevent dangerous configurations

---

## Core Components

### Variable Types - The Parameter Formats

Just like a form has different field types (text box, number field, dropdown), Terraform variables have types that define what format of data they accept.

```
PARAMETER FORMAT TYPES
───────────────────────

PRIMITIVE TYPES (Single Values)
─────────────────────────────────────────────────────────────────

  string    →  "Text field"           │  "us-east-1"
  number    →  "Number field"         │  10, 3.14
  bool      →  "Checkbox (yes/no)"    │  true, false


COLLECTION TYPES (Multiple Items)
─────────────────────────────────────────────────────────────────

  list(string)  →  "Ordered checklist"
                   ["first", "second", "third"]
                   Access by position: list[0]

  set(string)   →  "Unique items bag"
                   No duplicates, no order
                   ["a", "b"] same as ["b", "a"]

  map(string)   →  "Key-value labels"
                   { Environment = "prod", Team = "platform" }
                   Access by key: map["Environment"]


STRUCTURAL TYPES (Specific Shape)
─────────────────────────────────────────────────────────────────

  object({          →  "Structured form"
    name   = string      Fixed fields with types
    port   = number      Must match exactly
    enable = bool
  })

  tuple([           →  "Ordered mixed list"
    string,              Each position has a type
    number,
    bool
  ])
```

**In HCL:**
```hcl
# Primitive
variable "environment" {
  type    = string
  default = "dev"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

# Collection
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

variable "instance_tags" {
  type = map(string)
  default = {
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}

# Structural
variable "database_config" {
  type = object({
    engine         = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  
  default = {
    engine         = "postgres"
    instance_class = "db.t3.micro"
    storage_gb     = 20
    multi_az       = false
  }
}
```

---

### Variable Precedence - Who Gets the Final Say

When multiple sources provide a value for the same variable, Terraform follows a strict hierarchy. Think of it like a meeting where later speakers override earlier ones:

```
VARIABLE PRECEDENCE (Later Wins!)
─────────────────────────────────────────────────────────────────

  LOWEST PRIORITY (Easily overridden)
  │
  │  1. Default value in variable block
  │        variable "env" { default = "dev" }
  │
  │  2. Environment variable
  │        export TF_VAR_env="staging"
  │
  │  3. terraform.tfvars file (auto-loaded)
  │        env = "staging"
  │
  │  4. *.auto.tfvars files (auto-loaded, alphabetical)
  │        prod.auto.tfvars → env = "prod"
  │
  │  5. -var-file flag (in order specified)
  │        terraform apply -var-file="override.tfvars"
  │
  │  6. -var flag (in order specified)
  │        terraform apply -var="env=production"
  │
  ▼
  HIGHEST PRIORITY (Always wins!)

─────────────────────────────────────────────────────────────────

EXAMPLE:
  
  default = "dev"          ← Would be used if nothing else
  TF_VAR_env = "staging"   ← Overrides default
  terraform.tfvars: "test" ← Overrides environment var
  -var="env=prod"          ← WINS! Final value is "prod"
```

**Key insight:** The `-var` flag is often used for CI/CD overrides or quick testing because it always wins.

---

### Validation Blocks - Building Code Enforcement

Validation blocks are your building code inspector. They check that client parameters meet safety requirements before any construction begins.

```
BUILDING CODE ENFORCEMENT
─────────────────────────────────────────────────────────────────

CLIENT REQUEST                    BUILDING CODE CHECK
───────────────                   ─────────────────────────────
"I want 100 floors"        →      ❌ REJECTED
                                  "Maximum 50 floors per code"

"I want 25 floors"         →      ✅ APPROVED
                                  Proceed with construction

"CIDR: not-a-cidr"         →      ❌ REJECTED
                                  "Must be valid CIDR format"

"CIDR: 10.0.0.0/16"        →      ✅ APPROVED
                                  Valid format, proceed
```

**Basic validation:**
```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

**Multiple validations (checked in order):**
```hcl
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for VPC"
  
  # First: Is it even a valid CIDR?
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block format."
  }
  
  # Second: Is it the right size?
  validation {
    condition     = tonumber(split("/", var.vpc_cidr)[1]) >= 16 && tonumber(split("/", var.vpc_cidr)[1]) <= 24
    error_message = "VPC CIDR must be between /16 and /24."
  }
}
```

**Common validation patterns:**
```hcl
# Regex pattern matching
variable "bucket_name" {
  type = string
  
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must be 3-63 characters, lowercase, numbers, dots, hyphens."
  }
}

# Numeric range
variable "port" {
  type = number
  
  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "Port must be between 1 and 65535."
  }
}

# String length
variable "project_name" {
  type = string
  
  validation {
    condition     = length(var.project_name) >= 3 && length(var.project_name) <= 20
    error_message = "Project name must be 3-20 characters."
  }
}
```

---

### Sensitive & Ephemeral - Protecting Secrets

Not all parameters should be visible or stored. Terraform provides different levels of protection:

```
SECRET PROTECTION LEVELS
─────────────────────────────────────────────────────────────────

LEVEL 1: Plain Variable (No Protection)
─────────────────────────────────────────
  variable "instance_type" {
    type = string
  }
  
  ✗ Shown in CLI output
  ✗ Stored in state file
  ✗ Visible in logs
  
  Use for: Non-sensitive values (instance types, regions)


LEVEL 2: Sensitive Variable (Hidden from Output)
─────────────────────────────────────────
  variable "api_key" {
    type      = string
    sensitive = true
  }
  
  ✓ Hidden in CLI output (shows "sensitive")
  ✗ Still stored in state file!
  ✓ Hidden in logs
  
  Use for: Values that shouldn't appear in terminal
          but state file is encrypted/protected


LEVEL 3: Ephemeral Variable (Never Stored) [TF 1.10+]
─────────────────────────────────────────
  variable "db_password" {
    type      = string
    sensitive = true
    ephemeral = true
  }
  
  ✓ Hidden in CLI output
  ✓ NOT stored in state file
  ✓ Only exists during runtime
  
  Use for: Passwords, tokens, secrets
          Maximum protection

─────────────────────────────────────────────────────────────────
```

**Important limitation:** Ephemeral variables can only be passed to resources/providers that support ephemeral values. For other cases, use external secret managers (Vault, AWS Secrets Manager).

---

### tfvars Patterns - Organizing Client Specifications

How you organize your tfvars files determines how manageable your infrastructure becomes:

```
TFVARS ORGANIZATION PATTERNS
─────────────────────────────────────────────────────────────────

PATTERN 1: Environment-Based Files
──────────────────────────────────
project/
├── terraform.tfvars        # Shared defaults (auto-loaded)
├── dev.tfvars              # terraform apply -var-file="dev.tfvars"
├── staging.tfvars          
└── prod.tfvars             

Usage: terraform apply -var-file="prod.tfvars"


PATTERN 2: Auto-Loading with Naming Convention
──────────────────────────────────────────────
project/
├── terraform.tfvars        # Auto-loaded (base settings)
├── 01-network.auto.tfvars  # Auto-loaded (alphabetical order)
├── 02-compute.auto.tfvars  # Auto-loaded
└── secrets.auto.tfvars     # Auto-loaded (gitignored!)

Usage: terraform apply (all auto-loaded)


PATTERN 3: Workspace-Aligned Files
──────────────────────────────────
project/
├── terraform.tfvars        # Base settings
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars

Usage: terraform apply -var-file="environments/${terraform.workspace}.tfvars"


PATTERN 4: Secrets Separation
─────────────────────────────
project/
├── terraform.tfvars        # Committed to git
└── secrets.auto.tfvars     # In .gitignore!

secrets.auto.tfvars:
  db_password    = "actual-password"
  api_key        = "actual-key"
```

**What to put where:**

| File | Content | Git? |
|------|---------|------|
| terraform.tfvars | Default/shared values | ✅ Commit |
| {env}.tfvars | Environment-specific | ✅ Commit |
| secrets.auto.tfvars | Actual secrets | ❌ Gitignore |

---

## How Things Connect

Let's see how all the variable concepts work together in a real project:

```
VARIABLE FLOW IN ACTION
─────────────────────────────────────────────────────────────────

1. DECLARATION (variables.tf)
   │
   │  variable "instance_count" {
   │    type        = number
   │    description = "Number of instances"
   │    default     = 2
   │
   │    validation {
   │      condition     = var.instance_count >= 1 && var.instance_count <= 10
   │      error_message = "Must be 1-10."
   │    }
   │  }
   │
   ▼
2. VALUE SOURCES (multiple possible)
   │
   │  default = 2              ← Fallback
   │  terraform.tfvars = 3     ← Overrides default
   │  TF_VAR_instance_count=4  ← Overrides tfvars
   │  -var="instance_count=5"  ← WINS
   │
   ▼
3. VALIDATION CHECK
   │
   │  Is 5 >= 1 && 5 <= 10?
   │  ✅ YES → Continue
   │  ❌ NO  → Error, stop here
   │
   ▼
4. USAGE (main.tf)
   │
   │  resource "aws_instance" "web" {
   │    count = var.instance_count  # Uses 5
   │    ...
   │  }
   │
   ▼
5. OUTPUT
   
   Creates 5 EC2 instances
```

---

## Key Takeaways

1. **Types enforce data format** at plan time
   - Use `object({})` for structured data with known fields
   - Use `map(type)` for flexible key-value pairs
   - Terraform validates types before any API calls

2. **Precedence order matters** - later sources win
   - Default → Environment → tfvars → auto.tfvars → -var-file → -var
   - Use `-var` for CI/CD overrides
   - Remember: environment variables require `TF_VAR_` prefix

3. **Validation blocks are your safety net**
   - Multiple validations run in order (fail fast)
   - Use `can()` to safely test expressions
   - Common functions: `contains()`, `regex()`, `length()`, `tonumber()`

4. **Protect secrets with the right level**
   - `sensitive = true` hides from output (still in state!)
   - `ephemeral = true` never stored (TF 1.10+)
   - For older TF, use external secret managers

5. **tfvars patterns keep configs manageable**
   - terraform.tfvars for shared values
   - Environment-specific files for differences
   - NEVER commit actual secrets to git

6. **Maps and objects DON'T merge** - they replace entirely
   - Pass complete values or use `merge()` explicitly
   - Common gotcha in interviews!

---

*Written on January 23, 2026*
