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
| 2026-04-19 | [ArgoCD + Helm in Production Patterns](./2026/april/kubernetes/2026-04-19-argocd-helm-production-patterns.md) | Three Helm-chart-source patterns (same-repo, external+inline, multi-source with `$values` ref), values precedence chain (`valueFiles` < `values` < `valuesObject` < `parameters`), per-env hierarchies (common/region/env/cluster), repo structure (app vs config repo, folder-per-env vs anti-pattern branch-per-env), ArgoCD Image Updater (image-list + helm.image-name/tag annotations, semver/digest/latest/name strategies, argocd vs git write-back, git-write-branch for PR gating), promotion dev->stg->prod (Image Updater in dev only, PR-based promotion, Kargo/GitOps Promoter), Argo Rollouts (Rollout CRD replacing Deployment, canary with traffic routing NGINX/ALB/Istio, blue-green with active/preview, AnalysisTemplate Prometheus gating with auto-abort), six production gotchas (Helm hooks reinterpreted as sync hooks running every sync, release ownership conflict between helm CLI and ArgoCD, mutating-webhook drift with ignoreDifferences, CRD ordering sync waves -10/-5/0, secrets never in values.yaml use ESO/Sealed/SOPS, Image Updater branch mismatch) |
| 2026-04-17 | [ArgoCD Advanced -- Multi-App & Multi-Cluster Patterns](./2026/april/kubernetes/2026-04-17-argocd-advanced-patterns.md) | App of Apps bootstrapping and nested patterns, all seven ApplicationSet generators (List/Cluster/Git-dirs/Git-files/Matrix/Merge/SCM/PR) with goTemplate + preservedFields + RollingSync, multi-cluster registration and hub-and-spoke topology, Application Controller sharding (legacy/round-robin/consistent-hashing), AppProject as tenant boundary with Casbin RBAC and JWT tokens, Dex vs direct OIDC, three secrets strategies (Sealed Secrets vs ESO vs SOPS) with decision matrix, notifications triggers/templates/services/subscriptions, advanced hook patterns, production Terraform |
| 2026-04-16 | [ArgoCD Fundamentals -- GitOps Model](./2026/april/kubernetes/2026-04-16-argocd-gitops-fundamentals.md) | Four CNCF OpenGitOps principles (Declarative/Versioned/Pulled/Reconciled), push vs pull CD, ArgoCD's seven components (API Server, Repo Server, Application Controller, Redis, Dex, ApplicationSet, Notifications), Application CRD (source/destination/project/syncPolicy), sync status vs health status, auto-sync toggles (automated/prune/selfHeal/allowEmpty), per-resource sync options (Replace/ServerSideApply/CreateNamespace/PruneLast), the critical Helm bridge (helm template not helm install -- no release Secrets, no helm history, lookup returns empty), valuesObject vs values, sync phases/waves/hooks with health-gating, ArgoCD vs Helm hook semantics, custom Lua health checks, annotation vs label tracking, production gotchas |
| 2026-04-15 | [Helm Advanced Patterns](./2026/april/kubernetes/2026-04-15-helm-advanced-patterns.md) | Subcharts/dependencies with conditions/tags/alias/import-values, global values propagation and value-scoping rules, full hook lifecycle (weights, delete policies, hooks-aren't-release-resources gotcha), library charts (type: library), OCI registries vs classic HTTP repos with ECR specifics, Helmfile orchestration (environments, needs, apply vs sync), three-tool testing stack (helm test vs ct vs helm-unittest), post-renderers and lookup escape hatches, decision frameworks for umbrella vs Helmfile vs ArgoCD |
| 2026-04-13 | [Helm Fundamentals](./2026/april/kubernetes/2026-04-13-helm-fundamentals.md) | Chart/release/repository mental model, chart anatomy (Chart.yaml, values.yaml, templates/, charts/, crds/, _helpers.tpl), Go template engine with built-in objects, named templates via define/include (why include beats template), values precedence stack, install/upgrade/rollback lifecycle with revision history as Secrets, three-tool debugging (lint vs template vs --dry-run) |
| 2026-04-10 | [SQS, SNS & Messaging Patterns](./2026/april/aws/2026-04-10-sqs-sns-messaging-patterns.md) | SQS Standard vs FIFO queues, visibility timeout Lambda interaction (6x rule), DLQs with redrive, long polling, MessageGroupId ordering, SNS Standard/FIFO topics, subscription filter policies (attribute vs body), SNS subscription DLQs, fan-out/buffering/priority/request-response patterns, SQS vs SNS vs EventBridge decision framework and composition patterns, production Terraform |
| 2026-04-09 | [Amazon EventBridge & Event-Driven Architecture](./2026/april/aws/2026-04-09-eventbridge-event-driven.md) | Event buses (default/custom/partner), content-based filtering (prefix/suffix/numeric/wildcard/$or), targets with input transformers and DLQs, Schema Registry and discovery, Archive and Replay, EventBridge Scheduler (rate/cron/one-time with universal targets), EventBridge Pipes (source-filter-enrich-target), API Destinations, cross-account/cross-region, EventBridge vs SNS vs SQS decision framework, choreography vs orchestration, production Terraform |
| 2026-04-08 | [AWS Step Functions -- Workflow Orchestration](./2026/april/aws/2026-04-08-step-functions-workflow-orchestration.md) | Standard vs Express workflows, ASL 8 state types, Retry/Catch error handling with exponential backoff and jitter, three integration patterns (Request Response/.sync/.waitForTaskToken), Distributed Map for S3-scale processing, input/output five-stage pipeline, versioning/aliases with canary deployments, SDK direct integrations, cost optimization (nesting Express inside Standard), production Terraform with templatefile() ASL |
| 2026-04-07 | [AWS API Gateway Deep Dive](./2026/april/aws/2026-04-07-api-gateway-deep-dive.md) | REST vs HTTP vs WebSocket API comparison, stages, throttling token bucket, caching, Lambda/Cognito/JWT authorizers, usage plans, integration types, canary deployments, WAF, mTLS, production Terraform |
| 2026-04-06 | [AWS Lambda Deep Dive](./2026/april/aws/2026-04-06-lambda-deep-dive.md) | Firecracker microVM execution model, cold start mitigation (SnapStart vs Provisioned Concurrency), concurrency model (reserved/provisioned/unreserved), three invocation models, event source mapping, Layers, versions/aliases with weighted routing, VPC Hyperplane, production Terraform |
| 2026-04-03 | [Storage DR & Data Replication](./2026/april/aws/2026-04-03-storage-dr-data-replication.md) | S3 CRR deep dive (replication rules, Batch Replication, RTC 15-min SLA, delete marker security), DynamoDB Global Tables MREC vs MRSC, PITR, Export to S3. Replication-is-not-backup principle, defense-in-depth layers, production Terraform |
| 2026-04-02 | [Database Disaster Recovery Patterns](./2026/april/aws/2026-04-02-database-disaster-recovery.md) | RDS backup internals (snapshots + PITR), Aurora Backtrack, Aurora Global Database failover (planned switchover vs unplanned with epoch-based write fencing), RDS Proxy failover acceleration, cross-region read replica promotion, DR decision framework by RPO/RTO |
| 2026-04-01 | [AWS Backup & Centralized Protection](./2026/april/aws/2026-04-01-aws-backup-centralized-protection.md) | Centralized backup orchestration across 20+ services. Backup plans, vault hierarchy (Standard/Governance/Compliance/Air-Gapped), cross-region and cross-account copies, Organization policies, restore testing, Audit Manager, production Terraform |
| 2026-03-31 | [Multi-Region Architecture Patterns](./2026/march/aws/2026-03-31-multi-region-architecture-patterns.md) | Data plane vs control plane principle, static stability, Active-Active vs Active-Passive decision framework, multi-region data patterns (Aurora Global DB, DynamoDB Global Tables, S3 CRR, ElastiCache Global Datastore), routing stack, CAP/PACELC, anti-patterns, production Terraform |
| 2026-03-30 | [AWS Disaster Recovery Strategies](./2026/march/aws/2026-03-30-disaster-recovery-strategies.md) | Four DR strategies on cost-recovery spectrum (Backup & Restore, Pilot Light, Warm Standby, Multi-Site Active-Active), RPO/RTO as business decisions, Pilot Light vs Warm Standby distinction, Active-Active write strategies, DRS for Warm Standby at Pilot Light cost |
| 2026-03-27 | [ElastiCache & Caching Strategies](./2026/march/aws/2026-03-27-elasticache-caching-strategies.md) | Redis/Valkey vs Memcached, caching strategies (lazy loading, write-through, write-behind, TTL), cache invalidation and thundering herd mitigations, Cluster Mode, ElastiCache Serverless, Global Datastore, Redis data structures, ElastiCache vs DAX decision |
| 2026-03-26 | [DynamoDB Deep Dive](./2026/march/aws/2026-03-26-dynamodb-deep-dive.md) | Partition key design and hot partition problem, GSI vs LSI, capacity modes (on-demand vs provisioned), DAX in-memory cache, DynamoDB Streams CDC, Global Tables (MREC vs MRSC), single-table design, transactions, TTL, PITR, DynamoDB vs RDS decision framework |
| 2026-03-25 | [RDS & Aurora Deep Dive](./2026/march/aws/2026-03-25-rds-aurora-deep-dive.md) | RDS Multi-AZ (classic vs DB Cluster), Read Replicas, Aurora shared storage architecture (6-copy quorum), Aurora replicas, I/O-Optimized vs Standard, Serverless v2 ACU scaling, Global Database, Aurora Cloning, RDS Proxy connection pooling, Aurora vs RDS decision |
| 2026-03-24 | [Block & File Storage -- EBS, EFS, FSx](./2026/march/aws/2026-03-24-ebs-efs-fsx-storage-selection.md) | EBS volume types (gp3/io2 Block Express/st1/sc1), IOPS vs throughput, Snapshots and DLM, Multi-Attach, EFS performance and throughput modes, storage classes. FSx for Windows/Lustre/NetApp ONTAP/OpenZFS, storage selection decision framework |
| 2026-03-23 | [S3 Advanced Deep Dive](./2026/march/aws/2026-03-23-s3-advanced-deep-dive.md) | Eight storage classes with Intelligent-Tiering, lifecycle policies (transition waterfall, minimum duration constraints), replication (CRR/SRR, Batch Replication, RTC), versioning, access points, Object Lock (Governance/Compliance), event notifications, Transfer Acceleration |
| 2026-03-18 | [Global Accelerator & Edge Services](./2026/march/aws/2026-03-18-global-accelerator-edge-services.md) | Anycast IPs, AWS backbone routing, four-level hierarchy (accelerator/listener/endpoint group/endpoint), traffic dials vs endpoint weights, standard vs custom routing, Route53 vs CloudFront vs GA three-way comparison, cross-region failover patterns |
| 2026-03-16 | [ALB vs NLB vs GLB](./2026/march/aws/2026-03-16-alb-vs-nlb-vs-glb.md) | ALB Layer 7 content-based routing with WAF/OIDC, NLB Layer 4 with static IPs and PrivateLink, GLB Layer 3 GENEVE encapsulation for third-party appliances, target group types, health checks, cross-zone load balancing, sticky sessions, architectural patterns |
| 2026-03-13 | [CloudFront Deep Dive](./2026/march/aws/2026-03-13-cloudfront-deep-dive.md) | Four-tier cache hierarchy (POPs, RECs, Origin Shield), cache behaviors, cache vs origin request policies, OAC, VPC Origins, Lambda@Edge vs CloudFront Functions, signed URLs/cookies, cache invalidation vs versioned file names, logging and monitoring |
| 2026-03-12 | [Route53 Advanced -- Hybrid DNS](./2026/march/aws/2026-03-12-route53-advanced-hybrid-dns.md) | Private Hosted Zones, split-view DNS, DNSSEC (KSK/ZSK two-key model), Resolver endpoints (inbound/outbound), forwarding rules, centralized hybrid DNS architecture, Route 53 Profiles, multi-VPC DNS patterns, query resolution order |
| 2026-03-11 | [Route53 Routing Policies & Health Checks](./2026/march/aws/2026-03-11-route53-routing-policies.md) | Seven routing policies, alias vs non-alias records, four health check types, complex nested routing trees, active-active vs active-passive failover, data plane vs control plane reliability, failover timing analysis |
| 2026-03-10 | [AWS Network & Application Protection](./2026/march/aws/2026-03-10-aws-network-application-protection.md) | WAF (Web ACLs, managed rules, Bot Control, ATP), Shield Standard vs Advanced, Network Firewall (stateless/stateful Suricata, domain filtering, TLS inspection), Firewall Manager (policy types, sandwich model), layered defense architecture |
| 2026-03-09 | [AWS KMS & Encryption Deep Dive](./2026/march/aws/2026-03-09-aws-kms-encryption-deep-dive.md) | Key ownership models, key types (symmetric/asymmetric/HMAC), key policies, grants, envelope encryption, cross-account access, key rotation, multi-region keys, KMS vs CloudHSM, encryption context, S3/EBS/RDS/Lambda encryption integration patterns |
| 2026-03-06 | [AWS Security Services -- Threat Detection](./2026/march/aws/2026-03-06-aws-security-services-threat-detection.md) | GuardDuty (data sources, protection plans, Extended Threat Detection), AWS Config (rules, conformance packs, auto-remediation), Inspector (EC2/ECR/Lambda scanning), Security Hub (ASFF, security standards, cross-region aggregation), detection-to-remediation pipeline |
| 2026-03-05 | [IAM Advanced Patterns](./2026/march/aws/2026-03-05-iam-advanced-patterns.md) | Seven IAM policy types, policy evaluation flowcharts (same vs cross-account), permission boundaries, session policies, cross-account AssumeRole with External IDs, SAML/OIDC federation, IAM Identity Center, full restriction stack |
| 2026-03-03 | [AWS Control Tower & Landing Zones](./2026/march/aws/2026-03-03-aws-control-tower-landing-zones.md) | Landing zone setup (Security OU, Log Archive, Audit), three control behaviors (preventive/detective/proactive), Account Factory and AFT GitOps pipeline, drift detection, Control Tower vs manual Organizations |
| 2026-03-02 | [AWS Organizations & SCPs](./2026/march/aws/2026-03-02-aws-organizations-scps.md) | Organization structure (management account, OUs), multi-account strategy, SCP mechanics (permission ceilings, Allow intersection, Deny inheritance), deny-list vs allow-list, real-world SCP examples, delegated administrator, consolidated billing |
| 2026-02-25 | [AWS Site-to-Site VPN](./2026/february/aws/2026-02-25-aws-site-to-site-vpn.md) | IPSec tunnels, Customer Gateway, VGW vs TGW termination, static vs BGP routing, VPN CloudHub, Accelerated VPN, dual-tunnel HA, ECMP on TGW, DX vs VPN decision matrix |
| 2026-02-24 | [AWS Direct Connect](./2026/february/aws/2026-02-24-aws-direct-connect.md) | Dedicated vs Hosted connections, Private/Public/Transit VIFs, Direct Connect Gateway, LAG, BGP failover, three resiliency models, MACsec, SiteLink, DX + TGW integration |
| 2026-02-23 | [Transit Gateway Deep Dive](./2026/february/aws/2026-02-23-transit-gateway-deep-dive.md) | TGW architecture, 5 attachment types, route table mechanics (association vs propagation), network segmentation, centralized egress and inspection with appliance mode, inter-region peering, multi-account RAM sharing |
| 2026-02-20 | [VPC Peering vs Transit Gateway vs PrivateLink](./2026/february/aws/2026-02-20-vpc-peering-vs-transit-gateway.md) | VPC Peering limitations (transitive routing, full mesh scaling), Transit Gateway architecture (route tables, segmentation), PrivateLink provider-consumer model, cost comparison, decision framework |
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
| [2026](./2026/) | 51 | Docker (Networking, Storage, Multi-Stage Builds, Runtimes), Terraform (State, Modules, Workspaces, Functions, Data Sources, Providers, Variables, Outputs, Security, Testing, Cloud/Enterprise, Project Structure), AWS (EC2, EC2 Purchasing Options, EC2 Networking, Auto Scaling, ECS, ECS Advanced, VPC Advanced, VPC Peering vs Transit Gateway, Transit Gateway Deep Dive, Direct Connect, Site-to-Site VPN, Organizations & SCPs, Control Tower & Landing Zones, IAM Advanced Patterns, Security Services -- Threat Detection, KMS & Encryption, Network & Application Protection, Route53 Routing Policies & Health Checks, Route53 Advanced -- Hybrid DNS, CloudFront Deep Dive, ALB vs NLB vs GLB, Global Accelerator & Edge Services, S3 Advanced Deep Dive, Block & File Storage -- EBS, EFS, FSx, RDS & Aurora Deep Dive, DynamoDB Deep Dive, ElastiCache & Caching Strategies, Disaster Recovery Strategies, Multi-Region Architecture Patterns, AWS Backup & Centralized Protection, Lambda Deep Dive, API Gateway Deep Dive, Step Functions Workflow Orchestration, EventBridge & Event-Driven Architecture, SQS SNS & Messaging Patterns), Kubernetes (Helm Fundamentals, Helm Advanced Patterns, ArgoCD Fundamentals) |
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
