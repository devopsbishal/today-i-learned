# Multi-Region Architecture Patterns -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-31
**Difficulty:** Advanced

## Overview

This study plan covers multi-region architecture patterns on AWS -- moving beyond the "what services replicate" knowledge from your DR studies into the "how do I compose these services into a coherent multi-region system" territory. After completing this plan, you will understand the four fundamentals for evaluating multi-region readiness, the architectural trade-offs between active-active and active-passive patterns, data consistency models (CAP/PACELC applied to AWS services), cross-region dependency management, multi-region deployment strategies, organizational failover decision frameworks, and real-world lessons from the October 2025 US-EAST-1 outage.

## Prerequisites (Already Covered)

You have deep knowledge of the individual building blocks. This plan will NOT re-teach these services -- it focuses on how they compose together:
- **DR Strategies:** The four DR strategies (Backup/Restore, Pilot Light, Warm Standby, Multi-Site Active-Active), RPO/RTO, three Active-Active write patterns (Write Global, Write Local, Write Partitioned), AWS DRS, failover automation, failback procedures
- **Data Tier:** Aurora Global Database (storage-level replication, managed switchover, write forwarding), DynamoDB Global Tables (MREC/MRSC, last-writer-wins), ElastiCache Global Datastore, S3 CRR/SRR
- **Traffic Routing:** Route53 (failover, latency, geolocation, geoproximity routing, health checks), Global Accelerator (anycast IPs, traffic dials, endpoint weights), CloudFront
- **Compute:** Auto Scaling, ECS, EKS, ALB/NLB

This plan builds on all of that -- it shifts focus from individual service capabilities to system-level architecture decisions, operational patterns, and the organizational considerations that determine success or failure of multi-region deployments.

## Resources

### 1. AWS Prescriptive Guidance -- Multi-Region Fundamentals (Full Guide) -- 30 min
- **URL:** https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-multi-region-fundamentals/introduction.html
- **Type:** Official Docs (Prescriptive Guidance)
- **Summary:** The single most important resource for today's topic. This 300-level guide covers the four fundamentals you must evaluate before going multi-region: (1) understanding requirements -- availability tiers (Platinum 99.99% vs Gold 99.9% vs Silver 99.5%) and continuity metrics mapped to cost tiers, (2) understanding data -- the CAP theorem applied to AWS with async vs sync replication trade-offs, Read Local/Write Global as the dominant pattern, why write-intensive workloads favor primary-standby over active-active, (3) understanding workload dependencies -- eliminating cross-region dependencies including the critical Route53 control-plane-in-us-east-1 gotcha, avoiding shared fate with third-party services, and (4) operational readiness -- sequential deployment across regions, failover decision automation, and multi-region observability. Start with the "Single-Region Resilience" prerequisite section, then read all four fundamentals. Allocate the full 30 minutes -- this is dense and foundational. Navigate through all pages using the left sidebar (Introduction through Conclusion).

### 2. Well-Architected Reliability Pillar -- Data Plane vs Control Plane During Recovery (REL11-BP04) -- 15 min
- **URL:** https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_withstand_component_failures_avoid_control_plane.html
- **Type:** Official Docs (Well-Architected)
- **Summary:** A critical architectural principle that transforms how you think about multi-region failover. Control planes (CRUDL APIs for creating resources, launching instances, changing DNS records) are complex orchestration systems more likely to be unavailable during the very outages that trigger failover. Data planes (serving traffic, answering DNS queries, reading data) have higher availability SLAs. The page provides specific examples: using pre-provisioned EC2 instead of Auto Scaling during recovery, using ARC routing controls instead of editing Route53 records, adding Kubernetes pods (data plane) instead of nodes (control plane). This principle -- static stability through pre-provisioned resources -- is the architectural foundation of Warm Standby and Active-Active patterns you studied yesterday. The concept of "your failover mechanism must not depend on the thing that is failing" is the key insight for exam and interview scenarios.

