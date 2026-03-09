# AWS KMS & Encryption Deep Dive -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-09
**Difficulty:** Intermediate

## Overview

This study plan covers AWS Key Management Service (KMS) end-to-end: key types and hierarchy, envelope encryption mechanics, key policies and IAM integration, grants, cross-account access, key rotation, multi-region keys, CloudHSM comparison, and how KMS integrates with other AWS services. After completing it, you will understand how KMS authorization works (key policies + IAM + grants), why envelope encryption exists, and how to design key management strategies for multi-account environments -- all critical for Solutions Architect and Cloud Engineer interviews.

**Prerequisites you already have:** IAM policy evaluation (8 policy types, resource-based policies, cross-account AssumeRole, trust policies), AWS Organizations and SCPs, Security Hub and AWS Config. This plan builds directly on that IAM foundation -- KMS key policies are resource-based policies that follow the same evaluation logic you already know.

## Resources

### 1. AWS KMS Concepts -- KMS Keys (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html
- **Type:** Official Docs
- **Summary:** The foundational reference that explains the three key ownership models (customer managed, AWS managed, AWS owned), the internal key hierarchy (KMS key, HSM backing key, derived encryption key), key identifiers (key ID, key ARN, alias ARN, alias name), and how key material is structured inside KMS. Read this first to build the correct mental model of what a "KMS key" actually is before diving into policies and encryption mechanics.

### 2. AWS KMS Key Type Reference -- Symmetric, Asymmetric, HMAC (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/kms/latest/developerguide/symm-asymm-compare.html
- **Type:** Official Docs
- **Summary:** A comparison table of the three KMS key types with their supported operations, limitations, and feature availability. Key takeaways: symmetric keys (AES-256-GCM) are the default and only type that supports data key generation, automatic rotation, and custom key stores; asymmetric keys (RSA, ECC) support signing/verification and public-key encryption but cannot be auto-rotated; HMAC keys are for message authentication only. This page is the single source of truth for "which key type supports what" -- a common interview question.

### 3. KMS Cryptography Essentials -- Envelope Encryption (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/kms/latest/developerguide/kms-cryptography.html
- **Type:** Official Docs
- **Summary:** Explains the cryptographic foundations of KMS: AES-256-GCM for symmetric operations, the key derivation function used per-call, and most importantly the envelope encryption pattern with diagrams. This is where you learn WHY KMS does not encrypt your data directly (4 KB limit on direct encryption) and instead generates data keys that your application uses locally. Understanding the GenerateDataKey flow (plaintext data key + encrypted data key returned, plaintext used locally then discarded, encrypted data key stored alongside ciphertext) is essential for interviews.

