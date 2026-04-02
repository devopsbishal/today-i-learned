# AWS Backup & Centralized Protection -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-01
**Difficulty:** Intermediate

## Overview

This study plan covers AWS Backup as the centralized backup orchestration layer across 20+ AWS services, including backup plans, vault types (standard, locked, logically air-gapped), cross-region and cross-account protection with AWS Organizations, Audit Manager for compliance, and restore testing for recovery validation. After completing this plan, you will understand how to design and implement an enterprise-grade backup strategy that goes beyond individual service-native backups.

**Prerequisites already covered:** DR strategies (Backup & Restore, Pilot Light, Warm Standby, Multi-Site Active-Active), Multi-Region Architecture Patterns, RDS/Aurora backups and snapshots, DynamoDB PITR and Global Tables, S3 versioning and CRR, EBS snapshots, EFS replication, FSx backups, KMS encryption, AWS Organizations and SCPs. This session builds directly on that foundation -- focusing on AWS Backup as the centralized orchestration layer rather than re-explaining individual service backup mechanisms or DR strategy selection.

## Resources

### 1. What is AWS Backup? (Official Docs Overview) -- 15 min
- **URL:** https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html
- **Type:** Official Docs
- **Summary:** The definitive starting point -- covers what AWS Backup is, its core features (centralized management, policy-based backup, lifecycle policies, cross-region/cross-account copies, incremental backups, encryption), and the full list of 20+ supported resource types. Since you already understand each service's native backup mechanism, focus on how AWS Backup unifies them under a single control plane with consistent scheduling, retention, and monitoring.

### 2. AWS Backup Feature Availability by Resource Type -- 10 min
- **URL:** https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html
- **Type:** Official Docs (Reference Matrix)
- **Summary:** A critical reference matrix showing which features (cross-region copy, cross-account copy, continuous backup/PITR, cold storage lifecycle, restore testing, logically air-gapped vault, malware protection) are available for each resource type. Not every service supports every feature -- for example, continuous backup is only available for S3, RDS, Aurora, and DynamoDB. Skim the matrix to build a mental model of capability boundaries, then bookmark for future reference.

### 3. Backup and Recovery using AWS Backup (Prescriptive Guidance) -- 20 min
- **URL:** https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/aws-backup.html
- **Type:** Official Docs (Prescriptive Guidance)
- **Summary:** AWS Prescriptive Guidance on designing backup solutions with AWS Backup -- covers tag-based backup plan assignment, organization-wide policy management, SCP-based vault protection, and the relationship between AWS Backup and service-native backups. This bridges the gap between "what the service does" (Resource 1) and "how to architect with it." Read this after the overview to understand the design patterns before diving into specific features.

### 4. Managing AWS Backup Across Multiple Accounts (Cross-Account Management) -- 20 min
- **URL:** https://docs.aws.amazon.com/aws-backup/latest/devguide/manage-cross-account.html
- **Type:** Official Docs
- **Summary:** Covers the enterprise backbone of AWS Backup -- cross-account management via AWS Organizations. Explains backup policies (Organization-level policies that deploy backup plans to member accounts via OUs), delegated administrator accounts (up to 5), cross-account monitoring from a central account, cross-account backup copies with vault access policies, and how organization-level resource opt-in rules override member account settings. Since you already understand Organizations, OUs, and SCPs, this will click quickly -- focus on the policy merging behavior and how backup policies inherit/override across the OU hierarchy.

### 5. AWS Backup Vault Lock (WORM Compliance) -- 15 min
- **URL:** https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html
- **Type:** Official Docs
- **Summary:** Deep dive into Vault Lock's two modes -- Governance mode (removable by privileged IAM users, no cooling-off period) vs Compliance mode (immutable after a mandatory 3-day minimum grace/cooling-off period, cannot be removed by anyone including root or AWS). Covers min/max retention period enforcement, CLI configuration, and the critical distinction that once Compliance mode's grace period expires, the vault lock is permanent and irreversible. You already know S3 Object Lock (Governance vs Compliance) from your S3 deep dive -- Vault Lock follows the exact same two-mode pattern.

