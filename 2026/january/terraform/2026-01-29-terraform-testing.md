# Terraform Testing - The Model Building Approach

> Understanding Terraform testing through a real-world analogy of architects building scale models and running simulations before actual construction begins, ensuring blueprints work correctly without the cost of real building failures.

---

## TL;DR

| Terraform Concept | Real-World Analogy |
|-------------------|-------------------|
| `terraform validate` | Spell-checking the blueprint (syntax, no real checks) |
| `terraform plan` | Full blueprint review with city records (needs permits office) |
| `terraform test` | Building scale models in the workshop (native, TF 1.6+) |
| Terratest | Full-scale test construction (build real, then demolish) |
| Mocking | Paper models instead of real materials |
| Precondition | "Check materials BEFORE construction starts" |
| Postcondition | "Verify building AFTER construction completes" |
| `check` block | Ongoing building inspections (warnings, not failures) |
| Test file (`.tftest.hcl`) | Test blueprint specifications |
| `run` block | Individual test case (one model to build) |
| `assert` | Quality checkpoint ("roof must be red") |
| `defer` in Terratest | "Always demolish test building, even if test fails" |
| Unit test | Test one room design in isolation |
| Integration test | Test how rooms connect together |
| E2E test | Build entire building, invite test occupants |
| Test variables | Custom specs for test models |

---

## The Big Picture

Imagine you're an **architect** who needs to verify building designs work correctly before spending millions on real construction. You have several options:

```
THE TESTING PYRAMID - BLUEPRINT VERIFICATION
───────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                          ╱╲                                 │
│                         ╱  ╲                                │
│                        ╱ E2E ╲      TERRATEST               │
│                       ╱ Tests ╲     Build real building,    │
│                      ╱─────────╲    test with real people,  │
│                     ╱Integration╲   then demolish           │
│                    ╱   Tests     ╲  (Slowest, most real)    │
│                   ╱───────────────╲                         │
│                  ╱   Unit Tests    ╲  TERRAFORM TEST        │
│                 ╱  (with mocking)   ╲ Scale models,         │
│                ╱─────────────────────╲ paper mockups        │
│               ╱   Syntax Validation   ╲ (Fast, simulated)   │
│              ╱  (validate, fmt, plan)  ╲                    │
│             ╱───────────────────────────╲ VALIDATE/PLAN     │
│            ╱      Spell check blueprints ╲ (Fastest)        │
│           ╱──────────────────────────────╲                  │
│                                                             │
│   SPEED:    ⚡ Fast ──────────────────────────► 🐢 Slow     │
│   REALISM:  📄 Paper ─────────────────────────► 🏗️ Real    │
│   COST:     💰 Cheap ─────────────────────────► 💰💰💰      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Without testing:**
- Blueprints have typos nobody catches
- Buildings collapse after construction (production failures)
- Expensive fixes after the fact
- No confidence in design changes

**With testing:**
- Typos caught immediately (validate)
- Scale models reveal design flaws (terraform test)
- Full test construction proves everything works (Terratest)
- Confident refactoring and updates

---

## Core Components

### Level 1: Syntax Validation - Spell-Checking Blueprints

The fastest checks that catch obvious mistakes without any external connections.

```
TERRAFORM VALIDATE vs PLAN
──────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   terraform validate                                        │
│   ──────────────────                                        │
│                                                             │
│   "Spell-check the blueprints"                              │
│                                                             │
│   ✅ Checks:                                                │
│   ├── Syntax errors (missing brackets, typos)               │
│   ├── Type mismatches (string vs number)                    │
│   ├── Invalid references (unknown variables)                │
│   ├── Module structure issues                               │
│   └── Attribute validation                                  │
│                                                             │
│   ❌ Does NOT:                                               │
│   ├── Connect to AWS/Azure/GCP                              │
│   ├── Require credentials                                   │
│   ├── Check if resources exist                              │
│   └── Validate actual values (bad AMI ID)                   │
│                                                             │
│   ⚡ Speed: Instant (seconds)                               │
│   🔑 Credentials: Not required                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   terraform plan                                            │
│   ──────────────                                            │
│                                                             │
│   "Full blueprint review with city records"                 │
│                                                             │
│   ✅ Checks everything validate does, PLUS:                 │
│   ├── Connects to cloud provider APIs                       │
│   ├── Reads current state                                   │
│   ├── Validates resource values (AMI exists?)               │
│   ├── Checks permissions (can you create this?)             │
│   ├── Detects drift (manual changes)                        │
│   └── Calculates exact changes needed                       │
│                                                             │
│   🐢 Speed: Slower (depends on resources)                   │
│   🔑 Credentials: Required                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Usage:**
```bash
# Quick syntax check (CI pre-commit)
terraform validate

# Full validation with change preview
terraform plan

# Save plan for later apply
terraform plan -out=tfplan

# Plan as JSON for automated validation
terraform plan -out=tfplan && terraform show -json tfplan > plan.json
```

