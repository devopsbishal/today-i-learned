# EKS IAM - The Employee Badge System

> Understanding how pods get AWS permissions using IRSA and Pod Identity - continuing the office campus analogy.

---

## TL;DR

| EKS IAM Concept | Office Campus Analogy |
|-----------------|----------------------|
| Pod needing AWS access | Employee needing access to secure vault |
| IAM Role | Access level / clearance |
| ServiceAccount | Employee ID card |
| OIDC Provider | External badge verification authority |
| JWT Token | Employee badge with QR code |
| IRSA | Badge verified through external authority |
| Pod Identity Agent | Building's own security desk |
| Pod Identity Association | Security desk's access list |
| Temporary Credentials | Visitor pass (expires in hours) |
| Static Access Keys (old way) | Permanent master key (dangerous!) |

---

## The Problem: Pods Need AWS Access

Your pods (desks/workers) often need to access AWS services:

```
┌─────────────────────────────────────────────────────────────────┐
│                         YOUR CAMPUS (EKS)                       │
│                                                                 │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐      │
│   │ Payment Pod │     │ Report Pod  │     │ Upload Pod  │      │
│   │             │     │             │     │             │      │
│   │ Needs:      │     │ Needs:      │     │ Needs:      │      │
│   │ • Secrets   │     │ • RDS read  │     │ • S3 write  │      │
│   │ • DynamoDB  │     │             │     │             │      │
│   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘      │
│          │                   │                   │              │
│          └───────────────────┼───────────────────┘              │
│                              │                                  │
│                              ▼                                  │
│                    "How do we get into                         │
│                     the AWS vault?"                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────┐
                    │   AWS SERVICES  │
                    │   (The Vault)   │
                    │                 │
                    │ • S3            │
                    │ • RDS           │
                    │ • Secrets Mgr   │
                    │ • DynamoDB      │
                    └─────────────────┘
```

---

## The Old Ways (Don't Do This!)

### Option 1: Static Access Keys

Like giving everyone a **permanent master key** to the vault:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Admin creates Access Keys (ACCESS_KEY_ID + SECRET_KEY)       │
│                              │                                  │
│                              ▼                                  │
│   Stores in Kubernetes Secret                                  │
│                              │                                  │
│                              ▼                                  │
│   Mounts into Pod                                              │
│                              │                                  │
│                              ▼                                  │
│   Pod uses keys forever... 😰                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Problems:**

| Issue | Risk |
|-------|------|
| Keys never expire | Leaked key = permanent access |
| Hard to rotate | Manual process, downtime |
| Keys can be committed to git | "Oops, pushed to GitHub" |
| Same key for many pods | One breach = all compromised |

---

### Option 2: Node IAM Role

Like giving the **whole building** one access badge:

```
┌─────────────────────────────────────────────────────────────────┐
│                         NODE (Building)                         │
│                    IAM Role: super-role                         │
│                    (S3 + RDS + Secrets + DynamoDB)              │
│                                                                 │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│   │ Payment Pod │  │ Random Pod  │  │ Hacked Pod  │           │
│   │ Needs: S3   │  │ Needs: None │  │ 😈          │           │
│   │ Gets: ALL ❌│  │ Gets: ALL ❌│  │ Gets: ALL ❌│           │
│   └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                                 │
│   Every pod in building gets FULL access!                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Problems:**

| Issue | Risk |
|-------|------|
| No pod-level isolation | All pods = same permissions |
| Violates least privilege | Pods get more than they need |
| Breach of one pod = access to everything | 💀 |

---

## The Solution: Per-Pod Identity

Give each employee their **own badge** with **only the access they need**:

```
┌─────────────────────────────────────────────────────────────────┐
│                         NODE (Building)                         │
│                                                                 │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│   │ Payment Pod │  │ Report Pod  │  │ Upload Pod  │           │
│   │ Badge: 🟢   │  │ Badge: 🔵   │  │ Badge: 🟡   │           │
│   │ Access: S3  │  │ Access: RDS │  │ Access: S3  │           │
│   │ + DynamoDB  │  │ (read only) │  │ (write only)│           │
│   └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                                 │
│   Each pod = unique badge = specific permissions ✅            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

AWS provides **two ways** to do this:
1. **IRSA** - IAM Roles for Service Accounts
2. **Pod Identity** - EKS Pod Identity (newer, simpler)

---

## IRSA: External Badge Verification

Like using an **external badge authority** (OIDC Provider) to verify employee badges.

### The Players

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YOUR CAMPUS                    BADGE AUTHORITY                │
│  (EKS Cluster)                  (OIDC Provider)                │
│  ┌─────────────────┐           ┌─────────────────┐             │
│  │                 │           │                 │             │
│  │ Issues badges   │──────────▶│ Verifies badges │             │
│  │ (JWT tokens)    │           │ for AWS         │             │
│  │                 │           │                 │             │
│  └─────────────────┘           └────────┬────────┘             │
│                                         │                       │
│                                         │ "Yes, this badge      │
│                                         │  is legitimate"       │
│                                         ▼                       │
│                                ┌─────────────────┐             │
│                                │   AWS VAULT     │             │
│                                │   (STS Guard)   │             │
│                                │                 │             │
│                                │ "Welcome! Here's│             │
│                                │  your temp pass"│             │
│                                └─────────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How IRSA Works

