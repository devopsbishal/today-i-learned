# Terraform Functions & Expressions - The Architect's Calculator & Toolbox

> Understanding Terraform functions, expressions, and meta-arguments through a real-world analogy of an architect using calculators, drafting tools, and smart automation in their design workflow.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| Built-in functions | Architect's calculator & drafting tools |
| `count` | "Build 5 identical apartments" (numbered units: #1, #2, #3...) |
| `for_each` | "Build apartments for each tenant on this list" (named units: Unit-Alice, Unit-Bob) |
| Index shifting (count issue) | Renumbering all apartments when one tenant moves out |
| Stable keys (for_each benefit) | Tenant names stay the same, no renumbering needed |
| `dynamic` block | Architect's copy machine for repetitive room layouts |
| `for` expression | Spreadsheet formula to transform a column of data |
| Conditional expression `? :` | "If luxury building → marble floors, else → laminate" |
| `merge()` | Combining two specification sheets into one |
| `lookup()` | Looking up material cost in a price catalog |
| `length()` | Counting items on a supply list |
| `contains()` | Checking if a material is in approved vendor list |
| `toset()` | Removing duplicate materials from order list |
| `flatten()` | Unpacking nested boxes of supplies into one pile |
| `try()` | "Try this measurement, if missing use default" |
| `can()` | "Can I read this blueprint?" (yes/no check) |
| `format()` | Label maker: "Building-${name}-Floor-${num}" |
| `join()` / `split()` | Combining/separating address components |

---

## The Big Picture

Imagine you're an **architect designing a residential complex**. You have blueprints (`.tf` files), but you also need tools to make your work efficient:

```
ARCHITECT'S WORKSTATION
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                    YOUR DRAFTING TABLE                       │
│                                                              │
│   📋 Blueprints (main.tf)                                    │
│   │                                                          │
│   ├── 🧮 CALCULATOR (Functions)                              │
│   │   "Calculate square footage, count rooms, format labels" │
│   │                                                          │
│   ├── 📠 COPY MACHINE (count / for_each)                     │
│   │   "Make 10 copies of this apartment design"              │
│   │                                                          │
│   ├── 📄 TEMPLATE DUPLICATOR (dynamic blocks)                │
│   │   "Repeat this room layout for each bedroom needed"      │
│   │                                                          │
│   ├── 📊 SPREADSHEET (for expressions)                       │
│   │   "Transform this list of tenant names to unit labels"   │
│   │                                                          │
│   └── ❓ DECISION GUIDE (conditionals)                       │
│       "If penthouse → rooftop access, else → standard roof"  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Without these tools:**
- You'd manually copy apartment layouts 50 times
- You'd hand-write labels for each unit
- You'd calculate room totals on paper
- You'd have no way to customize designs per client

**With these tools:**
- One template → 50 apartments automatically
- Smart labels: "Unit-${tenant_name}-Floor-${floor_num}"
- Automatic calculations and transformations
- Dynamic customization based on requirements

---

## Core Components

### count vs for_each - Two Ways to Use the Copy Machine

You need to build 4 identical apartment units. You have two approaches:

#### `count` - The Numbered Ticket System

Think of `count` like a **deli counter ticket system**. Each customer gets a number: #1, #2, #3, #4.

```
APARTMENT BUILDING: NUMBERED UNITS
──────────────────────────────────

Day 1: Four tenants move in
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│ #0  │ │ #1  │ │ #2  │ │ #3  │
│Alice│ │ Bob │ │Carol│ │Dave │
└─────┘ └─────┘ └─────┘ └─────┘

Day 30: Bob (#1) moves out...

THE NIGHTMARE BEGINS! 😱

All numbers shift down:
┌─────┐ ┌─────┐ ┌─────┐
│ #0  │ │ #1  │ │ #2  │
│Alice│ │Carol│ │Dave │  ← Carol & Dave had to MOVE apartments!
└─────┘ └─────┘ └─────┘

❌ Carol was in #2, now she's in #1 (new lease, new address!)
❌ Dave was in #3, now he's in #2 (new lease, new address!)
❌ Their mail goes to wrong units, utilities disconnected!
```

**Terraform equivalent:**
```hcl
# Creates: aws_instance.server[0], [1], [2], [3]
resource "aws_instance" "server" {
  count         = 4
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2023
  instance_type = "t3.micro"
  tags = {
    Name = "Server-${count.index}"  # Server-0, Server-1, Server-2, Server-3
  }
}

# If you remove server[1], terraform destroys [1] and RECREATES [2] and [3]
# because their indices shift!
```

#### `for_each` - The Named Mailbox System

Think of `for_each` like **named mailboxes**. Each tenant has their name permanently on the box.

```
APARTMENT BUILDING: NAMED UNITS
───────────────────────────────

Day 1: Four tenants move in
┌───────┐ ┌─────┐ ┌───────┐ ┌──────┐
│ Alice │ │ Bob │ │ Carol │ │ Dave │
│  🏠   │ │ 🏠  │ │  🏠   │ │  🏠  │
└───────┘ └─────┘ └───────┘ └──────┘

Day 30: Bob moves out...

SIMPLE REMOVAL! ✅

┌───────┐ ┌───────┐ ┌──────┐
│ Alice │ │ Carol │ │ Dave │
│  🏠   │ │  🏠   │ │  🏠  │
└───────┘ └───────┘ └──────┘

✅ Only Bob's unit was removed
✅ Alice, Carol, Dave: unchanged!
✅ Same addresses, same leases, same everything!
```

**Terraform equivalent:**
```hcl
# Creates: aws_instance.server["alice"], ["bob"], ["carol"], ["dave"]
resource "aws_instance" "server" {
  for_each      = toset(["alice", "bob", "carol", "dave"])
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2023
  instance_type = "t3.micro"
  tags = {
    Name = "Server-${each.key}"  # Server-alice, Server-bob, etc.
  }
}

# If you remove "bob" from the list, ONLY server["bob"] is destroyed
# Alice, Carol, Dave remain untouched!
```

#### When to Use Which?

```
DECISION CHART
──────────────

┌─────────────────────────────────────────────────────────────┐
│ "Will I ever add or remove items from this list?"           │
│                                                             │
│     YES ─────────────────────────┐                          │
│                                  ▼                          │
│                         Use for_each ✅                     │
│                         (stable named keys)                 │
│                                                             │
│     NO, always same count ───────┐                          │
│                                  ▼                          │
│                         count is fine ⚪                    │
│                         (but for_each still safer)          │
└─────────────────────────────────────────────────────────────┘
```

---

### dynamic Blocks - The Room Template Duplicator

You're designing a floor with multiple bedrooms. Without automation:

```
FLOOR PLAN: THE HARD WAY
────────────────────────

bedroom {
  size = "12x12"
  window = true
}

bedroom {
  size = "12x12"
  window = true
}

bedroom {
  size = "12x12"
  window = true
}

# Copy-pasting 50 times for a hotel floor... 😫
```

**The `dynamic` block is like a template duplicator:**

```
FLOOR PLAN: THE SMART WAY
─────────────────────────

TEMPLATE DUPLICATOR SETTINGS:
┌─────────────────────────────────────────┐
│ Template: "bedroom"                     │
│ Repeat for: [Master, Guest1, Guest2]    │
│                                         │
│ For each copy:                          │
│   - Read name from list                 │
│   - Apply to template                   │
└─────────────────────────────────────────┘

OUTPUT:
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   bedroom    │ │   bedroom    │ │   bedroom    │
│   "Master"   │ │   "Guest1"   │ │   "Guest2"   │
└──────────────┘ └──────────────┘ └──────────────┘
```

**Terraform equivalent (Security Group Rules):**
```hcl
variable "allowed_ports" {
  default = [80, 443, 8080, 3000]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  # The template duplicator
  dynamic "ingress" {
    for_each = var.allowed_ports    # List to iterate
    
    content {                        # Template to duplicate
      from_port   = ingress.value    # Current port number
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

# Creates 4 ingress rules automatically!
```

**Key parts of dynamic blocks:**
```
dynamic "BLOCK_NAME" {       ← What nested block to generate
  for_each = COLLECTION      ← What to iterate over
  iterator = custom_name     ← (Optional) rename the iterator
  
  content {                  ← The template for each block
    field = BLOCK_NAME.value ← Access current item
    field = BLOCK_NAME.key   ← Access current key/index
  }
}
```

---

### for Expression - The Spreadsheet Formula

Imagine you have a spreadsheet of tenant data and need to transform it:

```
SPREADSHEET TRANSFORMATION
──────────────────────────

Original Column A:          Formula:                 Result Column B:
┌─────────────┐            ┌──────────────┐         ┌─────────────────┐
│ alice       │            │              │         │ ALICE           │
│ bob         │  ───────►  │ =UPPER(A)    │ ──────► │ BOB             │
│ carol       │            │              │         │ CAROL           │
└─────────────┘            └──────────────┘         └─────────────────┘
```

**Terraform `for` expression:**
```hcl
variable "names" {
  default = ["alice", "bob", "carol"]
}

# List output (use square brackets [])
locals {
  upper_names = [for name in var.names : upper(name)]
  # Result: ["ALICE", "BOB", "CAROL"]
}

# Map output (use curly braces {})
locals {
  name_map = {for name in var.names : name => upper(name)}
  # Result: {"alice" = "ALICE", "bob" = "BOB", "carol" = "CAROL"}
}
```

**Practical example - Transform list to resource-ready map:**
```hcl
variable "instances" {
  default = [
    { name = "web", type = "t3.small" },
    { name = "api", type = "t3.micro" },
    { name = "db",  type = "t3.large" }
  ]
}

locals {
  # Transform list to map (keyed by name)
  instance_map = {for inst in var.instances : inst.name => inst}
  # Result: {
  #   "web" = { name = "web", type = "t3.small" }
  #   "api" = { name = "api", type = "t3.micro" }
  #   "db"  = { name = "db",  type = "t3.large" }
  # }
}

# Now use with for_each (requires map or set!)
resource "aws_instance" "server" {
  for_each      = local.instance_map
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2023
  instance_type = each.value.type

  tags = {
    Name = each.key
  }
}
```

**Key difference: `for` vs `for_each`**
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   for EXPRESSION                    for_each META-ARGUMENT      │
│   ──────────────                    ────────────────────        │
│                                                                 │
│   📊 Transform data                 📋 Create resources         │
│   Used in: locals, outputs          Used on: resources, modules │
│   Returns: new list or map          Creates: multiple instances │
│                                                                 │
│   [for x in list : upper(x)]        for_each = toset(list)      │
│         ↓                                   ↓                   │
│   ["ALICE", "BOB"]                  resource["alice"]           │
│                                     resource["bob"]             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Conditional Expressions - The Decision Guide

When designing buildings, you have decision points based on requirements:

```
ARCHITECT'S DECISION FLOWCHART
──────────────────────────────

Question: Is this a luxury building?

        ┌─────────────────────┐
        │ Luxury Building?    │
        └─────────┬───────────┘
                  │
         YES      │       NO
          ↓       │        ↓
    ┌─────────┐   │   ┌─────────┐
    │ Marble  │   │   │Laminate │
    │ Floors  │   │   │ Floors  │
    └─────────┘   │   └─────────┘
```

**Terraform conditional syntax:**
```hcl
condition ? value_if_true : value_if_false
```

**Practical examples:**
```hcl
# 1. Choose instance size based on environment
instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

# 2. Conditionally create a resource (0 = don't create)
resource "aws_instance" "bastion" {
  count = var.enable_bastion ? 1 : 0
  # ...
}

# 3. Set value or use default
name = var.custom_name != "" ? var.custom_name : "default-server"

# 4. Multiple instances in prod, single in dev
resource "aws_instance" "web" {
  count = var.environment == "prod" ? 3 : 1
  # ...
}
```

**Important rule:** Both sides must return the **same type**!
```hcl
# ❌ WRONG - string vs number
value = var.use_number ? 42 : "forty-two"

# ✅ CORRECT - both strings
value = var.use_long ? "forty-two" : "42"
```

---

### Built-in Functions - The Architect's Calculator

Your calculator has specialized buttons for different calculations:

```
ARCHITECT'S SCIENTIFIC CALCULATOR
─────────────────────────────────

┌─────────────────────────────────────────────────────┐
│                                                     │
│   STRING FUNCTIONS              COLLECTION FUNCS    │
│   ─────────────────            ────────────────     │
│   [format] [join]              [merge] [lookup]     │
│   [split]  [upper]             [length] [contains]  │
│   [replace][trim]              [flatten] [distinct] │
│                                                     │
│   TYPE CONVERSION               ERROR HANDLING      │
│   ───────────────              ──────────────       │
│   [toset]  [tolist]            [try]   [can]        │
│   [tomap]  [tonumber]          [coalesce]           │
│                                                     │
│   FILE FUNCTIONS                                    │
│   ──────────────                                    │
│   [file]  [templatefile]                            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### merge() - Combining Specification Sheets

```
SPECIFICATION MERGE
───────────────────

Sheet 1: Default Tags           Sheet 2: Custom Tags
┌─────────────────────┐        ┌─────────────────────┐
│ ManagedBy: terraform│        │ Name: my-server     │
│ Environment: dev    │   +    │ Environment: prod   │ ← Overrides!
└─────────────────────┘        └─────────────────────┘
                    ↓
           MERGED RESULT
┌─────────────────────────────┐
│ ManagedBy: terraform        │
│ Environment: prod           │ ← Later value wins!
│ Name: my-server             │
└─────────────────────────────┘
```

```hcl
variable "default_tags" {
  default = {
    ManagedBy   = "terraform"
    Environment = "dev"
  }
}

resource "aws_instance" "server" {
  instance_type = "t3.micro"
  
  tags = merge(var.default_tags, {
    Name        = "my-server"
    Environment = "prod"  # Overrides the default
  })
}
```

#### lookup() - The Price Catalog

```
MATERIAL PRICE CATALOG
──────────────────────

┌──────────────┬─────────┐
│ Material     │ $/sqft  │
├──────────────┼─────────┤
│ marble       │ $50     │
│ hardwood     │ $25     │
│ laminate     │ $10     │
└──────────────┴─────────┘

Query: "What's the price of granite?"
Answer: Not found → Use default: $15

lookup(catalog, "granite", $15) → $15
lookup(catalog, "marble", $15)  → $50
```

```hcl
variable "instance_sizes" {
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }
}

resource "aws_instance" "server" {
  # If environment not in map, default to t3.micro
  instance_type = lookup(var.instance_sizes, var.environment, "t3.micro")
}
```

#### try() and can() - Error Handling

```
BLUEPRINT READING SAFETY
────────────────────────

try(): "Try to read this measurement. If page is missing, use default."
can(): "Can I read this blueprint page?" (yes/no answer)

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   try(blueprint.page5.room_size, "10x10")                       │
│                                                                 │
│   Page 5 exists?                                                │
│     YES → Return room_size                                      │
│     NO  → Return "10x10" (default)                              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   can(blueprint.page5.room_size)                                │
│                                                                 │
│   Page 5 exists?                                                │
│     YES → Return true                                           │
│     NO  → Return false                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```hcl
# try() - Returns value or fallback
locals {
  port = try(var.config.server.port, 8080)
  # If var.config.server.port errors, return 8080
}

# can() - Returns boolean
variable "port" {
  validation {
    condition     = can(tonumber(var.port))
    error_message = "Port must be a valid number."
  }
}
```

---

## How Things Connect

```
THE COMPLETE WORKFLOW
─────────────────────

┌──────────────────────────────────────────────────────────────────────┐
│                         YOUR TERRAFORM FILE                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. INPUT DATA                                                       │
│     variable "servers" {                                             │
│       default = ["web", "api", "db"]                                 │
│     }                                                                │
│                              │                                       │
│                              ▼                                       │
│  2. TRANSFORM (for expression + functions)                           │
│     locals {                                                         │
│       server_map = {for s in var.servers : s => {                    │
│         name = format("server-%s", s)                                │
│         type = lookup(var.sizes, s, "t3.micro")                      │
│       }}                                                             │
│     }                                                                │
│                              │                                       │
│                              ▼                                       │
│  3. CREATE RESOURCES (for_each)                                      │
│     resource "aws_instance" "server" {                               │
│       for_each      = local.server_map                               │
│       instance_type = each.value.type                                │
│       tags = merge(var.default_tags, {                               │
│         Name = each.value.name                                       │
│       })                                                             │
│     }                                                                │
│                              │                                       │
│                              ▼                                       │
│  4. CONDITIONAL LOGIC                                                │
│     resource "aws_eip" "server" {                                    │
│       for_each = var.environment == "prod" ? local.server_map : {}   │
│       instance = aws_instance.server[each.key].id                    │
│     }                                                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **`count` = numbered list, `for_each` = named map**
   - Use `for_each` to avoid destructive index shifting
   - Convert lists to sets with `toset()` for `for_each`

2. **`dynamic` blocks = generate repeated nested blocks**
   - Use for variable-length configurations (security rules, tags, etc.)
   - Don't overuse - reduces readability

3. **`for` expressions = data transformation**
   - `[]` brackets → output a list
   - `{}` braces → output a map
   - Often used to prepare data for `for_each`

4. **Conditionals = ternary decisions**
   - `count = condition ? 1 : 0` to conditionally create resources
   - Both sides must return the same type

5. **Functions = your toolbox**
   - `merge()` for combining maps (later wins on conflict)
   - `lookup()` for map access with defaults
   - `try()`/`can()` for error handling

---

*Written on January 19, 2026*
