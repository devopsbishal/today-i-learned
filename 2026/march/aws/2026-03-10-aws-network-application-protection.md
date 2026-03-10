# AWS Network & Application Protection Services -- The Medieval Castle Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), and installed the locksmith shop (KMS). But all of those systems operate **inside** your walls -- they control who enters, detect anomalies after they happen, ensure configuration compliance, and protect data at rest. None of them answer the question that comes before all of that: **what happens when hostile traffic arrives at your front door?** When a botnet sends 10 million requests per second to your application. When a SQL injection payload hides inside a legitimate-looking HTTP request. When a compromised internal workload tries to exfiltrate data to a command-and-control server. That is the domain of AWS's network and application protection services -- WAF, Shield, Network Firewall, and Firewall Manager -- and together they form the layered defensive perimeter of your cloud environment.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| AWS WAF | The gate inspector who reads every visitor's paperwork before letting them through | Inspects HTTP/HTTPS requests at Layer 7, matching against rules for SQL injection, XSS, bad IPs, excessive request rates, and bot signatures -- attached to CloudFront, ALB, API Gateway, and other edge/entry resources |
| Web ACL | The inspection checklist posted at the castle gate | A container that holds an ordered list of rules and a default action (Allow or Block); each protected resource is associated with exactly one Web ACL |
| WAF rule groups | A laminated card of pre-written inspection criteria the guard carries | A reusable, named collection of rules that can be included in multiple Web ACLs -- either AWS Managed Rule Groups (maintained by AWS) or your own custom rule groups |
| WCU (Web ACL Capacity Units) | The weight limit on the guard's inspection toolkit belt -- simple tools (IP checks) weigh almost nothing, but heavy tools (regex scanners, XSS detectors) fill up the belt fast | A capacity budget that caps rule complexity per Web ACL (1500 WCU default); complex match conditions like regex or body inspection cost more WCU than simple IP checks |
| WAF Bot Control | A specialist who checks whether the visitor is a real person or a disguised robot | A managed rule group that classifies traffic as verified bots (Googlebot), common bots (curl, scrapers), or automated sessions, and lets you apply different actions per category |
| AWS Shield Standard | The castle's outer wall -- always there, no extra charge | Automatic Layer 3/4 DDoS protection applied to every AWS resource by default, defending against SYN floods, UDP reflection, and other volumetric/protocol attacks |
| AWS Shield Advanced | Hiring a garrison of elite knights with a war chest to cover siege damages | Paid Layer 3/4/7 DDoS protection with the DDoS Response Team (DRT), proactive engagement, health-based detection, automatic L7 mitigation via WAF, and cost protection credits for scaling charges during an attack |
| DDoS Response Team (DRT) | A 24/7 rapid-response military unit that deploys to your castle during a siege | AWS security engineers who can write WAF rules on your behalf during an active DDoS attack, analyze attack patterns, and tune your defenses -- requires Business or Enterprise Support |
| AWS Network Firewall | A fortified checkpoint inside the castle walls that inspects every cart and messenger passing between districts | A managed, stateful network firewall service deployed inside your VPC that inspects all traffic (Layer 3-7) using a stateless 5-tuple engine and a Suricata-compatible stateful IPS/IDS engine |
| Firewall endpoint | A guard tower in a dedicated subnet where every packet must pass through | A Network Firewall-managed ENI deployed in a firewall subnet that traffic is routed through via VPC route tables; you deploy one per AZ |
| Stateless engine | The outer gate guard who checks the basics at a glance | 5-tuple inspection (source/dest IP, source/dest port, protocol) with fixed capacity units; fast but shallow -- best used to forward traffic to the stateful engine |
| Stateful engine | The inner chamber inspector who opens the crate and reads the manifests | Suricata-compatible deep packet inspection supporting IPS/IDS signatures, domain filtering, TLS SNI inspection, and protocol-aware analysis |
| Centralized inspection VPC | A single fortified checkpoint district that all traffic between castle districts must pass through | A dedicated VPC attached to Transit Gateway where all east-west and north-south traffic is routed through Network Firewall endpoints before reaching its destination |
| AWS Firewall Manager | The king's marshal who ensures every castle in the realm follows the same defense standards | A central policy management service that deploys WAF, Shield Advanced, Network Firewall, security group, and DNS Firewall policies across all accounts in an AWS Organization, auto-remediating new resources |

---

## The Big Picture

Think of your AWS environment as a **medieval kingdom** -- a realm of many castles (accounts and VPCs), each housing different operations. The kingdom faces threats from outside (internet-borne attacks) and from within (compromised workloads, lateral movement). To defend against both, the kingdom employs **four layers of defense**, each operating at a different level:

1. **The outer wall** (Shield Standard) -- massive stone walls that absorb brute-force siege weapons (volumetric DDoS). They exist around every castle by default, maintained by the crown at no cost.
2. **The gate inspector** (WAF) -- stationed at the castle gates (CloudFront, ALB, API Gateway), this inspector reads every visitor's documents (HTTP requests) and checks them against a rulebook for forgeries (SQL injection), forbidden names (IP blocklists), suspicious behavior (rate limiting), and disguised robots (Bot Control).
3. **The inner checkpoint** (Network Firewall) -- a fortified inspection point inside the castle walls where every cart, messenger, and supply shipment passing between districts (VPCs or subnets) is stopped and inspected. Unlike the gate inspector who only reads documents (HTTP), this checkpoint opens crates and examines the cargo itself (deep packet inspection at Layers 3-7).
4. **The king's marshal** (Firewall Manager) -- the central authority who rides between all castles in the realm, ensuring every one follows the same defensive standards. If a new castle is built, the marshal automatically posts guards, erects walls, and distributes the current rulebook.

The critical insight: **these services are not alternatives -- they are complementary layers.** Shield absorbs volumetric floods before they reach your resources. WAF filters malicious application-layer requests at entry points. Network Firewall inspects all traffic flowing within and between VPCs. Firewall Manager ensures all three are consistently deployed across every account. Miss any layer and you have a gap in your defense.