### 3. AWS Architecture Blog -- Creating an Organizational Multi-Region Failover Strategy -- 15 min
- **URL:** https://aws.amazon.com/blogs/architecture/creating-an-organizational-multi-region-failover-strategy/
- **Type:** Blog (AWS Official)
- **Summary:** Shifts perspective from individual service failover to organizational-level decision-making -- the gap between "our database can fail over" and "our business can fail over." Presents four failover strategies at increasing granularity: component-level (individual service failover, most flexible but creates untested combinations), individual application (each team decides, decentralized but may leave dependencies stranded), dependency graph (fail over the minimum set of applications supporting a capability, removes modal behavior), and portfolio failover (fail over everything regardless of what is impaired, simplest operationally but most expensive). The key insight is that component-level failover creates combinatorial complexity where Region A runs some services while Region B runs others -- a configuration you have never tested and that may not work. Most mature organizations evolve toward dependency-graph or portfolio-level failover.

### 4. AWS Architecture Blog -- Creating a Multi-Region Application with AWS Services (3-Part Series) -- 25 min
- **URL (Part 1 - Compute, Networking, Security):** https://aws.amazon.com/blogs/architecture/creating-a-multi-region-application-with-aws-services-part-1-compute-and-security/
- **URL (Part 2 - Data and Replication):** https://aws.amazon.com/blogs/architecture/creating-a-multi-region-application-with-aws-services-part-2-data-and-replication/
- **URL (Part 3 - Application Management and Monitoring):** https://aws.amazon.com/blogs/architecture/creating-a-multi-region-application-with-aws-services-part-3-application-management-and-monitoring/
- **Type:** Blog (AWS Official)
- **Summary:** The practical implementation counterpart to the Prescriptive Guidance fundamentals. This 3-part series builds a concrete multi-region application step by step, filtering through 200+ AWS services to identify those with specific multi-region features. Part 1 covers the compute/networking/security layer (Global Accelerator, CloudFront, Transit Gateway, cross-region IAM, Secrets Manager replication). Part 2 covers the data layer (DynamoDB Global Tables, ElastiCache Global Datastore, Aurora Global Database, S3 CRR, AWS Backup cross-region copies) -- since you already know these services individually, focus on how they compose together and the consistency trade-offs discussed. Part 3 covers the operational layer (CloudFormation StackSets for cross-region deployment, CloudWatch cross-region dashboards, Systems Manager multi-region automation). Skim Part 1 quickly (you know these services), spend more time on Parts 2 and 3 for the composition and operational patterns. Part 2 was updated August 2024.

### 5. AWS Whitepaper -- CAP Theorem (Availability and Beyond) -- 10 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/cap-theorem.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** AWS's own framing of the CAP theorem for distributed systems. Network partitions between regions are inevitable (P is non-negotiable), so the real choice is between consistency (CP: return errors during partitions to guarantee correctness) and availability (AP: always return responses but accept stale/inconsistent data). Maps directly to your existing knowledge: DynamoDB Global Tables with MREC is AP (last-writer-wins, always available, eventually consistent), DynamoDB Global Tables with MRSC is CP (strong consistency across 3 regions but higher latency and potential unavailability during partitions), Aurora Global Database is AP for reads (eventual consistency in secondary regions) but effectively CP for writes (single writer). Understanding where each AWS service falls on the CAP spectrum is essential for exam questions that ask "which service guarantees consistency across regions" -- the answer is almost always "none fully, but here are the trade-offs."

