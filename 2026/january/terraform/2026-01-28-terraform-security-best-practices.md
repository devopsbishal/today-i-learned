# Terraform Security Best Practices - The Blueprint Security Audit

> Understanding Terraform security through a real-world analogy of a security auditor reviewing blueprints for fire code violations, checking for hidden vault combinations in documents, and ensuring construction materials pass safety inspections before any building begins.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Hardcoded secrets in `.tf` files | Writing vault combinations on public blueprints |
| Secrets in state file | Registry office storing vault codes in plain text ledger |
| `sensitive = true` | "Don't read aloud in meetings" (but still written in ledger) |
| Remote backend encryption | Locking the ledger in an encrypted safe |
| tfsec | Quick fire code inspector (fast, focused) |
| Checkov | Full compliance auditor (thorough, multi-framework) |
| SAST for IaC | Blueprint safety inspection before construction begins |
| Policy-as-code | Written building codes that inspectors enforce |
| Shift-left security | Catch violations at drafting table, not after building is complete |
| Pre-commit hooks | Architect's assistant checking for obvious errors |
| tfsec:ignore | "Approved exception to fire code" with documented reason |
| checkov:skip | "Variance granted" with justification on file |
| CI/CD security gates | Inspector must approve before construction proceeds |
| tfsec (Trivy) | Quick structural inspector (part of larger safety firm) |
| Sentinel | Enterprise policy enforcement (construction permits office) |

---

## The Big Picture

Imagine you're running an **architecture firm** that designs buildings across the city. Before any construction begins, you need **security auditors** who review your blueprints for safety violations:

```
THE BLUEPRINT SECURITY PROCESS
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                    YOUR ARCHITECTURE FIRM                    │
│                                                              │
│   📋 BLUEPRINTS (Terraform .tf files)                        │
│   │   "What we want to build"                                │
│   │                                                          │
│   │   ⚠️  SECURITY CONCERNS:                                 │
│   │   │                                                      │
│   │   ├── 🔐 Vault codes written on blueprints?              │
│   │   │       (Hardcoded secrets in .tf files)               │
│   │   │                                                      │
│   │   ├── 🚪 Emergency exits blocked?                        │
│   │   │       (S3 buckets without encryption)                │
│   │   │                                                      │
│   │   ├── 🔥 Fire suppression missing?                       │
│   │   │       (Security groups open to 0.0.0.0/0)            │
│   │   │                                                      │
│   │   └── 📜 Building codes violated?                        │
│   │           (Non-compliant configurations)                 │
│   │                                                          │
│   ▼                                                          │
│   🔍 SECURITY AUDITORS (SAST Tools)                          │
│   │                                                          │
│   ├── tfsec (Quick Inspector)                                │
│   │   "Fast fire code check, catches common violations"      │
│   │                                                          │
│   ├── Checkov (Full Auditor)                                 │
│   │   "Comprehensive compliance review, checks all codes"    │
│   │                                                          │
│   └── Sentinel (Permit Office - Enterprise)                  │
│       "Must approve before construction can begin"           │
│                                                              │
│   ▼                                                          │
│   ✅ APPROVED → Proceed to construction                      │
│   ❌ REJECTED → Fix violations, resubmit blueprints          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Without security auditing:**
- Vault combinations end up on public blueprints (secrets in VCS)
- Emergency exits get blocked (misconfigured security groups)
- Fire codes violated (no encryption, public buckets)
- Problems discovered AFTER building is occupied (production breach)

**With security auditing:**
- Blueprints reviewed before construction begins (shift-left)
- Violations caught at drafting table (pre-commit, CI/CD)
- Approved exceptions documented (skip with justification)
- Consistent safety standards enforced (policy-as-code)

---

## Core Components

### The Secrets Problem - Vault Codes on Blueprints

The most critical security issue: **never write secrets directly in your Terraform files**.

```
THE VAULT CODE PROBLEM
──────────────────────

WRONG: Writing vault codes on public blueprints
─────────────────────────────────────────────────────────────────

  main.tf (committed to Git):
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   resource "aws_db_instance" "production" {                 │
  │     engine   = "postgres"                                   │
  │     username = "admin"                                      │
  │     password = "SuperSecret123!"  ← 💀 VAULT CODE EXPOSED!  │
  │   }                                                         │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  WHAT HAPPENS:
  ├── Everyone with repo access sees the password
  ├── Password exists in Git history FOREVER
  ├── CI/CD logs might expose it
  ├── Any breach of repo = breach of database
  └── No audit trail of who used the password