**When to use each:**
```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   SCENARIO                        USE                       │
│   ────────                        ───                       │
│                                                             │
│   Pre-commit hook                 validate (fast, no creds) │
│   CI pipeline (quick check)       validate                  │
│   Before applying changes         plan                      │
│   PR review                       plan (show changes)       │
│   Detecting drift                 plan                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Level 2: Native Testing - Scale Models in the Workshop

Terraform 1.6+ introduced native testing with `.tftest.hcl` files. Think of it as building scale models in your workshop.

```
TERRAFORM TEST FRAMEWORK
────────────────────────

FILE STRUCTURE:
─────────────────────────────────────────────────────────────────

  my-module/
  ├── main.tf              # The blueprint
  ├── variables.tf         # Customizable parameters
  ├── outputs.tf           # What we expose
  │
  └── tests/               # Test workshop
      ├── basic.tftest.hcl      # Basic model tests
      ├── validation.tftest.hcl # Input validation tests
      └── complete.tftest.hcl   # Full integration tests


TEST FILE ANATOMY:
─────────────────────────────────────────────────────────────────

  # tests/basic.tftest.hcl
  
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │   # Optional: Override variables for testing            │
  │   variables {                                           │
  │     environment = "test"                                │
  │     instance_type = "t3.micro"                          │
  │   }                                                     │
  │                                                         │
  │   # Each "run" block is one test case                   │
  │   run "verify_instance_type" {                          │
  │     command = plan    # Just plan, don't apply          │
  │                                                         │
  │     assert {                                            │
  │       condition     = aws_instance.web.instance_type == "t3.micro"
  │       error_message = "Instance type must be t3.micro"  │
  │     }                                                   │
  │   }                                                     │
  │                                                         │
  │   run "verify_tags" {                                   │
  │     command = plan                                      │
  │                                                         │
  │     assert {                                            │
  │       condition     = aws_instance.web.tags["Environment"] == "test"
  │       error_message = "Environment tag not set"         │
  │     }                                                   │
  │   }                                                     │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

**Running tests:**
```bash
# Run all tests
terraform test

# Run specific test file
terraform test -filter=tests/basic.tftest.hcl

# Verbose output
terraform test -verbose
```

**Test with real apply (integration test):**
```hcl
# tests/integration.tftest.hcl

run "deploy_and_verify" {
  command = apply  # Actually create resources!

  assert {
    condition     = aws_instance.web.public_ip != null
    error_message = "Instance must have public IP"
  }
}

# Resources are automatically destroyed after test
```

---

### Mocking - Paper Models Instead of Real Materials

Mocking lets you test without creating real cloud resources. Like using paper models instead of real construction materials.

```
MOCKING IN TERRAFORM TEST
─────────────────────────

WITHOUT MOCKING:
─────────────────────────────────────────────────────────────────

  terraform test
      │
      ├── terraform init
      ├── terraform apply  ← Creates REAL AWS resources!
      │       │
      │       ├── EC2 instance created ($$$)
      │       ├── RDS database created ($$$)
      │       └── Takes 5-10 minutes
      │
      ├── Run assertions
      └── terraform destroy
  
  ⏱️ Time: 5-10 minutes
  💰 Cost: Real money
  🌐 Requires: AWS credentials, network


WITH MOCKING:
─────────────────────────────────────────────────────────────────

  terraform test
      │
      ├── terraform init
      ├── Mock provider intercepts
      │       │
      │       ├── Fake EC2 data generated locally
      │       ├── Fake RDS data generated locally
      │       └── No API calls!
      │
      └── Run assertions
  
  ⏱️ Time: 2-5 seconds
  💰 Cost: Free
  🌐 Requires: Nothing (runs offline!)
```

