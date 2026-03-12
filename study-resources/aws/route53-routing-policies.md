# Route53 Routing Policies and Health Checks -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-03-11
**Difficulty:** Intermediate

## Overview

This study plan covers Amazon Route53's seven routing policies (Simple, Weighted, Latency, Failover, Geolocation, Geoproximity, Multivalue Answer), the health check system that underpins them, and how to combine policies into complex routing trees using alias records and Traffic Flow. After completing it, you will understand which routing policy to select for any given architecture scenario, how health checks interact with each policy to remove unhealthy endpoints from DNS responses, the difference between active-active and active-passive failover patterns, and how to layer policies together (e.g., latency at the top level routing to weighted records within each region) for production multi-region architectures -- all of which builds directly toward your Week 5 DR strategies and your Week 8 multi-region project.

## Resources

### 1. Choosing a Routing Policy -- Official Overview ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html
- **Type:** Official Docs
- **Summary:** The gateway page that lists all eight routing policies (Simple, Failover, Geolocation, Geoproximity, Latency, IP-based, Multivalue Answer, Weighted) with one-sentence descriptions of when to use each; read this first to build the mental map of what options exist, then use it as a navigation hub -- each policy links to its own detailed page, and you will visit the most important ones in later resources; the brevity is intentional, this is about orientation before depth

### 2. Choosing the Right Routing Policy with Route 53 -- AWS Fundamentals Blog ⏱️ 15 min
- **URL:** https://awsfundamentals.com/blog/how-to-pick-the-correct-routing-policy-with-route-53
- **Type:** Blog (AWS Fundamentals)
- **Summary:** A well-structured decision guide that walks through five core routing policies (Simple, Weighted, Failover, Latency, Geolocation) with clear use-case explanations and diagrams showing how traffic flows in each pattern; the value here is the practical framing -- Weighted for blue/green deployments and canary releases, Failover for DR with health checks, Latency for multi-region performance optimization, Geolocation for compliance and content localization; this fills the gap between the terse official overview and the dense per-policy doc pages by explaining the "why" behind each choice

### 3. Weighted, Latency, Failover, and Geoproximity Routing -- Official Docs Deep Dive ⏱️ 25 min
- **URL (start here):** https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html
- **Type:** Official Docs (read four pages sequentially)
- **Summary:** Read these four official pages in order: **Weighted** (traffic distribution by percentage, weight-of-zero fallback behavior, health check integration), **Latency** (https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html -- how Route53 measures latency to AWS regions, not to individual endpoints), **Failover** (https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html -- primary/secondary record designation, mandatory health checks on primary), and **Geoproximity** (https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geoproximity.html -- the bias mechanism that expands or shrinks routing regions, requires Traffic Flow, supports both AWS and non-AWS resources with latitude/longitude); pay special attention to the weighted routing zero-weight fallback (if all nonzero-weight records are unhealthy, Route53 considers zero-weight records) and the geoproximity bias math (positive bias halves the perceived distance, negative bias doubles it) -- these are common exam and interview trip-ups

### 4. Active-Active and Active-Passive Failover -- Official Docs ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html
- **Type:** Official Docs
- **Summary:** The essential page for understanding Route53's two failover models: active-active (use any policy except failover -- weighted, latency, geolocation, multivalue -- and attach health checks so unhealthy records are removed from responses, all healthy resources serve traffic simultaneously) versus active-passive (use the failover routing policy specifically, designate primary and secondary records, secondary only serves traffic when primary fails health checks); the page also covers three active-passive configuration patterns -- single primary/secondary, multiple primary resources behind weighted records wrapped in failover alias records, and the weighted-record alternative -- which directly connects to the complex routing trees you will study next

### 5. Health Checks in Complex Configurations -- Official Docs ⏱️ 20 min
- **URL:** https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-complex-configs.html
- **Type:** Official Docs
- **Summary:** The most architecturally important page in this list -- it shows how to build a decision tree by layering routing policies: latency routing at the top level selects the best region, weighted routing in the middle distributes traffic across instances within that region, and health checks at the bottom monitor each individual endpoint; when a branch becomes entirely unhealthy, Route53 backs up the tree and tries the next branch; the key concept is "Evaluate Target Health" on alias records, which lets health status propagate up through the tree without creating separate health checks for each alias; read the diagrams carefully to understand how Route53 traverses the tree -- this is exactly the pattern you will use in your Week 8 multi-region DR project with failover between primary and DR regions