### 6. AWS Architecture Blog -- What to Consider When Selecting a Region for Your Workloads -- 10 min
- **URL:** https://aws.amazon.com/blogs/architecture/what-to-consider-when-selecting-a-region-for-your-workloads/
- **Type:** Blog (AWS Official)
- **Summary:** Covers the four factors for AWS Region selection in priority order: (1) Compliance -- data residency regulations override all other factors (GDPR requires EU regions, certain government workloads require GovCloud), (2) Latency -- proximity to users drives performance for real-time applications, (3) Cost -- pricing varies significantly between regions due to local economies, real estate, power costs, customs on imported equipment (Sao Paulo and Asia Pacific regions are notably more expensive), (4) Service availability -- not all services or features are available in all regions (check the AWS Regional Services List). For multi-region specifically, the article helps you think about which regions to pair: geographic distance affects replication lag, cost differences compound when running two full stacks, and compliance may restrict your secondary region choices. Quick read since the concepts are straightforward, but important for exam scenarios asking "how do you select regions for a multi-region deployment."

### 7. INE Blog -- AWS October 2025 Outage: Multi-Region and Cloud Lessons Learned -- 15 min
- **URL:** https://ine.com/blog/aws-october-2025-outage-multi-region-and-cloud-lessons-learned
- **Type:** Article (Third-Party, Technical)
- **Summary:** Analysis of the October 2025 US-EAST-1 outage -- a 15-hour regional failure caused by internal DNS resolution failures affecting DynamoDB endpoints, which cascaded to 113 AWS services. Major platforms (Snapchat, Fortnite, Venmo, Robinhood) went down. The article validates every principle from the Prescriptive Guidance: Multi-AZ within a single region could not protect against this region-wide failure, organizations with manual failover processes experienced 3-6 hours of downtime versus 3-6 minutes with automation, monitoring hosted in the same region that failed was useless (the teams learned they were down from customers, not dashboards), and control-plane dependencies meant some "multi-region" architectures could not actually fail over because their provisioning mechanisms depended on us-east-1. This is the real-world proof that multi-region architecture is not theoretical -- it converts the Prescriptive Guidance fundamentals from best practices into concrete lessons. Excellent material for interview scenarios.

**Total estimated time: ~120 minutes**

## Study Tips

- **Build a mental model of composition, not services.** You already know what Aurora Global Database, DynamoDB Global Tables, and Route53 do individually. Today's goal is to think in terms of system architecture: "If I choose Active-Active with DynamoDB Global Tables (AP, eventual consistency) for the data tier, what does that imply for my compute layer (must be stateless, deployed in both regions), my routing layer (latency-based Route53 or Global Accelerator), my deployment pipeline (sequential, one region at a time), and my observability (independent monitoring in each region plus external canaries)?" Practice composing full architectures, not listing services.

- **Internalize the control plane vs data plane distinction.** This is the single most exam-relevant principle from today's study. When designing failover, ask for every component: "Does my failover mechanism depend on a control plane operation?" If yes, redesign. Route53 health-check-triggered failover uses the data plane (good). Editing Route53 records manually uses the control plane (bad). Launching new EC2 instances uses the control plane (bad). Routing to pre-provisioned instances uses the data plane (good). ARC routing controls use the data plane (good).

- **Start every multi-region design with the question "Do I actually need this?"** The Prescriptive Guidance is explicit: multi-region architectures cost roughly 2x and, if built incorrectly, can actually decrease availability. The first fundamental is understanding requirements -- if your workload needs 99.9% availability (8.77 hours downtime/year), multi-AZ within a single region likely suffices. Multi-region becomes justified at 99.99% (52.6 minutes/year) and is essential only when business requirements or compliance demand it. Leading with this framing in interviews demonstrates architectural maturity.

## Next Steps

- **Tomorrow (Day 3):** AWS Backup and Centralized Protection -- backup plans, vaults, cross-region and cross-account backup, Organization-level policies
- **Day 4:** Database DR Patterns -- RDS automated backups, snapshots, Aurora Global Database failover mechanics, cross-region read replica promotion
- **Day 5:** Storage DR and Data Replication -- S3 CRR for DR, DynamoDB Global Tables consistency, PITR recovery
- **Week 8 (Build Phase):** Implement a Multi-Region Architecture project combining the patterns from this week into a complete Terraform deployment with failover routing, cross-region data replication, and a failover runbook
