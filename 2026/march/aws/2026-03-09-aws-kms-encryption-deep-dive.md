# AWS KMS & Encryption Deep Dive -- The Locksmith Shop Analogy

> You have wired the airport security checkpoints (IAM), connected the surveillance cameras (GuardDuty), hired the building inspectors (Config), and centralized the operations center (Security Hub). But every one of those systems protects the **perimeter and posture of your environment** -- who can enter, what they can do, whether the configuration is correct. None of them answer the most fundamental security question: **if someone steals the hard drive, can they read what is on it?** That is the domain of encryption, and AWS Key Management Service (KMS) is the locksmith shop at the center of it all. KMS does not encrypt your data directly -- it manages the keys that encrypt your data, and it does so through a pattern called envelope encryption that every AWS architect must understand cold.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| KMS key (formerly called Customer Master Key or CMK) | A master key locked inside a vault that never leaves the vault | A logical resource in KMS backed by cryptographic material inside FIPS-validated HSMs; the key material never leaves KMS unencrypted |
| Customer managed key | A custom master key you commissioned from the locksmith | A KMS key you create, own, and control -- you set the key policy, rotation schedule, and can disable or delete it |
| AWS managed key | A master key the locksmith made for you when you moved in | A KMS key created automatically by an AWS service (alias `aws/s3`, `aws/ebs`, etc.) -- AWS manages the policy and rotation, you cannot change them |
| AWS owned key | A master key the locksmith uses in the back room that you never see | A KMS key that AWS owns and uses internally to protect your data in some services; it does not appear in your account, you cannot audit or control it |
| Symmetric key (AES-256-GCM) | A single key that both locks and unlocks the same lock | The default KMS key type -- one key for both encryption and decryption; required for envelope encryption, auto-rotation, and custom key stores |
| Asymmetric key (RSA/ECC) | A mailbox with a slot (public key) and a key to open it (private key) | A KMS key with a public/private pair; the public key is downloadable for encryption or signature verification outside AWS |
| HMAC key | A wax seal stamped with your family crest on a letter | A KMS key that generates and verifies message authentication codes -- proves data integrity (seal is unbroken) and authenticity (only the holder of the signet ring could have stamped it) without encryption |
| Key policy | The access list posted on the vault door | A resource-based policy attached to every KMS key that is the **primary** access control mechanism; unlike S3 or SQS, even the account root has zero access unless the key policy grants it |
| Grants | A temporary signed note authorizing a specific locksmith action | A delegated, scoped permission that AWS services use for temporary access to KMS keys; eventually consistent, revocable, limited to specific operations |
| Envelope encryption | Locking your data with a disposable padlock, then locking that padlock's key inside the vault | KMS generates a data key, you encrypt data locally with it, discard the plaintext data key, and store the encrypted data key alongside the ciphertext -- KMS never sees your data |
| GenerateDataKey | Asking the locksmith to cut a new padlock and give you both the padlock and a vault-sealed copy of the padlock key | Returns both a plaintext data key (use it, then discard) and an encrypted copy of that data key (store alongside ciphertext) |
| Key rotation (automatic) | The locksmith periodically re-cuts the master key but keeps every old version on the ring | KMS creates new backing key material on schedule (90-2560 days) while retaining all previous versions; the key ID, ARN, policies, and aliases stay the same |
| Multi-Region keys | Duplicate master keys installed in branch offices, all cut from the same mold | A primary key replicated to other regions with identical key material; ciphertext encrypted in one region can be decrypted in another without cross-region API calls |
| Cross-account KMS access | Two vaults in different buildings that must both authorize a visitor | Requires the key policy in the owning account to name the external account AND an IAM policy in the external account to grant the caller permission -- the same two-sided pattern you learned for cross-account AssumeRole |
| CloudHSM | Your own private locksmith workbench inside the vault, with no one else allowed in the room | Single-tenant, dedicated HSMs under your exclusive control; FIPS 140-3 Level 3 validated; you manage the keys, AWS manages the hardware |
| Custom key store | A KMS storefront that routes all key operations to your private CloudHSM workbench | Connects KMS to your CloudHSM cluster so you get KMS API convenience with CloudHSM-level isolation |
| Key identifiers | Four ways to address a letter to the locksmith | Key ID (short UUID), Key ARN (full resource path), Alias name (`alias/my-key`), Alias ARN -- cross-account operations require Key ARN |

---

## The Big Picture

In the IAM doc, you learned that KMS key policies are **resource-based policies** -- checkpoint 3 in the airport security analogy. You know that resource-based policies can grant same-account access without an identity policy. But KMS has a twist that makes it unique among all AWS services: **a KMS key grants zero permissions by default, even to the account root principal.** Unlike an S3 bucket where the account root implicitly has access, a KMS key with an empty key policy is a locked door with no handles on either side. The key policy is not optional decoration -- it is the only way in.

Think of your AWS environment as a city with a **central locksmith shop** (KMS). Every business in the city (AWS service) that needs to protect valuables (data) sends their valuables to the locksmith to be locked up. But the locksmith does not actually lock your valuables directly -- that would require shipping everything to the shop, which is slow and creates a bottleneck (KMS has a 4 KB limit on direct encryption). Instead, the locksmith uses a clever system:

1. You ask the locksmith for a **disposable padlock** (data key)
2. The locksmith cuts a new padlock and gives you **two things**: the padlock itself (plaintext data key) and a **vault-sealed envelope** containing a copy of the padlock's key (encrypted data key)
3. You lock your valuables with the padlock at your own premises (encrypt data locally)
4. You **destroy the padlock** (discard the plaintext data key from memory)
5. You store the vault-sealed envelope next to your locked valuables (encrypted data key alongside ciphertext)
6. Later, when you need your valuables, you send the sealed envelope back to the locksmith, who opens it inside the vault and hands you the padlock key (KMS decrypts the data key), and you unlock your valuables locally

This is **envelope encryption** -- the core pattern behind every KMS integration in AWS. The locksmith (KMS) never sees your valuables (data). The locksmith only ever handles the padlock keys (data keys). This separation is what makes KMS scalable, secure, and fast.

```
ENVELOPE ENCRYPTION -- THE CORE PATTERN
=============================================================================

  ENCRYPTION FLOW
  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  YOUR APPLICATION                         KMS (Locksmith Vault)    │
  │                                                                     │
  │  Step 1: "I need a data key"                                       │
  │          ─────────────────────────────▶  GenerateDataKey(KeyId)     │
  │                                                                     │
  │  Step 2: KMS returns TWO things:                                   │
  │          ◄─────────────────────────────                             │
  │          ┌─────────────────────┐  ┌──────────────────────────┐     │
  │          │ Plaintext data key  │  │ Encrypted data key       │     │
  │          │ (the padlock)       │  │ (vault-sealed envelope)  │     │
  │          └────────┬────────────┘  └──────────┬───────────────┘     │
  │                   │                          │                      │
  │  Step 3: Encrypt data locally               │                      │
  │          with plaintext data key             │                      │
  │          ┌───────────────────┐               │                      │
  │          │ Your Data         │               │                      │
  │          │ "Hello World"     │               │                      │
  │          └───────┬───────────┘               │                      │
  │                  │ AES-256-GCM               │                      │
  │                  ▼                           │                      │
  │          ┌───────────────────┐               │                      │
  │          │ Encrypted Data    │               │                      │
  │          │ 0x7f3a9b...       │               │                      │
  │          └───────┬───────────┘               │                      │
  │                  │                           │                      │
  │  Step 4: DISCARD plaintext data key from memory                    │
  │                                                                     │
  │  Step 5: Store together:                                           │
  │          ┌───────────────────────────────────────────┐              │
  │          │  Encrypted Data  +  Encrypted Data Key    │              │
  │          │  0x7f3a9b...        (vault-sealed envelope)│              │
  │          └───────────────────────────────────────────┘              │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘

  DECRYPTION FLOW
  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  Step 1: Send encrypted data key to KMS                            │
  │          ─────────────────────────────▶  Decrypt(CiphertextBlob)    │
  │                                                                     │
  │  Step 2: KMS decrypts the data key inside the HSM                  │
  │          ◄─────────────────────────────                             │
  │          ┌─────────────────────┐                                    │
  │          │ Plaintext data key  │                                    │
  │          └────────┬────────────┘                                    │
  │                   │                                                 │
  │  Step 3: Decrypt data locally with plaintext data key              │
  │          ┌───────────────────┐                                      │
  │          │ "Hello World"     │                                      │
  │          └───────────────────┘                                      │
  │                                                                     │
  │  Step 4: DISCARD plaintext data key from memory                    │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘

  WHY THIS PATTERN?
  • KMS never sees your data -- only the data key
  • No 4 KB size limit on your data (you encrypt locally)
  • Data key operations are fast (local AES-256-GCM)
  • Only the data key wrapping/unwrapping hits KMS (low latency)
  • If KMS is unavailable, cached plaintext data keys still work
    (this is how EBS volumes stay readable during brief KMS outages)
```