### 6. Route53 Health Check Types -- Official Docs ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-types.html
- **Type:** Official Docs
- **Summary:** Covers the four health check types: **Endpoint** (HTTP/HTTPS/TCP probes from global health checker locations, configurable interval of 10s or 30s, failure threshold of 1-10 consecutive failures, optional string matching in response body), **Calculated** (aggregates multiple child health checks with AND/OR/threshold logic -- "healthy if at least 3 of 5 children are healthy"), **CloudWatch Alarm** (monitors the CloudWatch data stream directly rather than waiting for alarm state transition, useful for private resources not reachable by Route53's public health checkers), and **Route53 Application Recovery Controller** (simple on/off routing controls for manual or automated regional failover); the calculated health check type is critical for complex architectures where you need a single parent health check that represents an entire stack's health across multiple components

### 7. Route53 Health Checks: Types, DNS Failover, and Setup -- StormIT Blog ⏱️ 15 min
- **URL:** https://www.stormit.cloud/blog/route-53-health-check/
- **Type:** Blog (StormIT)
- **Summary:** A practical, visual guide that complements the official docs by walking through an actual active-passive failover setup with screenshots -- two regions (Frankfurt and Ireland) with ALBs and Auto Scaling groups, health check creation with 10-second interval and failure threshold of 1, failover routing record configuration with primary and secondary designations; also covers the active-active approach using weighted routing with health checks, explains the 75-90 second DNS failover window (reducible to 25-30 seconds with fast-interval health checks), and includes a useful FAQ section on common issues like health checks showing healthy when endpoints are down (check security groups -- Route53 health checkers come from published IP ranges that must be allowed through)

### 8. Creating Disaster Recovery Mechanisms Using Amazon Route 53 -- AWS Blog ⏱️ 5 min
- **URL:** https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/
- **Type:** Blog (AWS Networking & Content Delivery)
- **Summary:** A concise AWS blog post covering three DR failover strategies built on Route53: automated failover using health checks (data plane operation, no dependency on control plane APIs), manual failover using Route53 Application Recovery Controller routing controls (enables deliberate failover decisions without modifying DNS records), and the "Standby Takes Over Primary" (STOP) pattern where the DR region promotes itself; the critical architectural insight is that Route53 health checks and ARC routing controls operate on the data plane while the Route53 console and management APIs run on the control plane in us-east-1 -- so your failover mechanism must never depend on the control plane being available during the event that triggered the failover

## Study Tips

- Build a comparison table as you read: for each of the seven routing policies, note the use case, whether it supports health checks, whether it can be combined with other policies via alias records, and the key configuration parameter (weight for Weighted, region for Latency, continent/country for Geolocation, bias value for Geoproximity). This table will become your interview cheat sheet and your Terraform reference when building records tomorrow.
- Pay close attention to the difference between Geolocation and Geoproximity: Geolocation routes based on the user's mapped location (continent, country, or US state -- hard boundaries, no overlap) while Geoproximity routes based on the geographic distance between user and resource (soft boundaries that can be shifted with bias values). Geolocation is for compliance ("EU users must hit EU servers"), Geoproximity is for performance optimization with fine-grained control ("shift 20% more traffic to the new us-west-2 deployment"). Mixing these up is a common interview mistake.
- Trace the health check propagation path in the complex configuration diagrams: an individual endpoint health check fails, the weighted record for that endpoint is removed from the pool, if all weighted records in a region become unhealthy the latency alias record's "Evaluate Target Health" propagates the unhealthy status upward, and Route53 backs out to the next-best-latency region. Understanding this cascade is essential for designing reliable multi-region architectures.

## Next Steps

- **Route53 Advanced -- Hybrid DNS (tomorrow):** Covers Private Hosted Zones, split-view DNS, DNSSEC, and Resolver Endpoints for on-premises integration -- builds directly on today's routing policy knowledge by adding the private DNS dimension and hybrid connectivity patterns that connect to your Direct Connect and VPN learning from Weeks 1-2.
- **Terraform implementation (tonight):** Deploy a Route53 hosted zone with weighted routing records using `aws_route53_record` with `weighted_routing_policy` blocks, attach `aws_route53_health_check` resources to each record, and verify that Route53 removes unhealthy records from responses; the `set_identifier` parameter is required for all routing policies except Simple.
- **DR architecture preview:** The failover patterns you learned today (especially the complex configuration tree and ARC routing controls) will be the foundation of your Week 5 DR strategies and Week 8 multi-region project -- when you reach those topics, you will design the Route53 layer first and build the compute/database layers underneath it.
