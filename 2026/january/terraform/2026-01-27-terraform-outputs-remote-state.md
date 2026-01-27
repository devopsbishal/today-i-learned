# Terraform Outputs & Remote State - The Building Specs Shared Between Architects

> Understanding Terraform outputs and remote state data sources through a real-world analogy of architects sharing building specifications and blueprints across different projects.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Output | Published building specs on the bulletin board |
| `output` block | "Here's the info other architects might need" |
| `value` attribute | The actual specification (address, ID, connection point) |
| `description` | Label explaining what this spec is for |
| `sensitive = true` | "Don't read this aloud in meetings" (hidden from logs) |
| Root module output | Specs posted on the PUBLIC bulletin board |
| Child module output | Internal memo (only parent module can see) |
| `terraform_remote_state` | Walking to another architect's office to read their bulletin board |
| `backend` config | Which office building (S3 bucket, Terraform Cloud) |
| `config` block | Office address and room number (bucket, key, region) |
| `defaults` argument | "If their board is empty, assume these values" |
| `outputs` attribute | Reading specific specs from their bulletin board |
| Cross-stack reference | Team A's network specs used by Team B's application |
| SSM Parameter Store | City's central bulletin board (any department can post/read) |
| Data source alternative | Calling city hall directly instead of reading another architect's notes |

---

## The Big Picture

Imagine you're in a **large architecture firm** with multiple teams working on different parts of a development project. The **networking team** designs roads and utilities, the **application team** designs office buildings, and the **database team** designs data centers.

Each team needs to **share specifications** with other teams:

```
ARCHITECTURE FIRM - SHARING BUILDING SPECS
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                    THE ARCHITECTURE FIRM                     │
│                                                              │
│  ┌─────────────────────┐      ┌─────────────────────┐       │
│  │  NETWORKING TEAM    │      │  APPLICATION TEAM   │       │
│  │     (VPC Stack)     │      │    (App Stack)      │       │
│  │                     │      │                     │       │
│  │  Creates:           │      │  Needs to know:     │       │
│  │  • VPC              │      │  • VPC ID           │       │
│  │  • Subnets          │      │  • Subnet IDs       │       │
│  │  • Security Groups  │      │  • Security Group   │       │
│  │                     │      │                     │       │
│  │  ┌───────────────┐  │      │  ┌───────────────┐  │       │
│  │  │ BULLETIN BOARD│  │      │  │ READS FROM:   │  │       │
│  │  │ (Outputs)     │  │ ───► │  │ remote_state  │  │       │
│  │  │               │  │      │  │               │  │       │
│  │  │ vpc_id: xyz   │  │      │  │ "Give me the  │  │       │
│  │  │ subnet_ids:   │  │      │  │  vpc_id from  │  │       │
│  │  │   [a, b, c]   │  │      │  │  networking"  │  │       │
│  │  │ sg_id: abc    │  │      │  │               │  │       │
│  │  └───────────────┘  │      │  └───────────────┘  │       │
│  └─────────────────────┘      └─────────────────────┘       │
│                                                              │
│  STATE FILE (The Office Files)                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ s3://terraform-state/network/terraform.tfstate          ││
│  │ s3://terraform-state/app/terraform.tfstate              ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Without outputs and remote state:**
- Teams would hardcode values: "VPC ID is vpc-abc123" (breaks when it changes!)
- No formal contract between teams
- Manual communication: "Hey, what's the VPC ID again?"
- Tight coupling: everyone needs to know internal details

**With outputs and remote state:**
- Teams publish what others need (and ONLY what they need)
- Clear interface: "Here's what I provide, here's what I need"
- Dynamic references: always get the latest value
- Loose coupling: consume the interface, not the internals

---

## Core Components

### Outputs - The Bulletin Board

An output is like **posting specifications on your team's bulletin board** for other teams (or your future self) to reference.

```
NETWORKING TEAM'S BULLETIN BOARD
─────────────────────────────────

