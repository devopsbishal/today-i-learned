# AWS Control Tower and Landing Zones -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-03
**Difficulty:** Intermediate

## Overview

This study plan covers AWS Control Tower as the automation layer on top of the AWS Organizations and SCP concepts you learned yesterday. After completing it, you will understand how Control Tower automates landing zone creation (the Security OU, Log Archive account, Audit account, and mandatory controls you would otherwise configure manually), how the three control types (preventive via SCPs, detective via Config Rules, proactive via CloudFormation Hooks) enforce governance at different points in the resource lifecycle, how Account Factory standardizes account provisioning, and how Account Factory for Terraform (AFT) integrates Control Tower with the Terraform workflows you already know. The key mental shift today: yesterday you learned the raw building blocks (Organizations, OUs, SCPs); today you learn the opinionated orchestration layer that wires them together according to AWS best practices.

## Resources

### 1. What Is AWS Control Tower? -- AWS Official Docs ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html
- **Type:** Official Docs
- **Summary:** Start here to establish the mental model. This page defines Control Tower's four pillars -- landing zone, controls (guardrails), Account Factory, and dashboard -- and explains how Control Tower orchestrates Organizations, IAM Identity Center, Service Catalog, Config, and CloudFormation StackSets under the hood. You already understand Organizations and SCPs from yesterday; this page shows you the orchestration layer that automates what you would otherwise build manually. Pay attention to the relationship diagram: Control Tower does not replace Organizations, it sits on top of it. Also note the drift detection concept -- Control Tower monitors whether someone has manually changed the guardrails or account configurations it deployed, which is analogous to Terraform detecting drift between state and reality.

### 2. How AWS Control Tower Works -- AWS Official Docs ⏱️ 20 min
- **URL:** https://docs.aws.amazon.com/controltower/latest/userguide/how-control-tower-works.html
- **Type:** Official Docs
- **Summary:** This page walks through what actually happens when you set up a landing zone: Control Tower creates the Security OU (containing the Log Archive account for centralized CloudTrail and Config logs, and the Audit account for cross-account security tooling), optionally creates a Sandbox OU, configures IAM Identity Center with preconfigured permission sets, and deploys mandatory preventive and detective controls. Connect this directly to yesterday's learning: the Security OU with Log Archive and Audit accounts maps exactly to the recommended OU structure from the AWS multi-account whitepaper you studied. The key addition today is understanding CloudFormation StackSets as the deployment mechanism -- Control Tower uses StackSets to push baseline configurations (Config rules, CloudTrail settings, IAM roles) into every enrolled account across all governed regions. Also note the critical detail that preventive controls (SCPs) do not apply to the management account, which you already learned yesterday.

### 3. Control Behavior and Guidance -- AWS Control Tower Control Reference ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/controltower/latest/controlreference/control-behavior.html
- **Type:** Official Docs (Control Reference)
- **Summary:** A concise but essential reference page that defines the two independent dimensions for classifying controls: behavior (preventive, detective, proactive) and guidance (mandatory, strongly recommended, elective). Preventive controls are implemented as SCPs -- you already know exactly how these work from yesterday's SCP evaluation deep dive. Detective controls are AWS Config rules that monitor resource compliance after deployment and report violations to the Control Tower dashboard. Proactive controls are CloudFormation Hooks that intercept resource creation before it happens, rejecting non-compliant resources at deploy time. The three guidance levels determine which controls are auto-enabled (mandatory), which AWS recommends you enable (strongly recommended), and which are situational (elective). This taxonomy is essential for navigating the control catalog and for interview questions that ask you to differentiate the three control types.

### 4. Control Tower Guardrails Overview: Preventive, Detective, and Proactive -- Automat-it Blog ⏱️ 25 min
- **URL:** https://www.automat-it.com/blog/control-tower-guardrails-overview-preventive-detective-and-proactive/
- **Type:** Blog (Automat-it, AWS Partner)
- **Summary:** This is the resource that bridges theory and practice for the three control types. Written by an AWS expert at an AWS Partner company (published April 2024), it provides concrete demonstrations with screenshots for each control type: a detective control flagging unencrypted EBS volumes via Config rules, a preventive control blocking public EBS snapshot sharing via SCPs, and a proactive control rejecting S3 buckets without secure transport requirements via CloudFormation Hooks. What makes this valuable beyond the official docs is that it shows the actual control enablement workflow in the console, the resulting Config rule evaluations, and the CloudFormation deployment failures when proactive controls block non-compliant resources. This practical walkthrough makes the abstract control taxonomy tangible and gives you concrete examples for interview answers.