RIGHT: Vault codes stored securely, referenced indirectly
─────────────────────────────────────────────────────────────────

  main.tf (safe to commit):
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   data "aws_secretsmanager_secret_version" "db_password" {  │
  │     secret_id = "production/database/password"              │
  │   }                                                         │
  │                                                             │
  │   resource "aws_db_instance" "production" {                 │
  │     engine   = "postgres"                                   │
  │     username = "admin"                                      │
  │     password = data.aws_secretsmanager_secret_version       │
  │                    .db_password.secret_string               │
  │   }                       ↑                                 │
  │                           │                                 │
  │              Password fetched at runtime from               │
  │              secure vault, never in code!                   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  WHAT HAPPENS:
  ├── Code is safe to commit, no secrets visible
  ├── Password retrieved at apply time from secure vault
  ├── Access to vault is controlled and audited
  ├── Password rotation doesn't require code changes
  └── Separation of duties: devs write code, ops manage secrets
```

**Methods to inject secrets securely:**

```
SECURE SECRET INJECTION METHODS
───────────────────────────────

METHOD 1: Environment Variables
─────────────────────────────────────────────────────────────────
  
  # Set in shell (not committed)
  export TF_VAR_db_password="SuperSecret123!"
  
  # Reference in Terraform
  variable "db_password" {
    type      = string
    sensitive = true
  }
  
  ✅ Secret not in code
  ⚠️  Still ends up in state file
  ⚠️  Visible in shell history


METHOD 2: AWS Secrets Manager (Recommended)
─────────────────────────────────────────────────────────────────
  
  data "aws_secretsmanager_secret_version" "db" {
    secret_id = "prod/db/password"
  }
  
  resource "aws_db_instance" "main" {
    password = data.aws_secretsmanager_secret_version.db.secret_string
  }
  
  ✅ Secret not in code
  ✅ Centralized management
  ✅ Rotation support
  ⚠️  Still ends up in state file


METHOD 3: HashiCorp Vault
─────────────────────────────────────────────────────────────────
  
  provider "vault" {
    address = "https://vault.company.com"
  }
  
  data "vault_generic_secret" "db" {
    path = "secret/data/production/database"
  }
  
  resource "aws_db_instance" "main" {
    password = data.vault_generic_secret.db.data["password"]
  }
  
  ✅ Dynamic secrets possible
  ✅ Fine-grained access control
  ✅ Audit logging
  ⚠️  Still ends up in state file


METHOD 4: Ephemeral Variables (Terraform 1.10+)
─────────────────────────────────────────────────────────────────
  
  variable "db_password" {
    type      = string
    ephemeral = true  # ← NEVER stored in state!
  }
  
  ✅ Secret not in code
  ✅ Secret NOT in state file!
  ⚠️  Only available in TF 1.10+
  ⚠️  Limited use cases (provider auth, etc.)
```

---

### The State File Problem - Ledger Security

Even when you inject secrets properly, **they still end up in the state file in plain text**.

```
THE STATE FILE REALITY
──────────────────────

SCENARIO: You inject a database password securely

  data "aws_secretsmanager_secret_version" "db" {
    secret_id = "prod/db/password"
  }
  
  resource "aws_db_instance" "main" {
    password = data.aws_secretsmanager_secret_version.db.secret_string
  }

WHAT ENDS UP IN terraform.tfstate:
─────────────────────────────────────────────────────────────────

  {
    "resources": [
      {
        "type": "aws_secretsmanager_secret_version",
        "instances": [{
          "attributes": {
            "secret_string": "SuperSecret123!"  ← 💀 PLAIN TEXT!
          }
        }]
      },
      {
        "type": "aws_db_instance",
        "instances": [{
          "attributes": {
            "password": "SuperSecret123!"  ← 💀 PLAIN TEXT!
          }
        }]
      }
    ]
  }

THE LEDGER (STATE FILE) CONTAINS VAULT CODES!
```

**Protecting the state file:**

```
STATE FILE SECURITY LAYERS
──────────────────────────