---

## Part 1: Key Ownership Models -- Who Owns the Locksmith's Keys?

Not all keys in KMS are created equal. There are three ownership tiers, and the distinction matters for control, auditability, and cost.

### Customer Managed Keys -- Your Custom Master Key

**The Analogy**: You walked into the locksmith shop and commissioned a custom master key. You specified the lock type (symmetric, asymmetric, or HMAC). You wrote the access list for who can use it (key policy). You decide when to rotate it, whether to disable it, and eventually when to destroy it. The locksmith stores it in the vault and does what you say.

**The Technical Reality**: A customer managed key (CMK) is a KMS key that you create explicitly via the console, CLI, API, or Terraform. You have full control over:
- The key policy (who can use and manage the key)
- Key rotation schedule (automatic every 90-2560 days, on-demand, or manual)
- Enabling and disabling the key
- Tagging (for ABAC-based access control and cost allocation)
- Scheduling deletion (7-30 day waiting period)
- Granting cross-account access

**Cost**: $1/month per key + $0.03 per 10,000 API requests. There is no free tier for key storage.

```hcl
# Terraform: Create a customer managed KMS key with a key policy
resource "aws_kms_key" "application_data" {
  description             = "Encrypts application data in the production account"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  rotation_period_in_days = 180  # Rotate every 180 days (range: 90-2560)
  multi_region            = false

  # Key policy -- THE most important part. This is the vault door access list.
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Statement 1: Enable IAM policies in this account to grant KMS access.
      # WITHOUT this statement, no IAM policy in the account can grant access
      # to this key -- not even policies attached to the account root.
      {
        Sid    = "EnableIAMPolicies"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::111111111111:root"
        }
        Action   = "kms:*"
        Resource = "*"  # "*" in a key policy means "this key" (self-referential)
      },
      # Statement 2: Key administrators -- can manage but NOT use the key
      {
        Sid    = "AllowKeyAdministration"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::111111111111:role/KeyAdminRole"
        }
        Action = [
          "kms:Create*",
          "kms:Describe*",
          "kms:Enable*",
          "kms:List*",
          "kms:Put*",
          "kms:Update*",
          "kms:Revoke*",
          "kms:Disable*",
          "kms:Get*",
          "kms:Delete*",
          "kms:TagResource",
          "kms:UntagResource",
          "kms:ScheduleKeyDeletion",
          "kms:CancelKeyDeletion"
        ]
        Resource = "*"
      },
      # Statement 3: Key users -- can use the key for crypto operations
      {
        Sid    = "AllowKeyUsage"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::111111111111:role/AppServiceRole",
            "arn:aws:iam::111111111111:role/LambdaExecutionRole"
          ]
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey",
          "kms:GenerateDataKeyWithoutPlaintext",
          "kms:DescribeKey"
        ]
        Resource = "*"
      },
      # Statement 4: Allow AWS services to create grants on behalf of key users.
      # This is how S3, EBS, RDS, etc. get temporary access to use the key.
      {
        Sid    = "AllowGrantsForAWSServices"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::111111111111:role/AppServiceRole"
          ]
        }
        Action = [
          "kms:CreateGrant",
          "kms:ListGrants",
          "kms:RevokeGrant"
        ]
        Resource = "*"
        Condition = {
          Bool = {
            "kms:GrantIsForAWSResource" = "true"
          }
        }
      }
    ]
  })

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    Purpose     = "application-data-encryption"
  }
}

# Create a human-readable alias
resource "aws_kms_alias" "application_data" {
  name          = "alias/prod-application-data"
  target_key_id = aws_kms_key.application_data.key_id
}
```

### AWS Managed Keys -- The Pre-Made Key

**The Analogy**: When you moved into the building (started using an AWS service), the locksmith automatically made a master key for you and labeled it with your suite number (`aws/s3`, `aws/ebs`, `aws/rds`). You can use it, but you cannot change the lock, the access list, or the rotation schedule. It is convenient but rigid.

**The Technical Reality**: AWS managed keys are created automatically when an AWS service first needs encryption in your account. They are identifiable by their `aws/service-name` alias format. Key differences from customer managed keys:

| Feature | Customer Managed | AWS Managed |
|---------|-----------------|-------------|
| Key policy control | Full control | AWS manages (you cannot change it) |
| Rotation | Configurable (90-2560 days) or on-demand | Automatic every year (not configurable) |
| Cross-account access | Yes (via key policy) | No -- cannot be used from another account |
| Deletion | You can schedule deletion | Cannot be deleted |
| Cost | $1/month + API calls | No monthly charge; API calls still apply |
| CloudTrail visibility | Full audit trail | Full audit trail |
| Grant management | Full control | AWS services manage grants |

> **Interview tip**: "When would you choose a customer managed key over an AWS managed key?" The answer is whenever you need cross-account access, custom key policies, custom rotation schedules, ABAC-based access control, the ability to disable the key, or when you want to share one key across multiple services. In practice, production environments almost always use customer managed keys for auditability and control.

### AWS Owned Keys -- The Invisible Key

**The Analogy**: The locksmith has a set of master keys in the back room that you never see, never audit, and never control. When some services encrypt your data "for free" without any KMS charges, they use these invisible keys. You are protected, but you have no visibility into the cryptographic operations.

**The Technical Reality**: AWS owned keys belong to AWS, not your account. They do not appear in your KMS console, you cannot view them in CloudTrail, and you cannot manage them. Services like DynamoDB (when not using SSE-KMS), SQS default encryption, and some internal service operations use AWS owned keys. The trade-off: zero cost, zero control.

```
KEY OWNERSHIP COMPARISON
=============================================================================

  Feature              Customer Managed    AWS Managed         AWS Owned
  ─────────────────    ─────────────────   ─────────────────   ───────────
  Created by           You                 AWS service         AWS (internal)
  Key policy           You control         AWS controls        N/A
  Visible in console   Yes                 Yes                 No
  CloudTrail logs      Yes                 Yes                 No
  Cross-account        Yes                 No                  No
  Rotation control     Yes (90-2560 days)  No (yearly, fixed)  N/A
  Can disable/delete   Yes                 No                  No
  Monthly cost         $1/key              Free                Free
  API call cost        $0.03/10K           $0.03/10K           Free
  Use case             Production data     Quick start/dev     Background encryption
```

---

## Part 2: Key Types -- What Kind of Lock Do You Need?

