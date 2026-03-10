# AWS Network and Application Protection Services -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-10
**Difficulty:** Intermediate

## Overview

This study plan covers the four AWS services that form the network and application perimeter defense layer: AWS WAF (Layer 7 web request filtering), AWS Shield (DDoS protection at Layers 3/4/7), AWS Network Firewall (VPC-level deep packet inspection at Layers 3-7), and AWS Firewall Manager (centralized policy management across accounts). After completing it, you will understand what each service protects against, how they complement each other across different network layers, how Network Firewall integrates with Transit Gateway in centralized inspection architectures (building on your Transit Gateway deep dive from Feb 23), and how Firewall Manager ties into your AWS Organizations structure to enforce protection policies across all accounts.

## Resources

### 1. What Are AWS WAF, AWS Shield Advanced, and AWS Firewall Manager? -- Official Overview ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html
- **Type:** Official Docs
- **Summary:** The single best starting page for building the mental model of how all four services relate to each other -- WAF filters web requests at Layer 7, Shield protects against DDoS at Layers 3/4/7, Network Firewall provides deep packet inspection at Layers 3-7 within VPCs, and Firewall Manager centrally deploys and manages all of them across your Organization; read this first to understand why each exists and what gap it fills before diving into any individual service

### 2. AWS WAF or AWS Shield? -- Decision Guide ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/decision-guides/latest/waf-or-shield/waf-or-shield.html
- **Type:** Official Docs (Decision Guide)
- **Summary:** A structured comparison across eight dimensions (purpose, protection layer, deployment model, customization, traffic inspection, pricing, support) that clarifies the WAF vs Shield boundary -- WAF inspects HTTP request content (headers, body, query strings) for application-layer attacks like SQLi and XSS, while Shield detects network-level volumetric and protocol attacks; the key insight is that Shield Advanced actually integrates with WAF for Layer 7 DDoS mitigation, so they work together rather than being alternatives

### 3. How AWS WAF Works -- Concepts and Components ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html
- **Type:** Official Docs
- **Summary:** The core conceptual page for WAF covering Web ACLs (the container that holds your rules and associates with a protected resource), rules (match statements + actions), rule groups (reusable bundles), actions (Allow, Block, Count, CAPTCHA, Challenge), WCU capacity units (how AWS meters rule complexity), and the eight resource types WAF can protect (CloudFront, ALB, API Gateway REST, AppSync, Cognito, App Runner, Verified Access, Amplify); after this page, read the linked AWS Managed Rule Groups page (https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups.html) to understand the pre-built rule sets for common threats -- the Baseline rule groups (Core Rule Set, Admin Protection, Known Bad Inputs) and use-case-specific groups (SQL Database, Linux/Windows OS, PHP/WordPress) are the ones you will use most often

### 4. AWS Shield Advanced Capabilities ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-capabilities.html
- **Type:** Official Docs
- **Summary:** The most feature-complete page on Shield Advanced covering all nine capabilities: WAF integration for L7 protection, automatic application-layer DDoS mitigation (deploys WAF rate-based rules automatically during attacks), health-based detection (Route 53 health checks lower detection thresholds so mitigation triggers faster), protection groups (logical resource groupings for aggregate detection), enhanced CloudWatch visibility, centralized management via Firewall Manager, the Shield Response Team (SRT -- 24/7 experts who will write WAF rules for you during an active attack, requires Business or Enterprise Support), proactive engagement (SRT contacts you when health checks degrade during a detected event), and cost protection (service credits for DDoS-related scaling charges on EC2, ELB, CloudFront, Global Accelerator, Route 53)

### 5. AWS Network Firewall Best Practices Guide ⏱️ 25 min
- **URL:** https://aws.github.io/aws-security-services-best-practices/guides/network-firewall/
- **Type:** Official Best Practices Guide (AWS GitHub)
- **Summary:** The most practical resource in this list -- covers the three deployment models (distributed per-VPC, centralized inspection VPC with Transit Gateway, combined), stateless vs stateful rule engines (recommendation: set stateless default action to "forward to stateful" and do all real filtering in stateful rules), strict vs action-order rule evaluation, Suricata-compatible rule syntax, HOME_NET variable configuration for centralized deployments, TLS SNI inspection and JA3 fingerprinting for encrypted traffic analysis, domain allow-listing patterns, Transit Gateway appliance mode (critical for symmetric routing through the firewall -- connects directly to what you learned in your TGW deep dive), logging configuration (alert logs vs flow logs), CloudWatch dashboard queries, and cost optimization strategies; this single guide replaces what would otherwise be 5-6 separate doc pages