LAYER 1: Remote Backend (Centralized Security)
─────────────────────────────────────────────────────────────────

  terraform {
    backend "s3" {
      bucket         = "my-terraform-state"
      key            = "prod/terraform.tfstate"
      region         = "us-east-1"
      encrypt        = true              ← Encryption at rest
      dynamodb_table = "terraform-locks" ← Locking (optional now)
      use_lockfile   = true              ← Modern locking
    }
  }
  
  ANALOGY: Ledger stored in a locked, encrypted safe
           at a secure registry office


LAYER 2: S3 Bucket Security
─────────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   S3 BUCKET: my-terraform-state                             │
  │                                                             │
  │   ✅ Block all public access                                │
  │   ✅ Enable versioning (audit trail)                        │
  │   ✅ Enable server-side encryption (AES-256 or KMS)         │
  │   ✅ Enable access logging                                  │
  │   ✅ Restrict bucket policy to specific IAM roles           │
  │   ✅ Enable MFA delete for critical state                   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  aws s3api put-public-access-block \
    --bucket my-terraform-state \
    --public-access-block-configuration \
      BlockPublicAcls=true,\
      IgnorePublicAcls=true,\
      BlockPublicPolicy=true,\
      RestrictPublicBuckets=true


LAYER 3: IAM Access Control
─────────────────────────────────────────────────────────────────

  WHO CAN ACCESS STATE:
  ├── CI/CD Pipeline role (apply changes)
  ├── Senior engineers (troubleshooting)
  └── Break-glass emergency role (incidents)
  
  WHO CANNOT ACCESS:
  ├── Junior developers
  ├── Other teams
  └── Any unauthorized personnel
  
  ANALOGY: Only authorized clerks can access the secure ledger


LAYER 4: Encryption Keys (KMS)
─────────────────────────────────────────────────────────────────

  resource "aws_kms_key" "terraform_state" {
    description             = "Key for Terraform state encryption"
    deletion_window_in_days = 30
    enable_key_rotation     = true
    
    policy = jsonencode({
      # Only specific roles can decrypt
    })
  }
  
  ANALOGY: Even if someone steals the safe,
           they can't open it without the key
```

---

### The `sensitive = true` Attribute - What It Actually Protects

```
SENSITIVE ATTRIBUTE: MISUNDERSTOOD PROTECTION
──────────────────────────────────────────────

variable "db_password" {
  type      = string
  sensitive = true  ← What does this actually do?
}


WHAT IT PROTECTS:
─────────────────────────────────────────────────────────────────

  ✅ CLI Output (terraform plan/apply)
  
     # aws_db_instance.main will be created
     + resource "aws_db_instance" "main" {
         + password = (sensitive value)  ← Hidden!
       }

  ✅ Log Files (CI/CD logs)
  
     No accidental exposure in build logs

  ✅ Propagation Enforcement
  
     output "db_password" {
       value = var.db_password
     }
     
     ERROR: Output "db_password" contains sensitive value
            Add `sensitive = true` to suppress this warning


WHAT IT DOES NOT PROTECT:
─────────────────────────────────────────────────────────────────

  ❌ State File (terraform.tfstate)
  
     {
       "attributes": {
         "password": "SuperSecret123!"  ← Still plain text!
       }
     }

  ❌ Plan Files (terraform plan -out=tfplan)
  
     Binary plan file contains secrets in recoverable format

  ❌ Error Messages (sometimes)
  
     Provider errors might expose sensitive values

  ❌ Terraform Console
  
     > var.db_password
     "SuperSecret123!"  ← Visible if you have state access


THE ANALOGY:
─────────────────────────────────────────────────────────────────

  sensitive = true is like saying:
  
  "Don't read this vault code aloud in meetings"
  
  BUT the code is still written in the ledger (state file).
  Anyone with ledger access can read it.
