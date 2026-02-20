# VPC Peering vs Transit Gateway -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-02-20
**Difficulty:** Intermediate-Advanced

## Overview

This study plan covers the three primary mechanisms for connecting VPCs in AWS -- VPC Peering, Transit Gateway, and AWS PrivateLink -- with a focus on understanding when to use each, how their routing models differ, and how they interact with the CIDR planning you learned yesterday. After completing it, you will be able to articulate the architectural trade-offs between point-to-point peering and hub-and-spoke Transit Gateway designs, explain why PrivateLink solves the overlapping CIDR problem, and make defensible connectivity decisions for organizations ranging from 2 VPCs to 100+. This plan assumes you already understand VPC CIDR planning, subnet tiers, route tables, and VPC endpoints from the VPC Advanced study session.

## Resources

### 1. How VPC Peering Connections Work -- Official Documentation (AWS Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html
- **Type:** Official Docs
- **Summary:** The definitive reference for VPC peering mechanics, covering the full connection lifecycle (initiating-request through active to deleted), the critical limitation of no transitive peering (if A peers with B and B peers with C, A cannot reach C through B), and the edge-to-edge routing restriction (you cannot use a peered VPC's internet gateway, NAT Gateway, VPN connection, or Direct Connect link). Also covers the overlapping CIDR prohibition -- you cannot peer two VPCs if any of their CIDR blocks overlap, even secondary CIDRs you do not intend to route. This directly connects to yesterday's CIDR planning lesson: the non-overlapping requirement is not just a best practice, it is a hard technical constraint enforced at peering creation time. Read the limitations section carefully -- these constraints are the primary reason Transit Gateway and PrivateLink exist.

### 2. What is a Transit Gateway and How It Works -- Official Documentation (AWS Docs) -- 25 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html
- **Type:** Official Docs
- **Summary:** The comprehensive conceptual overview of Transit Gateway as a regional virtual router operating at layer 3. Covers resource attachments (VPCs, VPN connections, Direct Connect gateways, peering connections, network firewalls), route table mechanics (static vs propagated routes, route evaluation order, ECMP support), and availability zone behavior. The most valuable sections are the six real-world architecture examples: centralized router connecting multiple VPCs, isolated routers for network segmentation (separate route tables for prod vs dev), shared services VPC accessible from all spoke VPCs, Transit Gateway peering across regions, centralized internet egress through a dedicated VPC, and appliance-based traffic inspection using appliance mode. Pay special attention to the route table association and propagation model -- each attachment is associated with exactly one route table (which determines where its outbound traffic goes) but can propagate its routes to multiple route tables (which determines who can reach it). This association-vs-propagation distinction is the key mental model for Transit Gateway routing.

### 3. Multi-VPC Whitepaper: VPC Peering, Transit Gateway, and PrivateLink Sections (AWS Whitepaper) -- 25 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/vpc-peering.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** Three sections of the AWS multi-VPC whitepaper (updated April 2024) that form the authoritative comparison of all three connectivity options. Start with the VPC Peering section (URL above) which covers the full mesh scaling problem -- connecting 100 VPCs requires 4,950 peering connections using the formula n(n-1)/2, with a maximum of 125 active peering connections per VPC. Then read the Transit Gateway section at https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-gateway.html which covers the hub-and-spoke model supporting thousands of VPCs, cross-account sharing via AWS RAM, route table segmentation for environment isolation, and hybrid connectivity consolidation. Finally read the PrivateLink section at https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/aws-privatelink.html which explains the provider-consumer model, the critical capability of working with overlapping CIDRs (because PrivateLink uses ENIs in the consumer VPC, not IP routing between VPCs), and how real architectures use all three technologies together. The whitepaper concludes that enterprises typically use PrivateLink for API-style client-server connectivity, VPC peering for low-latency direct connections between a small number of VPCs, and Transit Gateway to simplify connectivity at scale plus consolidate hybrid access.

### 4. AWS PrivateLink Concepts -- Official Documentation (AWS Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html
- **Type:** Official Docs
- **Summary:** The reference for understanding PrivateLink's provider-consumer architecture, which is fundamentally different from peering and Transit Gateway. With peering and TGW, both VPCs get full layer-3 IP connectivity to each other's CIDR ranges. With PrivateLink, the service provider creates a Network Load Balancer fronting their application and registers it as a VPC endpoint service; the consumer creates an interface endpoint (ENI with a private IP in their own subnet) that tunnels traffic to the provider's NLB. Traffic is unidirectional (consumer initiates to provider), the two VPCs never exchange routes, and overlapping CIDRs are perfectly fine because the ENI lives in the consumer's address space. This is the solution when teams independently provisioned VPCs with the same 10.0.0.0/16 CIDR (the spreadsheet anti-pattern from yesterday's lesson) and now need connectivity. Also covers the connection lifecycle where the provider must explicitly accept or auto-accept connection requests, giving the service owner control over who can connect.

### 5. VPC Peering vs Transit Gateway and Beyond -- Real-World Case Study (Ably Engineering Blog) -- 20 min
- **URL:** https://ably.com/blog/aws-vpc-peering-vs-transit-gateway-and-beyond
- **Type:** Blog (Industry -- Ably)
- **Summary:** A detailed engineering case study from Ably (a real-time messaging platform operating across 8 AWS regions) documenting their architectural decision between VPC peering and Transit Gateway for their production network. Unlike the AWS docs which explain what each service does, this article shows how a real team evaluated the trade-offs. Ably chose VPC peering over Transit Gateway because they process over 500,000 packets per second through existing connections, and Transit Gateway's regional peering limit of 5 million packets per second gave them less than 10x headroom for growth. The cost analysis is revealing: Transit Gateway would add approximately $20,000 per petabyte of extra data processing charges. The article also covers their IPAM strategy (per-region, per-environment IP pooling), multi-account architecture using AWS RAM to share VPCs, and IPv6 dual-stack implementation. This is essential reading because it demonstrates that the "right" answer depends entirely on your traffic patterns, scale, and cost constraints -- there is no universally correct choice.

### 6. Field Notes: Working with Route Tables in AWS Transit Gateway (AWS Architecture Blog) -- 15 min
- **URL:** https://aws.amazon.com/blogs/architecture/field-notes-working-with-route-tables-in-aws-transit-gateway/
- **Type:** Blog (Official AWS)
- **Summary:** A practical deep-dive into Transit Gateway route table mechanics from the AWS Architecture Blog. Covers the two key concepts that confuse most people: route table association (which route table does this attachment's traffic get evaluated against when leaving the TGW) and route propagation (which route tables learn about this attachment's CIDR so they can route traffic toward it). Walks through packet flow when source and destination are in the same route table vs different route tables, which is the foundation for network segmentation patterns like isolating production from development. This is the bridge between understanding Transit Gateway conceptually (resource 2) and being able to design real route table configurations for your Terraform build this evening. The diagrams showing packet flow through association and propagation are particularly valuable -- draw them out yourself as you read.

### 7. Integrating Transit Gateway with PrivateLink and Route 53 Resolver (AWS Networking Blog) -- 10 min
- **URL:** https://aws.amazon.com/blogs/networking-and-content-delivery/integrating-aws-transit-gateway-with-aws-privatelink-and-amazon-route-53-resolver/
- **Type:** Blog (Official AWS)
- **Summary:** Shows how the three connectivity technologies work together in a real architecture rather than as isolated choices. The pattern: spoke VPCs connect through Transit Gateway to a shared-services VPC that hosts centralized PrivateLink endpoints for AWS services (like S3 interface endpoints), and Route 53 Resolver forwards DNS queries so that spoke VPCs resolve service endpoints to the centralized PrivateLink ENI IPs rather than public IPs. This is the enterprise pattern you will see in every large AWS organization -- it centralizes endpoint costs (one set of interface endpoints shared across all VPCs instead of per-VPC endpoints) and provides consistent DNS resolution. Understanding this integration pattern is critical for the Transit Gateway deep-dive coming on Day 3 and for Week 7 when you build the multi-account landing zone project.

## Study Tips

- **Build a decision matrix as you read.** Create a table with rows for each connectivity option (VPC Peering, Transit Gateway, PrivateLink) and columns for: max connections, transitive routing support, cross-region support, overlapping CIDR support, data transfer cost, hourly cost, bandwidth limits, on-premises access, and best use case. Fill it in from each resource. By the end, you should be able to glance at this matrix and immediately identify the right tool for any scenario. This matrix will reappear in interview questions.

- **Calculate the full mesh scaling problem yourself.** Use n(n-1)/2 to compute peering connections needed for 5, 10, 20, 50, and 100 VPCs (10, 45, 190, 1225, 4950). Then compare with Transit Gateway where each VPC needs just one attachment regardless of how many other VPCs exist. The inflection point where Transit Gateway becomes cheaper despite its per-GB data processing charge ($0.02/GB) is the key cost question -- work through the math for your specific traffic patterns to build intuition.

- **Connect today's learning to yesterday's CIDR planning.** The non-overlapping CIDR requirement you studied yesterday exists primarily because of VPC peering and Transit Gateway. PrivateLink is the escape hatch when CIDR planning was not done properly. As you read about each technology, think back to the hierarchical CIDR allocation table from yesterday and consider how each connectivity option would work across those VPCs.

## Next Steps

After completing this study plan, consider exploring:

1. **Transit Gateway deep dive (Day 3)** -- Tomorrow covers Transit Gateway route tables in depth, including route propagation, segmented routing with multiple route tables (prod, dev, shared-services isolation), TGW peering across regions, and TGW Connect for SD-WAN integration. Today's conceptual understanding of association vs propagation is the prerequisite.

2. **Terraform lab: VPC Peering between 2 VPCs** -- This evening's Block 5 build. Create two VPCs with non-overlapping CIDRs from yesterday's IPAM hierarchy, establish a peering connection, update route tables in both VPCs, deploy EC2 instances in each, and test connectivity with ping. The Terraform resources are aws_vpc_peering_connection, aws_vpc_peering_connection_accepter (for cross-account), and aws_route entries pointing to the peering connection ID.

3. **PrivateLink hands-on** -- After the peering lab, try creating a simple PrivateLink service: deploy an NLB in one VPC fronting a simple web server, create a VPC endpoint service, then create an interface endpoint in a second VPC and access the service through the endpoint DNS name. This makes the provider-consumer model concrete.

4. **Cost modeling exercise** -- Build a spreadsheet comparing monthly costs for connecting 10 VPCs using full-mesh peering (45 connections, data transfer only) vs Transit Gateway (10 attachments at $0.05/hr each + $0.02/GB) for various traffic volumes (100 GB, 1 TB, 10 TB per month). The crossover point where TGW becomes more cost-effective despite per-GB charges reveals why cost alone should not drive the decision.
