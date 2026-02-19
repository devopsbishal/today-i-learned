# VPC Advanced: CIDR Planning and Subnet Strategy -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-02-19
**Difficulty:** Intermediate-Advanced

## Overview

This study plan covers enterprise-level VPC design focusing on CIDR planning for multi-VPC environments with 10+ VPCs, subnet tier architecture (public, private, isolated), route table design patterns, NAT Gateway high availability including the new regional NAT Gateway feature, and VPC endpoint strategy. After completing it, you will be able to design a non-overlapping IP addressing scheme for a multi-account AWS organization, architect multi-AZ subnet layouts with proper route table associations, and make informed decisions about NAT Gateway placement and VPC endpoint types. This plan assumes you already understand basic VPC concepts (subnets, route tables, IGW, NAT, security groups) and builds toward enterprise-scale design decisions.

## Resources

### 1. VPC CIDR Blocks -- Official Documentation (AWS Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html
- **Type:** Official Docs
- **Summary:** The definitive reference for VPC CIDR block rules that govern all your planning decisions. Covers primary vs secondary CIDR blocks (you can add secondary CIDRs later but cannot shrink them), the allowed range of /16 to /28, the RFC 1918 private ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16), prohibited CIDR blocks (notably 172.17.0.0/16 which conflicts with Docker and SageMaker), and the critical restriction that if your VPC has a CIDR from one RFC 1918 range you cannot add secondary CIDRs from other RFC 1918 ranges without first adding a public routable CIDR. Pay close attention to the VPC peering restrictions on CIDR additions -- these constraints directly impact how you plan multi-VPC environments. Read the subnet sizing page linked from here as well, and internalize the 5 reserved IPs per subnet (network address, VPC router, DNS server, future use, broadcast).

### 2. Building a Scalable and Secure Multi-VPC AWS Network Infrastructure -- IP Addressing Section (AWS Whitepaper) -- 25 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** The authoritative AWS whitepaper for enterprise network architecture, updated April 2024. Focus on the "IP address planning and management" section, then skim the centralized egress and centralized VPC endpoint sections. The IP addressing section covers hierarchical IP addressing schemes (allocating by region, then by business unit, then by environment), using Amazon VPC IPAM for centralized CIDR management across accounts, and preventing overlaps in multi-account organizations. The centralized egress section at https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-egress-to-internet.html explains the architecture where spoke VPCs route all egress traffic through a dedicated egress VPC with NAT Gateways via Transit Gateway -- a pattern you will encounter in every enterprise AWS environment. The centralized VPC endpoints section at https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-access-to-vpc-private-endpoints.html covers sharing interface endpoints across VPCs to reduce cost. Skim rather than deep-read the full whitepaper -- focus on IP planning, centralized egress, and centralized endpoints.

### 3. Amazon VPC IP Address Manager Best Practices (AWS Networking Blog) -- 15 min
- **URL:** https://aws.amazon.com/blogs/networking-and-content-delivery/amazon-vpc-ip-address-manager-best-practices/
- **Type:** Blog (Official AWS)
- **Summary:** The practical companion to the whitepaper's IP addressing section. Covers how IPAM organizes IP space into a hierarchy of pools (top-level pool split into regional pools, then into environment pools like dev/staging/prod), how allocation rules enforce minimum and maximum netmask lengths to prevent teams from requesting oversized CIDRs, and how IPAM integrates with AWS Organizations for cross-account IP management. The key architectural insight is the four-tier pool hierarchy: top-level pool (entire 10.0.0.0/8 allocation) to regional pools (10.0.0.0/10 for us-east-1, 10.64.0.0/10 for eu-west-1) to business unit pools to environment pools. This is how enterprises prevent the "spreadsheet of CIDR allocations" anti-pattern that leads to overlapping ranges and VPC peering failures. Also covers monitoring CIDR utilization to identify when pools are running low, so you can plan secondary CIDR additions before hitting capacity.