### 6. Logically Air-Gapped Vaults -- 15 min
- **URL:** https://docs.aws.amazon.com/aws-backup/latest/devguide/logicallyairgappedvault.html
- **Type:** Official Docs
- **Summary:** The newest and most secure vault type -- backups are stored in an AWS service-owned account (logically isolated from your account), always have Compliance-mode Vault Lock enabled by default, and can be shared cross-account via AWS RAM for recovery. Covers creation (CLI/console), encryption options (AWS-owned keys by default, or customer-managed KMS keys), cross-account sharing for restore, and how this compares to standard vaults with Vault Lock. This is AWS's answer to ransomware protection -- even if an attacker compromises your account and deletes all resources, the air-gapped vault in the service account survives.

### 7. Restore Testing for Recovery Validation -- 10 min
- **URL:** https://docs.aws.amazon.com/aws-backup/latest/devguide/restore-testing.html
- **Type:** Official Docs
- **Summary:** Covers the automated restore testing feature that validates your backups actually work -- because untested backups are Schrodinger's backups (a concept from your DR strategies doc). Explains restore testing plans (scheduled frequency, recovery point selection, resource assignment), automatic resource cleanup after validation, optional validation windows (1-168 hours) for running custom checks before cleanup, and supported resource types (Aurora, DynamoDB, EBS, EC2, EFS, FSx, RDS, S3, and more). Best practice: run restore tests in a dedicated test account.

### 8. AWS Backup Audit Manager (Compliance Frameworks) -- 15 min
- **URL:** https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-audit-manager.html
- **Type:** Official Docs
- **Summary:** Covers the compliance auditing layer -- create frameworks with controls that continuously evaluate whether your resources meet backup requirements (e.g., "Are all EC2 instances backed up?", "Are all backups encrypted?", "Do backups meet minimum frequency?"). Explains the three-step workflow (create frameworks with control templates, view compliance status, generate daily reports to S3), AWS Config prerequisite, and integration with AWS Audit Manager for broader compliance reporting. After reading the overview, skim the linked "Controls and remediation" page for the specific control templates available.

## Study Tips

- **Map to what you know:** You have already studied each service's native backup mechanism (RDS automated snapshots, DynamoDB PITR, EBS snapshots, S3 versioning/CRR, EFS replication). As you read, focus on what AWS Backup adds on top -- centralized scheduling, cross-account copies, Vault Lock, audit compliance -- rather than re-learning the per-service mechanics.
- **Think in three vault tiers:** Standard vault (basic, deletable), Vault Lock vault (governance or compliance WORM), and Logically Air-Gapped vault (service-account isolated, always compliance-locked). Every exam and architecture question about backup security maps to choosing the right tier.
- **Cross-account is the enterprise differentiator:** The most architecturally significant capability of AWS Backup is not the backup itself (services already do that natively) -- it is the centralized cross-account policy enforcement via Organizations. Pay special attention to how backup policies flow through the OU hierarchy and how delegated administrators reduce management account blast radius.

## Next Steps

- **Database DR Patterns** (next topic in syllabus) -- Apply AWS Backup knowledge to RDS automated backups vs manual snapshots vs AWS Backup-managed snapshots, Aurora Global Database failover procedures, and cross-region read replica promotion patterns.
- **Storage DR & Data Replication** -- Connect AWS Backup cross-region copies with S3 CRR/SRR, DynamoDB Global Tables, and EFS cross-region replication for a complete data protection strategy.
- **Terraform implementation** -- Build an AWS Backup plan with cross-region copy rules, a Vault Lock-protected vault, tag-based resource selection, and Organization backup policies (reference the `aws_backup_plan`, `aws_backup_vault`, `aws_backup_vault_lock_configuration`, and `aws_backup_selection` resources from your DR strategies Terraform example).
- **Cost modeling** -- AWS Backup pricing is consumption-based (per-GB-month storage varying by resource type, restore fees, cross-region transfer). Key optimization levers: cold storage lifecycle for long-term retention (~75% savings), incremental backups, S3 backup tiering (up to 30% savings launched Nov 2025), and avoiding redundant backup copies when service-native backups already provide sufficient RPO.
