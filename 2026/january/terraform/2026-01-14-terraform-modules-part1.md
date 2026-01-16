# Terraform Modules Part 1 - The Architectural Blueprint Library

> Understanding Terraform modules through a real-world analogy of an architecture firm with a library of reusable floor plans.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Module | Reusable floor plan template |
| Root module | The main building project using floor plans |
| Child module | A floor plan pulled from the library |
| Module source | Where to find the floor plan (local drawer, public library) |
| Input variables | Customization options (room size, window count) |
| Output values | Specifications shared after building (address, dimensions) |
| Module call | "I want to use the 3-bedroom floor plan" |
| Module arguments | "But make the kitchen 200 sqft and add a balcony" |
| `module.name.output` | "What's the final square footage of that unit?" |
| Local modules | Floor plans in your office filing cabinet |
| Remote modules | Floor plans from public architecture registry |
| Module composition | A hotel blueprint using room, bathroom, and hallway plans |
| `for_each` on module | "Build 10 units using this floor plan" |
| Required variable | "You MUST specify the lot size" |
| Optional variable (default) | "Ceiling height defaults to 9ft unless specified" |

---

## The Big Picture

Imagine you work at an **architecture firm** that designs and builds many buildings across the city. Every project needs common elements: apartments, office floors, parking garages, utility rooms.

Without a system, every architect would design these from scratch every time:

```
Project: Downtown Tower
├── Architect A designs apartment layout... (40 hours)
├── Architect B designs parking garage... (30 hours)
└── Architect C designs utility room... (20 hours)

Project: Uptown Complex
├── Architect D designs apartment layout... (40 hours) ← Same thing!
├── Architect E designs parking garage... (30 hours) ← Same thing!
└── Architect F designs utility room... (20 hours) ← Same thing!

Total: 180 hours of duplicate work!
```

**The solution: A Blueprint Library**

Your firm creates a **library of reusable floor plans**. Each floor plan is a template with customization options:

```
BLUEPRINT LIBRARY
├── 3-bedroom-apartment/
│   ├── floor_plan.dwg       ← The design
│   ├── customization.doc    ← "Choose: 1-3 bathrooms, balcony Y/N"
│   └── specifications.doc   ← "Final sq ft: X, rooms: Y"
│
├── parking-garage/
│   ├── floor_plan.dwg
│   ├── customization.doc    ← "Choose: 50-500 spaces"
│   └── specifications.doc   ← "Total spaces: X, levels: Y"
│
└── utility-room/
    ├── floor_plan.dwg
    ├── customization.doc    ← "Choose: residential/commercial"
    └── specifications.doc   ← "Electrical capacity: X"
```

Now, any architect can build a complete building by **combining floor plans**:

```
Project: Downtown Tower
├── USE 3-bedroom-apartment (20 units, with balcony) ← 2 minutes!
├── USE parking-garage (200 spaces)                  ← 2 minutes!
└── USE utility-room (commercial)                    ← 2 minutes!

Total: 6 minutes instead of 90 hours!
```

**This is exactly what Terraform modules do!**

- **Module** = Reusable floor plan
- **variables.tf** = Customization options
- **outputs.tf** = Specifications after building
- **Root module** = Your main project that uses floor plans
- **Module call** = "Use this floor plan with these customizations"

---

## Core Components

### The Module = A Reusable Floor Plan

A Terraform module is simply a **directory containing `.tf` files** - just like a floor plan folder in your library.

```
Your Desk (Root Module)              Blueprint Library (Modules)
─────────────────────                ─────────────────────────
main.tf                              modules/
variables.tf                         ├── ec2-instance/
outputs.tf                           │   ├── main.tf        (floor plan)
                                     │   ├── variables.tf   (customizations)
                                     │   └── outputs.tf     (specifications)
                                     │
                                     └── vpc/
                                         ├── main.tf
                                         ├── variables.tf
                                         └── outputs.tf
```

**Module structure breakdown:**