### Symmetric Keys (AES-256-GCM) -- The Default

**The Analogy**: A single key that both locks and unlocks the same padlock. Both the sender and the receiver use the same key. The locksmith (KMS) keeps this key inside the vault -- it never comes out. When you need to use it, you either send small data to the vault for direct encryption (up to 4 KB) or ask the vault for a disposable padlock (data key) that works with the same mechanism.

**The Technical Reality**: Symmetric encryption keys use AES-256-GCM, the gold standard for symmetric encryption. This is the default and most commonly used key type in KMS. The raw key material never leaves the HSM boundary.

Key characteristics:
- **256-bit AES key with GCM mode** (authenticated encryption -- provides both confidentiality and integrity)
- **Required for envelope encryption** -- only symmetric keys support `GenerateDataKey` and `GenerateDataKeyWithoutPlaintext`
- **Required for automatic key rotation**
- **Required for custom key stores** (CloudHSM-backed) -- this is the only symmetric-exclusive feature among these
- **Supported for multi-region key replication** (all key types -- symmetric, asymmetric, and HMAC -- support multi-region)
- The plaintext key material **never leaves KMS** -- you send data in or request data keys out

### Asymmetric Keys (RSA / ECC) -- The Mailbox

**The Analogy**: A mailbox with a slot that anyone can push mail into (public key) and a locked compartment that only the owner can open (private key). Unlike the single-key padlock, the person encrypting the data does not need access to KMS at all -- they just need the public key, which is freely downloadable. This is perfect for external parties who need to send you encrypted data but should not have AWS credentials.

**The Technical Reality**: Asymmetric keys consist of a mathematically linked public-private key pair. KMS keeps the private key inside the HSM; you can download the public key for use outside AWS.

Two use cases:
1. **Encrypt/Decrypt**: External party encrypts with the public key, you decrypt via KMS (RSA only)
2. **Sign/Verify**: KMS signs with the private key, anyone verifies with the public key (RSA and ECC)

| Algorithm | Key Specs | Use Case |
|-----------|-----------|----------|
| **RSA** | RSA_2048, RSA_3072, RSA_4096 | Encryption/decryption OR signing/verification (choose one per key) |
| **ECC** | ECC_NIST_P256, ECC_NIST_P384, ECC_NIST_P521, ECC_SECG_P256K1 | Signing/verification only (smaller keys, faster operations than RSA for signing) |
| **SM2** | SM2 (China Regions only) | Encryption/decryption, signing/verification, key agreement |

Limitations:
- Cannot be used for envelope encryption (no `GenerateDataKey`)
- **Cannot be automatically rotated** -- you must do manual rotation
- Cannot be used in custom key stores
- **Supports imported key material** (RSA, ECC private keys) since 2023, but imported asymmetric keys can only ever have one key material version (you can reimport the same material but not different material)

### HMAC Keys -- The Wax Seal

**The Analogy**: A wax seal stamped with your family crest on a letter. It does not hide the contents (no encryption) but it proves two things: the letter has not been tampered with since the seal was applied (integrity -- the seal is unbroken), and it came from someone who holds the family signet ring (authenticity -- only the secret key holder could have generated the MAC). Unlike a simple tamper-evident sticker anyone could apply, only the holder of the ring can create the seal.

**The Technical Reality**: HMAC keys generate and verify Hash-based Message Authentication Codes. They do not encrypt data -- they produce a fixed-length tag that proves a message has not been tampered with.

- Supported specs: HMAC_224, HMAC_256, HMAC_384, HMAC_512
- Operations: `GenerateMac` and `VerifyMac`
- Use case: API request signing, token validation, message integrity verification
- Advantage over doing HMAC locally: the secret key never leaves KMS, providing stronger security guarantees

```
KEY TYPE COMPARISON
=============================================================================

  Feature                 Symmetric         Asymmetric        HMAC
  ─────────────────────   ─────────────     ─────────────     ─────────────
  Algorithm               AES-256-GCM       RSA / ECC         HMAC-SHA
  Envelope encryption     YES               No                No
  Direct encrypt/decrypt  Yes (4 KB max)    Yes (varies)      No
  Signing                 No                Yes               MAC generation
  Auto-rotation           YES               No                No
  Manual rotation         Yes               Yes (required)    Yes (required)
  Custom key store        YES               No                No
  Multi-Region            YES               Yes               Yes
  Import key material     Yes               Yes (since 2023)  Yes (since 2023)
  Public key download     No                YES               No
  Use outside AWS         No (key in HSM)   Yes (public key)  No (key in HSM)
  Default key type        YES               No                No
```

---

## Part 3: Key Policies and IAM Integration -- The Vault Door Access List

This is the single most important KMS concept for interviews: **KMS key policies work differently from every other AWS resource-based policy.**

### The Unique KMS Authorization Model

**The Analogy**: In most buildings in the city (S3, SQS, Lambda), the building owner (account root) has a master key that works on every door, even if the door has no explicit access list. But the locksmith shop is different. The vault door (KMS key) has **no master key override**. If the access list on the vault door does not name you, you cannot enter -- period. Even the building owner (account root) is locked out unless the access list says otherwise.

**The Technical Reality**: Every KMS key must have exactly one key policy. Unlike S3 bucket policies (which are optional -- the account root can always access the bucket regardless), KMS keys start with **zero access**. The key policy is the primary authorization mechanism. There are three ways to grant access to a KMS key, and they can work independently or together:

1. **Key policy alone** -- the key policy directly names the principal and grants access. Use this for cross-account access to specific principals
2. **Key policy + IAM policy** -- the key policy enables IAM policies (by granting access to the account root), and then IAM policies grant access to specific principals. **This is the standard pattern** because it lets you manage KMS access through the same IAM policies and roles you already use for everything else
3. **Key policy + Grants** -- the key policy allows a principal to create grants, and grants delegate temporary access to other principals. Primarily used by AWS services (EBS, S3, RDS), not human-authored policies

### The Default Key Policy -- Why It Matters

When you create a KMS key via the console, AWS attaches a default key policy with this critical statement:

```json
{
  "Sid": "Enable IAM User Permissions",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:root"
  },
  "Action": "kms:*",
  "Resource": "*"
}
```

This statement does **not** give the root user access to the key. It does something more subtle: it **enables IAM policies in the account to grant KMS permissions**. Without this statement, no IAM policy -- not even `AdministratorAccess` -- can grant access to the key.

> **Interview trap**: "The default key policy grants kms:* to the account root. Does that mean the root user can use the key?" Yes, the root user can, because the root user is the principal in the key policy. But the deeper point is that this statement also enables IAM policies to work. If you remove this statement and do not name specific IAM principals in the key policy, you have created an **unmanageable key** -- no one can access it except through the key policy itself, and if the key policy does not name any IAM principals, the key becomes orphaned (only AWS Support can help recover it).

### The Three Authorization Mechanisms

