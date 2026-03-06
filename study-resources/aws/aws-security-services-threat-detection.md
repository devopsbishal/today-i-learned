# AWS Security Services -- Threat Detection -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-06
**Difficulty:** Intermediate

## Overview

This study plan covers four AWS security services that form the detection and compliance layer of an enterprise AWS environment: GuardDuty (threat detection), Security Hub (findings aggregation), AWS Config (configuration tracking and compliance rules), and Inspector (vulnerability assessment). After completing it, you will understand what each service detects, how they feed findings into Security Hub as a central pane of glass, how they integrate with AWS Organizations via delegated administrators (building on what you learned about the Security Tooling account in your Control Tower doc), and how automated remediation works through EventBridge.

## Resources

### 1. AWS Decision Guide -- Choosing Security Services ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/decision-guides/latest/security-on-aws-how-to-choose/choosing-aws-security-services.html
- **Type:** Official Docs
- **Summary:** Start here to build the mental model of how GuardDuty, Inspector, Config, and Security Hub each serve a distinct purpose and work together as layers -- GuardDuty detects threats, Inspector finds vulnerabilities, Config tracks compliance, and Security Hub aggregates everything into one dashboard; this page gives you the "why each exists" context before diving into any individual service

### 2. What Is Amazon GuardDuty? ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html
- **Type:** Official Docs
- **Summary:** The official overview of GuardDuty covering its architecture, foundational data sources (CloudTrail management events, VPC Flow Logs, DNS logs), optional protection plans (S3, EKS, RDS, Lambda, Malware, Runtime Monitoring), Extended Threat Detection for multi-stage attacks, and multi-account management via Organizations -- pay special attention to how GuardDuty consumes logs you never have to configure (it reads them directly from AWS infrastructure, not from your account)

### 3. AWS Config Terminology and Concepts ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/config/latest/developerguide/config-concepts.html
- **Type:** Official Docs
- **Summary:** The single best page for understanding Config's core machinery: configuration items (point-in-time snapshots of every resource), configuration recorder, configuration history and snapshots, configuration stream via SNS, managed vs custom rules, proactive vs detective evaluation modes, conformance packs (rule bundles deployed as YAML templates), and aggregators for multi-account/multi-region visibility -- you already saw Config Rules as detective controls in your Control Tower doc, and this page fills in the underlying mechanics

### 4. How AWS Config Works ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/config/latest/developerguide/how-does-config-work.html
- **Type:** Official Docs
- **Summary:** A concise visual walkthrough of the Config data flow: resource change detected via Describe/List API calls, configuration item created, rules evaluated, compliance status recorded, delivery to S3 and SNS -- read this right after the concepts page to see the pieces in motion

### 5. What Is Amazon Inspector? ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html
- **Type:** Official Docs
- **Summary:** Overview of Inspector v2 (the modern version, completely rebuilt from Inspector Classic): automatic discovery and continuous scanning of EC2 instances, ECR container images, and Lambda functions for software vulnerabilities (CVEs) and unintended network exposure; covers agent-based (SSM Agent) vs agentless scanning, environment-aware risk scoring that adjusts NVD severity based on your actual network topology, and multi-account management via delegated administrator

### 6. Introduction to AWS Security Hub ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html
- **Type:** Official Docs
- **Summary:** The official overview of Security Hub CSPM as the central aggregation point: collects findings from GuardDuty, Inspector, Config, Macie, Firewall Manager, and third-party tools; normalizes everything into AWS Security Finding Format (ASFF); evaluates your posture against security standards (AWS FSBP, CIS Benchmarks, PCI DSS, NIST); supports central configuration via Organizations so one delegated admin can push policies to all accounts and regions

### 7. AWS Security Services Best Practices -- Security Hub ⏱️ 25 min
- **URL:** https://aws.github.io/aws-security-services-best-practices/guides/security-hub/
- **Type:** Official Best Practices Guide (AWS GitHub)
- **Summary:** The most practical resource in this list -- covers real-world Security Hub deployment patterns: setting up a delegated administrator in the Security Tooling account (which you already know from your Control Tower and Organizations docs), central configuration policies that auto-enroll new accounts, cross-region finding aggregation, automation rules for suppressing or enriching findings, EventBridge integration for automated remediation, and SIEM/SOAR integration patterns; this is the bridge from "what does it do" to "how do I run it in production"

### 8. AWS Security Reference Architecture -- Security Tooling Account ⏱️ 25 min
- **URL:** https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/security-tooling.html
- **Type:** Official Prescriptive Guidance
- **Summary:** The capstone resource that shows how all four services (plus CloudTrail, Detective, Macie, EventBridge, and KMS) are deployed together in the Security Tooling account of the AWS Security Reference Architecture; covers the delegated administrator pattern for every service, the finding flow from source services through Security Hub to EventBridge for automated remediation, cross-account log storage in the Log Archive account, and the separation of duties between security administration and log management -- this ties directly into the OU structure and account topology you learned in your Organizations and Control Tower docs

## Study Tips

- Draw the finding flow diagram as you read: GuardDuty/Inspector/Config each generate findings independently, all feed into Security Hub via ASFF normalization, Security Hub publishes to EventBridge, EventBridge triggers Lambda or SNS for remediation/notification. Understanding this pipeline is the single most important takeaway.
- For each service, note three things: (1) what data source it consumes, (2) what it detects/evaluates, and (3) where its findings go. This framework makes interview questions straightforward.
- Connect everything back to your existing knowledge: the Security Tooling account from Control Tower, the delegated administrator pattern from Organizations, and the cross-account IAM roles from your IAM Advanced Patterns doc. These services are the *reason* that account structure exists.

## Next Steps

- **KMS and Encryption (tomorrow, Mar 7):** All of these security services encrypt findings and logs with KMS keys -- understanding key policies, grants, and cross-account KMS access will complete the security foundation.
- **Terraform implementation:** Deploy AWS Config rules (enforce encryption, enforce tagging) in Terraform during your evening build block; the `aws_config_config_rule` and `aws_config_configuration_recorder` resources map directly to the concepts you learned today.
- **Automated remediation deep dive:** After the KMS topic, explore building a GuardDuty finding -> EventBridge -> Lambda auto-remediation pipeline; the official doc at https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_findings_eventbridge.html walks through the EventBridge rule pattern.
