# AWS Transit Gateway Deep Dive -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-02-23
**Difficulty:** Intermediate-Advanced

## Overview

This study plan goes deep on AWS Transit Gateway architecture, building on the comparison-level understanding from the VPC Peering vs Transit Gateway session (Feb 20). You already know what TGW is, how association differs from propagation at a conceptual level, and why TGW replaces full-mesh peering. Today you will master the internal mechanics: how route tables enforce network segmentation with multiple isolation zones, how centralized egress and inspection architectures route traffic through dedicated VPCs, how inter-region peering extends your network globally with static routes, what appliance mode does at the packet level, and the quotas and cost patterns that constrain real-world designs. After completing this plan, you will be able to design and implement a multi-VPC Transit Gateway topology with route table segmentation -- the exact Terraform build planned for this evening.

## Resources

### 1. Transit Gateway Design Best Practices -- AWS Official Docs ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html
- **Type:** Official Docs
- **Summary:** A concise but authoritative list of 9 design best practices from AWS covering subnet sizing for TGW attachments (use dedicated /28 subnets), why NACLs on TGW subnets should be left open (apply filtering on workload subnets instead), when to use single vs multiple route tables, why BGP VPN connections with multipath should be preferred, that you never need multiple TGWs for HA within a region (TGW is highly available by design), and the MTU mismatch risk when migrating from VPC peering. Read this first as a checklist of principles that will frame everything else you read today.

### 2. Centralized Egress to Internet -- AWS Multi-VPC Whitepaper ⏱️ 25 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/using-nat-gateway-for-centralized-egress.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** The definitive reference for the centralized egress pattern where spoke VPCs route 0.0.0.0/0 through TGW to a dedicated egress VPC containing NAT Gateways and an IGW. This section explains the two-route-table design in detail: RT1 (associated with spoke attachments) has a static 0.0.0.0/0 pointing to the egress VPC attachment plus optional blackhole routes for 10.0.0.0/8 to prevent inter-VPC traffic; RT2 (associated with the egress VPC attachment) has propagated spoke routes so return traffic reaches the correct spoke. The egress VPC's own subnet route tables then forward traffic from TGW ENIs to NAT Gateways in the same AZ. Critically, it covers the cost trade-off: centralized egress saves NAT Gateway hourly costs (one set instead of N) but adds TGW data processing charges ($0.02/GB), and in edge cases with massive egress volume from a single VPC, keeping NAT local may be cheaper. Also covers NAT Gateway scaling limits: 55,000 simultaneous connections per IP per destination, up to 8 IPs (440,000 connections), and 5 Gbps baseline scaling to 100 Gbps.

### 3. Centralized Network Security with Transit Gateway -- AWS Multi-VPC Whitepaper ⏱️ 20 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-network-security-for-vpc-to-vpc-and-on-premises-to-vpc-traffic.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** Covers the centralized inspection architecture for east-west (VPC-to-VPC) and north-south (on-prem-to-VPC) traffic using a dedicated security VPC. Explains the four security zones (untrusted, production, development, security), the two-subnet-per-AZ design in the inspection VPC (one for TGW attachment, one for firewall endpoints), and why Network Firewall cannot inspect traffic in the same subnet as its endpoints. Most importantly, this is where you will learn appliance mode in depth: when enabled on a TGW attachment, TGW uses a flow hash algorithm to select a single network interface for the entire life of a connection, ensuring symmetric routing so both directions of a flow pass through the same AZ's firewall. Without appliance mode, return traffic might traverse a different AZ, breaking stateful inspection. Also covers the Gateway Load Balancer alternative for third-party appliances (Palo Alto, Fortinet) where GWLB provides transparent L4 load balancing with 5-tuple/3-tuple flow stickiness.

### 4. Transit Gateway Peering Attachments -- AWS Official Docs ⏱️ 15 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html
- **Type:** Official Docs
- **Summary:** The reference for inter-region and intra-region TGW peering. Key details: peering uses static routes only (no dynamic propagation across peered TGWs), so you must manually add routes in each TGW's route table pointing to the peering attachment; traffic is encrypted with AES-256 at the virtual network layer for inter-region peering and double-encrypted (virtual + physical) on links outside AWS physical control; each pair of TGWs can have only one peering attachment between them; unique ASNs are recommended for each TGW; and DNS resolution does not work across inter-region peering (you cannot resolve private hosted zone records from a VPC in another region via peering alone -- you need Route 53 Resolver endpoints, which you will cover in Week 3). This is directly relevant to your Week 5 multi-region DR architecture and Week 7 multi-account landing zone project.