```
KMS AUTHORIZATION -- THREE INDEPENDENT PATHS
=============================================================================

  ┌────────────────────────────────────────────────────────────────────┐
  │  KMS KEY                                                          │
  │                                                                    │
  │  ┌──────────────────────────────────────────────────────────┐     │
  │  │  KEY POLICY (always required, always evaluated)          │     │
  │  │                                                          │     │
  │  │  Path 1: Direct Grant                                   │     │
  │  │  ┌────────────────────────────────────────────┐         │     │
  │  │  │ "Principal": "arn:aws:iam::...:role/App"   │         │     │
  │  │  │ "Action": "kms:Decrypt"                    │─── ✅   │     │
  │  │  │ Key policy alone is sufficient             │         │     │
  │  │  └────────────────────────────────────────────┘         │     │
  │  │                                                          │     │
  │  │  Path 2: Enable IAM Policies                            │     │
  │  │  ┌────────────────────────────────────────────┐         │     │
  │  │  │ "Principal": "arn:aws:iam::...:root"       │         │     │
  │  │  │ "Action": "kms:*"                          │         │     │
  │  │  │ This enables IAM policies to grant access  │──┐      │     │
  │  │  └────────────────────────────────────────────┘  │      │     │
  │  │                                                   │      │     │
  │  │  Path 3: Enable Grants                           │      │     │
  │  │  ┌────────────────────────────────────────────┐  │      │     │
  │  │  │ "Action": "kms:CreateGrant"                │  │      │     │
  │  │  │ "Condition": GrantIsForAWSResource = true  │──┼─┐    │     │
  │  │  └────────────────────────────────────────────┘  │ │    │     │
  │  │                                                   │ │    │     │
  │  └───────────────────────────────────────────────────┘ │    │     │
  │                                                    │   │    │     │
  │            ┌───────────────────────────────────┐   │   │    │     │
  │            │  IAM POLICY (identity-based)      │   │   │    │     │
  │            │  "Action": "kms:Decrypt"          │◄──┘   │    │     │
  │            │  "Resource": key ARN              │── ✅   │    │     │
  │            │  Key policy MUST enable IAM       │       │    │     │
  │            └───────────────────────────────────┘       │    │     │
  │                                                        │    │     │
  │            ┌───────────────────────────────────┐       │    │     │
  │            │  GRANT                            │◄──────┘    │     │
  │            │  Grantee: role/ServiceRole         │           │     │
  │            │  Operations: Decrypt, Encrypt     │── ✅       │     │
  │            │  Temporary, scoped, revocable     │           │     │
  │            └───────────────────────────────────┘           │     │
  │                                                             │     │
  └─────────────────────────────────────────────────────────────┘     │
  └────────────────────────────────────────────────────────────────────┘

  IMPORTANT: All three paths are subject to the same IAM evaluation
  logic you learned on Mar 5. An explicit Deny in any policy (SCP,
  identity policy, key policy) always wins. The key policy is simply
  the FIRST gate -- without it, the other two paths cannot open.
```

---

## Part 4: Grants -- Temporary Signed Notes

### The Analogy

When S3 needs to encrypt your data, it cannot walk into the locksmith shop on its own -- it does not have a permanent key to the vault. Instead, you (the key owner) sign a **temporary note** saying "S3 may cut one padlock using my master key for this specific job." That note is a grant. S3 presents the note at the locksmith shop, performs the operation, and the note is either retired (cleaned up) or remains for future operations on that resource. The key point: grants are temporary, scoped, and revocable -- unlike a permanent entry on the vault door's access list (key policy).

### The Technical Reality

Grants are an alternative to key policies and IAM policies for granting KMS access. They are the primary mechanism AWS services use to get temporary access to your KMS keys.

Key characteristics:
- **Eventually consistent** -- a newly created grant may take up to 5 minutes to take effect. To use a grant immediately, pass the **grant token** returned by `CreateGrant` in subsequent API calls
- **Scoped to specific operations** -- you specify exactly which KMS operations the grantee can perform (Encrypt, Decrypt, GenerateDataKey, etc.)
- **Per-key limit**: 50,000 grants per KMS key (a hard limit -- exceeding it causes `LimitExceededException`)
- **Revocable at any time** via `RevokeGrant` or `RetireGrant`
- **No cost** for creating or managing grants
- **Not visible in the key policy** -- grants are a separate access channel

```json
// Example: Creating a grant for a Lambda function to decrypt data
{
  "KeyId": "arn:aws:kms:us-east-1:111111111111:key/1234abcd-12ab-34cd-56ef-1234567890ab",
  "GranteePrincipal": "arn:aws:iam::111111111111:role/LambdaDecryptRole",
  "Operations": ["Decrypt", "DescribeKey"],
  "RetiringPrincipal": "arn:aws:iam::111111111111:role/KeyAdminRole",
  "Constraints": {
    "EncryptionContextSubset": {
      "department": "finance"  // Encryption context explained in Part 12 -- for now,
                               // know it is a set of key-value pairs that provide
                               // additional authentication and auditing
    }
  }
}
```

The `Constraints` field is powerful: it limits the grant to operations where the encryption context includes specific key-value pairs. This enables fine-grained access control -- the grantee can only decrypt data that was encrypted with a matching encryption context.

> **Interview tip**: "How do AWS services like EBS use KMS keys?" Through grants. When you create an encrypted EBS volume, EBS creates a grant on your KMS key that allows it to decrypt the data key when the volume is attached to an instance. The grant is tied to the specific volume and instance. This is why you see `kms:CreateGrant` in the key policy's service usage statement -- without it, the service cannot obtain temporary access.

---

## Part 5: Cross-Account KMS Access -- Two Vaults, Two Access Lists

### The Analogy

You already know the cross-account AssumeRole pattern from the IAM doc: Building A's employee needs to enter Building B, so Building B's visitor approval form (trust policy) must name the visitor, AND Building A must give the employee permission to visit (identity policy). KMS cross-account access follows the **same two-sided pattern**, but with KMS-specific nuances.

### The Two-Policy Requirement

```
CROSS-ACCOUNT KMS ACCESS -- TWO POLICIES MUST AGREE
=============================================================================

  ACCOUNT A (Key Owner: 111111111111)       ACCOUNT B (Key User: 222222222222)
  ┌─────────────────────────────────┐      ┌─────────────────────────────────┐
  │                                 │      │                                 │
  │  KMS Key: arn:aws:kms:...:key/ │      │  IAM Role: DataProcessorRole   │
  │           1234abcd-...          │      │                                 │
  │                                 │      │  Identity Policy:              │
  │  Key Policy:                    │      │  {                              │
  │  {                              │      │    "Effect": "Allow",           │
  │    "Effect": "Allow",           │      │    "Action": [                  │
  │    "Principal": {               │      │      "kms:Decrypt",             │
  │      "AWS": "arn:aws:iam::     │      │      "kms:DescribeKey"          │
  │        222222222222:root"       │  ◄───│    ],                           │
  │    },                           │      │    "Resource":                  │
  │    "Action": [                  │      │      "arn:aws:kms:us-east-1:   │
  │      "kms:Decrypt",             │      │       111111111111:key/         │
  │      "kms:DescribeKey"          │      │       1234abcd-..."            │
  │    ],                           │      │  }                              │
  │    "Resource": "*"              │      │                                 │
  │  }                              │      │  NOTE: Must use KEY ARN, not   │
  │                                 │      │  alias. Aliases are account-    │
  │                                 │      │  local and do not resolve       │
  │                                 │      │  cross-account.                 │
  └─────────────────────────────────┘      └─────────────────────────────────┘
         │                                          │
         │  BOTH must say YES                       │
         ▼                                          ▼
     KEY POLICY CHECK:                         IAM POLICY CHECK:
     Does the key policy allow                 Does Account B's IAM policy
     Account B (or a specific                  grant kms:Decrypt on the
     principal in Account B)                   key ARN in Account A?
     to perform kms:Decrypt?
```

### Cross-Account Nuances

1. **Key ARN is mandatory**: You cannot reference a key in another account by alias name or alias ARN. Aliases are account-scoped. Cross-account references must use the full key ARN: `arn:aws:kms:REGION:ACCOUNT:key/KEY-ID`.

2. **Two granularity options in the key policy**:
   - Grant access to the **entire external account** (`222222222222:root`) -- then IAM policies in that account decide which principals can use the key
   - Grant access to a **specific principal** (`222222222222:role/DataProcessorRole`) -- more restrictive, does not depend on IAM policies in the external account