**One-time Setup:**

```
1. Register OIDC Provider with AWS
   └── "AWS, trust badges from my EKS cluster"

2. Create IAM Role with trust policy
   └── "Allow ServiceAccount X from cluster Y to assume this role"

3. Create ServiceAccount with annotation
   └── "This ServiceAccount uses IAM Role Z"
```

**Every pod request:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. Pod starts with ServiceAccount                             │
│     └── Kubelet injects JWT token (badge) into pod             │
│                                                                 │
│  2. JWT Token contains:                                        │
│     ┌─────────────────────────────────────────────┐            │
│     │ Issuer: oidc.eks.us-east-1.../ABC123       │            │
│     │ Subject: system:serviceaccount:            │            │
│     │          production:payment-sa             │            │
│     │ Expires: 1 hour (auto-refreshed by kubelet) │            │
│     └─────────────────────────────────────────────┘            │
│                                                                 │
│  3. AWS SDK in pod calls STS:                                  │
│     "AssumeRoleWithWebIdentity" + token                        │
│                                                                 │
│  4. STS asks OIDC Provider: "Is this badge real?"              │
│     OIDC: "Yes ✓"                                              │
│                                                                 │
│  5. STS returns temporary credentials (1-12 hours)             │
│                                                                 │
│  6. Pod accesses AWS services!                                 │
│                                                                 │
│  7. SDK auto-refreshes before expiry                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### IRSA Setup Steps

```
AWS Side:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. Enable OIDC Provider for cluster                           │
│     eksctl utils associate-iam-oidc-provider --cluster my-eks  │
│                                                                 │
│  2. Create IAM Policy (what can be accessed)                   │
│     {                                                           │
│       "Effect": "Allow",                                        │
│       "Action": ["s3:GetObject"],                              │
│       "Resource": "arn:aws:s3:::my-bucket/*"                   │
│     }                                                           │
│                                                                 │
│  3. Create IAM Role with trust policy (who can assume)         │
│     {                                                           │
│       "Effect": "Allow",                                        │
│       "Principal": {                                            │
│         "Federated": "arn:aws:iam::123:oidc-provider/..."     │
│       },                                                        │
│       "Action": "sts:AssumeRoleWithWebIdentity",               │
│       "Condition": {                                            │
│         "StringEquals": {                                       │
│           "...:sub": "system:serviceaccount:prod:my-sa"        │
│         }                                                       │
│       }                                                         │
│     }                                                           │
│                                                                 │
│  4. Attach policy to role                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Kubernetes Side:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  5. Create ServiceAccount with annotation                      │
│                                                                 │
│     apiVersion: v1                                              │
│     kind: ServiceAccount                                        │
│     metadata:                                                   │
│       name: my-sa                                               │
│       namespace: prod                                           │
│       annotations:                                              │
│         eks.amazonaws.com/role-arn: arn:aws:iam::123:role/X    │
│                                                                 │
│  6. Use ServiceAccount in Pod                                  │
│                                                                 │
│     spec:                                                       │
│       serviceAccountName: my-sa                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pod Identity: Building's Own Security Desk

Like having a **security desk in your building** that handles badge verification directly.

### The Difference

```
IRSA:
  Pod → "Here's my badge" → AWS STS → "Let me call OIDC..." → OIDC → "Valid!" → Credentials

Pod Identity:
  Pod → "I need access" → Security Desk (Agent) → AWS → Credentials
                          ↑
                    Agent handles everything!
```

### How Pod Identity Works

```
┌─────────────────────────────────────────────────────────────────┐
│                         NODE (Building)                         │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │            POD IDENTITY AGENT (Security Desk)             │ │
│  │                     (DaemonSet)                           │ │
│  │                                                           │ │
│  │  • Has list of who can access what (associations)        │ │
│  │  • Intercepts credential requests from pods              │ │
│  │  • Calls AWS on behalf of pods                           │ │
│  │  • Returns temporary credentials                         │ │
│  │                                                           │ │
│  └───────────────────────────────────────────────────────────┘ │
│         ▲                                                       │
│         │ "I need AWS access"                                  │
│         │                                                       │
│  ┌─────────────┐                                               │
│  │    POD      │  ← Just asks, agent handles the rest!        │
│  │  (my-sa)    │                                               │
│  └─────────────┘                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pod Identity Setup Steps