### 5. Transit Gateway Quotas -- AWS Official Docs ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html
- **Type:** Official Docs
- **Summary:** The authoritative quota reference that every TGW architect must internalize. Key numbers: 5 TGWs per account per region (adjustable), 5,000 attachments per TGW (adjustable), 20 route tables per TGW (adjustable), 10,000 total combined routes across all route tables (contact SA/TAM to increase), 50 peering attachments per TGW (adjustable), only 1 peering attachment between any two TGWs, up to 100 Gbps bandwidth per VPC attachment per AZ, 7.5 million packets per second per attachment per AZ, and MTU of 8,500 bytes for VPC/DX/peering attachments but only 1,500 bytes for VPN. The 10,000 total routes limit is the one that catches people at scale -- it is shared across ALL route tables, not per table. Knowing these numbers cold is essential for interviews and for designing architectures that will not hit ceilings.

### 6. Centralize Network Connectivity Using Transit Gateway -- AWS Prescriptive Guidance ⏱️ 20 min
- **URL:** https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/centralize-network-connectivity-using-aws-transit-gateway.html
- **Type:** Official Docs (Prescriptive Guidance)
- **Summary:** A step-by-step implementation pattern for building a multi-account Transit Gateway architecture. Walks through the full lifecycle: creating the TGW in a network services account, setting up a Site-to-Site VPN attachment, enabling cross-account sharing via AWS Organizations, creating a RAM resource share for the TGW, having spoke accounts create VPC attachments, accepting those attachments, configuring route table entries in both the TGW and the spoke VPC route tables, and testing connectivity. The key insight for multi-account is that the TGW lives in a centralized networking account and is shared via RAM -- spoke accounts can attach their VPCs but cannot modify the TGW route tables, preserving centralized control. This pattern directly feeds into your Week 7 multi-account landing zone project and tonight's Terraform build where you will create the TGW in a shared context with 3 VPCs.

### 7. Using Gateway Load Balancer with Transit Gateway for Centralized Security -- AWS Multi-VPC Whitepaper ⏱️ 20 min
- **URL:** https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/using-gwlb-with-tg-for-cns.html
- **Type:** Official Docs (Whitepaper)
- **Summary:** Complements Resource 3 by covering the GWLB-based inspection pattern for when you need third-party virtual appliances instead of AWS Network Firewall. The traffic flow is: spoke VPC sends traffic to TGW, TGW routes to the security VPC attachment, the security VPC forwards to a GWLB endpoint, GWLB distributes across registered appliance targets using transparent GENEVE encapsulation, appliances inspect and return traffic through GWLB, then back to TGW. Key architectural detail: appliance mode MUST be enabled on the TGW for east-west inspection so that both directions of a flow traverse the same appliance in the same AZ. GWLB uses 5-tuple or 3-tuple hashing for session stickiness. The decision between Network Firewall (managed, stateful L3-L7) and GWLB + appliances (self-managed, custom logic) is a common interview question. Both scale to 100 Gbps per endpoint but differ in operational overhead and feature flexibility.

## Study Tips

- **Trace packet flows on paper for every architecture.** For each pattern (centralized egress, centralized inspection, inter-region peering), draw the full path a packet takes: source instance route table entry pointing to TGW, which TGW route table the source attachment is associated with, which route in that table matches, which target attachment the route points to, the destination VPC's route table, and any intermediate hops through NAT Gateways or firewall endpoints. If you can trace the packet path end-to-end without looking at the docs, you understand the architecture. This is exactly what interviewers test.

- **Build a TGW route table segmentation diagram for tonight's Terraform build.** Before writing any code, sketch three route tables (shared-rt, prod-rt, dev-rt) with their associations and propagations. For each attachment (shared-services VPC, prod VPC, dev VPC), write down which route table it is associated with and which route tables it propagates to. Verify that prod cannot reach dev by confirming dev's CIDR never appears in prod-rt. This diagram becomes your Terraform implementation plan.

- **Memorize the top 5 quotas.** In interviews you will be asked about TGW limits. Know these cold: 5,000 attachments per TGW, 20 route tables per TGW, 10,000 total routes across all route tables, 50 peering attachments per TGW, and 100 Gbps per VPC attachment per AZ. The 10,000 total routes limit is the most surprising -- it is shared across all route tables, not per table.

## Next Steps

After completing this study plan, consider exploring:

1. **Tonight's Terraform build** -- Transit Gateway + 3 VPCs (shared-services, prod, dev) with route table segmentation. Implement the three-route-table pattern from the Feb 20 doc's Terraform section, but add centralized egress routing (0.0.0.0/0 in spoke route tables pointing to a shared-services NAT Gateway through TGW).

2. **Direct Connect (Day 4, tomorrow)** -- How Direct Connect Gateway attachments connect to TGW, enabling on-premises networks to reach all VPCs through the hub. The DX Gateway attachment is one of the five attachment types you studied today.

3. **Site-to-Site VPN (Day 5)** -- VPN attachments to TGW with ECMP for aggregated bandwidth across multiple tunnels. The best practices doc (Resource 1) recommended BGP VPN with multipath, and tomorrow you will learn why.

4. **Week 7 multi-account landing zone** -- The prescriptive guidance pattern (Resource 6) is the foundation for the multi-account networking architecture you will design in Week 7, combining TGW with AWS Organizations, RAM sharing, and the segmentation patterns you learned today.