┌────────────────────────────────────────────┐
│           📋 PUBLISHED SPECS               │
│                                            │
│  VPC ID                                    │
│  ─────────────────────────────             │
│  Value: vpc-0abc123def456                  │
│  Description: "Main VPC for production"    │
│                                            │
│  ─────────────────────────────             │
│  SUBNET IDs                                │
│  ─────────────────────────────             │
│  Value: [subnet-aaa, subnet-bbb]           │
│  Description: "Private subnets for apps"   │
│                                            │
│  ─────────────────────────────             │
│  🔒 DATABASE PASSWORD (SENSITIVE)          │
│  ─────────────────────────────             │
│  Value: ********** (hidden!)               │
│  Note: "Don't read aloud, check files"     │
│                                            │
└────────────────────────────────────────────┘
```

**Terraform equivalent:**
```hcl
# Networking team's outputs.tf

output "vpc_id" {
  description = "Main VPC for production workloads"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "Private subnets for application deployment"
  value       = aws_subnet.private[*].id
}

output "database_password" {
  description = "RDS master password - handle with care!"
  value       = random_password.db.result
  sensitive   = true  # Won't show in terminal output
}
```

#### Output Attributes

| Attribute | Purpose | Required? |
|-----------|---------|-----------|
| `value` | The actual data to expose | ✅ Yes |
| `description` | Human-readable explanation | ❌ No (but recommended!) |
| `sensitive` | Hide from CLI output | ❌ No (default: false) |
| `depends_on` | Wait for resource before computing | ❌ No (rarely needed) |
| `precondition` | Validate before exposing (TF 1.2+) | ❌ No |

#### Root vs Child Module Outputs

```
OUTPUT VISIBILITY
─────────────────────────────────────────────────────────────────

ROOT MODULE                          CHILD MODULE
(Your main configuration)            (Called module)
                                     
outputs.tf                           modules/vpc/outputs.tf
┌─────────────────────┐              ┌─────────────────────┐
│ output "vpc_id" {   │              │ output "vpc_id" {   │
│   value = ...       │              │   value = ...       │
│ }                   │              │ }                   │
└─────────────────────┘              └─────────────────────┘
        │                                    │
        ▼                                    ▼
   VISIBLE TO:                          VISIBLE TO:
   • CLI output                         • Parent module ONLY
   • terraform_remote_state             • NOT in CLI
   • terraform output command           • NOT in remote_state
   
   "PUBLIC bulletin board"              "INTERNAL memo"
```

**Key insight:** Only ROOT module outputs are accessible via `terraform_remote_state`. Child module outputs must be "re-exported" by the root module.

```hcl
# Root module re-exporting child module output
module "network" {
  source = "./modules/vpc"
}

# Re-export to make it available externally
output "vpc_id" {
  value = module.network.vpc_id  # Now it's on the PUBLIC board
}
```

---

### Remote State Data Source - Reading Another Team's Bulletin Board

The `terraform_remote_state` data source is like **walking to another team's office** and reading their published specifications.

```
CROSS-TEAM SPEC SHARING
─────────────────────────────────────────────────────────────────

APPLICATION TEAM: "I need the VPC ID to deploy my EC2 instances"

ACTION: Walk to Networking Team's office, read their bulletin board

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  APPLICATION TEAM'S CONFIG                                   │
│                                                              │
│  data "terraform_remote_state" "network" {                   │
│    backend = "s3"                     ← Which building?      │
│                                         (S3, Terraform Cloud)│
│    config = {                                                │
│      bucket = "company-tf-state"      ← Which office?        │
│      key    = "network/terraform.tfstate"  ← Which room?     │
│      region = "us-east-1"             ← Which city?          │
│    }                                                         │
│  }                                                           │
│                                                              │
│  # Now read from their bulletin board:                       │
│  resource "aws_instance" "app" {                             │
│    subnet_id = data.terraform_remote_state.network           │
│                    .outputs.private_subnet_ids[0]            │
│  }                     ▲                                     │
│                        │                                     │
│              "Read the first subnet from                     │
│               networking team's published specs"             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### Backend Configuration by Type

**For S3 Backend:**
```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"           # S3 bucket name
    key    = "network/terraform.tfstate"    # Path to state file
    region = "us-east-1"                    # Bucket region
  }
}
```

**For Terraform Cloud:**
```hcl
data "terraform_remote_state" "network" {
  backend = "remote"
  config = {
    organization = "my-company"
    workspaces = {
      name = "network-prod"
    }
  }
}
```

#### The Defaults Argument - Handling Empty Bulletin Boards

What if you try to read another team's specs, but they haven't posted anything yet?