```

---

### SAST Tools - The Security Auditors

**SAST (Static Application Security Testing)** for IaC means scanning your Terraform code for security issues **before** any infrastructure is created.

```
THE TWO MAIN AUDITORS
─────────────────────

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   tfsec (Deprecated - now absorbed into Trivy)                  │
│   ─────────────────────────                                     │
│                                                                 │
│   🏃 QUICK FIRE CODE INSPECTOR                                  │
│                                                                 │
│   • Fast (Go-based, lightweight)                                │
│   • ~300+ built-in rules                                        │
│   • Terraform + CloudFormation focus                            │
│   • Custom rules via YAML/JSON                                  │
│   • Great for pre-commit hooks (speed matters)                  │
│   • Owned by Aqua Security                                      │
│   • NOTE: tfsec has been deprecated and absorbed into Trivy.    │
│     New projects should use `trivy config` instead.             │
│                                                                 │
│   ANALOGY: Quick walkthrough inspector                          │
│            "I'll check fire exits and smoke detectors"          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Checkov                                                       │
│   ───────                                                       │
│                                                                 │
│   📋 FULL COMPLIANCE AUDITOR                                    │
│                                                                 │
│   • Comprehensive (Python-based)                                │
│   • ~1000+ built-in rules                                       │
│   • Multi-framework: Terraform, K8s, Dockerfile, Helm, ARM      │
│   • Custom rules via Python or YAML                             │
│   • Built-in compliance: CIS, SOC2, HIPAA, PCI-DSS, NIST        │
│   • Owned by Palo Alto (Bridgecrew)                             │
│                                                                 │
│   ANALOGY: Full building code compliance auditor                │
│            "I'll check EVERYTHING against ALL codes"            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


COMPARISON TABLE:
─────────────────────────────────────────────────────────────────

  ASPECT              tfsec               Checkov
  ──────              ─────               ───────
  Speed               ⚡ Very fast         🐢 Slower
  Language            Go                  Python
  Rule count          ~300+               ~1000+
  Frameworks          TF, CFN             TF, K8s, Docker, Helm...
  Custom rules        YAML/JSON           Python, YAML
  Compliance          Basic               CIS, SOC2, HIPAA, PCI...
  Best for            Pre-commit, quick   CI/CD, compliance
  
  
CAN THEY WORK TOGETHER? YES!
─────────────────────────────────────────────────────────────────

  Different rule sets = different catches
  
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   Developer Workstation     CI/CD Pipeline                  │
  │   ─────────────────────     ──────────────                  │
  │                                                             │
  │   git commit                push to main                    │
  │       │                         │                           │
  │       ▼                         ▼                           │
  │   pre-commit hook           build job                       │
  │       │                         │                           │
  │       ▼                         ▼                           │
  │   ┌─────────┐               ┌─────────┐                     │
  │   │  tfsec  │               │ Checkov │                     │
  │   │ (fast!) │               │ (full)  │                     │
  │   └─────────┘               └─────────┘                     │
  │       │                         │                           │
  │   Quick feedback            Comprehensive                   │
  │   before commit             before deploy                   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

### Using tfsec - Quick Fire Code Inspection

```
TFSEC IN ACTION
───────────────

INSTALLATION:
─────────────────────────────────────────────────────────────────

  # macOS
  brew install tfsec
  
  # Or use Trivy (tfsec is now part of Trivy)
  brew install trivy


BASIC USAGE:
─────────────────────────────────────────────────────────────────

  # Scan current directory
  tfsec .
  
  # Scan with specific format
  tfsec . --format json
  tfsec . --format sarif  # For GitHub Security tab
  
  # Scan with minimum severity
  tfsec . --minimum-severity HIGH


EXAMPLE OUTPUT:
─────────────────────────────────────────────────────────────────

  $ tfsec .

  Result #1 HIGH Security group rule allows egress to 0.0.0.0/0
  ─────────────────────────────────────────────────────────────
  
    main.tf:15-20
  
     15 │   resource "aws_security_group_rule" "allow_all" {
     16 │     type              = "egress"
     17 │     cidr_blocks       = ["0.0.0.0/0"]  ← VIOLATION
     18 │     from_port         = 0
     19 │     to_port           = 65535
     20 │   }
  
    ID:          aws-ec2-no-public-egress-sgr
    Impact:      Your port is egressing data to the internet
    Resolution:  Set a more restrictive cidr range
  
    More info: https://aquasecurity.github.io/tfsec/latest/checks/aws/ec2/...
  
  
  Result #2 CRITICAL S3 bucket does not have encryption enabled
  ─────────────────────────────────────────────────────────────
  
    main.tf:25-28
  
     25 │   resource "aws_s3_bucket" "data" {
     26 │     bucket = "my-data-bucket"
     27 │     acl    = "private"
     28 │   }                        ← NO ENCRYPTION!
  
    ID:          aws-s3-encryption-customer-key
    Resolution:  Enable server-side encryption


SUPPRESSING FALSE POSITIVES:
─────────────────────────────────────────────────────────────────

  INLINE SUPPRESSION (with reason!):
  
  #tfsec:ignore:aws-s3-encryption-customer-key:exp:2026-06-01
  # Reason: Using S3 default encryption configured at account level
  resource "aws_s3_bucket" "data" {
    bucket = "my-data-bucket"
  }
  
  EXPLANATION:
  ├── tfsec:ignore:<RULE_ID>  - Which rule to skip
  ├── :exp:YYYY-MM-DD         - Optional expiration date
  └── # Reason: ...           - ALWAYS document why!


CONFIG FILE SUPPRESSION (.tfsec/config.yml):
─────────────────────────────────────────────────────────────────

  # .tfsec/config.yml
  severity_overrides:
    aws-ec2-no-public-egress-sgr: LOW  # Downgrade severity
  
  exclude:
    - aws-s3-encryption-customer-key  # Skip this check globally
```

