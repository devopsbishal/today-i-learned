# AWS Disaster Recovery Strategies (The 4 DR Strategies) -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-30
**Difficulty:** Intermediate

## Overview

This study plan covers the four AWS disaster recovery strategies -- Backup/Restore, Pilot Light, Warm Standby, and Multi-Site Active-Active -- with focus on RPO/RTO targets, cost vs. recovery trade-offs, the AWS services that power each strategy, and when to choose one over another. After completing this plan, you will be able to map business requirements to the correct DR strategy, estimate costs, and design multi-region recovery architectures using services you already know (Aurora Global Database, DynamoDB Global Tables, S3 CRR, Route53 failover, etc.).

## Prerequisites (Already Covered)

You have solid foundations in the building blocks of DR architectures:
- **Networking:** Route53 failover routing and health checks, Global Accelerator, CloudFront
- **Storage:** S3 CRR/SRR, EBS snapshots with DLM, EFS replication
- **Database:** RDS Multi-AZ (synchronous) vs Read Replicas (async), Aurora Global Database (sub-1s storage-level replication, managed switchover vs unplanned failover), DynamoDB Global Tables (active-active, MREC last-writer-wins), ElastiCache Global Datastore
- **Compute:** Auto Scaling, ECS, EKS

This plan builds on that knowledge -- it will not re-explain these services but will show how they fit together within each DR strategy.

## Resources

### 1. AWS Well-Architected Reliability Pillar -- Use Defined Recovery Strategies (REL13-BP02) -- 20 min
- **URL:** https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html
- **Type:** Official Docs
- **Summary:** The single most authoritative page for DR strategy selection. Contains architecture diagrams for all four strategies (Figures 17-22), clear RPO/RTO ranges for each tier, the critical distinction between Pilot Light and Warm Standby ("pilot light cannot process requests without additional action taken first, while warm standby can handle traffic at reduced capacity immediately"), 6-step implementation framework, and coverage of AWS Elastic Disaster Recovery. This is your mental-model foundation -- read it carefully before everything else.

### 2. AWS Whitepaper -- Disaster Recovery Options in the Cloud -- 25 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** The definitive AWS whitepaper chapter on DR strategies. Goes deeper than the Reliability Pillar page with detailed service mappings for each strategy: which AWS services handle data backup (AWS Backup, S3 CRR, EBS snapshots), data replication (Aurora Global Database with sub-1s lag, DynamoDB Global Tables), infrastructure provisioning (CloudFormation, CDK), and traffic routing (Route53, Global Accelerator). Pay special attention to the cost-complexity spectrum diagram and the service-by-service breakdown for each strategy tier.

### 3. AWS Architecture Blog -- DR Part I: Strategies for Recovery in the Cloud -- 15 min
- **URL:** https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-i-strategies-for-recovery-in-the-cloud/
- **Type:** Blog (AWS Official)
- **Summary:** The overview article in the 4-part AWS Architecture Blog DR series (updated 2024). Provides the high-level comparison framework: the RPO/RTO spectrum from hours (Backup/Restore) down to near-zero (Active-Active), with clear visual positioning of each strategy on the cost-vs-recovery curve. Efficient primer that connects the dots between the four strategies before you dive into each one individually. Skim if the whitepaper already felt comprehensive.

### 4. AWS Architecture Blog -- DR Part III: Pilot Light and Warm Standby -- 20 min
- **URL:** https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iii-pilot-light-and-warm-standby/
- **Type:** Blog (AWS Official)
- **Summary:** Deep dive into the two most commonly confused strategies. Explains exactly what "core infrastructure" means in Pilot Light (live data replication but scaled-to-zero compute) vs. what "reduced capacity" means in Warm Standby (functional stack handling a fraction of production traffic). Covers the failover sequence for each: Pilot Light requires deploying infrastructure then scaling out, while Warm Standby only needs scaling out existing resources -- directly impacting RTO. Published January 2024.

