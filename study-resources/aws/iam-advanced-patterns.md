# IAM Advanced Patterns -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-05
**Difficulty:** Intermediate

## Overview

This study plan covers the advanced IAM mechanisms that sit between Organizations-level controls (SCPs, which you already know) and the actual permissions a principal receives: permission boundaries, session policies, cross-account AssumeRole patterns, SAML/OIDC federation, and IAM Identity Center. After completing it, you will understand how AWS evaluates all seven policy types together, when to use each restriction layer, and how to design identity federation for a multi-account organization.

## Prerequisites (Already Covered)

- AWS Organizations and SCPs (Mar 2) -- you understand the org hierarchy, deny-list vs allow-list SCPs, and how SCPs set the maximum permissions ceiling at the account/OU level
- Control Tower and Landing Zones (Mar 3) -- you understand preventive/detective/proactive controls, Account Factory, and IAM Identity Center at a high level
- EKS IRSA and Pod Identity (Dec 10) -- you have seen OIDC federation in practice for Kubernetes workloads

## Resources

### 1. Policies and Permissions in IAM (Official Docs Overview) -- 20 min
- **URL:** https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
- **Type:** Official Docs
- **Summary:** The single most important reference page for this topic. It defines all seven IAM policy types (identity-based, resource-based, permission boundaries, SCPs, RCPs, ACLs, session policies), explains how they interact, and establishes the mental model of "intersection vs union" that governs all IAM evaluation. Read this first to build the conceptual framework that every other resource builds on. Pay special attention to the table showing which policy types grant permissions versus which only restrict them.

### 2. Policy Evaluation Logic (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- **Type:** Official Docs
- **Summary:** Explains the three-step evaluation process (authenticate, process request context, evaluate policies) and provides the critical flowcharts showing how identity-based policies, resource-based policies, permission boundaries, SCPs, and session policies interact within a single account. Also follow the link to the cross-account evaluation logic page (https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html) -- cross-account evaluation works differently because resource-based policies alone can grant access across accounts without requiring an identity-based Allow. Understanding these flowcharts is essential for tomorrow's security services topic and for interview scenarios.

### 3. Permissions Boundaries for IAM Entities (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html
- **Type:** Official Docs
- **Summary:** The definitive reference on permission boundaries. Includes the delegation scenario where a central admin (Maria) empowers a developer (Zhang) to create IAM users but only within the guardrails of a pre-defined boundary. Contains four JSON policy examples showing the boundary policy, the delegator's own boundary, and the delegator's permissions policy. The diagrams showing effective permissions as the intersection of identity-based policies and boundaries are critical for building intuition. Focus on the section explaining how resource-based policies interact differently with boundaries depending on whether the principal is a user, role, or federated session.

### 4. When and Where to Use IAM Permissions Boundaries (AWS Security Blog) -- 15 min
- **URL:** https://aws.amazon.com/blogs/security/when-and-where-to-use-iam-permissions-boundaries/
- **Type:** Blog (Official AWS)
- **Summary:** A practical complement to the docs. This post explains the real-world "why" behind permission boundaries: they solve the bottleneck of a centralized IAM team handling every role-creation request. It walks through the pattern of allowing developers to self-service IAM role creation while the security team maintains a boundary that prevents privilege escalation. Covers the confused deputy problem and how boundaries prevent developers from creating roles more powerful than their own. Read this after the docs to solidify the practical use cases.

### 5. IAM Tutorial: Cross-Account Access with Roles (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html
- **Type:** Official Docs (Tutorial)
- **Summary:** A step-by-step walkthrough of the foundational cross-account pattern: creating a role in the destination account with a trust policy that allows principals from the originating account to assume it, then granting sts:AssumeRole in the originating account. Covers all three access methods (console role-switching, CLI with sts assume-role, and programmatic API calls). The tutorial uses a concrete S3 bucket sharing scenario. Since you will be building cross-account IAM roles in Terraform tonight, this tutorial gives you the exact mechanics.