3. **Some operations are not available cross-account**: You cannot call `CreateKey`, `ListKeys`, `PutKeyPolicy`, or `ScheduleKeyDeletion` cross-account -- these are account-level management operations.

4. **SCPs apply on both sides**: Account A's SCPs must allow the KMS operations on the key. Account B's SCPs must allow the caller to perform KMS operations. This is the same layered evaluation you learned on Mar 5.

5. **Grants work cross-account too**: The key policy can allow an external principal to create grants, and those grants can delegate access to other principals in the external account.

---

## Part 6: Key Rotation -- Changing the Locks Without Changing the Address

### The Analogy

Imagine the locksmith periodically re-cuts your master key -- new ridges, new pattern -- but keeps every old version on a key ring. The vault door address (key ID), the name on the door (alias), and the access list (key policy) all stay the same. When someone brings in an old padlock to be opened, the locksmith checks the key ring, finds the version that matches, and opens it. New padlocks are always cut using the latest version of the master key. This is seamless rotation: old data stays readable, new data uses stronger key material.

### Three Rotation Types

| Rotation Type | How It Works | Applies To | Key ID Changes? |
|---------------|-------------|-----------|-----------------|
| **Automatic** | KMS creates new backing key material on a schedule (90-2560 days, configurable) | Symmetric encryption keys only | No -- same key ID, ARN, aliases, policies |
| **On-demand** | You trigger immediate rotation via `RotateKeyOnDemand` API. Hard limit: 10 on-demand rotations per key | Symmetric keys (including imported key material -- but imported keys require a multi-step workflow: import new material in `PENDING_ROTATION` state first, then call `RotateKeyOnDemand`) | No -- same key ID, ARN, aliases, policies |
| **Manual** | You create an entirely new KMS key and update the alias to point to it | All key types (required for asymmetric and HMAC) | YES -- new key ID, but alias stays the same if you update it |

### How Automatic Rotation Works Internally

```
AUTOMATIC KEY ROTATION -- BACKING KEY VERSIONS
=============================================================================

  KMS Key: arn:aws:kms:us-east-1:111111111111:key/1234abcd-...
  Alias:   alias/prod-application-data
  Key ID:  1234abcd-...  (NEVER CHANGES)

  ┌─────────────────────────────────────────────────────────────────┐
  │  Backing Key Ring (inside KMS HSMs)                             │
  │                                                                 │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │
  │  │ Version 1     │  │ Version 2     │  │ Version 3     │      │
  │  │ Created: 2024 │  │ Created: 2025 │  │ Created: 2026 │      │
  │  │               │  │               │  │  (CURRENT)    │      │
  │  │ Used to       │  │ Used to       │  │ Used for all  │      │
  │  │ DECRYPT data  │  │ DECRYPT data  │  │ new ENCRYPT   │      │
  │  │ encrypted     │  │ encrypted     │  │ operations    │      │
  │  │ in 2024       │  │ in 2025       │  │               │      │
  │  └───────────────┘  └───────────────┘  └───────────────┘      │
  │                                                                 │
  │  The ciphertext metadata includes which backing key version    │
  │  was used, so KMS automatically selects the correct version    │
  │  for decryption. You never need to specify the version.        │
  └─────────────────────────────────────────────────────────────────┘

  KEY POINT: Old backing keys are NEVER deleted. Every ciphertext
  ever encrypted by this KMS key remains decryptable, regardless
  of how many rotations have occurred.
```

### Manual Rotation with Aliases

For asymmetric and HMAC keys (which do not support automatic rotation), manual rotation means creating a new key and updating the alias:

```hcl
# Manual rotation strategy using aliases
# Step 1: You already have the old key with an alias
resource "aws_kms_key" "signing_key_v2" {
  description              = "Signing key v2 (rotated 2026-03-09)"
  customer_master_key_spec = "ECC_NIST_P256"
  key_usage                = "SIGN_VERIFY"
  # ... key policy ...
}

# Step 2: Update the alias to point to the new key
# Applications using the alias automatically start using the new key
resource "aws_kms_alias" "signing_key" {
  name          = "alias/prod-signing-key"
  target_key_id = aws_kms_key.signing_key_v2.key_id
  # Was pointing to signing_key_v1, now points to v2
}

# IMPORTANT: Do NOT delete the old key. Data signed with v1 still needs
# the old public key for verification. Update the old key's key policy to
# remove Sign permissions (forcing new signatures to use v2) while keeping
# Verify permissions so existing signatures remain verifiable.
```

---

## Part 7: Multi-Region Keys -- Branch Office Locksmith

### The Analogy

Your locksmith shop has branch offices in multiple cities (regions). Normally, each branch cuts its own independent keys -- a key from the New York shop cannot open a lock made by the London shop. But for your most important customers, the locksmith offers a special service: **duplicate master keys** cut from the same mold and distributed to each branch. A padlock locked in New York can be opened in London without shipping the padlock back to New York. This is the multi-region key.

### The Technical Reality

Multi-region keys share the same key material across regions while remaining independent KMS keys with their own key policies, grants, and aliases in each region.

```
MULTI-REGION KEY ARCHITECTURE
=============================================================================

  PRIMARY KEY (us-east-1)                  REPLICA KEY (eu-west-1)
  ┌─────────────────────────────┐         ┌─────────────────────────────┐
  │ Key ID:  mrk-1234abcd...    │         │ Key ID:  mrk-1234abcd...    │
  │ ARN:     arn:aws:kms:       │         │ ARN:     arn:aws:kms:       │
  │          us-east-1:...:     │         │          eu-west-1:...:     │
  │          key/mrk-1234abcd   │         │          key/mrk-1234abcd   │
  │                             │         │                             │
  │ Key Material: 0xABCDEF...  │────────▶│ Key Material: 0xABCDEF...  │
  │ (IDENTICAL)                 │  Shared │ (IDENTICAL)                 │
  │                             │         │                             │
  │ Key Policy: (independent)   │         │ Key Policy: (independent)   │
  │ Aliases:   (independent)    │         │ Aliases:   (independent)    │
  │ Grants:    (independent)    │         │ Grants:    (independent)    │
  │                             │         │                             │
  │ Can: Create replicas,       │         │ Cannot: Create replicas,    │
  │      update key material    │         │         update key material │
  └─────────────────────────────┘         └─────────────────────────────┘
                                                       │
                                          REPLICA KEY (ap-southeast-1)
                                          ┌─────────────────────────────┐
                                          │ Key ID:  mrk-1234abcd...    │
                                          │ Key Material: 0xABCDEF...  │
                                          │ (IDENTICAL -- same mold)    │
                                          └─────────────────────────────┘
```

Key characteristics:
- **Same key ID** across all regions (always starts with `mrk-` prefix)
- **Same key material** -- ciphertext encrypted in one region can be decrypted in another
- **Independent key policies, aliases, and grants** per region -- replication copies the cryptographic material, not the authorization
- **Primary-replica model**: only the primary can create new replicas; any replica can be promoted to primary
- **Cannot convert** a single-region key to multi-region (must create as multi-region from the start)
- **AWS managed keys are always single-region** -- only customer managed keys can be multi-region

Use cases:
1. **Disaster recovery**: Encrypted data replicated to DR region can be decrypted locally without cross-region KMS calls
2. **Global applications**: DynamoDB global tables, S3 cross-region replication with client-side encryption
3. **Low-latency decryption**: Applications in multiple regions decrypt locally instead of calling back to the key's home region