---

### Using Checkov - Full Compliance Audit

```
CHECKOV IN ACTION
─────────────────

INSTALLATION:
─────────────────────────────────────────────────────────────────

  # pip
  pip install checkov
  
  # brew
  brew install checkov
  
  # Docker
  docker run -t -v $(pwd):/tf bridgecrew/checkov -d /tf


BASIC USAGE:
─────────────────────────────────────────────────────────────────

  # Scan directory
  checkov -d .
  
  # Scan with specific framework
  checkov -d . --framework terraform
  
  # Scan with compliance framework
  checkov -d . --check CIS_AWS
  checkov -d . --check SOC2
  checkov -d . --check HIPAA
  
  # Output formats
  checkov -d . -o json
  checkov -d . -o sarif


EXAMPLE OUTPUT:
─────────────────────────────────────────────────────────────────

  $ checkov -d .
  
  
       _               _              
      ___| |__   ___  ___| | _______   __
     / __| '_ \ / _ \/ __| |/ / _ \ \ / /
    | (__| | | |  __/ (__|   < (_) \ V / 
     \___|_| |_|\___|\___|_|\_\___/ \_/  
  
  By Prisma Cloud | version: 3.1.0
  
  terraform scan results:
  
  Passed checks: 45
  Failed checks: 3
  Skipped checks: 1
  
  Check: CKV_AWS_20: "Ensure S3 bucket has encryption enabled"
    FAILED for resource: aws_s3_bucket.data
    File: /main.tf:25-28
    Guide: https://docs.prismacloud.io/en/enterprise-edition/...
  
  Check: CKV_AWS_23: "Ensure every security group rule has a description"
    FAILED for resource: aws_security_group_rule.allow_all
    File: /main.tf:15-20


SUPPRESSING WITH CHECKOV:
─────────────────────────────────────────────────────────────────

  INLINE SUPPRESSION:
  
  resource "aws_s3_bucket" "public_website" {
    # checkov:skip=CKV_AWS_20:Bucket hosts public static website, encryption not required
    bucket = "my-public-website"
  }
  
  BREAKDOWN:
  ├── checkov:skip=<CHECK_ID>   - Which check to skip
  └── :<REASON>                 - Justification (REQUIRED!)


CONFIG FILE SUPPRESSION (.checkov.yaml):
─────────────────────────────────────────────────────────────────

  # .checkov.yaml
  skip-check:
    - CKV_AWS_20  # Skip encryption check globally
    - CKV_AWS_23  # Skip SG description check
  
  soft-fail:
    - CKV_AWS_145  # Warn but don't fail pipeline
  
  framework:
    - terraform
  
  compact: true
  
  
SKIP VIA CLI (for CI/CD):
─────────────────────────────────────────────────────────────────

  checkov -d . --skip-check CKV_AWS_20,CKV_AWS_23
```

---

### Shift-Left Security - Catch Issues Early