### 5. AWS Architecture Blog -- DR Part IV: Multi-Site Active-Active -- 20 min
- **URL:** https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/
- **Type:** Blog (AWS Official)
- **Summary:** Covers the most complex and expensive DR strategy. Discusses multi-region active-active architecture with Route53 latency/geolocation routing, data consistency challenges (DynamoDB Global Tables last-writer-wins, Aurora Global Database write forwarding), and why you still need backups even with active-active (to protect against data corruption propagating across regions). Connects directly to your existing knowledge of Aurora Global Database and DynamoDB Global Tables. Published August 2024.

### 6. AWS Cloud Operations Blog -- Establishing RPO and RTO Targets for Cloud Applications -- 10 min
- **URL:** https://aws.amazon.com/blogs/mt/establishing-rpo-and-rto-targets-for-cloud-applications/
- **Type:** Blog (AWS Official)
- **Summary:** Practical guidance on how to actually determine RPO/RTO targets for your applications -- the business-side of DR that most technical resources skip. Covers the process of working with stakeholders to translate business impact into concrete recovery objectives, then mapping those objectives to DR strategy tiers. Important for interview scenarios where you need to justify strategy selection based on business requirements, not just technical preference.

### 7. AWS Disaster Recovery Workshop -- Strategy Overview and Hands-On Labs -- 10 min
- **URL:** https://disaster-recovery.workshop.aws/en/intro/disaster-recovery.html
- **Type:** Official Workshop
- **Summary:** AWS's free hands-on DR workshop with labs organized by strategy tier. Browse the strategy overview and lab structure to understand the practical implementation patterns. The workshop covers CloudFront, DynamoDB, Aurora Global Database, Route53, S3 bi-directional replication, and EKS multi-region. Bookmark this for the evening Terraform build session -- the architecture patterns map directly to infrastructure-as-code implementations. Focus on reading the intro and strategy sections now; save the hands-on labs for later.

**Total estimated time: ~120 minutes**

## Study Tips

- **Draw the 4-strategy spectrum yourself.** After reading resources 1-2, sketch a diagram from left (Backup/Restore, high RPO/RTO, low cost) to right (Active-Active, near-zero RPO/RTO, high cost). For each strategy, write: what is always running in the DR region, what needs to be provisioned during failover, and the RPO/RTO range. This single diagram will anchor every DR conversation in interviews.

- **Map services you already know to DR roles.** As you read, categorize each AWS service by its DR function: data backup (AWS Backup, EBS snapshots, S3 versioning), data replication (Aurora Global Database, DynamoDB Global Tables, S3 CRR, ElastiCache Global Datastore), traffic routing (Route53 failover, Global Accelerator endpoint groups), and infrastructure provisioning (CloudFormation StackSets, ASG scaling policies). This prevents the "too many services" overwhelm.

- **Focus on the Pilot Light vs. Warm Standby distinction.** This is the most common interview question and exam trap. The key difference is not about which services are replicated -- both replicate data. The difference is whether compute and networking infrastructure is running (Warm Standby) or needs to be provisioned from scratch (Pilot Light). Think of it as: Pilot Light = data is live, everything else is off. Warm Standby = a smaller copy of production is running and serving health checks.

## Next Steps

- **Tomorrow (Day 2):** Multi-Region Architecture Patterns -- Active-Active vs. Active-Passive design, data replication strategies, conflict resolution, and region selection criteria
- **Day 3:** AWS Backup and Centralized Protection -- backup plans, vaults, cross-region and cross-account backup, Organization-level policies
- **Day 4:** Database DR Patterns -- RDS automated backups, snapshots, Aurora Global Database failover mechanics, cross-region read replica promotion
- **Day 5:** Storage DR and Data Replication -- S3 CRR for DR, DynamoDB Global Tables consistency, PITR recovery
- **Week 8 (Build Phase):** You will implement a Multi-Region DR Architecture project, designing primary and DR region infrastructure, failover routing, and a complete failover runbook