### 4. Decide on AWS Account VPC Subnet CIDR Strategy (Cloud Posse Reference Architecture) -- 15 min
- **URL:** https://docs.cloudposse.com/layers/network/design-decisions/decide-on-aws-account-vpc-subnet-cidr-strategy/
- **Type:** Blog (Community -- Cloud Posse)
- **Summary:** A real-world, opinionated subnet CIDR strategy from Cloud Posse, a well-known cloud infrastructure consultancy. Unlike the AWS docs which describe what is possible, this resource prescribes a specific allocation pattern: each account gets a /16, each region within an account gets a /18, and subnets are carved as /21 blocks for EKS workloads (2,046 usable IPs per subnet, critical for pod networking) and /24 blocks for single-purpose subnets. This is one of the few resources that shows you a complete, concrete CIDR allocation table from the account level down to individual subnets, making it invaluable for understanding how the abstract IPAM hierarchy translates into actual numbers. Their warning about never going below /19 for Kubernetes subnets is critical -- underprovisioned subnets cause pod scheduling failures that are painful to remediate in production.

### 5. Example Routing Options -- Route Table Design Patterns (AWS Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/userguide/route-table-options.html
- **Type:** Official Docs
- **Summary:** A collection of 11 routing scenarios that demonstrates how route tables work in practice. The critical patterns for this study plan are: (1) the NAT Gateway routing pattern showing how private subnets route 0.0.0.0/0 to a NAT Gateway while using more specific routes for S3 gateway endpoints and VPC peering connections to avoid unnecessary NAT charges, (2) the Transit Gateway routing pattern showing route propagation from VPN connections, and (3) the gateway route table pattern for inbound traffic inspection through middlebox appliances. Pay special attention to longest prefix match -- AWS always uses the most specific route, which is why a gateway endpoint route for S3 (prefix list) takes priority over the 0.0.0.0/0 NAT route. Also note that static routes take priority over propagated routes when destinations are identical, which matters when combining Transit Gateway with VPN.

### 6. Regional NAT Gateways for Automatic Multi-AZ Expansion (AWS Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html
- **Type:** Official Docs
- **Summary:** A significant architectural simplification announced in November 2025. Traditional zonal NAT Gateways required one NAT Gateway per AZ, one public subnet per AZ to host them, and separate route table entries per AZ -- leading to complex infrastructure code and higher costs. Regional NAT Gateways automatically expand and contract across AZs based on workload presence, support up to 32 IP addresses per AZ (vs 8 for zonal), and eliminate the requirement for public subnets entirely. In automatic mode, AWS manages IP allocation and AZ expansion; in manual mode, you control IP addresses and AZ presence. The key architectural impact: you can now use a single route table entry (0.0.0.0/0 pointing to one regional NAT Gateway ID) for all private subnets across all AZs, instead of maintaining per-AZ route tables. This is the new recommended approach for most workloads and fundamentally changes how you design multi-AZ NAT architectures.

### 7. Choosing Your VPC Endpoint Strategy for Amazon S3 (AWS Architecture Blog) -- 15 min
- **URL:** https://aws.amazon.com/blogs/architecture/choosing-your-vpc-endpoint-strategy-for-amazon-s3/
- **Type:** Blog (Official AWS)
- **Summary:** A decision framework for the most common VPC endpoint question: when to use a gateway endpoint vs an interface endpoint for S3. Gateway endpoints are free and have no throughput limit but work only within a single VPC (traffic must originate from the VPC where the endpoint is created), require route table modifications, and cannot be accessed from on-premises or peered VPCs. Interface endpoints cost money (~$0.01/hour per AZ plus data processing) but create ENIs with private IPs that can be reached from on-premises via VPN/Direct Connect, from peered VPCs, and from other regions via Transit Gateway. The enterprise pattern is to use gateway endpoints for intra-VPC S3 traffic (free, no bandwidth limit) and interface endpoints in a shared services VPC for cross-VPC and on-premises S3 access. This dual strategy optimizes both cost and connectivity -- a common interview topic for solutions architect discussions.