### 4. KMS Access and Permissions -- Key Policies, IAM Policies, and Grants (Official Docs) -- 25 min
- **URL:** https://docs.aws.amazon.com/kms/latest/developerguide/control-access.html
- **Type:** Official Docs (start here, then follow links to Key Policies and Grants subpages)
- **Summary:** The authorization deep dive covering all three access mechanisms. Start with this overview page to understand how the three layers relate, then read the linked key policies page (https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html) to learn about the default key policy, why it enables IAM policies by granting kms permissions to the account root, and how the key policy is the single mandatory policy every KMS key must have. Then read the grants page (https://docs.aws.amazon.com/kms/latest/developerguide/grants.html) to understand grant tokens, eventual consistency, the 50,000-grant-per-key limit, and why AWS services use grants for temporary access. Since you already know IAM evaluation logic, focus on what makes KMS unique: key policies are resource-based policies, but unlike S3 bucket policies, KMS keys have NO default access -- even the account root has no permissions unless the key policy explicitly grants them.

### 5. Cross-Account KMS Access (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/kms/latest/developerguide/key-policy-modifying-external-accounts.html
- **Type:** Official Docs
- **Summary:** Step-by-step guide for the two-policy requirement for cross-account KMS access: (1) the key policy in the owning account must name the external account or specific principals, and (2) an IAM policy in the external account must grant the caller permission to use the key. This is directly analogous to the cross-account AssumeRole pattern you learned on Mar 5 (trust policy + identity policy), but with KMS-specific nuances: you can grant access to an entire external account or to specific roles, only certain operations work cross-account (no ListKeys or ScheduleKeyDeletion), and you must use key ARN (not alias) for cross-account references.

### 6. KMS Key Rotation (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html
- **Type:** Official Docs
- **Summary:** Covers the three rotation types: automatic (customizable 90-2560 day period, symmetric encryption keys only), on-demand (immediate rotation for symmetric keys including imported key material), and manual (create a new key and update references -- required for asymmetric and HMAC keys). The critical interview insight: rotation only changes the backing key material, not the logical KMS key -- the key ID, ARN, policies, and aliases all stay the same, and KMS automatically selects the correct key material version for decryption of older ciphertext.

### 7. Multi-Region Keys (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html
- **Type:** Official Docs
- **Summary:** Explains how multi-region keys share the same key ID and key material across regions while maintaining independent key policies, grants, and aliases per region. Covers the primary-replica relationship, use cases (DR decryption without cross-region calls, global signing, active-active applications), and limitations (cannot convert single-region to multi-region, AWS managed keys are always single-region, some services re-encrypt during cross-region operations like S3 CRR). This connects directly to the DR architecture you will study in Week 5.

### 8. Choosing an AWS Cryptography Service -- KMS vs CloudHSM Decision Guide (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/decision-guides/latest/cryptography-on-aws-how-to-choose/guide.html
- **Type:** Official Docs
- **Summary:** AWS's official decision framework comparing KMS, CloudHSM, Encryption SDK, Certificate Manager, Private CA, and Secrets Manager. For interview purposes, focus on the KMS vs CloudHSM comparison: KMS is multi-tenant (shared HSMs with logical separation), fully managed, integrates natively with 100+ AWS services, and now holds FIPS 140-2 Level 3 validation; CloudHSM is single-tenant (dedicated HSMs), gives you full control of key material, supports custom cryptographic algorithms, and is required when regulations mandate dedicated hardware. The decision usually comes down to: use KMS unless you need single-tenant HSMs, custom algorithms, or direct PKCS#11/JCE/CNG access.

## Study Tips

- **Connect KMS key policies to what you already know about IAM evaluation.** KMS key policies are resource-based policies (checkpoint 3 in the airport analogy from your Mar 5 doc). The unique twist: unlike S3 or SQS, KMS keys grant ZERO permissions by default -- the key policy must explicitly enable access, even for the owning account's root user. This is the single most important KMS concept for interviews.
- **Trace the envelope encryption flow end-to-end.** When studying Resource 3, mentally walk through this sequence: application calls GenerateDataKey, KMS returns plaintext data key + encrypted data key, application encrypts data locally with the plaintext key, discards the plaintext key from memory, stores encrypted data + encrypted data key together. On decryption: application sends encrypted data key to KMS, KMS returns plaintext data key, application decrypts data locally. Draw this flow on paper -- interviewers love asking you to explain it.
- **Build a comparison table as you read.** Create a quick-reference grid with rows for each key type (symmetric, asymmetric, HMAC) and columns for: supported operations, auto-rotation support, custom key store support, multi-region support, and imported key material support. This table will be invaluable during interview prep.

## Next Steps

- **AWS WAF, Shield, Network Firewall & Firewall Manager** -- the next topic in your Week 2 schedule, covering network and application layer protection services
- **Terraform KMS module** -- build a KMS customer managed key with a key policy, use it for S3 bucket encryption (SSE-KMS), and test cross-account access in your evening Terraform block
- **S3 Advanced Deep Dive (Week 4)** -- when you reach S3, you will see SSE-KMS, SSE-S3, and SSE-C encryption options in practice, connecting back to everything you learn today about key ownership models and envelope encryption
- **AWS KMS Best Practices (Prescriptive Guidance)** -- for deeper reading beyond this 2-hour plan, the full guide at https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-kms-best-practices/introduction.html covers centralized vs decentralized key management, ABAC for KMS, cost optimization, and monitoring strategies
