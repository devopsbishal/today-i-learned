# Today I Learned

A personal learning journal where I document what I learn each day. Writing things down helps me understand concepts better and gives me a reference to look back on when I need a refresher.

---

## 📚 What's This?

This is my collection of bite-sized learnings - concepts, insights, and "aha moments" I pick up along the way. Each entry is written in my own words, often with real-world analogies that help me remember.

## 📁 Structure

```
today-i-learned/
├── 2025/
│   ├── november/
│   │   └── 2025-11-30-aws-vpc.md
│   └── december/
├── 2026/
│   └── ...
└── Readme.md
```

Organized by: `year/month/YYYY-MM-DD-topic.md`

## 📝 Recent Learnings

| Date | Topic | Description |
|------|-------|-------------|
| 2026-03-09 | [AWS KMS & Encryption Deep Dive](./2026/march/aws/2026-03-09-aws-kms-encryption-deep-dive.md) | KMS key ownership models (customer managed, AWS managed, AWS owned), key types (symmetric AES-256-GCM, asymmetric RSA/ECC, HMAC), key policies as primary access control (unique zero-default behavior), IAM integration (three authorization paths), grants (temporary delegated access, eventual consistency, grant tokens), envelope encryption (GenerateDataKey flow, data key caching), cross-account KMS access (two-policy requirement, Key ARN mandatory), key rotation (automatic 90-2560 days, on-demand, manual with aliases), multi-region keys (primary-replica model, shared key material, independent policies), KMS vs CloudHSM (multi-tenant vs single-tenant, FIPS 140-2 Level 3, custom key stores), encryption context (AAD, audit trail, policy conditions), S3 encryption options (SSE-S3/SSE-KMS/SSE-C, Bucket Keys for cost optimization), EBS/RDS/Lambda/Secrets Manager integration patterns |
| 2026-03-06 | [AWS Security Services -- Threat Detection](./2026/march/aws/2026-03-06-aws-security-services-threat-detection.md) | GuardDuty (foundational data sources, protection plans for S3/EKS/RDS/Lambda/Malware/Runtime, finding type naming convention, severity levels, Extended Threat Detection for multi-stage attacks), AWS Config (configuration recorder, configuration items, managed vs custom rules, conformance packs, detective vs proactive evaluation, auto-remediation via SSM, multi-account aggregator), Amazon Inspector (EC2/ECR/Lambda scanning, CVE database, environment-aware risk scoring, network reachability findings, continuous scanning), Security Hub (ASFF normalization, security standards CIS/FSBP/PCI DSS/NIST, cross-region aggregation, automation rules, central configuration via Organizations), full detection-to-remediation pipeline (finding to EventBridge to Lambda/SSM/SNS), Security Tooling account architecture with delegated admin for all four services, Log Archive separation of duties |
| 2026-03-05 | [IAM Advanced Patterns](./2026/march/aws/2026-03-05-iam-advanced-patterns.md) | Seven IAM policy types and how they interact, policy evaluation flowcharts (same-account vs cross-account), permission boundaries (delegation pattern, privilege escalation prevention), session policies (per-session scoping), cross-account AssumeRole (trust policies, External IDs, confused deputy), SAML 2.0 federation flow, OIDC federation (GitHub Actions OIDC, EKS IRSA connection), IAM Identity Center (permission sets, identity sources, auto-created roles), full restriction stack (SCP > boundary > identity policy > session policy) |
| 2026-03-03 | [AWS Control Tower & Landing Zones](./2026/march/aws/2026-03-03-aws-control-tower-landing-zones.md) | Control Tower as orchestration layer on Organizations, landing zone setup (Security OU, Log Archive, Audit accounts), three control behaviors (preventive/SCPs, detective/Config Rules, proactive/CloudFormation Hooks), three guidance levels (mandatory, strongly recommended, elective), Account Factory provisioning, Account Factory for Terraform (AFT) GitOps pipeline, drift detection, StackSets deployment mechanism, Control Tower vs manual Organizations comparison |
| 2026-03-02 | [AWS Organizations & SCPs](./2026/march/aws/2026-03-02-aws-organizations-scps.md) | Organization structure (management account, OUs, root), multi-account strategy (Security/Infrastructure/Workloads/Sandbox OUs), SCP mechanics (permission ceilings, not grants), SCP evaluation logic (Allow intersection, Deny inheritance), deny-list vs allow-list strategy, real-world SCP examples (deny root, restrict regions, enforce encryption, prevent leaving org), SCPs vs Permission Boundaries vs IAM Policies, cross-account access patterns, delegated administrator, consolidated billing, Terraform examples |
| 2026-02-25 | [AWS Site-to-Site VPN](./2026/february/aws/2026-02-25-aws-site-to-site-vpn.md) | IPSec tunnel fundamentals, Customer Gateway (CGW), VGW vs TGW termination, static vs dynamic (BGP) routing, VPN CloudHub multi-site, Accelerated VPN (Global Accelerator), dual-tunnel HA, redundant CGW pattern, ECMP bandwidth aggregation on TGW, DX vs VPN decision matrix, CloudWatch monitoring and troubleshooting |
| 2026-02-24 | [AWS Direct Connect](./2026/february/aws/2026-02-24-aws-direct-connect.md) | Dedicated vs Hosted connections, Private/Public/Transit VIFs, Direct Connect Gateway (multi-VPC, multi-region), LAG, BGP communities for failover, three resiliency models (dev/test, high, maximum), MACsec encryption, SiteLink, DX + TGW integration via Transit VIF + DXGW, DX vs VPN trade-offs |
| 2026-02-23 | [Transit Gateway Deep Dive](./2026/february/aws/2026-02-23-transit-gateway-deep-dive.md) | TGW architecture (ENIs, /28 subnets), 5 attachment types, route table mechanics (association vs propagation), network segmentation, centralized egress, centralized inspection with appliance mode, GWLB integration, inter-region peering, multi-account RAM sharing, cost model, key quotas |
| 2026-02-20 | [VPC Peering vs Transit Gateway vs PrivateLink](./2026/february/aws/2026-02-20-vpc-peering-vs-transit-gateway.md) | VPC Peering limitations (transitive routing, edge-to-edge, full mesh scaling), Transit Gateway architecture (route tables, associations, propagations, segmentation), PrivateLink provider-consumer model, cost comparison, decision framework |
| 2026-02-19 | [VPC Advanced: CIDR Planning & Subnet Strategy](./2026/february/aws/2026-02-19-vpc-advanced-cidr-subnet-strategy.md) | CIDR planning for 10+ VPCs, subnet tiers (public/private/isolated), route table design, Regional NAT Gateway, VPC endpoints, IPAM |
| 2026-02-17 | [ECS Advanced](./2026/february/aws/2026-02-17-ecs-advanced.md) | Service discovery (Cloud Map), Service Connect, App Mesh (EOL), ECS Anywhere, advanced capacity provider strategies |
| 2026-02-16 | [ECS Fundamentals](./2026/february/aws/2026-02-16-ecs-fundamentals.md) | Task definitions, services, capacity providers, Fargate vs EC2, deployment strategies |
| 2026-02-12 | [Auto Scaling Deep Dive](./2026/february/aws/2026-02-12-auto-scaling-deep-dive.md) | Scaling policies (target tracking, step, simple), predictive scaling, warm pools & lifecycle hooks |
| 2026-02-09 | [EC2 Networking](./2026/february/aws/2026-02-09-ec2-networking.md) | ENI, ENA, Enhanced Networking, ENA Express/SRD, EFA, MTU & Jumbo Frames |
| 2026-02-08 | [EC2 Purchasing Options](./2026/february/aws/2026-02-08-ec2-purchasing-options.md) | Spot, Reserved Instances, Savings Plans & Capacity Reservations |
| 2026-02-05 | [EC2 Instance Types, Placement & Tenancy](./2026/february/aws/2026-02-05-ec2-instance-types-placement-tenancy.md) | Instance families, placement groups & dedicated hosts |
| 2026-02-02 | [Terraform Project Structure](./2026/january/terraform/2026-02-02-terraform-project-structure.md) | Mono-repo vs multi-repo, environments, naming |
| 2026-01-30 | [Terraform Cloud/Enterprise](./2026/january/terraform/2026-01-30-terraform-cloud-enterprise.md) | Remote runs, Sentinel, private registry |
| 2026-01-29 | [Terraform Testing](./2026/january/terraform/2026-01-29-terraform-testing.md) | Validate, plan, terraform test, Terratest |
| 2026-01-28 | [Terraform Security](./2026/january/terraform/2026-01-28-terraform-security-best-practices.md) | Secrets, state security, tfsec, Checkov |
| 2026-01-27 | [Terraform Outputs & Remote State](./2026/january/terraform/2026-01-27-terraform-outputs-remote-state.md) | Output patterns, cross-stack references |
| 2026-01-23 | [Terraform Variables](./2026/january/terraform/2026-01-23-terraform-variables-validation.md) | Variable types, validation, sensitive vars |
| 2026-01-22 | [Terraform Providers](./2026/january/terraform/2026-01-22-terraform-providers.md) | Configuration, aliases, version constraints |
| 2026-01-20 | [Terraform Data Sources & Locals](./2026/january/terraform/2026-01-20-terraform-data-sources-locals.md) | Querying resources, local values |
| 2026-01-19 | [Terraform Functions](./2026/january/terraform/2026-01-19-terraform-functions-expressions.md) | Built-in functions, for expressions |