```
BOOTSTRAPPING PROBLEM
─────────────────────────────────────────────────────────────────

DAY 1:
  App Team: "I need VPC ID from networking!"
  Network Team: "We haven't applied yet... board is empty"
  App Team: 💥 ERROR! No outputs found!

SOLUTION: Defaults
  App Team: "If board is empty, assume these placeholder values"
```

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
  
  # Fallback values if outputs don't exist yet
  defaults = {
    vpc_id            = ""         # Empty string as placeholder
    private_subnet_ids = []        # Empty list
  }
}

# Safe to reference even if network stack not applied:
resource "aws_instance" "app" {
  count     = length(data.terraform_remote_state.network.outputs.private_subnet_ids) > 0 ? 1 : 0
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
}
```

---

### The Security Consideration - Full State Access

**Critical insight:** When you read `terraform_remote_state`, you get read access to the **entire state file**, not just the outputs.

```
SECURITY IMPLICATION
─────────────────────────────────────────────────────────────────

What you WANT:                     What you GET:
┌──────────────────┐               ┌───────────────────────────┐
│ Just the VPC ID  │               │ ENTIRE STATE FILE         │
│ please!          │               │                           │
│                  │               │ • VPC ID ✓                │
│                  │               │ • Subnet IDs              │
│                  │               │ • Security group rules    │
│                  │               │ • IAM role ARNs           │
│                  │               │ • Database passwords! ⚠️   │
│                  │               │ • API keys! ⚠️             │
│                  │               │ • All resource attributes │
│                  │               │                           │
└──────────────────┘               └───────────────────────────┘

The app team can see EVERYTHING in network team's state!
```

**This is why alternatives exist:**

| Method | Access Scope | Use Case |
|--------|--------------|----------|
| `terraform_remote_state` | Entire state file | Trusted teams, same org |
| SSM Parameter Store | Individual parameters | Fine-grained access control |
| Data sources (aws_vpc, etc.) | Live resource lookup | Query by tags, no state coupling |
| Secrets Manager | Individual secrets | Cross-tool access, rotation |

---

## Alternatives to Remote State

### Option 1: SSM Parameter Store - The City's Central Board

Instead of each team having their own bulletin board, post to a **central city bulletin board** that anyone can read (with proper permissions).

```
CENTRAL BULLETIN BOARD (SSM)
─────────────────────────────────────────────────────────────────

NETWORKING TEAM POSTS:                APP TEAM READS:
                                      
resource "aws_ssm_parameter" "vpc" {  data "aws_ssm_parameter" "vpc" {
  name  = "/infra/prod/vpc_id"          name = "/infra/prod/vpc_id"
  type  = "String"                    }
  value = aws_vpc.main.id             
}                                     # Use it:
                                      vpc_id = data.aws_ssm_parameter
                                                   .vpc.value
```

**Advantages over remote_state:**
- Fine-grained IAM: control access per parameter
- Cross-tool: Lambda, EC2, scripts can all read
- No state file exposure

### Option 2: Data Sources - Call City Hall Directly

Instead of reading another architect's notes, **call the city directly** and ask about existing infrastructure.

```hcl
# Instead of reading network team's state:
data "aws_vpc" "production" {
  tags = {
    Name        = "production"
    Environment = "prod"
  }
}

