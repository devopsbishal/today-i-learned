# AWS Organizations and SCPs -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-02
**Difficulty:** Intermediate

## Overview

This study plan covers AWS Organizations as the foundation for multi-account governance: how the management account, organizational units, and service control policies work together to create security guardrails across dozens or hundreds of AWS accounts. After completing this plan, you will understand the Organization hierarchy (root, OUs, accounts), SCP inheritance and evaluation logic (deny-list vs allow-list strategies), the recommended OU structure (Security, Infrastructure, Workloads, Sandbox), real-world SCP patterns (deny root, restrict regions, enforce encryption), and how Organizations integrates with services like CloudTrail, Config, and RAM. This directly sets up tomorrow's Control Tower session and feeds into the Week 7 multi-account landing zone project where you will implement this architecture in Terraform.

## Resources

### 1. What is AWS Organizations? -- AWS Official Docs ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html
- **Type:** Official Docs
- **Summary:** The authoritative starting point that establishes the mental model for everything else. Covers the six core capabilities: centralized account management, OU-based grouping, policy-based governance (SCPs, tag policies, backup policies, AI services opt-out policies), delegated administration, resource sharing via RAM, and consolidated billing. Key concepts to internalize: the management account (formerly master account) is the only account that can create OUs and attach SCPs, but SCPs do not apply to the management account itself -- this is a critical security implication and a common interview question. Also introduces the concept of trusted access and delegated administrators, which let you push administrative tasks out of the management account to reduce its blast radius. Read this page and follow the "Terminology and concepts" link at the bottom to solidify vocabulary before moving on.

### 2. Organizing Your AWS Environment Using Multiple Accounts -- AWS Whitepaper ⏱️ 25 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** The definitive AWS whitepaper on multi-account strategy, updated April 2025. This is where you learn the "why" behind the recommended OU structure. Key sections to focus on: "Benefits of Using Multiple AWS Accounts" (security isolation, blast radius containment, billing separation, service quota independence), "Design Principles" (organize by security and operational needs, not org chart; apply policies to OUs not individual accounts; avoid deep nesting; keep the management account workload-free), and the recommended OU taxonomy. The whitepaper prescribes foundational OUs (Security OU with Log Archive and Security Tooling accounts; Infrastructure OU with Networking and Shared Services accounts) and workload OUs (Production, SDLC/Staging/Dev, Sandbox). It also covers the Suspended OU pattern for accounts pending closure and the Policy Staging OU for testing SCP changes safely. Read the first four sections deeply and skim the implementation section -- you will revisit that structure when you build the landing zone in Week 7.

### 3. SCP Evaluation -- AWS Official Docs ⏱️ 20 min
- **URL:** https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_evaluation.html
- **Type:** Official Docs
- **Summary:** The most important page for understanding how SCPs actually work at the evaluation level, complete with diagrams and seven progressive scenarios. The core mental model: Allow statements must exist at every level from root through each OU to the account (they are an intersection, not inherited), while Deny statements at any level block access for the entire subtree below (they are inherited). Default behavior: when you enable SCPs, AWS attaches a FullAWSAccess managed policy at every level -- this is the deny-list baseline where everything is allowed unless explicitly denied. If you remove FullAWSAccess without replacing it, all permissions are implicitly denied at that level and everything below it breaks. The seven scenarios walk through combinations of Allow and Deny at root, OU, and account levels, showing the effective permissions at each step. Work through all seven scenarios on paper, tracing the Allow intersection and Deny inheritance for each. This evaluation logic is the single most tested concept about Organizations in interviews.

### 4. How to Use Service Control Policies to Set Permission Guardrails -- AWS Security Blog ⏱️ 20 min
- **URL:** https://aws.amazon.com/blogs/security/how-to-use-service-control-policies-to-set-permission-guardrails-across-accounts-in-your-aws-organization/
- **Type:** Blog (AWS Security Blog)
- **Summary:** An AWS Security Blog post that bridges the gap between SCP theory and practical implementation. Covers the two SCP strategies in depth: the deny-list strategy (keep FullAWSAccess, layer on explicit Deny statements for what you want to block) versus the allow-list strategy (remove FullAWSAccess, create explicit Allow policies for only the services you want to permit). The deny-list approach is recommended for most organizations because it requires less maintenance -- you do not need to update policies every time AWS launches a new service. The allow-list approach is appropriate for highly regulated environments where only pre-approved services should be accessible. The article also covers the critical distinction that SCPs never grant permissions -- they only set the outer boundary of what IAM policies can grant. An IAM policy can grant ec2:RunInstances, but if the SCP does not allow EC2, the call is denied. This three-layer model (SCP boundary, IAM policy grant, resource-based policy) is essential for the Week 2 deep dive on IAM advanced patterns.

### 5. Best Practices for Organizational Units with AWS Organizations -- AWS Cloud Operations Blog ⏱️ 15 min
- **URL:** https://aws.amazon.com/blogs/mt/best-practices-for-organizational-units-with-aws-organizations/
- **Type:** Blog (AWS Cloud Operations Blog)
- **Summary:** A focused blog post from the AWS Management and Governance team on OU design patterns. Covers why OUs should be organized by function and compliance requirements rather than by business unit or team structure (because policies apply to OUs, not people). Walks through the recommended hierarchy: Security OU (Log Archive account for centralized CloudTrail and Config logs, Security Tooling account for GuardDuty delegated admin and Security Hub aggregation), Infrastructure OU (Networking account that owns Transit Gateway and shares it via RAM -- directly connects to your Week 1 TGW knowledge), Workloads OU split into Prod and Non-Prod sub-OUs, Sandbox OU with aggressive cost controls and auto-cleanup, and Suspended OU for accounts being decommissioned. Key insight: AWS recommends keeping OU depth to two levels maximum (root > OU > sub-OU) because deeper nesting makes SCP inheritance harder to reason about and debug.