### 6. Deployment Models for AWS Network Firewall -- AWS Blog ⏱️ 20 min
- **URL:** https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/
- **Type:** Blog (AWS Networking & Content Delivery)
- **Summary:** The definitive architecture blog post with detailed diagrams for each deployment pattern -- distributed (firewall endpoints in every VPC, simple routing but higher cost and operational overhead), centralized (dedicated inspection VPC with Transit Gateway as the hub, firewall endpoints in a firewall subnet per AZ, TGW attachment subnet per AZ, appliance mode enabled for symmetric routing), and combined (centralized for east-west inter-VPC traffic, distributed for internet ingress in specific VPCs); the routing table diagrams are essential -- they show exactly how traffic flows from a spoke VPC through TGW to the inspection VPC's firewall endpoint and back, which builds directly on the Transit Gateway route table concepts you already know

### 7. AWS WAF Best Practices Guide ⏱️ 15 min
- **URL:** https://aws.github.io/aws-security-services-best-practices/guides/waf/
- **Type:** Official Best Practices Guide (AWS GitHub)
- **Summary:** Covers the four phases of WAF operations: prerequisites (account setup, logging destinations), configuring rules (start with AWS Managed Rules in Count mode before switching to Block, always include at least one rate-based rule, use labels for cross-rule logic), monitoring (CloudWatch metrics per rule and per Web ACL, sampled requests for debugging false positives), and integration with other services (CloudFront, ALB, API Gateway); the practical advice on testing rules in Count mode first and using the WAF logging to S3/CloudWatch/Kinesis pipeline is exactly what you need for the Terraform implementation

### 8. AWS Firewall Manager Overview and Policy Types ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/waf/latest/developerguide/fms-chapter.html
- **Type:** Official Docs
- **Summary:** The capstone resource that ties everything together -- Firewall Manager is the central management layer that deploys WAF Web ACLs, Shield Advanced protections, Network Firewall policies, security group rules, network ACLs, and Route 53 Resolver DNS Firewall rules across all accounts in your Organization; it auto-applies policies to new accounts and resources as they are created, reports non-compliant resources to Security Hub (connecting back to your threat detection pipeline from Mar 6), and requires a delegated administrator account (the same Security Tooling account pattern you learned in Organizations and Control Tower); after reading the overview, skim the policy types page (https://docs.aws.amazon.com/waf/latest/developerguide/working-with-policies.html) to see how each policy type maps to a specific protection service

## Study Tips

- Draw a layered defense diagram as you read: Shield Standard sits at the AWS edge protecting L3/L4 automatically, Shield Advanced adds L7 DDoS detection and SRT support, WAF sits at CloudFront/ALB/API Gateway filtering application-layer requests, Network Firewall sits inside VPCs inspecting all traffic at L3-L7, and Firewall Manager orchestrates all of them across accounts. Understanding which service operates at which layer and which resource types it protects is the single most important takeaway for interview questions.
- Connect Network Firewall's centralized deployment model to your existing Transit Gateway knowledge: the inspection VPC is just another spoke attached to TGW, but with appliance mode enabled and route tables configured to hairpin traffic through the firewall endpoints before forwarding it to the destination VPC. If you can explain this traffic flow on a whiteboard, you can answer any Network Firewall architecture question.
- For each service, note the integration point with Firewall Manager: WAF policies push Web ACLs, Shield policies push Shield Advanced protections, Network Firewall policies push firewall rules to VPCs, and all non-compliance findings flow into Security Hub. This is the "single pane of glass" story that ties your entire Week 2 security learning together.

## Next Steps

- **Route 53 and CloudFront (next topic):** WAF Web ACLs are most commonly associated with CloudFront distributions, so understanding CloudFront's edge architecture will deepen your WAF knowledge -- you will see how WAF rules execute at edge locations before requests even reach your origin.
- **Terraform implementation:** Deploy a WAF Web ACL with AWS Managed Rules (Core Rule Set + Known Bad Inputs) attached to an ALB using the `aws_wafv2_web_acl` and `aws_wafv2_web_acl_association` resources; start rules in Count mode, verify with `aws_wafv2_web_acl_logging_configuration` sending to CloudWatch Logs, then switch to Block.
- **Centralized Network Firewall lab:** The AWS whitepaper section on centralized inspection (https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-network-security-for-vpc-to-vpc-and-on-premises-to-vpc-traffic.html) has the exact architecture you would build in Terraform -- inspection VPC with TGW attachment, firewall endpoints, and route tables that force east-west traffic through the firewall.
