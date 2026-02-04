# AWS EC2: Instance Types, Placement Groups, and Tenancy - 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-02-04
**Difficulty:** Intermediate

## Overview

This study plan covers three essential EC2 concepts: understanding instance type families and their naming conventions, strategically placing instances using placement groups for performance or availability, and choosing the right tenancy model for compliance and licensing requirements. After completing this plan, you will be able to decode any EC2 instance name, select appropriate placement strategies for different workloads, and make informed decisions between shared tenancy, dedicated instances, and dedicated hosts.

## Resources

### 1. Amazon EC2 Instance Type Naming Conventions (Official Docs) - 15 min
- **URL:** https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html
- **Type:** Official Docs
- **Summary:** The authoritative reference for understanding how EC2 instance names are constructed, breaking down the family, generation, processor options, and size components that make up names like `c7gn.xlarge`.

### 2. Amazon EC2 Instance Types Overview (Official Docs) - 20 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html
- **Type:** Official Docs
- **Summary:** Comprehensive documentation covering all instance families (general purpose, compute optimized, memory optimized, storage optimized, accelerated computing), their characteristics, and intended use cases. Essential for understanding which family to choose for different workloads.

### 3. EC2 Instance Families Comparison - 15 min
- **URL:** https://awsforengineers.com/blog/ec2-instance-families-comparison/
- **Type:** Blog
- **Summary:** A practical comparison of EC2 instance families with clear tables showing CPU-to-memory ratios, typical use cases, and guidance on when to choose each family. Helps translate the official documentation into real-world decision-making.

### 4. Placement Groups for Your Amazon EC2 Instances (Official Docs) - 15 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html
- **Type:** Official Docs
- **Summary:** The foundational documentation explaining what placement groups are, the three strategies available (cluster, spread, partition), and their core rules and limitations. Start here before diving into strategy details.

### 5. Placement Strategies for Your Placement Groups (Official Docs) - 20 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-strategies.html
- **Type:** Official Docs
- **Summary:** Deep dive into each placement strategy with specific limitations (7 instances per AZ for spread, 7 partitions per AZ for partition), supported instance types, and best practices for launching instances. Critical for understanding when capacity errors occur and how to avoid them.

### 6. EC2 Placement Groups: Optimizing Instance Placement for Performance and Availability - 15 min
- **URL:** https://dev.to/himanshusinghtomar/ec2-placement-groups-optimizing-instance-placement-for-performance-and-availability-5hhi
- **Type:** Article
- **Summary:** A practical guide that explains placement groups with clear comparisons and real-world scenarios. Includes decision criteria for choosing between cluster (HPC), spread (critical instances), and partition (distributed databases like Cassandra) strategies.

### 7. Amazon EC2 Dedicated Instances (Official Docs) - 10 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/dedicated-instance.html
- **Type:** Official Docs
- **Summary:** Documentation explaining dedicated instances, their isolation guarantees, pricing model (per-instance plus $2/hour regional fee), and limitations including no visibility into host placement and limited BYOL support.

### 8. Amazon EC2 Dedicated Hosts Overview (Official Docs) - 10 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/dedicated-hosts-overview.html
- **Type:** Official Docs
- **Summary:** Covers dedicated hosts including full BYOL support for Windows Server, SQL Server, RHEL, and SUSE licenses. Explains host affinity, visibility into socket/core counts, and when dedicated hosts are required for compliance or licensing.

## Study Tips

- **Decode instance names as you encounter them:** When you see `m7i.4xlarge` in documentation or the console, practice breaking it down (M = general purpose, 7 = generation, i = Intel processors, 4xlarge = size). This builds intuition faster than memorization.

- **Draw the placement group strategies:** Sketch cluster (all instances close together in one rack), spread (each instance on separate hardware, max 7 per AZ), and partition (groups of instances isolated by partition, max 7 partitions). Visual memory helps during interviews and real decisions.

- **Create a tenancy decision tree:** Map out: "Do I need BYOL for per-socket licenses?" (Yes = Dedicated Host), "Do I need dedicated hardware but not license control?" (Yes = Dedicated Instance), "Neither?" (Shared tenancy is fine).

## Next Steps

After completing this study plan, consider exploring:

1. **EC2 pricing models** - Understanding On-Demand, Reserved Instances, Savings Plans, and Spot Instances to optimize costs for different tenancy and placement configurations.

2. **AWS Compute Optimizer** - Learn how this service analyzes your workloads and recommends optimal instance types based on actual utilization patterns.

3. **EC2 Auto Scaling with placement groups** - Understand how Auto Scaling interacts with placement groups and what happens when capacity is insufficient.

4. **Hands-on practice** - Create each type of placement group in the AWS Console, launch instances, and observe the behavior when you stop/start instances or exceed limits.