### 5. Designing an AWS Control Tower Landing Zone -- AWS Prescriptive Guidance ⏱️ 25 min
- **URL:** https://docs.aws.amazon.com/prescriptive-guidance/latest/designing-control-tower-landing-zone/introduction.html
- **Type:** Official Docs (Prescriptive Guidance)
- **Summary:** The comprehensive design guide that covers all seven foundational pillars of a Control Tower landing zone: landing zone setup, account structure and OUs, controls and compliance, networking integration, authentication and authorization (IAM Identity Center), centralized logging and monitoring, and resource configuration management. Focus especially on the account structure section (maps to the OU hierarchy you studied yesterday but shows how Control Tower implements it), the controls section (how to select and layer mandatory, strongly recommended, and elective controls), and the networking section (which connects back to your Week 1 knowledge of Transit Gateway and VPC architecture). Skim the authentication section -- you will cover IAM Identity Center in depth during the IAM Advanced Patterns session on Day 3. This guide is the architectural blueprint you will reference when building the multi-account landing zone project in Week 7.

### 6. Account Factory for Terraform (AFT) Overview -- AWS Official Docs ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html
- **Type:** Official Docs
- **Summary:** A focused overview of AFT, the Terraform-native integration for Control Tower account provisioning. AFT adopts a GitOps model: you define account requests as Terraform files in a Git repository, push changes, and a Step Functions pipeline provisions the account through Control Tower's Account Factory, then applies customizations. This directly connects to your 14 days of Terraform knowledge -- AFT is how you would manage Control Tower accounts using the same Terraform workflows you already practice. Key concepts: the AFT management account (a dedicated account for the pipeline infrastructure), account request repos (where you define new accounts), global customizations (applied to all accounts), and account-specific customizations. AFT supports Terraform Community Edition, Cloud, and Enterprise. Read this to understand the architecture; you will implement it in Week 7.

### 7. Best Practices for Applying Controls with AWS Control Tower -- AWS Cloud Operations Blog ⏱️ 10 min
- **URL:** https://aws.amazon.com/blogs/mt/best-practices-for-applying-controls-with-aws-control-tower/
- **Type:** Blog (AWS Cloud Operations Blog)
- **Summary:** An AWS-authored best practices post that covers the operational side of managing controls at scale. Key recommendations include: start with mandatory controls (auto-enabled), then enable strongly recommended controls across all OUs, then selectively apply elective controls based on your compliance requirements; test controls in a non-production OU before applying to production (the same Policy Staging OU pattern from yesterday's SCP best practices); use the Controls Dedicated Experience in the console to search, filter, and enable controls across multiple OUs simultaneously; and align control selection with compliance frameworks (CIS Benchmarks, NIST 800-53, PCI DSS). This gives you the practical playbook for how a cloud team would roll out governance controls incrementally rather than enabling everything at once.

## Study Tips

- **Map every Control Tower automation to the manual equivalent you learned yesterday.** When the docs say "Control Tower creates a Security OU with Log Archive and Audit accounts," mentally trace back: yesterday you learned the recommended OU structure (Security OU, Infrastructure OU, Workloads OU) and the role of centralized logging. Control Tower automates two-thirds of that structure. When you see "preventive controls," think "these are SCPs with a nice UI and pre-written policy JSON." This mapping exercise is what solidifies the relationship between the two layers and is exactly what interviewers test when they ask "what does Control Tower do that Organizations does not?"

- **Build a comparison table as you read: Control Tower feature vs. the manual Organizations equivalent.** For example: Account Factory vs. manually creating accounts with `aws organizations create-account`; preventive controls vs. writing and attaching SCPs yourself; detective controls vs. manually deploying Config rules in every account; landing zone vs. manually creating OUs, shared accounts, and CloudTrail configuration. This table becomes your study artifact and interview cheat sheet.

- **Pay special attention to the three control implementation mechanisms.** Preventive = SCPs (you know these), Detective = Config Rules (you will deep-dive on Day 4 this week with Security Services), Proactive = CloudFormation Hooks (newer, blocks non-compliant resources before creation). The proactive type is the newest and least intuitive -- the Automat-it blog post (Resource 4) has the best practical demonstration. Understanding when each type fires in the resource lifecycle (before creation, at creation, after creation) is a common exam and interview question.

## Next Steps

After completing this study plan, consider exploring:

1. **Tonight's Terraform build** -- Implement SCPs that mirror what Control Tower's mandatory preventive controls enforce: deny root user access in member accounts, restrict API calls to approved regions (with exceptions for global services), and prevent disabling of CloudTrail and Config. This reinforces that Control Tower's guardrails are SCPs and Config rules you could write yourself -- Control Tower just manages them at scale.

2. **IAM Advanced Patterns (Day 3, tomorrow)** -- Control Tower uses IAM Identity Center for centralized access management. Tomorrow you will learn permission boundaries, cross-account roles, and federation -- these are the identity mechanisms that complement Control Tower's governance controls.

3. **Security Services: GuardDuty, Security Hub, Config (Day 4)** -- Detective controls in Control Tower are implemented as Config rules. On Day 4, you will deep-dive into Config, understanding how rules evaluate resources and how Security Hub aggregates findings across accounts. This completes the detective control picture that Control Tower introduces today.

4. **Week 7 multi-account landing zone project** -- Everything from this week (Organizations, Control Tower, IAM, security services, KMS) feeds directly into the Week 7 build where you will design and implement a full landing zone architecture in Terraform, including OU hierarchy, SCPs, centralized logging, Transit Gateway sharing, and security baselines.