```
THE SHIFT-LEFT PRINCIPLE
────────────────────────

TRADITIONAL (SHIFT-RIGHT):
─────────────────────────────────────────────────────────────────

  Developer → Commit → Build → Deploy → PRODUCTION
       │                                    │
       │                                    ▼
       │                               Security Scan
       │                                    │
       │                                    ▼
       │                               💥 BREACH!
       │                               (too late)
       │                                    │
       └────────── Fix takes weeks ─────────┘
  
  COST OF FIX: $$$$$$ (production incident)


SHIFT-LEFT (MODERN):
─────────────────────────────────────────────────────────────────

  Developer → Pre-commit → PR Check → Pre-deploy → Production
       │          │            │           │
       ▼          ▼            ▼           ▼
   IDE Plugin   tfsec       Checkov    Final gate
       │          │            │           │
       │     Fast scan   Comprehensive  Sentinel
       │     (seconds)   (minutes)      (policy)
       │          │            │           │
       ▼          ▼            ▼           ▼
   ❌ BLOCKED  ❌ BLOCKED   ❌ BLOCKED   ❌ BLOCKED
   (immediate)  (pre-commit) (CI/CD)    (pre-apply)
  
  COST OF FIX: $ (caught early, no breach)


WHY SHIFT-LEFT MATTERS:
─────────────────────────────────────────────────────────────────

  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │   STAGE                    COST TO FIX                     │
  │   ─────                    ───────────                     │
  │                                                            │
  │   Design/Development       $1 (trivial)                    │
  │   Code Review              $5                              │
  │   CI/CD Pipeline           $10                             │
  │   Staging Environment      $50                             │
  │   Production               $500+                           │
  │   Post-breach              $50,000+ (+ reputation)         │
  │                                                            │
  │   RULE: The later you catch it, the more expensive!        │
  │                                                            │
  └────────────────────────────────────────────────────────────┘
```

---

### CI/CD Integration - Automated Security Gates