> **Important nuance for S3 cross-region replication (CRR)**: Even with multi-region keys, S3 CRR **re-encrypts** objects in the destination region. S3 has no optimization that inspects whether source and destination keys share material -- the process is identical regardless of key type. The replication IAM role needs `kms:Decrypt` on the source key and `kms:Encrypt`/`kms:GenerateDataKey` on the destination key. Multi-region keys eliminate the need for cross-account or cross-region key access grants (since each region has its own independent replica key), but S3 still performs a full decrypt-then-encrypt cycle.

---

## Part 8: KMS vs CloudHSM -- Shared vs Private Locksmith

### The Analogy

KMS is a **shared locksmith shop** -- your keys are stored in vaults managed by the locksmith (AWS), and while each customer's keys are cryptographically isolated, the physical vaults (HSMs) are shared infrastructure. You trust the locksmith to keep your keys separate and secure.

CloudHSM is your **own private locksmith workbench** inside a locked room that only you can enter. AWS builds and maintains the room (hardware, networking, patching), but you hold the only keys to the workbench. AWS does not have access to your key material -- you are the sole custodian of the HSM credentials.

### Detailed Comparison

| Dimension | KMS | CloudHSM |
|-----------|-----|----------|
| **Tenancy** | Multi-tenant (shared HSMs with logical isolation) | Single-tenant (dedicated HSMs exclusively for you) |
| **FIPS validation** | FIPS 140-3 Level 3 (as of Feb 2025 -- upgraded from FIPS 140-2 Level 3 in 2023, which itself was upgraded from Level 2) | FIPS 140-3 Level 3 |
| **Key control** | AWS manages the HSM infrastructure; you manage key policies | You manage everything: keys, users, cryptographic operations |
| **Key accessibility** | Key material never leaves KMS HSMs; you interact via API | You have direct access to key material via PKCS#11, JCE, or CNG |
| **Integration** | Native integration with 100+ AWS services | Direct integration only; can use custom key store to bridge to KMS |
| **Custom algorithms** | No -- AES-256-GCM, RSA, ECC, HMAC only | Yes -- any algorithm supported by the HSM firmware |
| **High availability** | Fully managed (multi-AZ, redundant HSMs) | You must deploy >=2 HSMs across AZs for HA |
| **Pricing** | $1/key/month + API calls | ~$1.50/hr per HSM (~$1,100/month minimum for HA pair) |
| **Audit** | CloudTrail logs every API call | CloudTrail + CloudHSM client logs |
| **Best for** | 99% of use cases: S3, EBS, RDS, Lambda, Secrets Manager | Regulatory requirements mandating dedicated HSMs, custom crypto, PKCS#11 |

### Custom Key Stores -- Bridging KMS and CloudHSM

A custom key store connects KMS to your CloudHSM cluster. You get the **KMS API** (and all its service integrations) while the actual key material lives in **your dedicated HSMs**. This is the best-of-both-worlds pattern for organizations that need KMS convenience but regulatory compliance demands single-tenant hardware.

```
CUSTOM KEY STORE ARCHITECTURE
=============================================================================

  ┌──────────────────────────────────────────────────────────────┐
  │  Your Application                                            │
  │                                                              │
  │  aws kms encrypt --key-id alias/my-custom-key --plaintext ..│
  │       │                                                      │
  │       │  Normal KMS API call -- application code unchanged   │
  │       ▼                                                      │
  │  ┌───────────────────────────────────────┐                   │
  │  │  KMS Custom Key Store                 │                   │
  │  │                                       │                   │
  │  │  Receives KMS API call, routes the    │                   │
  │  │  cryptographic operation to CloudHSM  │                   │
  │  │       │                               │                   │
  │  │       ▼                               │                   │
  │  │  ┌─────────────────────────────────┐  │                   │
  │  │  │  CloudHSM Cluster               │  │                   │
  │  │  │  ┌──────────┐  ┌──────────┐    │  │                   │
  │  │  │  │  HSM 1   │  │  HSM 2   │    │  │                   │
  │  │  │  │  (AZ-a)  │  │  (AZ-b)  │    │  │                   │
  │  │  │  │          │  │          │    │  │                   │
  │  │  │  │  Key     │  │  Key     │    │  │                   │
  │  │  │  │  material│  │  material│    │  │                   │
  │  │  │  │  (yours) │  │  (yours) │    │  │                   │
  │  │  │  └──────────┘  └──────────┘    │  │                   │
  │  │  └─────────────────────────────────┘  │                   │
  │  └───────────────────────────────────────┘                   │
  │                                                              │
  │  Result: KMS API convenience + single-tenant HSM security    │
  └──────────────────────────────────────────────────────────────┘
```

### External Key Stores (XKS) -- Hold Your Own Key

Beyond CloudHSM-backed custom key stores, KMS supports **external key stores (XKS)** (launched November 2022). XKS allows KMS to use key material stored in your own HSM **outside AWS** -- on-premises or in another cloud provider.

```
EXTERNAL KEY STORE ARCHITECTURE
=============================================================================

  Your Application
       │
       │  Normal KMS API call
       ▼
  ┌───────────────────────────────┐
  │  KMS External Key Store       │
  │                               │
  │  Routes crypto operations     │
  │  to your external key manager │
  │       │                       │
  └───────┼───────────────────────┘
          │  HTTPS (TLS 1.3)
          ▼
  ┌───────────────────────────────┐
  │  XKS Proxy                    │    Runs in your infrastructure
  │  (you deploy and manage)      │    (on-prem, your VPC, or other cloud)
  │       │                       │
  └───────┼───────────────────────┘
          │
          ▼
  ┌───────────────────────────────┐
  │  Your External Key Manager    │    Thales, Entrust, Fortanix, etc.
  │  (HSM / key management)       │    Key material NEVER enters AWS
  └───────────────────────────────┘
```

This is the **"hold your own key" (HYOK)** pattern -- increasingly relevant for:
- **Data sovereignty** requirements (key material must not leave your jurisdiction)
- **Regulated industries** (financial services, healthcare) mandating external key custody
- **Multi-cloud architectures** where you want a single key manager across providers
- **"Kill switch" capability** -- disconnecting the XKS proxy immediately renders all data encrypted with those keys inaccessible in AWS

> **Custom key store vs External key store**: CloudHSM-backed custom key stores keep key material in AWS (but in your dedicated HSMs). External key stores keep key material **entirely outside AWS**. Both provide the KMS API convenience, but XKS adds latency because every crypto operation traverses the network to your external key manager.

> **Decision rule**: Use KMS for 99% of use cases. Choose CloudHSM-backed custom key stores when regulations mandate dedicated hardware within AWS. Choose external key stores (XKS) when key material must not reside in AWS at all. If you just need KMS service integrations with single-tenant HSMs in AWS, use a custom key store.

---

## Part 9: Key Identifiers -- Four Ways to Address the Locksmith

Every KMS key can be referenced by four identifiers, and choosing the right one matters:

| Identifier | Format | Example | Cross-Account? |
|-----------|--------|---------|---------------|
| **Key ID** | UUID | `1234abcd-12ab-34cd-56ef-1234567890ab` | No |
| **Key ARN** | Full ARN | `arn:aws:kms:us-east-1:111111111111:key/1234abcd-...` | **YES** |
| **Alias name** | `alias/` prefix | `alias/prod-application-data` | No |
| **Alias ARN** | Full alias ARN | `arn:aws:kms:us-east-1:111111111111:alias/prod-...` | No |

**Critical rule**: For cross-account operations, you **must** use the Key ARN. Aliases are account-scoped **in every form** -- both the short name (`alias/prod-data`) and the full alias ARN (`arn:aws:kms:region:account:alias/prod-data`) fail to resolve cross-account, despite the alias ARN containing the account ID. Only the Key ARN works. If you put any alias form in an IAM policy's Resource field for a cross-account scenario, the policy will match the wrong key (or no key) in the calling account.