```
LAYERED DEFENSE ARCHITECTURE -- FOUR SERVICES, FOUR LAYERS
=============================================================================

                        INTERNET / EXTERNAL TRAFFIC
                                   │
                                   ▼
  ┌────────────────────────────────────────────────────────────────────────┐
  │  LAYER 1: AWS SHIELD (The Outer Wall)                                 │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  Shield Standard (automatic, free)                              │  │
  │  │  Absorbs L3/L4 volumetric + protocol attacks                    │  │
  │  │  SYN floods, UDP reflection, DNS amplification                  │  │
  │  │                                                                 │  │
  │  │  Shield Advanced (paid, opt-in)                                 │  │
  │  │  + L7 DDoS detection  + DRT  + cost protection  + health-based  │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  └────────────────────────┬───────────────────────────────────────────────┘
                           │ Traffic that survives the wall
                           ▼
  ┌────────────────────────────────────────────────────────────────────────┐
  │  LAYER 2: AWS WAF (The Gate Inspector)                                │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  Attached to: CloudFront │ ALB │ API Gateway │ AppSync │ etc.   │  │
  │  │                                                                 │  │
  │  │  Web ACL evaluates HTTP requests against ordered rules:         │  │
  │  │  ├── AWS Managed Rules (SQLi, XSS, Bad Inputs, Bot Control)     │  │
  │  │  ├── Rate-based rules (throttle abusive IPs)                    │  │
  │  │  ├── IP set rules (allow/block known addresses)                 │  │
  │  │  ├── Geo-match rules (block/allow by country)                   │  │
  │  │  └── Custom rules (regex, size constraints, label matching)     │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  └────────────────────────┬───────────────────────────────────────────────┘
                           │ Requests that pass inspection
                           ▼
  ┌────────────────────────────────────────────────────────────────────────┐
  │  LAYER 3: AWS NETWORK FIREWALL (The Inner Checkpoint)                 │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  Deployed inside VPCs (firewall endpoints in dedicated subnets) │  │
  │  │                                                                 │  │
  │  │  Stateless Engine ──▶ 5-tuple matching (fast, shallow)          │  │
  │  │         │                                                       │  │
  │  │         ▼ (forward to stateful)                                 │  │
  │  │  Stateful Engine ──▶ Suricata IPS/IDS (deep, protocol-aware)    │  │
  │  │         ├── Domain filtering (allow/deny lists)                 │  │
  │  │         ├── TLS SNI inspection (see destination without decrypt)│  │
  │  │         ├── IPS signatures (detect/drop exploit traffic)        │  │
  │  │         └── Protocol anomaly detection                          │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  └────────────────────────┬───────────────────────────────────────────────┘
                           │ Clean traffic reaches workloads
                           ▼
  ┌────────────────────────────────────────────────────────────────────────┐
  │  LAYER 4: Security Groups + NACLs (You already know these)            │
  │  Instance-level and subnet-level packet filtering                     │
  └────────────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────────┐
  │  ORCHESTRATOR: AWS FIREWALL MANAGER (The King's Marshal)              │
  │  Deploys and enforces WAF, Shield, Network Firewall, Security Group,  │
  │  and DNS Firewall policies across ALL accounts in the Organization    │
  │  Reports non-compliant resources to Security Hub                      │
  └────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: AWS WAF -- The Gate Inspector

### The Analogy

AWS WAF is the **gate inspector** stationed at every entrance to your castle. Every visitor (HTTP request) must present their documents (headers, query string, body, URI path), and the inspector checks them against an ordered checklist (Web ACL). The inspector can:

- **Allow** the visitor through (allow action)
- **Turn them away** with a 403 (block action)
- **Let them through but note it in the logbook** (count action -- used for testing rules before enforcing them)
- **Challenge them to prove they are human** (CAPTCHA or Challenge action)
- **Stamp their hand with a label** that a downstream rule can check (labeling -- WAF's cross-rule communication mechanism)

The checklist is evaluated **top-to-bottom by priority number** (lowest number first). The moment a rule matches, its action fires and evaluation stops -- unless the action is Count or Label, in which case the request continues to the next rule. If no rule matches, the default action (Allow or Block) applies.

**Important nuance for rule groups**: When a managed rule group is added to a Web ACL, the `override_action` block controls the group's behavior at the Web ACL level. Setting `override_action { count {} }` overrides whatever action the matching rule *within* the group would have taken (Block, Allow, etc.) and converts it to Count. This is how you test an entire rule group in Count mode before enforcing it. When the group override is `none {}`, the individual rules' native actions fire normally. The `rule_action_override` block inside a managed rule group is used to selectively override *specific rules* within the group -- most useful when the group-level override is `none {}` (block mode) and you want to set certain noisy rules to Count for tuning.

### Web ACLs and Rules

A **Web ACL** is the central container. It holds rules (or references to rule groups), a default action, and is associated with one or more protected resources. Key constraints:

- A resource can be associated with **only one Web ACL** at a time
- A Web ACL can protect **multiple resources** of the same type
- **CloudFront Web ACLs must be created in us-east-1** (CloudFront is a global service with its control plane in us-east-1)
- Regional Web ACLs (for ALB, API Gateway, etc.) must be in the same region as the resource

### Rule Types

| Rule Type | What It Does | Castle Analogy | Example |
|-----------|-------------|----------------|---------|
| **Regular rule** | Matches request properties against conditions | "Check if the visitor's name is on the banned list" | Block requests where the URI contains `/admin` and the source IP is not in a trusted IP set |
| **Rate-based rule** | Counts requests from a single IP (or key) over a 5-minute window and triggers when the threshold is exceeded | "If the same visitor knocks on the gate more than N times in 5 minutes, stop letting them in (you set the threshold; minimum 100)" | Rate limit to 2000 requests per 5 minutes per IP (minimum threshold is 100, lowered from 2000 in late 2023) |
| **Managed rule group** | A pre-built set of rules maintained by AWS or a third-party Marketplace seller | "A printed inspection manual from the Royal Guard Academy" | AWS-AWSManagedRulesCommonRuleSet (Core Rule Set for OWASP Top 10) |
| **Custom rule group** | Your own reusable set of rules | "Your castle's own inspection manual, written by your guard captain" | A rule group combining geo-blocking, header validation, and rate limiting specific to your application |

### AWS Managed Rule Groups -- The Royal Guard's Inspection Manuals

These are the most commonly used rule groups. AWS maintains and updates them at no additional charge (except Bot Control, ATP, and ACFP, which have premium per-request pricing):

| Rule Group | What It Catches | WCU Cost |
|-----------|----------------|----------|
| **AWSManagedRulesCommonRuleSet** (Core Rule Set) | OWASP Top 10: cross-site scripting, file inclusion, path traversal, protocol violations | 700 |
| **AWSManagedRulesKnownBadInputsRuleSet** | Known malicious input patterns: Log4j/Log4Shell, SSRF patterns, Java deserialization | 200 |
| **AWSManagedRulesSQLiRuleSet** | SQL injection in headers, body, URI, and query strings | 200 |
| **AWSManagedRulesAdminProtectionRuleSet** | Blocks access to admin pages (e.g., `/admin`, `/wp-admin`) from unauthorized IPs | 100 |
| **AWSManagedRulesAmazonIpReputationList** | IP addresses with a poor reputation (botnets, scanners, etc.) sourced from Amazon threat intelligence | 25 |
| **AWSManagedRulesAnonymousIpList** | Traffic from VPNs, Tor exit nodes, proxies, and hosting providers | 50 |
| **AWSManagedRulesBotControlRuleSet** | Bot categorization and management (verified, common, targeted bots) | 50 (both common and targeted levels; the per-request *pricing* differs, not the WCU) |
| **AWSManagedRulesATPRuleSet** (Account Takeover Prevention) | Credential stuffing, brute-force login attempts, stolen credential use | 50 |
| **AWSManagedRulesACFPRuleSet** (Account Creation Fraud Prevention) | Fraudulent account creation patterns | 50 |

### Web ACL Capacity Units (WCU) -- The Guard's Toolkit Belt

**The Analogy**: The gate inspector carries a toolkit belt with a fixed weight limit. Simple tools (an IP checklist card) weigh almost nothing. Heavy tools (a regex scanner, an XSS detector, a body-inspection magnifying glass) fill up the belt fast. Once the belt is full, the inspector cannot add more tools -- regardless of how many visitors are in line. WCU is a **capacity budget**, not a time constraint; it limits the total complexity of rules you can add, not the per-request processing time.

**The Technical Reality**: Each Web ACL has a WCU budget of **1500 by default** (can be increased via support). Every rule or rule group consumes WCU based on its complexity:

| Rule Component | WCU Cost | Notes |
|---------------|----------|-------|
| Simple string match (exact, starts with, ends with) | 1 per match | Cheapest inspection |
| Size constraint | 1 | Checking Content-Length, URI length, etc. |
| IP set match | 1 | Even for sets with thousands of IPs |
| Geo match | 1 | Match by country code |
| Regex match | 3 per pattern | More expensive than string match |
| Regex pattern set | 25 per set | A set of up to 10 regex patterns |
| Rate-based rule | Varies | Based on the scope-down statement complexity |
| SQL injection match | 20 | Deep SQL pattern analysis |
| XSS match | 40 | Cross-site scripting pattern analysis |
| Body inspection (first 8 KB default) | Multiplied by text transformation count | Body is more expensive than header/URI |

> **Critical gotcha -- body inspection limits**: WAF only inspects the **first 8 KB** of the request body by default for regional Web ACLs (ALB, API Gateway, etc.), and **16 KB** for CloudFront Web ACLs. An attacker can pad the beginning of a request body with benign content and place a SQLi or XSS payload beyond the inspection boundary, bypassing inspection entirely. For **regional** Web ACLs, you can extend body inspection up to **64 KB** using the `body` field's `oversize_handling` configuration (note: this 64 KB extension is a regional feature, not a CloudFront feature). WAF offers three oversize handling behaviors: **CONTINUE** (inspect what fits, skip the rest -- the default, and the most dangerous), **MATCH** (treat oversized content as matching the rule -- effectively block any request too large to fully inspect), and **NO_MATCH** (treat oversized content as not matching). For security-sensitive rules like SQLi detection, consider setting oversize handling to MATCH so that oversized requests are blocked rather than silently passed. Always consider whether your application accepts request bodies larger than 8 KB -- if it does, you should extend the inspection limit, configure appropriate oversize handling, and add application-level input validation (parameterized queries) as a backstop.

> **Interview tip**: "Why can't I just add every AWS Managed Rule Group to my Web ACL?" Because you will blow the WCU budget. The Core Rule Set alone costs 700 WCU, and SQLi adds 200, and Known Bad Inputs adds 200. That is 1100 of your 1500 budget gone on three rule groups. You must be deliberate about which rule groups to include and may need to request a WCU increase for complex configurations.

### WAF Integration Points

WAF can protect eight resource types, but not all are equal:

| Resource Type | Region Requirement | Key Consideration |
|--------------|-------------------|-------------------|
| **Amazon CloudFront** | us-east-1 only | Most common deployment -- WAF rules execute at edge locations globally, blocking bad traffic before it reaches your origin |
| **Application Load Balancer** | Same region as ALB | Second most common -- protects web applications behind ALBs |
| **Amazon API Gateway (REST API)** | Same region | Protects REST APIs; does NOT support HTTP APIs or WebSocket APIs |
| **AWS AppSync** | Same region | Protects GraphQL APIs |
| **Amazon Cognito user pool** | Same region | Protects authentication endpoints (login, signup) |
| **AWS App Runner** | Same region | Protects App Runner services |
| **AWS Verified Access** | Same region | Protects Verified Access instances |
| **AWS Amplify** | Same region | Protects Amplify hosted apps |

> **Interview tip**: "Where should you attach WAF for the broadest protection?" CloudFront. A WAF Web ACL on CloudFront filters requests at the edge before they traverse the internet to your origin. This reduces origin load, improves latency, and means malicious traffic never reaches your VPC. This is why the CloudFront + WAF combination is the standard architecture for internet-facing applications.

> **Architectural distinction -- CloudFront WAF vs ALB WAF**: When WAF is attached to CloudFront (scope: `CLOUDFRONT`, must be in us-east-1), rules execute at **edge locations** before the request ever reaches your VPC. When WAF is attached to an ALB (scope: `REGIONAL`), rules execute **inside your VPC** after traffic has already traversed the internet and entered your network. This means CloudFront WAF blocks bad traffic before it consumes your VPC bandwidth or triggers origin auto-scaling, while ALB WAF only blocks it after it is already inside your perimeter. For internet-facing applications, always prefer CloudFront WAF; use ALB WAF only for applications that do not use CloudFront (e.g., internal ALBs or API-only services).

### Bot Control and Account Takeover Prevention

**Bot Control** is a premium WAF feature (additional cost per request inspected) that classifies traffic into three categories:

1. **Verified bots** -- legitimate crawlers like Googlebot and Bingbot (verified by reverse DNS lookup). Default action: allow.
2. **Common bots** -- generic HTTP libraries (curl, python-requests, headless browsers). You define the action.
3. **Targeted bots** -- sophisticated bots that mimic human behavior using browser automation. The "targeted" level uses browser fingerprinting and behavioral analysis.

**Account Takeover Prevention (ATP)** monitors your login endpoint for credential stuffing and brute-force attacks. You configure the login URL path and the request field names for username and password. ATP checks credentials against a stolen credential database maintained by AWS.

### WAF Logging and Testing -- Count Mode

**Always deploy new rules in Count mode first.** This is the single most important operational practice for WAF. Count mode lets a rule match and log without blocking traffic, so you can verify that your rules are not generating false positives against legitimate traffic.

The testing workflow:

1. Deploy the rule group or custom rule with action overridden to **Count**
2. Enable WAF logging (to S3, CloudWatch Logs, or Kinesis Data Firehose)
3. Run production traffic for 24-72 hours
4. Analyze sampled requests and logs for false positives
5. Tune rules (add exclusion rules, adjust match conditions)
6. Switch from Count to Block

```hcl
# Terraform: WAF Web ACL with managed rules -- some in Count mode for testing
resource "aws_wafv2_web_acl" "main" {
  name        = "production-web-acl"
  description = "Production Web ACL with layered managed rules"
  scope       = "REGIONAL"  # Use "CLOUDFRONT" for CloudFront distributions

  default_action {
    allow {}  # Allow by default, block explicitly via rules
  }

  # Rule 1: Rate limiting -- highest priority (lowest number)
  rule {
    name     = "rate-limit-per-ip"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000  # Max 2000 requests per 5-minute window per IP (minimum allowed: 100)
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitPerIP"
    }
  }

  # Rule 2: Block known bad IPs (Amazon IP reputation list)
  rule {
    name     = "aws-ip-reputation"
    priority = 2

    override_action {
      none {}  # Use the rule group's native actions (block)
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSIPReputation"
    }
  }

  # Rule 3: Core Rule Set (OWASP Top 10) -- IN COUNT MODE FOR TESTING
  rule {
    name     = "aws-core-rule-set"
    priority = 10

    override_action {
      count {}  # <-- Count mode: log matches but do not block
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        # rule_action_override selectively overrides specific rules within the group.
        # It is most useful when override_action is none {} (block mode) and you want
        # to set certain noisy rules to Count for tuning. Here it is shown for
        # illustration -- since this group's override_action is count {}, all rules
        # are already Count. Move this override to a block-mode group for real use.
        rule_action_override {
          name = "SizeRestrictions_BODY"
          action_to_use {
            count {}
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSCoreRuleSet"
    }
  }

  # Rule 4: SQL injection protection
  rule {
    name     = "aws-sql-injection"
    priority = 20

    override_action {
      none {}  # Block matches (this rule group is less prone to false positives)
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSSQLiRuleSet"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "ProductionWebACL"
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Associate the Web ACL with an ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.application.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}

# Enable WAF logging to CloudWatch Logs
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  log_destination_configs = [aws_cloudwatch_log_group.waf.arn]
  resource_arn            = aws_wafv2_web_acl.main.arn

  # Optional: filter logs to only capture blocked or counted requests
  logging_filter {
    default_behavior = "DROP"  # Do not log allowed requests (reduces cost)

    filter {
      behavior    = "KEEP"
      requirement = "MEETS_ANY"

      condition {
        action_condition {
          action = "BLOCK"
        }
      }

      condition {
        action_condition {
          action = "COUNT"
        }
      }
    }
  }
}

# CloudWatch Log Group for WAF logs (name MUST start with "aws-waf-logs-")
resource "aws_cloudwatch_log_group" "waf" {
  name              = "aws-waf-logs-production"
  retention_in_days = 30
}
```

### Labels -- WAF's Cross-Rule Communication

**The Analogy**: When the gate inspector checks a visitor's documents, sometimes the result is not a simple pass/fail. The inspector might stamp the visitor's hand with a label -- "this person came from country X" or "this request matched a SQL pattern but at low confidence." A second inspector downstream can read the stamp and make a decision based on it.

**The Technical Reality**: WAF labels are metadata strings that rules can add to a request during evaluation. Subsequent rules in the same Web ACL can match on those labels. This enables complex multi-stage logic:

1. Rule at priority 5 matches a condition and adds label `awswaf:managed:aws:core-rule-set:GenericLFI_BODY`
2. Rule at priority 6 checks for that label AND a custom condition (e.g., the request is to a specific URI) and decides to allow it (known false positive for that endpoint)

Labels are how AWS Managed Rule Groups communicate their findings without immediately blocking -- many managed rules add labels in Count mode so you can build your own decision logic on top.

---

## Part 2: AWS Shield -- The Outer Wall

### Shield Standard -- The Free Wall

**The Analogy**: Every castle in the kingdom has outer walls maintained by the crown at no cost. These walls automatically absorb the most common siege weapons -- battering rams (SYN floods), catapult barrages (UDP reflection attacks), and trebuchet bombardments (DNS amplification). You do not need to request these walls, maintain them, or even know the technical details of how they work. They are just there.

**The Technical Reality**: Shield Standard is automatically enabled for every AWS account at no additional cost. It protects all AWS resources against the most common Layer 3 (network) and Layer 4 (transport) DDoS attack vectors:

- **SYN/ACK floods** -- overwhelm TCP connection state tables
- **UDP reflection/amplification** -- spoof source IPs to reflect massive traffic volumes
- **DNS query floods** -- overwhelm DNS infrastructure

Shield Standard is strictly **Layer 3/Layer 4** -- it does NOT provide application-layer (HTTP flood) protection. HTTP flood detection and mitigation is a Shield Advanced feature. Note that services like CloudFront and ALB have some inherent HTTP flood resilience due to their scaling architecture, but that is infrastructure behavior, not a Shield Standard feature.

Shield Standard operates at the AWS edge network and is deeply integrated into services like CloudFront, Route 53, and Global Accelerator. For most workloads, Shield Standard is sufficient.

### Shield Advanced -- The Elite Garrison

**The Analogy**: Shield Advanced is hiring an **elite garrison of knights** for your castle. These knights bring capabilities far beyond what the basic walls provide: they station intelligence officers at every gate who detect subtle attack patterns the walls cannot see (Layer 7 DDoS detection). They bring a rapid-response team (DRT) that deploys experts to your castle during a siege. They have scouts on horseback (proactive engagement) who monitor your castle's health and ride ahead of the army to warn you. And critically, they bring a **war chest** (cost protection) -- if the siege forces you to hire extra defenders (auto-scaling), the garrison reimburses the cost.

**The Technical Reality**: Shield Advanced is a paid service ($3,000/month per Organization with a 1-year commitment, plus data transfer fees) that provides enhanced DDoS protection:

| Capability | What It Does | Why It Matters |
|-----------|-------------|----------------|
| **L3/L4/L7 DDoS protection** | Detects and mitigates application-layer HTTP floods in addition to network-layer attacks | Shield Standard only handles L3/L4; Advanced adds intelligent L7 detection |
| **DDoS Response Team (DRT/SRT)** | 24/7 AWS security engineers who can write WAF rules for you during an active attack | Requires Business or Enterprise Support plan; DRT can proactively manage your WAF rules. Note: AWS documentation now also refers to this team as the **Shield Response Team (SRT)** |
| **Proactive engagement** | DRT contacts you when Route 53 health checks degrade during a detected DDoS event | DRT reaches out to you rather than waiting for you to open a ticket |
| **Health-based detection** | Uses Route 53 health checks to detect when your application is degraded, lowering detection thresholds | Faster mitigation -- detects attacks based on application impact, not just traffic volume |
| **Automatic application-layer DDoS mitigation** | Automatically creates, tests, and deploys WAF rate-based rules during an L7 DDoS attack | WAF rules are deployed in Count mode first, then Block -- no manual intervention needed |
| **Cost protection** | AWS issues service credits for scaling charges during a DDoS attack | Covers EC2, ELB, CloudFront, Global Accelerator, and Route 53 scaling costs |
| **Protection groups** | Logically group resources for aggregate detection | Detect attacks that target multiple resources simultaneously but would be below threshold individually |
| **Advanced CloudWatch metrics** | Per-resource DDoS attack metrics and dashboards | Visibility into attack vectors, volume, and mitigation effectiveness |

### What Resources Shield Advanced Protects

Shield Advanced protection is applied **per resource**. You choose which resources to protect:

| Resource Type | Protection Layer |
|--------------|-----------------|
| Amazon CloudFront distributions | L3/L4/L7 |
| Amazon Route 53 hosted zones | L3/L4 |
| AWS Global Accelerator accelerators | L3/L4 |
| Elastic Load Balancers (ALB, NLB, CLB) | L3/L4 |
| Amazon EC2 instances (Elastic IP addresses) | L3/L4 |

> **Interview tip**: "What is the difference between Shield Standard and Shield Advanced?" Shield Standard is free, automatic, and handles L3/L4 DDoS. Shield Advanced is paid ($3,000/month/org), adds L7 DDoS detection with automatic WAF mitigation, provides the DRT, offers proactive engagement via health checks, and includes cost protection credits. Shield Advanced requires a resource-by-resource opt-in -- it does not automatically protect everything.

### Shield Advanced + WAF Integration

This is the critical architectural pattern: Shield Advanced uses WAF as its application-layer mitigation tool. When Shield Advanced detects an L7 DDoS attack:

1. Shield Advanced identifies the attack pattern (e.g., requests from a specific set of IPs, with specific headers, at anomalous rates)
2. It automatically creates a WAF rate-based rule that matches the attack pattern
3. The rule is deployed in **Count mode** first to verify it will not block legitimate traffic
4. After verification, the rule is switched to **Block mode**
5. When the attack subsides, the rule is automatically removed

This means Shield Advanced requires a WAF Web ACL to be associated with the protected resource for L7 mitigation to work. Shield Advanced without WAF can only mitigate L3/L4 attacks.

```
SHIELD ADVANCED L7 DDoS MITIGATION FLOW
=============================================================================

  Attack Detected (anomalous HTTP flood)
         │
         ▼
  ┌──────────────────────────────────────┐
  │  Shield Advanced Detection Engine    │
  │  (monitors CloudWatch + health       │
  │   checks + traffic baselines)        │
  └──────────────────┬───────────────────┘
                     │ Attack pattern identified
                     ▼
  ┌──────────────────────────────────────┐
  │  Automatic Mitigation                │
  │  1. Creates WAF rate-based rule      │
  │  2. Deploys in COUNT mode            │
  │  3. Validates no false positives     │
  │  4. Switches to BLOCK mode           │
  └──────────────────┬───────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────┐
  │  WAF Web ACL (on CloudFront/ALB)     │
  │  ┌────────────────────────────────┐  │
  │  │  Auto-deployed rate-based rule │  │
  │  │  + your existing rules         │  │
  │  └────────────────────────────────┘  │
  └──────────────────┬───────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────┐
  │  Attack subsides                     │
  │  → Auto-deployed rule is removed     │
  │  → Normal traffic flow resumes       │
  └──────────────────────────────────────┘
```

### Shield Advanced Pricing

Shield Advanced pricing has two components:

1. **Monthly subscription**: $3,000/month per Organization (not per account -- all accounts in the Organization share one subscription). 1-year commitment required.
2. **Data transfer fees**: Data transfer out (DTO) charges for protected resources, calculated per resource type.

The $3,000/month covers the subscription. The cost protection feature then provides credits for DDoS-related scaling costs (EC2 auto-scaling, ELB scaling, CloudFront burst). This is not a blanket refund -- you must file a cost protection claim and demonstrate the scaling was caused by a DDoS event.

---

## Part 3: AWS Network Firewall -- The Inner Checkpoint

### The Analogy

While WAF inspects visitors at the castle gate (HTTP requests at CloudFront/ALB), the **Network Firewall** is a fortified checkpoint **inside the castle walls** where every cart, messenger, and supply shipment passing between districts must be stopped and inspected. This checkpoint inspects all traffic -- not just HTTP. It reads TCP headers, examines UDP payloads, checks TLS handshakes, matches domain names, and runs a full intrusion prevention system (IPS) to detect known attack signatures.

The checkpoint has two inspection stations:

1. **The outer station** (stateless engine) -- a quick glance at each cart's manifest: source, destination, port, protocol, and direction. This is fast but shallow. In practice, the best pattern is to configure the outer station to simply **forward everything to the inner station** for deeper inspection.

2. **The inner station** (stateful engine) -- opens the crates, reads the documents, and runs every item against a massive catalog of known contraband (Suricata IPS rules). This station is connection-aware -- it tracks the full conversation between sender and receiver, not just individual packets.

### Architecture -- Firewall Endpoints in Dedicated Subnets

Network Firewall deploys as **firewall endpoints** -- managed ENIs that live in a dedicated firewall subnet in each Availability Zone. Traffic is routed through these endpoints via VPC route tables. The firewall itself does not live in your VPC -- the endpoint is just the entry point into the managed firewall service.

```
NETWORK FIREWALL ARCHITECTURE -- SINGLE VPC
=============================================================================

                    Internet Gateway
                          │
              ┌───────────┴───────────┐
              │  IGW Route Table       │
              │  10.0.1.0/24 → fw-ep  │  ◄── VPC Ingress Routing sends
              │  10.0.2.0/24 → fw-ep  │      inbound traffic to firewall
              └───────────┬───────────┘
                          │
                          ▼
  AZ-a                                           AZ-b
  ┌────────────────────────────┐   ┌────────────────────────────┐
  │  Firewall Subnet           │   │  Firewall Subnet           │
  │  10.0.100.0/28             │   │  10.0.101.0/28             │
  │  ┌──────────────────────┐  │   │  ┌──────────────────────┐  │
  │  │  Firewall Endpoint   │  │   │  │  Firewall Endpoint   │  │
  │  │  (managed ENI)       │  │   │  │  (managed ENI)       │  │
  │  └──────────┬───────────┘  │   │  └──────────┬───────────┘  │
  │             │              │   │             │              │
  │  FW Subnet Route Table:    │   │  FW Subnet Route Table:    │
  │  0.0.0.0/0 → IGW          │   │  0.0.0.0/0 → IGW          │
  └─────────────┬──────────────┘   └─────────────┬──────────────┘
                │                                │
                ▼                                ▼
  ┌────────────────────────────┐   ┌────────────────────────────┐
  │  Public Subnet (ALB)       │   │  Public Subnet (ALB)       │
  │  10.0.1.0/24               │   │  10.0.2.0/24               │
  │                            │   │                            │
  │  Subnet Route Table:       │   │  Subnet Route Table:       │
  │  0.0.0.0/0 → fw-endpoint  │   │  0.0.0.0/0 → fw-endpoint  │
  └────────────────────────────┘   └────────────────────────────┘
                │                                │
                ▼                                ▼
  ┌────────────────────────────┐   ┌────────────────────────────┐
  │  Private Subnet (App)      │   │  Private Subnet (App)      │
  │  10.0.10.0/24              │   │  10.0.11.0/24              │
  └────────────────────────────┘   └────────────────────────────┘
```

### Stateless Engine -- The Quick Glance

The stateless engine inspects packets individually (no connection tracking) using 5-tuple matching:

- Source IP / CIDR
- Destination IP / CIDR
- Source port
- Destination port
- Protocol (TCP, UDP, ICMP)

Stateless rules have a **capacity** budget (like WAF's WCU). The default maximum is **30,000 stateless capacity units** per firewall policy (the stateful engine also has a 30,000 capacity unit default). Each rule group has a capacity that counts against the policy maximum. Actions are:

| Action | What It Does |
|--------|-------------|
| **Pass** | Allow the packet through without stateful inspection |
| **Drop** | Silently discard the packet |
| **Forward to stateful rules** | Pass the packet to the stateful engine for deep inspection |

**Best practice**: Set the stateless engine's default action to **"Forward to stateful rules"** and do all real filtering in the stateful engine. **Why?** The stateless engine does not track connections -- it inspects each packet in isolation. If you drop inbound SYN packets from a blocked IP, you still need explicit rules for the return direction, because the stateless engine has no concept of "this is a response to an allowed connection." The stateful engine handles this automatically via connection tracking. The stateless engine is best reserved for simple, high-volume traffic that you definitively want to pass or drop without deeper inspection (e.g., dropping all traffic from a known bad CIDR).

### Stateful Engine -- The Deep Inspector

The stateful engine is powered by **Suricata**, the same open-source IPS/IDS engine used by security operations teams worldwide. This is what makes Network Firewall powerful -- you can use the full Suricata rule language.

The stateful engine supports three rule types:

| Rule Type | What It Does | Use Case |
|-----------|-------------|----------|
| **5-tuple rules** | Same as stateless but with connection tracking (stateful) | Block all outbound traffic to port 22 except to specific CIDRs |
| **Domain list rules** | Allow or deny traffic based on the destination domain name | Allow outbound HTTPS only to `*.amazonaws.com` and `api.github.com` |
| **Suricata-compatible IPS rules** | Full Suricata rule syntax with IPS signatures, protocol analyzers, and content matching | Detect and drop traffic matching known exploit signatures, detect C2 beacons, inspect HTTP headers |

AWS also provides **managed stateful rule groups** for Network Firewall -- similar to WAF's managed rule groups. These include threat signature rule groups for malware domains, botnet C2 servers, and known exploit patterns. You can subscribe to these via the Network Firewall console or reference them in Terraform, and AWS maintains and updates them automatically.

### Rule Evaluation Order -- Strict vs Action Order

Network Firewall has two evaluation modes for stateful rules:

| Mode | How It Works | When to Use |
|------|-------------|-------------|
| **Strict order** | Rules are evaluated by priority number (lowest first) across ALL rule groups; first match wins | **Recommended.** Predictable behavior. You control exactly which rule fires first. Supports a default drop action (drop everything that does not explicitly match an allow rule) |
| **Action order** | Rules are grouped by action: Pass rules evaluate first, then Drop, then Alert. Within each group, order is undefined | Legacy behavior. Less predictable because you cannot control order within an action group |

**Best practice**: Always use **strict order** with a **default drop action**. This creates a "deny all, allow explicitly" model -- the most secure baseline for network filtering.

### TLS Inspection -- Reading the Sealed Letter

**The Analogy**: Some messages arriving at the checkpoint are sealed in wax-sealed envelopes (TLS-encrypted). The inspector cannot read the contents without breaking the seal. Network Firewall offers two approaches: read the address on the outside of the envelope without breaking the seal (SNI inspection), or install an authorized seal-breaker and reseal the message after inspection (full TLS inspection).

**TLS SNI Inspection** (no decryption): When a TLS handshake begins, the client sends the **Server Name Indication (SNI)** -- the domain name it wants to connect to -- in plaintext. Network Firewall can match on this domain name to allow or block connections without decrypting the traffic. This is how domain list rules work for HTTPS traffic.

**Full TLS Inspection** (with decryption):

1. **Certificate**: You provide a CA certificate stored in ACM that the firewall uses to generate substitute certificates for inspected connections.
2. **MITM behavior**: The firewall decrypts the TLS traffic, inspects the plaintext against stateful rules, and re-encrypts it with a substitute certificate before forwarding.
3. **Client trust**: Clients must trust the inspection CA certificate. This works for east-west traffic between services you control (where you can install the CA cert), but NOT for internet-facing traffic where external clients would reject the substituted certificate.
4. **Performance and cost**: Full TLS inspection adds significant processing overhead and increases GB-processed charges. Factor this into capacity planning.

### Deployment Models -- Distributed vs Centralized

This is where your Transit Gateway knowledge from Feb 23 connects directly (see the [Transit Gateway deep dive](../../../2026/february/aws/2026-02-23-transit-gateway-deep-dive.md) for TGW route table configuration and appliance mode details).

**Distributed Model** -- a firewall in every VPC:

```
DISTRIBUTED DEPLOYMENT -- FIREWALL PER VPC
=============================================================================

  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
  │  VPC A (Prod)      │  │  VPC B (Staging)   │  │  VPC C (Dev)       │
  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │
  │  │ FW Endpoint  │  │  │  │ FW Endpoint  │  │  │  │ FW Endpoint  │  │
  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │
  │  FW Policy A       │  │  FW Policy B       │  │  FW Policy C       │
  │  (prod rules)      │  │  (staging rules)   │  │  (dev rules)       │
  └───────────────────┘  └───────────────────┘  └───────────────────┘
```

- **Pros**: Independent policies per VPC, no single point of inspection
- **Cons**: Higher cost (firewall endpoints per VPC per AZ), operational overhead managing multiple firewall policies
- **Use when**: VPCs have very different security requirements, or you need to minimize blast radius

**Centralized Model** -- one inspection VPC with Transit Gateway (think of it as a **dedicated customs house at the highway interchange** -- every road between separate castles must pass through this customs house before reaching its destination, rather than each castle maintaining its own internal checkpoint):

```
CENTRALIZED DEPLOYMENT -- INSPECTION VPC WITH TRANSIT GATEWAY
=============================================================================

  ┌────────────────────────────────────────────────────────────────────┐
  │  INSPECTION VPC                                                    │
  │                                                                    │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │  Firewall Subnet (AZ-a)          Firewall Subnet (AZ-b)     │  │
  │  │  ┌──────────────────┐            ┌──────────────────┐       │  │
  │  │  │ Firewall Endpoint│            │ Firewall Endpoint│       │  │
  │  │  └────────┬─────────┘            └────────┬─────────┘       │  │
  │  └───────────┼───────────────────────────────┼─────────────────┘  │
  │              │                               │                    │
  │  ┌───────────┴───────────────────────────────┴─────────────────┐  │
  │  │  TGW Attachment Subnet (AZ-a)    TGW Attachment Subnet(AZ-b)│  │
  │  │  Route: 0.0.0.0/0 → fw-endpoint                            │  │
  │  └────────────────────────────┬────────────────────────────────┘  │
  └───────────────────────────────┼────────────────────────────────────┘
                                  │
                        ┌─────────┴─────────┐
                        │  Transit Gateway   │
                        │  (Appliance Mode   │
                        │   ENABLED)         │  ◄── Critical for symmetric
                        └────┬────┬────┬────┘      routing through firewall
                             │    │    │
              ┌──────────────┘    │    └──────────────┐
              ▼                   ▼                   ▼
  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
  │  Spoke VPC A       │ │  Spoke VPC B       │ │  Spoke VPC C       │
  │  (Production)      │ │  (Staging)         │ │  (Development)     │
  │                    │ │                    │ │                    │
  │  Route:            │ │  Route:            │ │  Route:            │
  │  0.0.0.0/0 → TGW  │ │  0.0.0.0/0 → TGW  │ │  0.0.0.0/0 → TGW  │
  └───────────────────┘ └───────────────────┘ └───────────────────┘
```

- **Pros**: Single firewall policy for all traffic, lower cost (one set of firewall endpoints), centralized logging and monitoring
- **Cons**: Single point of inspection (must be highly available), all traffic hairpins through the inspection VPC (adds latency)
- **Use when**: Consistent security policy across VPCs, east-west traffic inspection required, cost optimization

**Critical detail -- TGW Appliance Mode**: When you enable appliance mode on the TGW attachment for the inspection VPC, Transit Gateway ensures that traffic flowing through the inspection VPC uses the **same AZ** for both the inbound and return path. Without appliance mode, TGW might send the request through AZ-a's firewall endpoint but route the response through AZ-b, causing asymmetric routing and dropped connections. This connects directly to the TGW appliance mode concept from your Feb 23 deep dive.

### Network Firewall Logging

Network Firewall supports three log types:

| Log Type | What It Captures | Use Case |
|----------|-----------------|----------|
| **Alert logs** | Traffic that matches a stateful rule with an Alert or Drop action | Security analysis, threat detection, compliance evidence |
| **Flow logs** | All traffic passing through the firewall (both directions) | Network forensics, traffic analysis, troubleshooting |
| **TLS logs** | TLS handshake metadata (SNI, certificate details, JA3 fingerprint) | Encrypted traffic analysis without decryption |

Logs can be sent to S3, CloudWatch Logs, or Kinesis Data Firehose. In the AWS SRA, alert and flow logs go to the Log Archive account.

```hcl
# Terraform: Network Firewall with centralized logging
resource "aws_networkfirewall_firewall" "inspection" {
  name                = "centralized-inspection-firewall"
  firewall_policy_arn = aws_networkfirewall_firewall_policy.main.arn
  vpc_id              = aws_vpc.inspection.id

  # Deploy firewall endpoints in dedicated subnets per AZ
  subnet_mapping {
    subnet_id = aws_subnet.firewall_az_a.id
  }

  subnet_mapping {
    subnet_id = aws_subnet.firewall_az_b.id
  }

  tags = {
    Environment = "security"
    ManagedBy   = "terraform"
  }
}

# Firewall policy with strict order evaluation and default drop
resource "aws_networkfirewall_firewall_policy" "main" {
  name = "centralized-inspection-policy"

  firewall_policy {
    # Stateless: forward everything to stateful engine
    stateless_default_actions          = ["aws:forward_to_sfe"]
    stateless_fragment_default_actions = ["aws:forward_to_sfe"]

    # Stateful: strict order with default drop
    stateful_engine_options {
      rule_order = "STRICT_ORDER"
    }

    stateful_default_actions = ["aws:drop_strict"]

    # Reference stateful rule groups
    stateful_rule_group_reference {
      priority     = 1
      resource_arn = aws_networkfirewall_rule_group.allow_domains.arn
    }

    stateful_rule_group_reference {
      priority     = 10
      resource_arn = aws_networkfirewall_rule_group.ips_rules.arn
    }
  }
}

# Domain allow-list rule group
resource "aws_networkfirewall_rule_group" "allow_domains" {
  capacity = 100
  name     = "allow-approved-domains"
  type     = "STATEFUL"

  rule_group {
    rule_variables {
      ip_sets {
        key = "HOME_NET"
        ip_set {
          definition = ["10.0.0.0/8"]  # All private RFC1918 space in your environment
        }
      }
    }

    rules_source {
      rules_source_list {
        generated_rules_type = "ALLOWLIST"
        target_types         = ["TLS_SNI", "HTTP_HOST"]
        targets = [
          ".amazonaws.com",  # Leading dot = wildcard prefix (matches any subdomain, e.g., s3.amazonaws.com)
          ".aws.amazon.com",
          "api.github.com",
          "registry.npmjs.org",
          ".datadog.com"
        ]
      }
    }
  }
}

# Suricata IPS rule group
resource "aws_networkfirewall_rule_group" "ips_rules" {
  capacity = 100
  name     = "ips-signatures"
  type     = "STATEFUL"

  rule_group {
    rules_source {
      # Suricata-compatible IPS rules
      rules_string = <<-EOT
        # Alert on any outbound connection to known C2 ports
        alert tcp $HOME_NET any -> $EXTERNAL_NET [4444,5555,6666,7777,8888] (msg:"Potential C2 communication on suspicious port"; sid:1000001; rev:1;)

        # Drop outbound SSH to non-approved destinations
        drop tcp $HOME_NET any -> !10.0.0.0/8 22 (msg:"Outbound SSH to non-internal destination blocked"; sid:1000002; rev:1;)

        # Alert on DNS queries to suspicious TLDs
        alert dns $HOME_NET any -> any any (msg:"DNS query to suspicious TLD"; dns.query; content:".xyz"; endswith; sid:1000003; rev:1;)
      EOT
    }
  }
}

# Logging configuration -- alert and flow logs to CloudWatch
resource "aws_networkfirewall_logging_configuration" "main" {
  firewall_arn = aws_networkfirewall_firewall.inspection.arn

  logging_configuration {
    log_destination_config {
      log_destination = {
        logGroup = aws_cloudwatch_log_group.nfw_alerts.name
      }
      log_destination_type = "CloudWatchLogs"
      log_type             = "ALERT"
    }

    log_destination_config {
      log_destination = {
        logGroup = aws_cloudwatch_log_group.nfw_flow.name
      }
      log_destination_type = "CloudWatchLogs"
      log_type             = "FLOW"
    }
  }
}
```

---

## Part 4: AWS Firewall Manager -- The King's Marshal

### The Analogy

The kingdom has dozens of castles (accounts), each with its own gates, walls, and checkpoints. The **king's marshal** rides between them, ensuring every castle follows the same defense standards. When a new castle is built (new account joins the Organization), the marshal automatically:

1. Posts gate inspectors with the standard rulebook (deploys WAF Web ACLs)
2. Activates the elite garrison subscription (enables Shield Advanced)
3. Builds the inner checkpoint (deploys Network Firewall rules)
4. Verifies the guard post staffing (audits security groups)
5. Reports any castle that does not meet standards to the central war council (sends non-compliance findings to Security Hub)

The marshal does not replace the individual castle captains (account-level security teams) -- the marshal ensures a **baseline** that every castle must meet. Individual captains can add their own rules on top.

### Prerequisites

Firewall Manager has three hard prerequisites:

1. **AWS Organizations**: Your accounts must be in an Organization (you set this up on Mar 2)
2. **AWS Config**: Must be enabled in every account and region where you want Firewall Manager policies to apply (you learned about Config on Mar 6)
3. **Firewall Manager administrator account**: A member account designated as the Firewall Manager admin (typically the Security Tooling account). This is set via the Organizations management account

```bash
# Designate the Security Tooling account as Firewall Manager administrator
# Run this from the management account
aws fms associate-admin-account \
  --admin-account 222222222222  # Security Tooling account ID
```

### Policy Types

Firewall Manager uses **policies** to deploy protections. Each policy type maps to a specific protection service:

| Policy Type | What It Deploys | Auto-Remediation |
|-------------|----------------|------------------|
| **WAF policy** | Web ACLs with rule groups to all in-scope resources (CloudFront, ALB, API Gateway) | Yes -- automatically creates and associates Web ACLs with new resources |
| **Shield Advanced policy** | Shield Advanced protection to specified resource types across all accounts | Yes -- automatically subscribes resources to Shield Advanced |
| **Network Firewall policy** | Firewall policies and rule groups to VPCs across accounts | Yes -- can auto-create firewall endpoints and configure route tables (distributed model) |
| **Security Group policy** | Audits or enforces security group configurations | Three sub-types: Common (apply baseline SGs), Audit (flag non-compliant SGs), Usage (remove unused SGs) |
| **Network ACL policy** | Audits or enforces NACL configurations | Yes -- can auto-remediate non-compliant NACLs |
| **Route 53 Resolver DNS Firewall policy** | DNS Firewall rule groups to VPCs | Yes -- associates DNS Firewall rule groups with VPCs |

> **What is DNS Firewall?** Route 53 Resolver DNS Firewall is an important complement to Network Firewall. It operates at the **DNS resolution layer** inside your VPCs -- blocking DNS queries to known malicious domains *before* any network connection is established. If a compromised workload tries to resolve `evil-c2-server.xyz`, DNS Firewall blocks the query at the Route 53 Resolver level, so the workload never gets an IP address to connect to. This is a defense-in-depth layer that works alongside Network Firewall's domain filtering: DNS Firewall blocks at resolution time, while Network Firewall blocks at connection time (catching cases where an IP is contacted directly without DNS).
| **Third-party firewall policy** | Palo Alto Cloud NGFW or Fortigate CNF | Yes -- integrates with third-party marketplace firewalls |

### Scope Configuration

Every Firewall Manager policy defines its scope -- which accounts and resources it applies to:

- **Include/exclude accounts**: Apply to all accounts, specific OUs, or exclude specific accounts
- **Resource type**: Which resource types the policy targets (e.g., ALBs, CloudFront distributions, VPCs)
- **Resource tags**: Include or exclude resources based on tags (e.g., only resources tagged `Environment=production`)

### WAF Policy -- First and Last Rule Groups

When Firewall Manager deploys a WAF policy, it creates a Web ACL that contains:

1. **First rule groups** -- rules the Firewall Manager admin defines that evaluate BEFORE any account-level rules. The account owner cannot reorder or override them.
2. **Account-level rules** -- rules the individual account owner adds to the Web ACL for application-specific logic.
3. **Last rule groups** -- rules the Firewall Manager admin defines that evaluate AFTER account-level rules. A safety net.

```
FIREWALL MANAGER WAF POLICY -- RULE EVALUATION ORDER
=============================================================================

  ┌──────────────────────────────────────────────────────────────────┐
  │  Web ACL (deployed by Firewall Manager)                         │
  │                                                                 │
  │  Priority 1-99:    FIRST RULE GROUPS (FM admin controls)        │
  │  ├── Rate limiting baseline                                     │
  │  ├── IP reputation blocklist                                    │
  │  └── Known bad inputs protection                                │
  │                                                                 │
  │  Priority 100-899: ACCOUNT-LEVEL RULES (account owner controls) │
  │  ├── Application-specific custom rules                          │
  │  ├── Additional managed rule groups                             │
  │  └── Geo-blocking for specific endpoints                        │
  │                                                                 │
  │  Priority 900-999: LAST RULE GROUPS (FM admin controls)         │
  │  ├── Core Rule Set (safety net)                                 │
  │  └── SQL injection baseline                                     │
  │                                                                 │
  │  Default Action: BLOCK (configured by FM admin)                 │
  └──────────────────────────────────────────────────────────────────┘
```

This "sandwich" model gives the security team centralized control over critical protections while allowing application teams flexibility for their own rules in the middle.

### Integration with Security Hub

Firewall Manager generates findings in Security Hub when it detects non-compliance:

- A new ALB is created without a WAF Web ACL
- A security group allows SSH from 0.0.0.0/0
- A VPC is missing Network Firewall endpoints
- A resource is not enrolled in Shield Advanced

These findings flow into the same ASFF-normalized pipeline you built with Security Hub, EventBridge, and Lambda/SSM on Mar 6. Firewall Manager can auto-remediate (attach the Web ACL, fix the security group) or simply report for manual action.

```hcl
# Terraform: Firewall Manager WAF policy
# This deploys a WAF Web ACL to all ALBs across the Organization
resource "aws_fms_policy" "waf_alb" {
  name                  = "org-waf-policy-alb"
  exclude_resource_tags = false
  remediation_enabled   = true  # Auto-remediate: attach Web ACLs to new ALBs

  # Apply to all accounts in the Organization
  include_map {
    orgunit = ["ou-xxxx-yyyyyyyy"]  # Production OU (this is the OU ID format from AWS Organizations)
  }

  # This targets both ALBs and NLBs, but WAF only supports ALBs.
  # Firewall Manager's WAF policy will only apply to ALBs; NLBs are skipped.
  resource_type = "AWS::ElasticLoadBalancingV2::LoadBalancer"

  security_service_policy_data {
    type = "WAFV2"

    managed_service_data = jsonencode({
      type = "WAFV2"

      defaultAction = {
        type = "ALLOW"
      }

      preProcessRuleGroups = [
        {
          ruleGroupArn  = null
          managedRuleGroupIdentifier = {
            vendorName            = "AWS"
            managedRuleGroupName  = "AWSManagedRulesAmazonIpReputationList"
          }
          overrideAction       = { type = "NONE" }
          ruleGroupType        = "ManagedRuleGroup"
          excludeRules         = []
          sampledRequestsEnabled = true
        },
        {
          ruleGroupArn  = null
          managedRuleGroupIdentifier = {
            vendorName            = "AWS"
            managedRuleGroupName  = "AWSManagedRulesCommonRuleSet"
          }
          overrideAction       = { type = "NONE" }
          ruleGroupType        = "ManagedRuleGroup"
          excludeRules         = []
          sampledRequestsEnabled = true
        }
      ]

      postProcessRuleGroups = []

      overrideCustomerWebACLAssociation = false  # Do not override existing Web ACLs
      sampledRequestsEnabledForDefaultActions = true
    })
  }

  tags = {
    ManagedBy = "firewall-manager"
    Purpose   = "organization-waf-baseline"
  }
}
```

---

## Part 5: Comparison and Decision Framework

### OSI Layer Coverage

| Service | Layer 3 (Network) | Layer 4 (Transport) | Layer 7 (Application) |
|---------|:-:|:-:|:-:|
| **Security Groups** | -- | Inbound/outbound port filtering | -- |
| **NACLs** | IP filtering | Port filtering | -- |
| **Shield Standard** | Volumetric DDoS | Protocol DDoS | -- |
| **Shield Advanced** | Volumetric DDoS | Protocol DDoS | HTTP flood DDoS |
| **AWS WAF** | -- | -- | HTTP request inspection (SQLi, XSS, bots, rate limiting) |
| **Network Firewall** | IP filtering | 5-tuple, IPS | Domain filtering, TLS inspection, protocol analysis |

### Decision Table -- When to Use What

| Scenario | Service | Why |
|----------|---------|-----|
| "I need to block SQL injection attacks on my web application" | **WAF** | WAF inspects HTTP request bodies, headers, and query strings for SQL injection patterns |
| "I need to protect against a volumetric DDoS attack" | **Shield Standard** (automatic) or **Shield Advanced** (for L7 DDoS, DRT, cost protection) | Shield operates at the edge and absorbs flood traffic before it reaches your resources |
| "I need to block all outbound traffic from my VPCs except to approved domains" | **Network Firewall** | Network Firewall domain filtering rules with strict order + default drop create an egress allow-list |
| "I need to inspect east-west traffic between VPCs for malicious patterns" | **Network Firewall** (centralized deployment with TGW) | Only Network Firewall can inspect non-HTTP traffic between VPCs; WAF only sees HTTP at edge/entry resources |
| "I need to ensure every ALB in my Organization has WAF protection" | **Firewall Manager** | FM auto-deploys WAF Web ACLs to new ALBs and reports non-compliant resources |
| "I need IPS/IDS capabilities with Suricata-compatible rules" | **Network Firewall** | The stateful engine runs Suricata, supporting the full rule language |
| "I need to rate-limit API requests per IP address" | **WAF** (rate-based rules) | WAF rate-based rules track requests per IP over 5-minute windows |
| "I need DDoS experts to help during an attack and cost protection for scaling" | **Shield Advanced** | DRT and cost protection are Shield Advanced-exclusive features |
| "I need to block traffic from specific countries" | **WAF** (geo-match rules) | WAF can match requests by the source country code from the GeoIP database |
| "I need to enforce the same security group rules across all accounts" | **Firewall Manager** (security group policy) | FM audits security groups and auto-remediates non-compliant ones |

### Where Each Service Sits in the Request Path

```
USER REQUEST JOURNEY -- FROM INTERNET TO WORKLOAD
=============================================================================

  User (Internet)
       │
       │ DNS resolution
       ▼
  Route 53 ◄─── Shield Standard protects Route 53
       │         Shield Advanced (optional) adds health-based detection
       │
       ▼
  CloudFront Edge Location
  ┌─────────────────────────────────────┐
  │  Shield Standard (always on, L3/L4) │
  │  Shield Advanced (optional, L3/L4/7)│
  │  WAF Web ACL (optional, L7)         │ ◄── WAF inspects HTTP here
  └──────────────────┬──────────────────┘     BEFORE reaching origin
                     │
                     │ Origin request (only clean traffic)
                     ▼
  VPC (Origin)
  ┌──────────────────────────────────────────────────────────┐
  │  Internet Gateway                                        │
  │       │                                                  │
  │       ▼                                                  │
  │  Network Firewall endpoint (if deployed)                 │
  │  ┌──────────────────────────────────────┐                │
  │  │  Stateless → Stateful (Suricata)     │ ◄── NFW        │
  │  └──────────────────┬───────────────────┘    inspects     │
  │                     │                        ALL traffic  │
  │                     ▼                                     │
  │  ALB                                                     │
  │  ┌──────────────────────────────────────┐                │
  │  │  Shield Standard (L3/L4)             │                │
  │  │  Shield Advanced (optional)          │                │
  │  │  WAF Web ACL (optional, L7)          │ ◄── Second WAF │
  │  └──────────────────┬───────────────────┘    checkpoint   │
  │                     │                        (if no CF)   │
  │                     ▼                                     │
  │  Security Group → EC2 / ECS / EKS                        │
  └──────────────────────────────────────────────────────────┘
```

---

## Cost Considerations

| Service | Pricing Model | Key Cost Drivers | Common Gotcha |
|---------|-------------|-----------------|---------------|
| **AWS WAF** | $5/month per Web ACL + $1/month per rule *or rule group reference* + $0.60 per million requests inspected | Request volume is the biggest driver at scale; Bot Control and ATP add per-request fees on top. A managed rule group (which contains many rules internally) counts as one $1/month charge when referenced in a Web ACL, not per rule within it | Bot Control targeted inspection can cost $10 per million requests -- easy to generate surprise bills on high-traffic sites |
| **Shield Standard** | Free | N/A | None -- it is always on and always free |
| **Shield Advanced** | $3,000/month per Organization (1-year subscription commitment, billed monthly) + data transfer out fees per protected resource | The flat subscription fee regardless of attack volume; DTO fees based on traffic through protected resources | The $3,000/month is per Organization, not per account. But you still pay DTO. And cost protection only covers DDoS-related scaling -- you must file a claim to get credits |
| **Network Firewall** | $0.395/hour per firewall endpoint (~$288/month) + $0.065 per GB processed | Endpoint hours (one per AZ, so 2-3 endpoints is $576-$864/month before any traffic) and GB processed | The per-AZ endpoint cost adds up fast. A centralized deployment with 2 AZs is $576/month in endpoint charges alone, before processing a single byte |
| **Firewall Manager** | $100/month per policy per Region | Number of policies multiplied by number of Regions | If you have 5 policy types across 4 Regions, that is $2,000/month just for Firewall Manager -- before the underlying services' costs |

---

## Key Takeaways

- **WAF is Layer 7 only and attaches to specific edge/entry resources.** It inspects HTTP/HTTPS requests at CloudFront, ALB, API Gateway, AppSync, Cognito, App Runner, Verified Access, and Amplify. It cannot inspect non-HTTP traffic and it cannot inspect traffic between VPCs. For those scenarios, use Network Firewall.

- **Shield Standard is free and automatic -- you do not need to enable it.** It protects all AWS resources against common L3/L4 DDoS attacks. Shield Advanced adds L7 DDoS detection (using WAF for mitigation), the DRT, proactive engagement, health-based detection, and cost protection. Shield Advanced costs $3,000/month per Organization and requires a 1-year commitment.

- **Shield Advanced needs WAF for Layer 7 protection.** The automatic application-layer DDoS mitigation feature creates WAF rate-based rules on the fly. Without a WAF Web ACL associated with the protected resource, Shield Advanced can only mitigate L3/L4 attacks.

- **Network Firewall deploys as endpoints in dedicated subnets -- traffic must be routed through them.** This is not a magic toggle. You must configure VPC route tables, IGW route tables (VPC Ingress Routing), and potentially TGW route tables to force traffic through the firewall endpoints. Without correct routing, traffic bypasses the firewall entirely.

- **For centralized Network Firewall with TGW, you MUST enable appliance mode.** Without appliance mode on the TGW attachment for the inspection VPC, return traffic may route through a different AZ than the request, causing asymmetric routing and dropped connections. This is the number one deployment mistake for centralized inspection architectures.

- **Set the Network Firewall stateless engine to "forward to stateful" and do all real filtering in the stateful engine.** The stateless engine is a quick pre-filter. The stateful Suricata engine is where domain filtering, IPS signatures, and protocol analysis happen. Use strict rule order with a default drop action for the most secure baseline.

- **Always deploy new WAF rules in Count mode first.** Run production traffic for 24-72 hours and analyze WAF logs for false positives before switching to Block. This is the most important operational practice for WAF. AWS Managed Rule Groups can generate false positives for specific applications -- use rule action overrides and label-based exclusion logic to handle them.

- **Firewall Manager requires Organizations + Config as prerequisites.** It deploys WAF, Shield Advanced, Network Firewall, security group, and DNS Firewall policies across all accounts. Its "first rule group / last rule group" sandwich model for WAF policies lets the security team enforce baselines while giving application teams flexibility for their own rules in the middle.

- **Firewall Manager reports non-compliance to Security Hub.** This connects your perimeter protection directly to the detection-to-remediation pipeline you built on Mar 6. A new ALB without WAF, a VPC without Network Firewall, a security group allowing SSH from 0.0.0.0/0 -- all generate Security Hub findings that can trigger automated remediation via EventBridge and Lambda.

- **CloudFront + WAF is the standard architecture for internet-facing applications.** WAF rules execute at CloudFront edge locations, blocking malicious traffic before it reaches your origin. This means blocked requests never consume origin bandwidth, never trigger origin auto-scaling, and never traverse the public internet to your VPC. Always deploy WAF at CloudFront, not just at the ALB behind it.

- **These services form a defense-in-depth stack, not alternatives.** Shield absorbs volumetric floods at the edge. WAF filters malicious HTTP requests at entry points. Network Firewall inspects all traffic within VPCs. Security Groups and NACLs provide instance/subnet-level filtering. Firewall Manager ensures consistent deployment across all accounts. Missing any layer creates a gap.

---

## Further Reading

- [AWS WAF Developer Guide -- How AWS WAF Works](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html) -- core concepts, rule evaluation order, and Web ACL structure
- [AWS Managed Rule Groups List](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html) -- complete list with WCU costs and rule descriptions
- [Shield Advanced Capabilities](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-capabilities.html) -- all nine capabilities in detail
- [Network Firewall Best Practices (AWS GitHub)](https://aws.github.io/aws-security-services-best-practices/guides/network-firewall/) -- the single best operational guide
- [Deployment Models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/) -- architecture diagrams for distributed, centralized, and combined models
- [AWS WAF Best Practices (AWS GitHub)](https://aws.github.io/aws-security-services-best-practices/guides/waf/) -- Count mode testing, logging, and rule tuning
- [AWS Firewall Manager Policy Types](https://docs.aws.amazon.com/waf/latest/developerguide/working-with-policies.html) -- all policy types with configuration details
- [Shield Advanced Pricing](https://aws.amazon.com/shield/pricing/) -- subscription, data transfer, and cost protection details
- [Network Firewall Deployment Models Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/using-aws-network-firewall-for-east-west-traffic-inspection.html) -- centralized and distributed architecture patterns in detail
