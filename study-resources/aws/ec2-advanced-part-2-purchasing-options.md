# EC2 Advanced Part 2: Spot Instances, Reserved Instances, Savings Plans, and Capacity Reservations -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-02-08
**Difficulty:** Intermediate

## Overview

This study plan covers the four main EC2 purchasing options beyond On-Demand pricing: Spot Instances for interruptible workloads at up to 90% discount, Reserved Instances for long-term commitment savings, Savings Plans for flexible commitment-based discounts, and On-Demand Capacity Reservations for guaranteeing availability. After completing this plan, you will understand how each purchasing model works, when to use each one, the tradeoffs between flexibility and savings, and how to combine them into a cost optimization strategy.

## Resources

### 1. Amazon EC2 Billing and Purchasing Options (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-purchasing-options.html
- **Type:** Official Docs
- **Summary:** The single-page overview that introduces all seven EC2 purchasing options with concise descriptions and selection guidance. Start here to build the mental map of how On-Demand, Spot, Reserved Instances, Savings Plans, and Capacity Reservations relate to each other before diving into any one model.

### 2. Spot Instances Overview and How Spot Instances Work (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html
- **Type:** Official Docs
- **Summary:** The foundational reference for Spot Instances covering key concepts (Spot capacity pools, Spot pricing, request types, interruption notices, rebalance recommendations), the differences between Spot and On-Demand, and how interruptions work with the 2-minute warning. Follow the link to "How Spot Instances work" from this page to understand the full lifecycle of a Spot request.

### 3. Best Practices for Amazon EC2 Spot (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html
- **Type:** Official Docs
- **Summary:** Eight actionable best practices including preparing for interruptions via EventBridge, requesting at least 10 instance types for flexibility, using attribute-based instance type selection, leveraging the price-capacity-optimized allocation strategy, and choosing the right Spot request method. Essential reading for understanding how to use Spot in production rather than just theory.

### 4. Reserved Instances for Amazon EC2 Overview (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html
- **Type:** Official Docs
- **Summary:** Comprehensive overview of Reserved Instances covering the four key pricing variables (instance attributes, term length, payment options, offering class), the difference between Standard and Convertible RIs, regional vs zonal scope and their capacity reservation implications, and how RI discounts are applied to running instances. Also covers the Reserved Instance Marketplace for selling unused commitments.

### 5. What Are Savings Plans and Savings Plan Types (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html
- **Type:** Official Docs
- **Summary:** The official introduction to Savings Plans explaining the commitment model (dollars per hour rather than instance configuration), payment options, and how to use Cost Explorer for purchase recommendations. From this page, follow the link to "Savings Plans types" to understand the critical differences between Compute Savings Plans (up to 66% off, maximum flexibility across families, regions, and even Fargate/Lambda) and EC2 Instance Savings Plans (up to 72% off, locked to an instance family in a region).

### 6. Compute Savings Plans and Reserved Instances Comparison (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/savingsplans/latest/userguide/sp-ris.html
- **Type:** Official Docs
- **Summary:** The definitive side-by-side comparison table showing how Compute Savings Plans, EC2 Instance Savings Plans, Convertible RIs, and Standard RIs differ across savings percentages, flexibility dimensions (family, size, tenancy, OS, region), and applicability to Fargate and Lambda. This page alone resolves most confusion about which commitment model to choose.

### 7. On-Demand Capacity Reservations Overview (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/capacity-reservation-overview.html
- **Type:** Official Docs
- **Summary:** Explains how On-Demand Capacity Reservations guarantee instance availability in a specific Availability Zone without requiring a long-term commitment. Covers the critical distinction that Capacity Reservations ensure capacity but do not provide a billing discount on their own, and how they can be combined with Savings Plans or Regional RIs to get both guaranteed capacity and a discount.

### 8. AWS Savings Plans vs Reserved Instances: A Roadmap (CloudZero) -- 15 min
- **URL:** https://www.cloudzero.com/blog/savings-plans-vs-reserved-instances/
- **Type:** Blog
- **Summary:** A practitioner-oriented comparison that goes beyond the official docs by explaining when to choose Savings Plans over Reserved Instances with real decision scenarios. Covers the hybrid strategy of using Savings Plans for variable workloads and RIs for stable baselines, addresses common mistakes in commitment purchasing, and provides practical guidance for organizations evaluating which model fits their usage patterns.

## Study Tips

- **Build a comparison matrix as you read:** Create a table with columns for each purchasing option (On-Demand, Spot, Standard RI, Convertible RI, Compute SP, EC2 Instance SP, Capacity Reservation) and rows for discount level, commitment required, flexibility, capacity guarantee, and ideal use case. Fill it in as you progress through the resources -- this becomes an invaluable reference.

- **Focus on the "commitment vs flexibility" spectrum:** The purchasing options form a clear spectrum from maximum flexibility and no discount (On-Demand) to maximum discount and minimum flexibility (Standard RI, All Upfront, 3-year). Understanding where each option sits on this spectrum is more valuable than memorizing exact discount percentages.

- **Pay attention to what "scope" means for RIs and Capacity Reservations:** A Zonal RI reserves capacity in a specific AZ and provides a discount, while a Regional RI provides a discount across all AZs in a region but does not reserve capacity. This distinction trips up many people and is a common exam and interview question. On-Demand Capacity Reservations fill the gap by providing capacity without a billing discount.

## Next Steps

After completing this study plan, consider exploring:

1. **AWS Cost Explorer and recommendations** -- Learn how to use Cost Explorer to analyze your usage patterns and get automated recommendations for Reserved Instance and Savings Plan purchases.

2. **EC2 Auto Scaling with Spot Instances** -- Understand how Auto Scaling groups handle mixed instance policies combining On-Demand and Spot, including capacity rebalancing and allocation strategies.

3. **AWS Compute Optimizer** -- Explore how this service analyzes actual utilization to recommend optimal instance types and purchasing options based on your workload patterns.

4. **Hands-on practice** -- Use the AWS Pricing Calculator to model different purchasing scenarios. Compare total cost of ownership for a workload using various combinations of On-Demand, Spot, RIs, and Savings Plans over 1-year and 3-year horizons.