```
CI/CD SECURITY PIPELINE
───────────────────────

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   GITHUB ACTIONS EXAMPLE                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

  # .github/workflows/terraform-security.yml
  
  name: Terraform Security Scan
  
  on:
    pull_request:
      paths:
        - '**.tf'
        - '**.tfvars'
  
  jobs:
    tfsec:
      name: tfsec scan
      runs-on: ubuntu-latest
      steps:
        - name: Checkout
          uses: actions/checkout@v4
        
        - name: tfsec
          uses: aquasecurity/tfsec-action@v1.0.0
          with:
            soft_fail: false  # Fail pipeline on issues
    
    checkov:
      name: Checkov scan
      runs-on: ubuntu-latest
      steps:
        - name: Checkout
          uses: actions/checkout@v4
        
        - name: Checkov
          uses: bridgecrewio/checkov-action@v12
          with:
            directory: .
            framework: terraform
            soft_fail: false
            output_format: sarif
            download_external_modules: true
        
        - name: Upload SARIF
          uses: github/codeql-action/upload-sarif@v2
          with:
            sarif_file: results.sarif


PRE-COMMIT HOOKS (Local Development):
─────────────────────────────────────────────────────────────────

  # .pre-commit-config.yaml
  
  repos:
    - repo: https://github.com/antonbabenko/pre-commit-terraform
      rev: v1.86.0
      hooks:
        - id: terraform_fmt
        - id: terraform_validate
        - id: terraform_tfsec
          args:
            - --args=--minimum-severity=HIGH
        - id: terraform_checkov
          args:
            - --args=--quiet
  
  # Install hooks
  pre-commit install
  
  # Now every commit runs security scan!


PIPELINE STAGES:
─────────────────────────────────────────────────────────────────

  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   STAGE 1: PRE-COMMIT (Developer machine)                │
  │   ─────────────────────────────────────────              │
  │   • terraform fmt (formatting)                           │
  │   • terraform validate (syntax)                          │
  │   • tfsec (quick security scan)                          │
  │                                                          │
  │   ⏱️ Speed: Seconds                                       │
  │   🎯 Goal: Immediate feedback                            │
  │                                                          │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │   STAGE 2: PR CHECKS (CI/CD)                             │
  │   ─────────────────────────────────────────              │
  │   • tfsec (full scan)                                    │
  │   • Checkov (compliance scan)                            │
  │   • terraform plan (validate changes)                    │
  │   • Cost estimation (Infracost)                          │
  │                                                          │
  │   ⏱️ Speed: Minutes                                       │
  │   🎯 Goal: Block bad PRs from merging                    │
  │                                                          │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │   STAGE 3: PRE-APPLY (Before terraform apply)            │
  │   ─────────────────────────────────────────              │
  │   • Sentinel policies (Enterprise)                       │
  │   • Final security gate                                  │
  │   • Manual approval (for prod)                           │
  │                                                          │
  │   ⏱️ Speed: Minutes                                       │
  │   🎯 Goal: Last line of defense                          │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

---

## How Things Connect

```
THE COMPLETE SECURITY FLOW
──────────────────────────

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   1. DEVELOPER WRITES TERRAFORM                                 │
│      │                                                          │
│      │  main.tf:                                                │
│      │  ┌─────────────────────────────────────────────────┐     │
│      │  │ resource "aws_s3_bucket" "data" {               │     │
│      │  │   bucket = "my-bucket"                          │     │
│      │  │   # Oops, forgot encryption!                    │     │
│      │  │ }                                               │     │
│      │  └─────────────────────────────────────────────────┘     │
│      │                                                          │
│      ▼                                                          │
│   2. PRE-COMMIT HOOK RUNS                                       │
│      │                                                          │
│      │  $ git commit -m "Add S3 bucket"                         │
│      │                                                          │
│      │  Running tfsec...                                        │
│      │  ❌ CRITICAL: S3 bucket missing encryption               │
│      │                                                          │
│      │  Commit blocked! Fix the issue.                          │
│      │                                                          │
│      ▼                                                          │
│   3. DEVELOPER FIXES                                            │
│      │                                                          │
│      │  main.tf:                                                │
│      │  ┌─────────────────────────────────────────────────┐     │
│      │  │ resource "aws_s3_bucket_server_side_encryption_configuration" │     │
│      │  │   bucket = aws_s3_bucket.data.id                │     │
│      │  │   rule {                                        │     │
│      │  │     apply_server_side_encryption_by_default {   │     │
│      │  │       sse_algorithm = "AES256"                  │     │
│      │  │     }                                           │     │
│      │  │   }                                             │     │
│      │  │ }                                               │     │
│      │  └─────────────────────────────────────────────────┘     │
│      │                                                          │
│      ▼                                                          │
│   4. PRE-COMMIT PASSES                                          │
│      │                                                          │
│      │  $ git commit -m "Add S3 bucket with encryption"         │
│      │  Running tfsec... ✅ Passed                              │
│      │                                                          │
│      ▼                                                          │
│   5. PR CREATED → CI/CD RUNS                                    │
│      │                                                          │
│      │  GitHub Actions:                                         │
│      │  ├── tfsec ✅                                            │
│      │  ├── Checkov ✅                                          │
│      │  ├── terraform plan ✅                                   │
│      │  └── All checks passed!                                  │
│      │                                                          │
│      ▼                                                          │
│   6. MERGE → DEPLOY                                             │
│      │                                                          │
│      │  terraform apply                                         │
│      │  ├── Creating S3 bucket... done                          │
│      │  ├── Enabling encryption... done                         │
│      │  └── Apply complete! ✅                                  │
│      │                                                          │
│      ▼                                                          │
│   7. SECURE INFRASTRUCTURE DEPLOYED 🎉                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Never hardcode secrets in `.tf` files**
   - Use environment variables, Secrets Manager, or Vault
   - Secrets in Git history are there FOREVER
   - Anyone with repo access = anyone with secrets

2. **State files contain secrets in plain text**
   - Use remote backend with encryption (S3 + SSE)
   - Restrict state access via IAM
   - Enable versioning for audit trail

3. **`sensitive = true` only hides output, not state**
   - Masks values in CLI and logs
   - Does NOT encrypt in state file
   - Use `ephemeral = true` (TF 1.10+) for true protection

4. **tfsec vs Checkov - use both!**
   - tfsec: Fast, great for pre-commit hooks
   - Checkov: Comprehensive, great for CI/CD compliance
   - Different rules = catch different issues

5. **Shift-left: catch issues early**
   - Pre-commit hooks for instant feedback
   - PR checks for team enforcement
   - Pre-apply gates for final defense

6. **Document all suppressions**
   - Never skip without a reason
   - Include ticket numbers for audit trail
   - Set expiration dates when possible

---

*Written on January 28, 2026*