## 📂 Archive

| Year | Entries | Topics |
|------|---------|--------|
| [2026](./2026/) | 33 | Docker (Networking, Storage, Multi-Stage Builds, Runtimes), Terraform (State, Modules, Workspaces, Functions, Data Sources, Providers, Variables, Outputs, Security, Testing, Cloud/Enterprise, Project Structure), AWS (EC2, EC2 Purchasing Options, EC2 Networking, Auto Scaling, ECS, ECS Advanced, VPC Advanced, VPC Peering vs Transit Gateway, Transit Gateway Deep Dive, Direct Connect, Site-to-Site VPN, Organizations & SCPs, Control Tower & Landing Zones, IAM Advanced Patterns, Security Services -- Threat Detection, KMS & Encryption) |
| [2025](./2025/) | 10 | AWS (VPC, EKS, VPC CNI, Networking, IAM, Storage, Autoscaling, Security, Observability, Upgrades) |

---

## 🧠 Why I Do This

1. **Learn by teaching** - Writing forces me to truly understand
2. **Future reference** - Quick refresher when concepts get fuzzy
3. **Track progress** - See how far I've come over time

---

### 🔗 Connect

[![GitHub](https://img.shields.io/badge/GitHub-@devopsbishal-181717?style=flat&logo=github)](https://github.com/devopsbishal)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Bishal%20Rayamajhi-0A66C2?style=flat&logo=linkedin)](https://www.linkedin.com/in/bishal-rayamajhi-02523a243/)