```
AWS Side:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. Install Pod Identity Agent add-on                          │
│     aws eks create-addon --cluster my-eks \                    │
│       --addon-name eks-pod-identity-agent                      │
│                                                                 │
│  2. Create IAM Policy (same as IRSA)                           │
│                                                                 │
│  3. Create IAM Role (SIMPLER trust policy!)                    │
│     {                                                           │
│       "Effect": "Allow",                                        │
│       "Principal": {                                            │
│         "Service": "pods.eks.amazonaws.com"    ← Generic!      │
│       },                                                        │
│       "Action": [                                              │
│         "sts:AssumeRole",                                      │
│         "sts:TagSession"                                       │
│       ]                                                         │
│     }                                                           │
│     No cluster-specific OIDC! Reusable across clusters!       │
│                                                                 │
│  4. Create Pod Identity Association                            │
│     aws eks create-pod-identity-association \                  │
│       --cluster-name my-eks \                                  │
│       --namespace prod \                                       │
│       --service-account my-sa \                                │
│       --role-arn arn:aws:iam::123:role/my-role                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Kubernetes Side:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  5. Create ServiceAccount (NO annotation needed!)              │
│                                                                 │
│     apiVersion: v1                                              │
│     kind: ServiceAccount                                        │
│     metadata:                                                   │
│       name: my-sa                                               │
│       namespace: prod                                           │
│     # No annotations! Association is in AWS.                   │
│                                                                 │
│  6. Use ServiceAccount in Pod (same as always)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## IRSA vs Pod Identity Comparison

| Aspect | IRSA | Pod Identity |
|--------|------|--------------|
| **OIDC Provider needed?** | Yes | No |
| **Trust policy per cluster?** | Yes (cluster-specific) | No (generic) |
| **SA → Role link stored in** | ServiceAccount annotation | AWS Association |
| **Reuse role across clusters?** | Must update trust policy | Just create new association |
| **Who calls STS?** | AWS SDK in pod | Pod Identity Agent |
| **Setup complexity** | More steps | Fewer steps |
| **Works outside EKS?** | Yes ✅ | No (EKS only) |
| **Launched** | 2019 | 2023 |

---

## Visual Comparison

```
IRSA FLOW:
┌──────┐    ┌──────────┐    ┌──────┐    ┌─────────────┐
│ Pod  │───▶│ AWS SDK  │───▶│ STS  │───▶│OIDC Provider│
│      │    │ (in pod) │    │      │◀───│  (verify)   │
│      │◀───│          │◀───│      │    └─────────────┘
└──────┘    └──────────┘    └──────┘
  Pod does the STS call


POD IDENTITY FLOW:
┌──────┐    ┌──────────┐    ┌──────┐
│ Pod  │───▶│  Agent   │───▶│ STS  │
│      │◀───│(DaemonSet│◀───│      │
│      │    │          │    │      │
└──────┘    └──────────┘    └──────┘
  Agent does the STS call
```

---

## When to Use Which?

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Is it EKS (AWS managed)?                                      │
│  │                                                              │
│  ├── YES                                                       │
│  │   │                                                          │
│  │   ├── New cluster / starting fresh?                         │
│  │   │   └── ✅ Pod Identity (simpler, AWS recommended)        │
│  │   │                                                          │
│  │   └── Existing IRSA setup?                                  │
│  │       └── Keep IRSA, migrate gradually if needed            │
│  │                                                              │
│  └── NO (self-managed K8s, EKS Anywhere, Hybrid, ROSA)        │
│      └── ✅ IRSA (only option)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Scenario | Recommendation |
|----------|----------------|
| New EKS cluster | Pod Identity ✅ |
| EKS Auto Mode | Pod Identity (default) |
| Multiple clusters, same roles | Pod Identity ✅ |
| Self-managed K8s on EC2 | IRSA |
| EKS Hybrid Nodes | IRSA |
| EKS Anywhere | IRSA |
| Existing IRSA working fine | Keep IRSA |

---

## Security Benefits (Both Methods)

| Old Way | IRSA / Pod Identity |
|---------|---------------------|
| Static keys (never expire) | Temporary credentials (auto-expire) |
| Same key for all pods | Per-pod unique identity |
| Manual rotation | Auto-rotation |
| Keys can leak | Nothing to leak! |
| Breach = permanent access | Breach = access expires soon |
| Violates least privilege | Each pod = only what it needs |

---

## Key Takeaways

1. **Never use static access keys** - They're a security nightmare
2. **Never rely on node IAM role for pods** - No isolation
3. **IRSA uses OIDC** - External verification, works anywhere
4. **Pod Identity uses Agent** - Simpler, EKS-only
5. **Both give temporary credentials** - Auto-rotating, short-lived
6. **Pod Identity for new EKS** - AWS recommended, simpler
7. **IRSA for non-EKS** - Only option for self-managed K8s

---

## Related Reading

- [AWS VPC - The Compound Analogy](../november/2025-11-30-aws-vpc.md)
- [AWS EKS - The Managed Office Building Analogy](./2025-12-02-aws-eks.md)
- [AWS VPC CNI - The Phone Extension System](./2025-12-03-aws-vpc-cni.md)
- [EKS Networking - The Mail & Delivery System](./2025-12-09-eks-networking.md)

---

*Written on December 10, 2025*