### 6. SCP Examples Repository -- AWS Samples (GitHub) ⏱️ 15 min
- **URL:** https://github.com/aws-samples/service-control-policy-examples
- **Type:** Reference (GitHub Repository)
- **Summary:** The official AWS-maintained collection of production-ready SCP examples organized into seven security domains: data perimeter guardrails (enforce trusted identities and resources), deny changes to security services (prevent disabling GuardDuty, Config, CloudTrail, Security Hub), privileged access controls (restrict root user actions in member accounts, deny creation of IAM access keys for root), region controls (deny all API calls outside approved regions with exceptions for global services like IAM, STS, CloudFront, Route 53), sensitive data protection (prevent S3 public access, enforce encryption), service-specific controls, and protection of cloud platform resources (prevent deletion of VPC flow logs, CloudTrail trails). Browse the deny-changes-to-security-services and region-controls folders closely -- these are the exact SCP patterns you will implement in Terraform tomorrow and revisit in Week 7.

### 7. Best Practices for SCPs in a Multi-Account Environment -- AWS Industries Blog ⏱️ 10 min
- **URL:** https://aws.amazon.com/blogs/industries/best-practices-for-aws-organizations-service-control-policies-in-a-multi-account-environment/
- **Type:** Blog (AWS Industries Blog)
- **Summary:** A concise best practices article that covers the operational side of managing SCPs at scale. Key recommendations: always test SCPs in a Policy Staging OU before rolling out to production OUs; use the IAM policy simulator and IAM Access Analyzer to validate SCP impact before attachment; prefer explicit Deny statements over restrictive Allow statements because each Deny control works independently; use Conditions in Deny statements to create exceptions (for example, deny all actions except when performed by a specific break-glass role using aws:PrincipalArn); and leverage IAM service last accessed data to identify unused permissions before tightening SCPs. Also covers the practical limit: SCPs have a 5,120 character maximum size per policy (recently increased from 2,048), and you can attach up to 5 SCPs per OU or account -- so plan your policy structure to stay within these constraints.

## Study Tips

- **Trace the permission evaluation chain for every scenario.** For any given API call, ask: does the SCP at root Allow it? Does the SCP at the OU Allow it? Does the SCP at the account Allow it? Is there a Deny at any level? Only after all SCP levels pass does the IAM policy evaluation even begin. Draw the Organization tree on paper, attach sample SCPs at each level, and trace whether ec2:RunInstances would succeed or fail for a user in a deeply nested account. This exercise builds the intuition that interviewers test.

- **Compare SCPs to what you already know about Terraform and Kubernetes RBAC.** SCPs are like Terraform's `prevent_destroy` lifecycle rule or Kubernetes NetworkPolicies -- they are guardrails that limit what is possible regardless of what individual permissions allow. The key difference is that SCPs never grant permissions, they only restrict them. Think of SCPs as a filter that sits above IAM: IAM policies define what a user can do, SCPs define the ceiling of what anyone in that account can do. The effective permission is always the intersection of SCP and IAM.

- **Map the recommended OU structure to tonight's Terraform build.** Before writing code, sketch the Organization tree: Root > Security OU (Log Archive, Security Tooling) > Infrastructure OU (Networking, Shared Services) > Workloads OU > Prod sub-OU > Dev sub-OU > Sandbox OU. Then decide which SCPs attach where: deny-root at root level, region-restriction at root level, deny-leave-organization at root level, allow-only-approved-services at Prod sub-OU, relaxed permissions at Sandbox OU with budget alerts. This diagram becomes your Terraform implementation plan.

## Next Steps

After completing this study plan, consider exploring:

1. **Tonight's Terraform build** -- Create an AWS Organization with OUs in Terraform using the `aws_organizations_organization`, `aws_organizations_organizational_unit`, and `aws_organizations_policy` resources. Implement three SCPs: deny root user access in member accounts, restrict API calls to approved regions, and prevent disabling of CloudTrail.

2. **Control Tower and Landing Zones (Day 2, tomorrow)** -- How Control Tower automates the Organization setup you learned today, providing pre-configured guardrails (preventive via SCPs, detective via Config Rules), Account Factory for standardized account provisioning, and a landing zone with the Security and Infrastructure OUs already wired up.

3. **IAM Advanced Patterns (Day 3)** -- Permission boundaries are the IAM-level equivalent of SCPs: they set a ceiling on what an IAM role can do, just as SCPs set a ceiling on what an account can do. Understanding when to use SCPs (organization-wide, coarse-grained) versus permission boundaries (per-role, fine-grained) versus IAM policies (per-identity, specific grants) is a critical Week 2 concept.

4. **Week 7 multi-account landing zone project** -- Everything you learn this week feeds directly into the Week 7 build where you will design the full OU hierarchy, write production SCPs, configure centralized logging to the Log Archive account, and wire up Transit Gateway sharing via RAM across accounts.