Alias best practices:
- Use aliases as the primary reference in application code -- they are human-readable and rotation-friendly
- When you manually rotate a key, update the alias to point to the new key. Applications using the alias see zero downtime
- Alias names must start with `alias/` and cannot start with `alias/aws/` (reserved for AWS managed keys)
- Each alias maps to exactly one key, but one key can have multiple aliases

---

## Part 10: Integration with AWS Services -- Where KMS Shows Up

Nearly every AWS service that stores data integrates with KMS. Here is the encryption model for the most frequently tested services:

### S3 Server-Side Encryption

| Encryption Option | Who Manages the Key | Who Encrypts | Key Visible in Your Account | Cost |
|-------------------|--------------------|--------------|-----------------------------|------|
| **SSE-S3** | AWS (S3-managed keys -- not KMS; the only S3 encryption option that does not involve KMS at all) | S3 | No | Free |
| **SSE-KMS** | You (customer managed) or AWS (AWS managed `aws/s3`) | S3 (via KMS) | Yes | $1/month + API calls |
| **SSE-C** | You (customer-provided) | S3 | No (you send the key with each request) | Free (but you manage keys) |
| **Client-side** | You (any method) | Your application | Depends on your implementation | Depends |

```hcl
# Terraform: S3 bucket with SSE-KMS using a customer managed key
resource "aws_s3_bucket" "encrypted_data" {
  bucket = "my-encrypted-data-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "encrypted_data" {
  bucket = aws_s3_bucket.encrypted_data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.application_data.arn
    }
    bucket_key_enabled = true  # IMPORTANT: reduces KMS API calls by 99%
  }
}
```

**S3 Bucket Keys**: When `bucket_key_enabled = true`, S3 generates a bucket-level key from your KMS key and uses it to encrypt objects. Instead of calling KMS for every `PutObject`, S3 calls KMS once to generate the bucket key, then derives per-object keys locally. This reduces KMS costs by up to 99% for high-volume buckets. The trade-off: CloudTrail logs show the bucket ARN instead of the object ARN in KMS events, which reduces per-object audit granularity.

### EBS Encryption

EBS encryption uses envelope encryption with KMS:
1. When you create an encrypted volume, EBS calls `GenerateDataKeyWithoutPlaintext` to get an encrypted data key (no plaintext returned because the volume is not attached yet -- the data key will only be decrypted later when an instance needs it)
2. When the volume is attached to an instance, EBS calls `Decrypt` to get the plaintext data key via a **grant**
3. The plaintext data key is stored in the hypervisor memory of the host -- **not** in the EC2 instance itself
4. All I/O between the instance and the volume is encrypted using this data key
5. When the volume is detached, the plaintext data key is discarded from hypervisor memory

> **Interview question**: "If KMS goes down, do your running EC2 instances with encrypted EBS volumes stop working?" No. The plaintext data key is cached in the hypervisor memory for the lifetime of the attachment. KMS is only needed when attaching/detaching volumes or creating snapshots. Running instances are unaffected by a KMS outage.

### Other Service Integrations

| Service | How It Uses KMS | Key Point |
|---------|----------------|-----------|
| **RDS** | Encrypts storage, automated backups, read replicas, and snapshots with a single KMS key | Once encrypted, a DB instance cannot be unencrypted; read replicas in the same region use the same key |
| **Lambda** | Encrypts environment variables at rest with a default `aws/lambda` key or your CMK | Environment variables are decrypted at function initialization, not per-invocation |
| **Secrets Manager** | Encrypts every secret version with a KMS key (default: `aws/secretsmanager`) | Rotation functions need `kms:Decrypt` and `kms:GenerateDataKey` on the secret's KMS key |
| **DynamoDB** | Table-level encryption with AWS owned, AWS managed (`aws/dynamodb`), or customer managed key | Customer managed keys required for cross-account table access via VPC endpoint policies |
| **CloudTrail** | SSE-KMS encryption for trail log files in S3 | Key policy must allow `cloudtrail.amazonaws.com` to call `GenerateDataKey` |
| **SNS / SQS** | Message encryption at rest using KMS | Requires publishers and consumers to have KMS permissions; cross-service encryption (SNS to SQS) requires both services to use the same KMS key or have appropriate cross-key grants |

---

## Part 11: GenerateDataKey vs GenerateDataKeyWithoutPlaintext

These two API calls are at the heart of envelope encryption, and understanding when each is used is critical.

| API Call | What It Returns | Who Uses It | When |
|----------|----------------|-------------|------|
| **GenerateDataKey** | Plaintext data key + Encrypted data key | Your application (client-side encryption) | When you need to encrypt data immediately |
| **GenerateDataKeyWithoutPlaintext** | Encrypted data key ONLY (no plaintext) | AWS services (EBS, RDS) | When the data key will be needed later, not now |

**Why the "without plaintext" variant?** Consider EBS: when you create an encrypted volume, EBS does not need the plaintext data key immediately -- the volume is not attached to any instance yet. EBS stores the encrypted data key in the volume metadata. Only when the volume is attached does EBS call `Decrypt` to unwrap the plaintext data key. This avoids holding plaintext key material in memory unnecessarily.

```python
# Example: Client-side envelope encryption with GenerateDataKey
import boto3
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

kms = boto3.client('kms')

# Step 1: Ask KMS for a data key
response = kms.generate_data_key(
    KeyId='alias/prod-application-data',
    KeySpec='AES_256'  # Request a 256-bit AES key
)

plaintext_key = response['Plaintext']       # Use this to encrypt, then discard
encrypted_key = response['CiphertextBlob']  # Store this alongside ciphertext

# Step 2: Encrypt your data locally (never sends data to KMS)
nonce = os.urandom(12)
aesgcm = AESGCM(plaintext_key)
ciphertext = aesgcm.encrypt(nonce, b"Sensitive application data", None)

# Step 3: DISCARD the plaintext key from memory
del plaintext_key

# Step 4: Store encrypted data + encrypted key + nonce together
# In production, serialize this into your storage format
envelope = {
    'encrypted_data': ciphertext,
    'encrypted_data_key': encrypted_key,
    'nonce': nonce,
    'key_id': response['KeyId']  # Tracks which KMS key was used
}

# --- Later, to decrypt ---

# Step 1: Send the encrypted data key back to KMS
decrypt_response = kms.decrypt(
    CiphertextBlob=envelope['encrypted_data_key']
    # No need to specify KeyId -- KMS metadata in the ciphertext
    # identifies which key to use (including the correct backing
    # key version if rotation has occurred)
)

plaintext_key = decrypt_response['Plaintext']

# Step 2: Decrypt data locally
aesgcm = AESGCM(plaintext_key)
original_data = aesgcm.decrypt(envelope['nonce'], envelope['encrypted_data'], None)

# Step 3: Discard plaintext key
del plaintext_key

print(original_data)  # b"Sensitive application data"
```

> **Production note**: The example above hand-rolls envelope encryption for pedagogical clarity. In production, use the **AWS Encryption SDK** (available for Python, Java, JavaScript, C, and .NET) instead of implementing the envelope pattern manually. The SDK handles data key caching (reducing `GenerateDataKey` calls), key commitment (preventing ciphertext from being decrypted with the wrong key), algorithm suite selection, and a standardized message format. For DynamoDB specifically, use the **AWS Database Encryption SDK**.

---

## Part 12: Encryption Context -- The Shipping Label

**The Analogy**: When you send a padlock to the locksmith for opening, you include a **shipping label** with key-value pairs like `{"department": "finance", "project": "payroll"}`. The locksmith records this label in the audit log (CloudTrail) and, critically, uses it as **additional authenticated data** in the encryption. If someone tries to decrypt the ciphertext with a different shipping label (or no label), the decryption fails -- even if they have the correct KMS key.