```
modules/ec2-instance/
│
├── main.tf              ← The actual floor plan design
│   │                       (resource definitions)
│   │
│   │   resource "aws_instance" "this" {
│   │     ami           = var.ami_id
│   │     instance_type = var.instance_type
│   │   }
│   │
│
├── variables.tf         ← Customization options form
│   │                       (what can be changed?)
│   │
│   │   variable "instance_type" {
│   │     default = "t2.micro"  ← "Ceiling 9ft unless specified"
│   │   }
│   │
│   │   variable "ami_id" {
│   │     # No default!          ← "You MUST specify lot size"
│   │   }
│   │
│
├── outputs.tf           ← Final specifications sheet
│   │                       (what info do we provide back?)
│   │
│   │   output "instance_id" {
│   │     value = aws_instance.this.id
│   │   }
│   │
│   │   output "public_ip" {
│   │     value = aws_instance.this.public_ip
│   │   }
│
└── README.md            ← User manual for the floor plan
```

**Analogy:**

```
┌─────────────────────────────────────────────────────────────┐
│            FLOOR PLAN: 3-Bedroom Apartment                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CUSTOMIZATION OPTIONS (variables.tf):                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ □ Square footage: _______ (REQUIRED)                │    │
│  │ □ Number of bathrooms: 2 (default, can override)    │    │
│  │ □ Balcony: Yes / No (default: No)                   │    │
│  │ □ Flooring type: Hardwood (default, can override)   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  FINAL SPECIFICATIONS (outputs.tf):                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ After building, you will receive:                   │    │
│  │ • Unit ID (apartment number)                        │    │
│  │ • Final square footage                              │    │
│  │ • Address                                           │    │
│  │ • Window count                                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  [See main.tf for actual floor plan design]                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Calling a Module = Using a Floor Plan

When you want to use a floor plan in your project, you "call" it:

```hcl
# main.tf (Your project - the root module)

module "web_server" {
  source = "./modules/ec2-instance"   # Where to find the floor plan
  
  # Customization options:
  ami_id        = "ami-12345"         # Required (no default)
  instance_type = "t3.medium"         # Override the default
  # instance_name uses default        # Not specified, uses default
}
```

**What happens:**

```
┌────────────────────────────────────────────────────────────┐
│                    YOUR PROJECT DESK                        │
│                    (Root Module)                            │
│                                                             │
│   "I want to use the EC2 floor plan"                        │
│                                                             │
│   module "web_server" {                                     │
│     source = "./modules/ec2-instance"  ─────────┐           │
│     ami_id = "ami-12345"               ─────────┼──┐        │
│     instance_type = "t3.medium"        ─────────┼──┼──┐     │
│   }                                             │  │  │     │
│                                                 │  │  │     │
└─────────────────────────────────────────────────┼──┼──┼─────┘
                                                  │  │  │
                    ┌─────────────────────────────┼──┼──┼─────┐
                    │  BLUEPRINT LIBRARY          │  │  │     │
                    │  modules/ec2-instance/      ▼  │  │     │
                    │                                │  │     │
                    │  1. Find floor plan            │  │     │
                    │                                ▼  │     │
                    │  2. Read customization:           │     │
                    │     var.ami_id = "ami-12345"      ▼     │
                    │     var.instance_type = "t3.medium"     │
                    │                                         │
                    │  3. Build according to plan             │
                    │     aws_instance.this { ... }           │
                    │                                         │
                    │  4. Return specifications:              │
                    │     output.instance_id = "i-abc123"     │
                    │     output.public_ip = "52.1.2.3"       │
                    │                                         │
                    └─────────────────────────────────────────┘
```

---

### Variables = Customization Options

Variables are the **customization form** for your floor plan.

**Required vs Optional:**

```hcl
# variables.tf in the module

# REQUIRED - No default, MUST be provided
# Like: "You MUST tell us the lot size before we can build"
variable "vpc_id" {
  type        = string
  description = "VPC where instance will be deployed"
  # No default! Caller must provide this.
}

# OPTIONAL - Has default, can be overridden
# Like: "Ceiling height is 9ft unless you want something different"
variable "instance_type" {
  type        = string
  default     = "t2.micro"
  description = "EC2 instance size"
}