# Use the live value:
resource "aws_instance" "app" {
  subnet_id = data.aws_vpc.production.id
  # ...
}
```

**Advantages:**
- No dependency on another team's state
- Always gets current live value
- No need for state file access

**Disadvantages:**
- Requires consistent tagging strategy
- More AWS API calls
- Can't get computed values that aren't in AWS

---

## How Things Connect

```
COMPLETE OUTPUT & REMOTE STATE FLOW
───────────────────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  PRODUCER STACK (Networking Team)                            │
│                                                              │
│  1. Create resources                                         │
│     ┌─────────────────────────────────────┐                  │
│     │ resource "aws_vpc" "main" {         │                  │
│     │   cidr_block = "10.0.0.0/16"        │                  │
│     │ }                                   │                  │
│     │                                     │                  │
│     │ resource "aws_subnet" "private" {   │                  │
│     │   count  = 3                        │                  │
│     │   vpc_id = aws_vpc.main.id          │                  │
│     │ }                                   │                  │
│     └───────────────────┬─────────────────┘                  │
│                         │                                    │
│                         ▼                                    │
│  2. Publish outputs (bulletin board)                         │
│     ┌─────────────────────────────────────┐                  │
│     │ output "vpc_id" {                   │                  │
│     │   value = aws_vpc.main.id           │                  │
│     │ }                                   │                  │
│     │                                     │                  │
│     │ output "private_subnet_ids" {       │                  │
│     │   value = aws_subnet.private[*].id  │                  │
│     │ }                                   │                  │
│     └───────────────────┬─────────────────┘                  │
│                         │                                    │
│                         ▼                                    │
│  3. Store in state (S3)                                      │
│     ┌─────────────────────────────────────┐                  │
│     │ s3://state/network/terraform.tfstate│                  │
│     │                                     │                  │
│     │ {                                   │                  │
│     │   "outputs": {                      │                  │
│     │     "vpc_id": { "value": "vpc-123" }│                  │
│     │     "private_subnet_ids": {         │                  │
│     │       "value": ["sub-a", "sub-b"]   │                  │
│     │     }                               │                  │
│     │   }                                 │                  │
│     │ }                                   │                  │
│     └───────────────────┬─────────────────┘                  │
│                         │                                    │
└─────────────────────────┼────────────────────────────────────┘
                          │
          ┌───────────────┘
          │ Read via terraform_remote_state
          ▼
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  CONSUMER STACK (Application Team)                           │
│                                                              │
│  1. Read remote state                                        │
│     ┌─────────────────────────────────────┐                  │
│     │ data "terraform_remote_state" "net" │                  │
│     │   backend = "s3"                    │                  │
│     │   config = {                        │                  │
│     │     bucket = "state"                │                  │
│     │     key    = "network/tf.tfstate"   │                  │
│     │   }                                 │                  │
│     │ }                                   │                  │
│     └───────────────────┬─────────────────┘                  │
│                         │                                    │
│                         ▼                                    │
│  2. Use outputs in resources                                 │
│     ┌─────────────────────────────────────┐                  │
│     │ resource "aws_instance" "app" {     │                  │
│     │   vpc_id    = data.terraform_remote │                  │
│     │               _state.net.outputs    │                  │
│     │               .vpc_id               │                  │
│     │                                     │                  │
│     │   subnet_id = data.terraform_remote │                  │
│     │               _state.net.outputs    │                  │
│     │               .private_subnet_ids[0]│                  │
│     │ }                                   │                  │
│     └─────────────────────────────────────┘                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Common Patterns

### Pattern 1: Layered Infrastructure Stacks

```
TYPICAL MULTI-STACK ARCHITECTURE
─────────────────────────────────────────────────────────────────

Layer 3: APPLICATION STACK
    │     Reads from: network, database
    │     Outputs: app_url, load_balancer_dns
    │
    ▼
Layer 2: DATABASE STACK
    │     Reads from: network
    │     Outputs: db_endpoint, db_port
    │
    ▼
Layer 1: NETWORK STACK
          Reads from: nothing (foundation)
          Outputs: vpc_id, subnet_ids, security_groups
```

### Pattern 2: Environment-Specific Remote State

```hcl
# Dynamic remote state based on environment
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "company-tf-state"
    key    = "${var.environment}/network/terraform.tfstate"  # dev/, staging/, prod/
    region = "us-east-1"
  }
}
```

### Pattern 3: Workspace-Aware Remote State

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "company-tf-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
  
  # Read from matching workspace
  workspace = terraform.workspace  # If network stack uses same workspaces
}
```

---

## Key Takeaways

1. **Outputs are your public API**
   - Only expose what consumers need
   - Add descriptions for documentation
   - Use `sensitive = true` for secrets (still in state though!)

2. **Root module outputs only**
   - Child module outputs aren't visible externally
   - Re-export in root module if needed for remote state

3. **Remote state = full state access**
   - Consumer sees EVERYTHING in producer's state
   - Security concern for cross-team sharing
   - Consider alternatives: SSM, data sources, Secrets Manager

4. **Use defaults for bootstrapping**
   - Prevents errors when producer hasn't applied yet
   - Enables gradual/parallel stack development

5. **Alternatives based on use case**
   - Same team, trusted: `terraform_remote_state`
   - Cross-team, fine-grained: SSM Parameter Store
   - Live lookup, no coupling: Data sources
   - Secrets: Secrets Manager or Vault

6. **Sensitive outputs propagate**
   - If producer marks output sensitive, it stays sensitive when consumed
   - But remember: still stored in plaintext in state!

---

*Written on January 27, 2026*