### 8. Enforce Non-Overlapping Private IP Address Ranges -- Well-Architected Framework (AWS Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_network_topology_non_overlap_ip.html
- **Type:** Official Docs
- **Summary:** The Well-Architected Framework's reliability pillar guidance on IP address management, framed as an auditable best practice (REL02-BP05). Covers the step-by-step process for achieving non-overlapping ranges: capture current CIDR consumption using APIs and IPAM discovery, document subnet usage per VPC per region, analyze for existing overlaps, and remediate using either migration to new ranges, Private NAT Gateway (for connectivity between VPCs with overlapping CIDRs), or AWS PrivateLink. Lists the common anti-patterns to avoid: using identical IP ranges on-premises and in AWS, tracking IPs manually in spreadsheets, not monitoring VPC IP utilization, and over-sizing or under-sizing CIDR blocks. Read this last as a checklist to validate your understanding of everything covered in the earlier resources.

## Study Tips

- **Build a concrete CIDR allocation table as you read.** Take a hypothetical organization with 3 environments (dev, staging, prod), 2 regions (us-east-1, eu-west-1), and 4 accounts per environment. Starting from 10.0.0.0/8, work out the full hierarchy: regional pools, account-level VPC CIDRs, and per-VPC subnet allocations with public, private, and isolated tiers across 3 AZs. Use https://visualsubnetcalc.com/ to validate your math. This exercise forces you to confront real constraints (how many /16s fit in a /8? only 256 -- so with 12 accounts across 2 regions, that is 24 VPCs minimum, and you need room for growth). Working the numbers on paper is what separates "I understand VPC CIDR planning" from actually being able to do it.

- **Draw the route table associations for a complete 3-tier, 3-AZ VPC.** Sketch a VPC with public subnets (route to IGW), private subnets (route to NAT), and isolated subnets (no internet route at all -- only local and VPC endpoint routes). Show which subnets share route tables and which need their own. With regional NAT Gateway, you can now share one route table across all private subnets in all AZs; with zonal NAT Gateways, each AZ needs its own private route table. Drawing this makes the operational difference between the two approaches visceral.

- **Compare the cost of NAT Gateway vs VPC endpoints for your most common traffic patterns.** NAT Gateway charges $0.045/hour plus $0.045/GB processed. If your private subnets send 1TB/month to S3, that is $45 in NAT data processing alone -- but a gateway endpoint is free. Interface endpoints cost $0.01/hour per AZ (about $7.20/month per AZ). Run these numbers for 3 AZs, 5 services, and 2TB of monthly S3 traffic to build intuition for when endpoints pay for themselves.

## Next Steps

After completing this study plan, consider exploring:

1. **Hands-on CIDR planning lab** -- Use Terraform with the AWS IPAM provider to create a hierarchical pool structure, allocate VPC CIDRs from pools, and deploy a multi-AZ VPC with public, private, and isolated subnet tiers. This makes the abstract CIDR hierarchy concrete and teaches you the Terraform resource relationships (aws_vpc_ipam, aws_vpc_ipam_pool, aws_vpc_ipam_pool_cidr, aws_vpc).

2. **Transit Gateway deep dive** -- Study Transit Gateway route tables, route propagation, attachment types, and multi-account sharing via AWS RAM. Transit Gateway is the hub that connects your carefully planned non-overlapping VPCs, and its routing model (separate route tables per attachment type) introduces its own design decisions.

3. **VPC Flow Logs and network observability** -- Learn how to use VPC Flow Logs to validate that your route table design and endpoint strategy are working as intended. Flow logs reveal whether traffic is taking the expected path (through endpoints vs NAT vs direct) and help identify misconfigured route table associations.

4. **IPv6 and dual-stack VPC design** -- All new VPCs should consider dual-stack for future-proofing. IPv6 eliminates the IP exhaustion problem entirely but introduces its own subnet sizing rules (/44 to /64 in /4 increments) and requires egress-only internet gateways instead of NAT for outbound-only access.