# OPTIONAL with validation
# Like: "Choose 1-5 bedrooms only"
variable "environment" {
  type    = string
  default = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

**Analogy - The Customization Form:**

```
┌─────────────────────────────────────────────────────────────┐
│         FLOOR PLAN CUSTOMIZATION FORM                        │
│         Module: ec2-instance                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REQUIRED FIELDS (must fill out):                            │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ VPC ID: _________________________ (required)        │     │
│  │         Cannot proceed without this!                │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  OPTIONAL FIELDS (defaults provided):                        │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ Instance Type: [t2.micro______] (default: t2.micro) │     │
│  │ Environment:   [dev___________] (default: dev)      │     │
│  │                ⚠️  Must be: dev, staging, or prod    │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  [Submit to build]                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### Outputs = Specifications After Building

Outputs are the **specifications sheet** you receive after the building is constructed.

```hcl
# outputs.tf in the module

output "instance_id" {
  value       = aws_instance.this.id
  description = "The EC2 instance ID"
}

output "public_ip" {
  value       = aws_instance.this.public_ip
  description = "Public IP address of the instance"
}

output "private_ip" {
  value       = aws_instance.this.private_ip
  description = "Private IP address of the instance"
}
```

**Accessing outputs from root module:**

```hcl
# Root module - after calling the module

# Use in another resource
resource "aws_route53_record" "web" {
  zone_id = data.aws_route53_zone.main.id
  name    = "web.example.com"
  type    = "A"
  records = [module.web_server.public_ip]  # ← Access the output!
}

# Expose to the world
output "web_server_address" {
  value = module.web_server.public_ip
}
```

**Analogy:**

```
After construction, you receive the specification sheet:

┌─────────────────────────────────────────────────────────────┐
│         COMPLETED BUILDING SPECIFICATIONS                    │
│         Project: web_server                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Building Details:                                           │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ Unit ID:        i-abc123def456                      │     │
│  │ Street Address: 52.1.2.3 (public IP)                │     │
│  │ Private Line:   10.0.1.45 (private IP)              │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  Access these specs using:                                   │
│    module.web_server.instance_id                             │
│    module.web_server.public_ip                               │
│    module.web_server.private_ip                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### The Data Flow = Customization → Build → Specifications

Here's the complete flow:

```
┌─────────────────────────────────────────────────────────────────┐
│                         ROOT MODULE                              │
│                     (Your Project Desk)                          │
│                                                                  │
│   Step 1: "I want to use this floor plan with these options"     │
│                                                                  │
│   module "web_server" {                                          │
│     source        = "./modules/ec2-instance"                     │
│     ami_id        = "ami-12345"     ────────────────┐            │
│     instance_type = "t3.medium"     ────────────────┼─┐          │
│   }                                                 │ │          │
│                                                     │ │          │
│   Step 4: "What's the IP address?"                  │ │          │
│                                                     │ │          │
│   output "server_ip" {                              │ │          │
│     value = module.web_server.public_ip  ◄──────────┼─┼────┐     │
│   }                                                 │ │    │     │
│                                                     │ │    │     │
└─────────────────────────────────────────────────────┼─┼────┼─────┘
                                                      │ │    │
┌─────────────────────────────────────────────────────┼─┼────┼─────┐
│                      CHILD MODULE                   │ │    │     │
│               (modules/ec2-instance/)               ▼ ▼    │     │
│                                                            │     │
│   Step 2: "Receive customizations"                         │     │
│                                                            │     │
│   variable "ami_id" { }        ← Receives "ami-12345"      │     │
│   variable "instance_type" { } ← Receives "t3.medium"      │     │
│                                                            │     │
│   Step 3: "Build according to plan"                        │     │
│                                                            │     │
│   resource "aws_instance" "this" {                         │     │
│     ami           = var.ami_id           # ami-12345       │     │
│     instance_type = var.instance_type    # t3.medium       │     │
│   }                                                        │     │
│                    │                                       │     │
│                    ▼                                       │     │
│   output "public_ip" {                                     │     │
│     value = aws_instance.this.public_ip  ─────────────────►│     │
│   }                                                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Module Composition - Buildings Made of Floor Plans

### The Concept: Nested Blueprints

Just like a hotel is built from room blueprints, bathroom blueprints, and hallway blueprints, a complex Terraform deployment can be built from smaller modules.

```
HOTEL BLUEPRINT (Composite Module)
│
├── Uses: GUEST ROOM floor plan (×200 rooms)
│         └── Bed, desk, TV, mini-fridge
│
├── Uses: BATHROOM floor plan (×200 bathrooms)
│         └── Shower, toilet, sink
│
├── Uses: HALLWAY floor plan (×20 hallways)
│         └── Carpet, lighting, fire exits
│
└── Uses: LOBBY floor plan (×1)
          └── Reception, seating, elevators
```

**In Terraform:**

```hcl
# modules/web-application/main.tf
# This is a COMPOSITE module - it calls other modules!

# Use the networking floor plan
module "networking" {
  source = "../networking"
  
  vpc_cidr    = var.vpc_cidr
  environment = var.environment
}

# Use the compute floor plan
module "compute" {
  source = "../ec2-cluster"
  
  # Connect to networking outputs!
  subnet_ids    = module.networking.private_subnet_ids
  vpc_id        = module.networking.vpc_id
  instance_type = var.instance_type
}

# Use the database floor plan
module "database" {
  source = "../rds"
  
  subnet_ids = module.networking.database_subnet_ids
  vpc_id     = module.networking.vpc_id
}
```

**Visual representation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                         ROOT MODULE                              │
│                                                                  │
│   module "my_app" {                                              │
│     source      = "./modules/web-application"                    │
│     environment = "production"                                   │
│     vpc_cidr    = "10.0.0.0/16"                                  │
│   }                                                              │
│                                                                  │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│               COMPOSITE MODULE: web-application                  │
│                    (Hotel Blueprint)                             │
│                                                                  │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│   │ module      │    │ module      │    │ module      │         │
│   │ "networking"│───►│ "compute"   │    │ "database"  │         │
│   │             │    │             │    │             │         │
│   │ vpc_id ─────┼────┼─► vpc_id    │    │             │         │
│   │ subnet_ids ─┼────┼─► subnet_ids│    │             │         │
│   │             │    │             │    │             │         │
│   └─────────────┘    └─────────────┘    └─────────────┘         │
│         │                                      ▲                 │
│         │                                      │                 │
│         └──────────────────────────────────────┘                 │
│                    (passes subnet_ids)                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   SIMPLE    │     │   SIMPLE    │     │   SIMPLE    │
    │   MODULE:   │     │   MODULE:   │     │   MODULE:   │
    │  networking │     │ ec2-cluster │     │    rds      │
    │             │     │             │     │             │
    │ (VPC, Subnet│     │ (EC2, ASG,  │     │ (RDS, SG,   │
    │  Route, IGW)│     │  SG, ELB)   │     │  Subnet Grp)│
    └─────────────┘     └─────────────┘     └─────────────┘
```

**Analogy:**

```
ARCHITECTURAL HIERARCHY:

Level 1: Simple Floor Plans (Simple Modules)
├── Guest Room Plan      → networking module
├── Bathroom Plan        → ec2-cluster module
└── Hallway Plan         → rds module

Level 2: Building Section Plans (Composite Modules)
├── Hotel Wing Plan      → web-application module
│   └── Uses: rooms + bathrooms + hallways
│
└── Conference Center    → api-backend module
    └── Uses: meeting rooms + lobby + kitchen

Level 3: Complete Campus Plan (Root Module)
└── Resort Complex
    └── Uses: Hotel Wing + Conference Center + Parking
```

---

## Using Same Module Multiple Times

### Method 1: Multiple Module Blocks

Like building multiple apartments using the same floor plan:

```hcl
# Building A - uses 3-bedroom floor plan
module "apartment_101" {
  source    = "./modules/apartment"
  unit      = "101"
  sqft      = 1200
  floor     = 1
}

# Building B - same floor plan, different customization
module "apartment_201" {
  source    = "./modules/apartment"
  unit      = "201"
  sqft      = 1400      # Bigger!
  floor     = 2
}

# Building C - same floor plan, penthouse version
module "apartment_ph1" {
  source    = "./modules/apartment"
  unit      = "PH1"
  sqft      = 2000
  floor     = 10
}
```

### Method 2: for_each (Building an Entire Floor)

Like saying "Build 10 apartments using this floor plan, here are the specs for each":

```hcl
locals {
  apartments = {
    "101" = { sqft = 1200, floor = 1 }
    "102" = { sqft = 1100, floor = 1 }
    "103" = { sqft = 1200, floor = 1 }
    "201" = { sqft = 1400, floor = 2 }
    "202" = { sqft = 1300, floor = 2 }
  }
}

module "apartments" {
  for_each = local.apartments
  source   = "./modules/apartment"
  
  unit  = each.key                  # "101", "102", etc.
  sqft  = each.value.sqft           # 1200, 1100, etc.
  floor = each.value.floor          # 1, 1, 1, 2, 2
}

# Access individual apartment
output "apartment_101_address" {
  value = module.apartments["101"].address
}

# Access all apartments
output "all_addresses" {
  value = { for k, v in module.apartments : k => v.address }
}
```

**Visual:**

```
┌────────────────────────────────────────────────────────────────┐
│  "Build apartments using this floor plan"                      │
│                                                                 │
│  module "apartments" {                                          │
│    for_each = local.apartments                                  │
│    source   = "./modules/apartment"                             │
│  }                                                              │
│                                                                 │
│         │          │          │          │          │           │
│         ▼          ▼          ▼          ▼          ▼           │
│    ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐      │
│    │ Apt    │ │ Apt    │ │ Apt    │ │ Apt    │ │ Apt    │      │
│    │ 101    │ │ 102    │ │ 103    │ │ 201    │ │ 202    │      │
│    │        │ │        │ │        │ │        │ │        │      │
│    │1200sqft│ │1100sqft│ │1200sqft│ │1400sqft│ │1300sqft│      │
│    │Floor 1 │ │Floor 1 │ │Floor 1 │ │Floor 2 │ │Floor 2 │      │
│    └────────┘ └────────┘ └────────┘ └────────┘ └────────┘      │
│                                                                 │
│  All built from the same floor plan!                            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Local vs Remote Modules

### Local Modules - Your Office Filing Cabinet

Floor plans stored in your own office:

```hcl
module "vpc" {
  source = "./modules/vpc"           # Same building
  source = "../shared-modules/vpc"   # Down the hall
}
```

```
Your Office
├── current-project/
│   └── main.tf              ← You are here
│
└── modules/                  ← Local library
    ├── vpc/
    ├── ec2-instance/
    └── rds/
```

### Remote Modules - Public Architecture Library

Floor plans from external sources:

```hcl
# Terraform Registry (like city's public blueprint library)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# GitHub (like another architecture firm's shared drive)
module "vpc" {
  source = "github.com/company/terraform-modules//vpc?ref=v1.0.0"
}

# S3 (like your firm's cloud storage)
module "vpc" {
  source = "s3::https://bucket.s3.amazonaws.com/modules/vpc.zip"
}
```

**Analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE TO FIND FLOOR PLANS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LOCAL (Your Office)                                             │
│  ┌──────────────────────────────────────────────┐                │
│  │  Filing Cabinet: ./modules/                  │                │
│  │  ├── vpc/         (your VPC design)          │                │
│  │  ├── ec2/         (your EC2 design)          │                │
│  │  └── rds/         (your RDS design)          │                │
│  │                                              │                │
│  │  ✓ Full control                              │                │
│  │  ✓ Instant access                            │                │
│  │  ✓ Can modify anytime                        │                │
│  └──────────────────────────────────────────────┘                │
│                                                                  │
│  REMOTE (External Libraries)                                     │
│  ┌──────────────────────────────────────────────┐                │
│  │  City Library: registry.terraform.io         │                │
│  │  ├── terraform-aws-modules/vpc/aws           │                │
│  │  └── (thousands of community floor plans)    │                │
│  │                                              │                │
│  │  ✓ Battle-tested by community                │                │
│  │  ✓ Maintained by experts                     │                │
│  │  ✗ Must download (terraform init)            │                │
│  │  ✗ Can't modify directly                     │                │
│  └──────────────────────────────────────────────┘                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

### Module Fundamentals
1. **Module = Directory with .tf files** - A reusable floor plan
2. **Root module = Your project** - The main building using floor plans
3. **Child module = Reusable component** - A floor plan from the library

### Data Flow
4. **Variables = Inputs** - Customization options on the form
5. **Outputs = Results** - Specifications after building
6. **module.name.output** - How to read specifications

### Variable Rules
7. **No default = Required** - "You MUST specify the lot size"
8. **Has default = Optional** - "Ceiling height defaults to 9ft"
9. **Validation** - "Choose 1-5 bedrooms only"

### Module Sources
10. **Local** - `./modules/vpc` - Your filing cabinet
11. **Registry** - `terraform-aws-modules/vpc/aws` - Public library
12. **Git** - `github.com/org/repo//path` - Another firm's shared drive

### Composition
13. **Simple modules** - Single-purpose floor plans (VPC, EC2, RDS)
14. **Composite modules** - Modules calling modules (hotel = rooms + lobby)
15. **for_each** - "Build 10 apartments using this plan"

### Best Practices
16. **Single responsibility** - One module, one purpose
17. **Clear naming** - `instance_type` not `type`
18. **Document everything** - README.md for each module
19. **Output everything useful** - Users will need various specs
20. **Version constraints** - Pin provider versions in modules

---

*Written on January 14, 2026*  