**Mocking syntax:**
```hcl
# tests/unit.tftest.hcl

# Mock the entire AWS provider
mock_provider "aws" {}

run "test_s3_bucket_naming" {
  command = plan

  assert {
    condition     = aws_s3_bucket.main.bucket == "my-app-${var.environment}"
    error_message = "Bucket naming convention incorrect"
  }
}
```

**Override mock values:**
```hcl
# tests/unit.tftest.hcl

mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id         = "i-mock12345"
      arn        = "arn:aws:ec2:us-east-1:123456789:instance/i-mock12345"
      public_ip  = "1.2.3.4"
      private_ip = "10.0.1.100"
    }
  }

  mock_resource "aws_s3_bucket" {
    defaults = {
      arn                = "arn:aws:s3:::test-bucket"
      bucket_domain_name = "test-bucket.s3.amazonaws.com"
    }
  }
}

run "verify_instance_outputs" {
  command = plan

  assert {
    condition     = output.instance_ip == "1.2.3.4"
    error_message = "Output should use mocked IP"
  }
}
```

**What CAN vs CANNOT be mocked:**
```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ✅ CAN MOCK:                                              │
│   ├── Provider resources (aws_instance, aws_s3_bucket)      │
│   ├── Data sources (aws_ami, aws_vpc)                       │
│   ├── Resource attributes (id, arn, ip addresses)           │
│   └── Computed values                                       │
│                                                             │
│   ❌ CANNOT MOCK (need real tests for):                     │
│   ├── Actual behavior (does EC2 boot correctly?)            │
│   ├── Network connectivity (can I reach the server?)        │
│   ├── IAM permissions (does the role actually work?)        │
│   ├── Provider-specific validation (valid AMI ID?)          │
│   └── Cross-service interactions                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Level 3: Preconditions & Postconditions - Construction Checkpoints

Built-in validation that runs during `plan` and `apply`.

```
PRECONDITIONS vs POSTCONDITIONS
───────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   TIMELINE OF TERRAFORM APPLY                               │
│                                                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐         │
│   │ BEFORE   │ ───► │ DURING   │ ───► │ AFTER    │         │
│   │ (Plan)   │      │ (Apply)  │      │ (Apply)  │         │
│   └──────────┘      └──────────┘      └──────────┘         │
│        │                                    │               │
│        ▼                                    ▼               │
│   PRECONDITION                        POSTCONDITION         │
│   "Check materials                    "Verify building      │
│    BEFORE building"                    AFTER completion"    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Precondition - Validate BEFORE apply:**
```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  lifecycle {
    # Check BEFORE attempting to create
    precondition {
      condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
      error_message = "Instance type must be t3.micro, t3.small, or t3.medium for this environment."
    }

    precondition {
      condition     = can(regex("^ami-", var.ami_id))
      error_message = "AMI ID must start with 'ami-'."
    }
  }
}
```

**Postcondition - Validate AFTER apply:**
```hcl
resource "aws_instance" "web" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.public_subnet_id
  associate_public_ip_address = true

  lifecycle {
    # Check AFTER resource is created
    postcondition {
      condition     = self.public_ip != null && self.public_ip != ""
      error_message = "Instance must have a public IP address assigned."
    }

    postcondition {
      condition     = self.instance_state == "running"
      error_message = "Instance must be in running state."
    }
  }
}
```

**The `self` keyword:**
```hcl
# In postconditions, use 'self' to reference the resource's own attributes

postcondition {
  condition     = self.arn != null    # ← 'self' = this resource
  error_message = "Resource must have an ARN"
}
```

**Check blocks - Warnings without failures:**
```hcl
# check blocks run AFTER apply but don't fail the apply
# Good for things you want to monitor but not block on

check "website_health" {
  data "http" "health" {
    url = "https://${aws_instance.web.public_ip}/health"
  }

  assert {
    condition     = data.http.health.status_code == 200
    error_message = "Website health check failed (warning only)"
  }
}
```