### 6. AWS Trust Policy Complete Guide (Dev.to / AWS Builders) -- 15 min
- **URL:** https://dev.to/aws-builders/aws-trust-policy-complete-guide-how-to-control-iam-role-access-in-2025-cfi
- **Type:** Blog (Community -- AWS Builders)
- **Summary:** Expands on the cross-account tutorial with a focus on trust policy patterns for different principal types: Service principals (for AWS services), AWS principals (for cross-account), and Federated principals (for SAML and OIDC). Includes six JSON policy examples covering external IDs for third-party access, MFA requirements, source IP restrictions, and condition-based controls. The "permission policy vs trust policy" comparison table is a useful reference. Read this to understand the full range of trust policy patterns beyond the basic cross-account scenario.

### 7. Identity Providers and Federation (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers.html
- **Type:** Official Docs
- **Summary:** The official overview of SAML 2.0 and OIDC federation with IAM. Explains how external identity providers (Okta, Azure AD, Google) issue tokens that are exchanged for temporary AWS credentials via STS (AssumeRoleWithSAML or AssumeRoleWithWebIdentity). Since you already understand OIDC from EKS IRSA, this page fills in the SAML side and shows how both federation types fit into the broader IAM architecture. Skim the SAML setup process and focus on understanding when to use direct IAM federation versus IAM Identity Center (the answer: prefer Identity Center for workforce access; use direct federation only for legacy or specialized scenarios).

### 8. What is IAM Identity Center? (Official Docs) -- 5 min
- **URL:** https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html
- **Type:** Official Docs
- **Summary:** The official landing page for IAM Identity Center (formerly AWS SSO). Read this as a quick capstone that ties together everything from this session. You already know Identity Center conceptually from the Control Tower doc -- this page adds the detail on permission sets (templates that auto-create IAM roles in target accounts), identity source options (Identity Center directory, Active Directory, external SAML IdP), and the AWS access portal. Follow the link to the permission sets concept page (https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html) if you have time -- it explains how permission sets are templates that generate IAM roles behind the scenes, which connects directly to the cross-account and trust policy patterns you studied earlier.

## Study Tips

- **Draw the evaluation flowchart by hand.** After reading resources 1 and 2, sketch the IAM policy evaluation flow from memory: request arrives, check for explicit deny across all policy types, check SCPs, check resource-based policies, check permission boundaries, check session policies, check identity-based policies. Being able to reproduce this flowchart from memory is the single most valuable thing you can do for both interviews and real-world debugging.

- **Map the restriction layers to what you already know.** You have a strong mental model from Organizations/SCPs. Think of the full stack as: SCPs (account ceiling) > Permission Boundaries (principal ceiling) > Session Policies (session ceiling) > Identity-Based Policies (actual grant). Each layer can only restrict, never expand, what the layer above allows. The effective permission is the intersection of all four plus any resource-based policy grants.

- **Focus on the "when to use which" decision.** The interview question is rarely "what is a permission boundary?" -- it is "your team needs to let developers create their own IAM roles without risking privilege escalation; what do you use?" (Permission boundary.) "You need to restrict a third-party vendor's access to only their S3 prefix during an assumed role session?" (Session policy.) "You need to prevent any account in the Sandbox OU from using iam:CreateUser?" (SCP.) Build this decision matrix as you read.

## Next Steps

- **Tomorrow: Security Services (GuardDuty, Security Hub, Config, Inspector)** -- these services detect and alert on IAM misconfigurations. Understanding today's IAM evaluation logic will help you understand what Config Rules and Security Hub controls are actually checking for.
- **Friday: KMS and Encryption** -- KMS key policies are resource-based policies with their own evaluation logic that interacts with IAM policies. Today's policy evaluation framework applies directly.
- **Tonight's Terraform build: Cross-account IAM roles + permission boundaries** -- implement the patterns from resources 3-6 in code. Create a role in a target account with a trust policy, grant AssumeRole in the source account, and attach a permission boundary that prevents the role from modifying its own boundary or creating roles without boundaries.