**The Technical Reality**: Encryption context is a set of non-secret key-value string pairs that you pass with `Encrypt`, `Decrypt`, and `GenerateDataKey` calls. It serves three purposes:

1. **Additional authenticated data (AAD)**: The encryption context is cryptographically bound to the ciphertext -- it is fed into the authenticated encryption algorithm as an input, not stored as metadata. Decryption with the wrong context (or no context) fails at the **cryptographic level**, not just as a policy check. Even a principal with direct `kms:Decrypt` permission and the correct ciphertext blob cannot decrypt without the matching context. Decryption requires the exact same context
2. **Audit trail**: The encryption context appears in plaintext in CloudTrail logs, making it easy to identify which operations accessed which data
3. **Access control via policy conditions**: Key policies and IAM policies can use `kms:EncryptionContext:` conditions to restrict who can encrypt/decrypt specific categories of data

```json
// IAM policy that restricts decryption to a specific encryption context
{
  "Effect": "Allow",
  "Action": "kms:Decrypt",
  "Resource": "arn:aws:kms:us-east-1:111111111111:key/1234abcd-...",
  "Condition": {
    "StringEquals": {
      "kms:EncryptionContext:department": "finance"
    }
  }
}
// This principal can ONLY decrypt ciphertext that was encrypted with
// {"department": "finance"} in the encryption context. Any other
// ciphertext -- even encrypted with the same KMS key -- is denied.
```

AWS services use encryption context automatically. For example:
- **EBS** includes `{"aws:ebs:id": "vol-0123456789abcdef0"}` in every KMS call
- **S3** (with SSE-KMS) includes `{"aws:s3:arn": "arn:aws:s3:::bucket/key"}`
- **Secrets Manager** includes `{"SecretARN": "arn:aws:secretsmanager:...:secret:name"}`

---

## Important Condition Keys and Operational Patterns

### `kms:ViaService` -- Restricting Key Usage to Specific Services

The `kms:ViaService` condition key restricts a KMS key so it can **only** be used through specific AWS services. This prevents direct API calls to `Encrypt`/`Decrypt` while still allowing service-integrated encryption.

```json
{
  "Effect": "Allow",
  "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": [
        "s3.us-east-1.amazonaws.com",
        "ebs.us-east-1.amazonaws.com"
      ]
    }
  }
}
```

This principal can only use the key through S3 or EBS -- direct calls to `kms:Encrypt` or `kms:Decrypt` will be denied. This is a high-signal interview topic for security-focused KMS questions.

### Safe Key Deletion -- The Disable-Monitor-Delete Pattern

Deleting a KMS key is **irreversible** -- all data encrypted under that key becomes permanently unrecoverable. The recommended practice:

1. **Disable the key first** (`kms:DisableKey`) -- this immediately prevents all cryptographic operations without destroying the key material
2. **Monitor CloudTrail** during the disabled period for any `AccessDeniedException` errors from services or applications still trying to use the key
3. **Schedule deletion** (`kms:ScheduleKeyDeletion`) with a waiting period of 7-30 days -- during this window, the key remains in `PendingDeletion` state and can be cancelled
4. **Set a CloudWatch alarm** on the `KMSKeyPendingDeletion` metric or create an EventBridge rule to catch any `Decrypt` attempts during the waiting period

> **Interview question**: "How would you safely delete a KMS key?" Disable first, monitor CloudTrail for usage, then schedule deletion with the maximum 30-day waiting period. If any service is still using the key, cancel the deletion before it completes.

---

## Cost Optimization Strategies

| Strategy | Impact | Implementation |
|----------|--------|---------------|
| **S3 Bucket Keys** | Reduces KMS API calls by up to 99% | `bucket_key_enabled = true` in bucket encryption config |
| **Use AWS managed keys for non-critical data** | Eliminates $1/month per-key charge | Accept the trade-off: no custom policies, no cross-account, no custom rotation |
| **Consolidate keys by workload** | Fewer keys = lower monthly cost | Use one key per workload or environment instead of one key per resource |
| **Cache data keys** | Reduces `GenerateDataKey` calls for high-throughput encryption | Use the AWS Encryption SDK, which caches data keys and reuses them for multiple encryption operations (configurable max age and max bytes) |
| **Monitor KMS API throttling** | Avoid `ThrottlingException` on high-volume keys | KMS has a shared per-region, per-account quota for cryptographic operations (typically 5,000-30,000 req/sec for symmetric ops, varies by region). Quotas are shared across ALL keys in the account. Use CloudWatch `ThrottleCount` metric. Request increases via Service Quotas console. S3 Bucket Keys exist specifically to mitigate this throttling for high-volume S3 workloads |

---

## Key Takeaways

- **KMS key policies are unlike any other resource-based policy.** Even the account root has zero access to a KMS key unless the key policy explicitly grants it. The default key policy's `"Principal": {"AWS": "arn:aws:iam::ACCOUNT:root"}` statement enables IAM policies to grant KMS access -- removing it creates an unmanageable key that only AWS Support can recover.

- **Envelope encryption is the core pattern behind every KMS integration.** KMS never encrypts your data directly (except for payloads under 4 KB). Instead, KMS generates data keys, you encrypt locally, discard the plaintext key, and store the encrypted data key alongside the ciphertext. `GenerateDataKey` returns both plaintext and encrypted versions; `GenerateDataKeyWithoutPlaintext` returns only the encrypted version for deferred use.

- **Three authorization paths: key policy, key policy + IAM policy, and grants.** Key policies can grant access directly to named principals. The default key policy's root statement enables IAM policies to work. Grants provide temporary, scoped access that AWS services use for operations like EBS volume attachment. All three paths are subject to the same IAM evaluation logic (explicit Deny always wins, SCPs apply, permission boundaries apply).

- **Cross-account KMS access requires two policies and a Key ARN.** The key policy in the owning account must name the external account or principal, and an IAM policy in the external account must grant the caller KMS permissions on the key ARN. Aliases do not work cross-account -- they are account-scoped.

- **Automatic key rotation changes the backing material, not the logical key.** The key ID, ARN, aliases, and policies stay the same. Old backing key versions are retained forever, so previously encrypted data remains decryptable. Automatic rotation works only for symmetric encryption keys (90-2560 day configurable period). Asymmetric and HMAC keys require manual rotation via alias swapping.

- **Multi-region keys share key material but not policies.** Same key ID (mrk- prefix) and identical key material across regions, but key policies, grants, and aliases are independent per region. A replica can be promoted to primary. S3 CRR still re-encrypts even with multi-region keys.

- **CloudHSM is for dedicated, single-tenant HSM requirements.** Use KMS for 99% of use cases. Choose CloudHSM when regulations mandate dedicated hardware, you need custom algorithms, or you require PKCS#11/JCE/CNG access. Custom key stores bridge the two -- KMS API convenience with CloudHSM-level isolation.

- **Encryption context is free, powerful, and underused.** It provides cryptographic binding (tamper detection), CloudTrail audit trails, and policy-based access control. AWS services include encryption context automatically, and you should too for client-side encryption.

- **S3 Bucket Keys reduce KMS costs by up to 99%.** For high-volume buckets using SSE-KMS, enable Bucket Keys to avoid per-object KMS API calls. The trade-off is reduced per-object audit granularity in CloudTrail.

- **KMS outages do not affect running workloads with cached data keys.** EBS volumes remain readable during KMS outages because the plaintext data key is cached in hypervisor memory. KMS is only needed for new volume attachments, snapshot operations, and new encryption operations.