---

### Level 4: Terratest - Full-Scale Test Construction

For maximum confidence, Terratest builds REAL infrastructure, runs tests, then destroys everything.

```
TERRATEST LIFECYCLE
───────────────────

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   1. SETUP                                                  │
│      │                                                      │
│      └── Configure terraform options                        │
│          (directory, variables, backend)                    │
│                                                             │
│   2. DEFER CLEANUP ⭐ CRITICAL!                             │
│      │                                                      │
│      └── defer terraform.Destroy(t, opts)                   │
│          "No matter what happens, tear it all down"         │
│                                                             │
│   3. APPLY                                                  │
│      │                                                      │
│      └── terraform.InitAndApply(t, opts)                    │
│          Creates REAL AWS resources                         │
│                                                             │
│   4. VERIFY                                                 │
│      │                                                      │
│      ├── terraform.Output() - check outputs                 │
│      ├── http_helper.HttpGet() - test endpoints             │
│      ├── ssh.Command() - run commands on instances          │
│      └── aws.GetInstance() - verify AWS state               │
│                                                             │
│   5. CLEANUP (from defer)                                   │
│      │                                                      │
│      └── terraform.Destroy()                                │
│          Destroys ALL resources                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Basic Terratest example:**
```go
// test/terraform_aws_test.go

package test

import (
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/http-helper"
)

func TestTerraformAwsWebServer(t *testing.T) {
    t.Parallel()

    // Configure Terraform options
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/web-server",
        Vars: map[string]interface{}{
            "environment":   "test",
            "instance_type": "t3.micro",
        },
    }

    // ⭐ ALWAYS defer destroy - runs even if test fails!
    defer terraform.Destroy(t, terraformOptions)

    // Apply Terraform (creates real resources)
    terraform.InitAndApply(t, terraformOptions)

    // Get output
    publicIP := terraform.Output(t, terraformOptions, "public_ip")

    // Verify the web server is responding
    url := "http://" + publicIP + ":8080"
    expectedBody := "Hello, World!"

    // Retry for eventual consistency
    http_helper.HttpGetWithRetry(
        t,
        url,
        nil,               // TLS config
        200,               // Expected status
        expectedBody,      // Expected body
        30,                // Max retries
        5*time.Second,     // Time between retries
    )
}
```

**Why `defer` is CRITICAL:**
```
WITHOUT DEFER:
─────────────────────────────────────────────────────────────────

  Apply ✅ 
      │
      ▼
  Verify ❌ FAILS! (test assertion fails)
      │
      ▼
  Test exits immediately
      │
      ▼
  Destroy NEVER RUNS! 💀
      │
      ▼
  Resources keep running → 💸 $$$$ BILL


WITH DEFER:
─────────────────────────────────────────────────────────────────

  defer terraform.Destroy() registered first
      │
      ▼
  Apply ✅
      │
      ▼
  Verify ❌ FAILS!
      │
      ▼
  Go runtime triggers deferred functions
      │
      ▼
  Destroy RUNS! ✅
      │
      ▼
  Resources cleaned up → No orphaned infrastructure
```

**Retry patterns for eventual consistency:**
```go
// AWS resources take time to be fully available

// HTTP retry
http_helper.HttpGetWithRetry(t, url, nil, 200, "expected", 30, 5*time.Second)

// Custom retry
retry.DoWithRetry(t, "Waiting for instance", 30, 10*time.Second, func() (string, error) {
    instance := aws.GetInstance(t, region, instanceID)
    if instance.State != "running" {
        return "", fmt.Errorf("Instance not running yet")
    }
    return "Instance is running", nil
})
```

---

## How Things Connect

```
THE COMPLETE TESTING STRATEGY
─────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   DEVELOPER WORKFLOW                                        │
│   │                                                         │
│   │  1. Write Terraform code                                │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  2. terraform fmt (auto-format)                         │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  3. terraform validate (syntax check)                   │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  4. terraform test (unit tests with mocks)              │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  5. git commit                                          │
│   │                                                         │
│   ▼                                                         │
│   CI/CD PIPELINE                                            │
│   │                                                         │
│   │  6. terraform validate                                  │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  7. terraform test (all tests)                          │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  8. tfsec / checkov (security scan)                     │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  9. terraform plan (preview changes)                    │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  10. Terratest (E2E in test account) [optional]         │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  11. Manual approval (for prod)                         │
│   │     │                                                   │
│   │     ▼                                                   │
│   │  12. terraform apply                                    │
│   │                                                         │
│   ▼                                                         │
│   PRODUCTION ✅                                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘


CHOOSING THE RIGHT TEST LEVEL
─────────────────────────────

┌────────────────┬──────────────────────────────────────────┐
│ WHAT TO TEST   │ WHICH TOOL                               │
├────────────────┼──────────────────────────────────────────┤
│ Syntax errors  │ terraform validate                       │
│ Type errors    │ terraform validate                       │
│ Module logic   │ terraform test (with mocks)              │
│ Input validation│ terraform test + variable validation    │
│ Output values  │ terraform test                           │
│ Security       │ tfsec, Checkov                           │
│ Real creation  │ terraform test (command = apply)         │
│ Full E2E       │ Terratest                                │
│ HTTP endpoints │ Terratest (http_helper)                  │
│ SSH access     │ Terratest (ssh module)                   │
│ Complex flows  │ Terratest (full Go flexibility)          │
└────────────────┴──────────────────────────────────────────┘
```

---

## Practical Examples

### Example 1: Variable Validation Test
```hcl
# tests/validation.tftest.hcl

variables {
  environment = "invalid"  # Should fail validation
}

run "reject_invalid_environment" {
  command = plan

  # Expect this to fail
  expect_failures = [
    var.environment
  ]
}

run "accept_valid_environment" {
  variables {
    environment = "prod"  # Override with valid value
  }

  command = plan

  # Should pass - no expect_failures
}
```

### Example 2: Module Output Test
```hcl
# tests/outputs.tftest.hcl

mock_provider "aws" {}

run "verify_bucket_arn_format" {
  command = plan

  assert {
    condition     = can(regex("^arn:aws:s3:::", output.bucket_arn))
    error_message = "Bucket ARN must be in correct format"
  }
}

run "verify_all_required_outputs" {
  command = plan

  assert {
    condition     = output.bucket_name != null
    error_message = "bucket_name output is required"
  }

  assert {
    condition     = output.bucket_arn != null
    error_message = "bucket_arn output is required"
  }
}
```

### Example 3: Integration Test with Real Apply
```hcl
# tests/integration.tftest.hcl

# Use real provider - will create actual resources
run "create_and_verify_s3_bucket" {
  command = apply

  assert {
    condition     = aws_s3_bucket.main.bucket_regional_domain_name != null
    error_message = "Bucket must have a regional domain name"
  }

  assert {
    condition     = aws_s3_bucket_versioning.main.versioning_configuration[0].status == "Enabled"
    error_message = "Bucket versioning must be enabled"
  }
}

# Resources automatically destroyed after test
```

---

## Key Takeaways

1. **Testing pyramid applies to Terraform**
   - Base: `terraform validate` (fast, syntax)
   - Middle: `terraform test` with mocks (medium, logic)
   - Top: Terratest (slow, real infrastructure)

2. **Use mocking for fast feedback**
   - `mock_provider` eliminates API calls
   - Tests run in seconds, not minutes
   - No credentials needed for unit tests

3. **Always use `defer` in Terratest**
   - Guarantees cleanup even on test failure
   - Prevents orphaned resources and runaway costs
   - First line after `terraformOptions`

4. **Preconditions vs Postconditions**
   - Precondition: Validate inputs BEFORE apply
   - Postcondition: Verify results AFTER apply
   - Use `self` to reference resource attributes

5. **Condition must be TRUE to pass**
   - Common mistake: inverted logic
   - `condition = var.env == "prod"` passes when env IS prod
   - Test your conditions!

6. **`terraform test` requires TF 1.6+**
   - Native HCL-based testing
   - Mocking support
   - Replaces need for Terratest in many cases

---

*Written on January 29, 2026*
