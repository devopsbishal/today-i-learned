# EC2 Networking: ENI, ENA, EFA, Enhanced Networking, and MTU -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-02-09
**Difficulty:** Intermediate

## Overview

This study plan covers the EC2 networking stack from the ground up: Elastic Network Interfaces (ENI) as the foundational virtual network card, Enhanced Networking with the Elastic Network Adapter (ENA) for high-throughput low-latency performance, ENA Express and the SRD protocol for single-flow bandwidth improvements, Elastic Fabric Adapter (EFA) for HPC and ML workloads requiring OS-bypass, and Maximum Transmission Unit (MTU) configuration including jumbo frames. After completing this plan, you will understand how each networking component builds on the one below it, when to use each adapter type, and how to optimize network performance for different EC2 workloads.

## Resources

### 1. Elastic Network Interfaces (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html
- **Type:** Official Docs
- **Summary:** The foundational reference for ENIs covering what a virtual network card is, its attributes (primary/secondary private IPs, Elastic IPs, security groups, MAC address, source/destination check), the distinction between primary and secondary interfaces, network cards on high-performance instances, and termination behavior. Start here because every other networking concept in this plan builds on top of ENIs.

### 2. Understanding ENI (Elastic Network Interface) in EC2 (Dev.to) -- 15 min
- **URL:** https://dev.to/himanshusinghtomar/understanding-eni-elastic-network-interface-in-ec2-1f8i
- **Type:** Article
- **Summary:** A practitioner-focused guide that reinforces the official docs with practical use cases for ENIs including multi-IP applications, workload isolation via separate security groups, high-availability failover by moving an ENI between instances, and network security appliance configurations. Includes AWS CLI examples for creating and attaching ENIs, making the concepts concrete.

### 3. Enhanced Networking on Amazon EC2 Instances (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html
- **Type:** Official Docs
- **Summary:** The overview page that explains what enhanced networking is and why it exists: using single root I/O virtualization (SR-IOV) to provide higher bandwidth, higher packet-per-second performance, and lower latency at no additional charge. Covers the two adapter types (ENA supporting up to 100 Gbps on Nitro instances, and the legacy Intel 82599 VF supporting up to 10 Gbps on older instances) and links to detailed configuration guides for each.

### 4. Enable Enhanced Networking with ENA on Your EC2 Instances (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking-ena.html
- **Type:** Official Docs
- **Summary:** The hands-on reference for ENA covering which instances support it (all Nitro-based plus select Xen-based), how to verify ENA support on your instance, driver requirements, and the key performance features: hardware checksum generation, multi-queue device interface for scalability, and receive-side steering for directing packets to the correct vCPU. Essential for understanding the mechanics behind enhanced networking rather than just knowing it exists.

### 5. Improve Network Performance Between EC2 Instances with ENA Express (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ena-express.html
- **Type:** Official Docs
- **Summary:** Covers ENA Express, powered by AWS Scalable Reliable Datagram (SRD) technology, which increases maximum single-flow bandwidth from 5 Gbps to 25 Gbps and reduces tail latency by up to 85% for high-throughput workloads. Explains how SRD distributes packets across multiple network paths with dynamic congestion detection, handles packet reordering at the network layer, and falls back to standard ENA when not configured on both endpoints. Important for understanding the most advanced ENA capability.

### 6. Elastic Fabric Adapter for AI/ML and HPC Workloads (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html
- **Type:** Official Docs
- **Summary:** The comprehensive guide to EFA covering its definition as an ENA with additional OS-bypass capabilities, the two interface modes (EFA with ENA for combined low-latency and IP networking, and EFA-only), supported software libraries (Libfabric, NCCL, Open MPI, Intel MPI), RDMA support for direct memory access, supported instance types across Nitro v3-v6, and the critical limitation that EFA traffic cannot cross Availability Zones. This is where the networking stack reaches its peak performance tier.

### 7. AWS Networking -- ENI vs EFA vs ENA (Digital Cloud Training) -- 10 min
- **URL:** https://digitalcloud.training/aws-networking-eni-vs-efa-vs-ena/
- **Type:** Blog
- **Summary:** A concise comparison article that puts all three networking components side by side, clarifying the hierarchy: ENI is the base virtual network card, ENA adds enhanced networking with up to 100 Gbps throughput, and EFA adds OS-bypass on top of ENA for HPC and ML workloads. Useful as a consolidation resource after reading the individual official docs, helping solidify the mental model of how these components relate to each other.

### 8. Network Maximum Transmission Unit (MTU) for Your EC2 Instance (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/network_mtu.html
- **Type:** Official Docs
- **Summary:** Explains MTU configuration including the default 1500 bytes for standard Ethernet, jumbo frames at 9001 bytes for current-generation instances, and when each is appropriate. Covers the critical nuances: jumbo frames work within a VPC and over Direct Connect but are reduced to 1500 for internet gateway and VPN traffic, inter-region peering supports 8500 bytes, and Path MTU Discovery (PMTUD) requires allowing ICMP through security groups. Essential for avoiding mysterious throughput issues in production.

## Study Tips

- **Build the layered mental model as you read:** Think of EC2 networking as a stack: ENI is the base (virtual network card with IPs and security groups), ENA sits on top (SR-IOV driver for enhanced throughput), ENA Express adds SRD on top of ENA (multi-path single-flow optimization), and EFA adds OS-bypass on top of ENA (kernel bypass for HPC). Sketch this hierarchy and annotate each layer with its maximum throughput and use case.

- **Pay attention to the "superset" relationships:** EFA is not a replacement for ENA -- it is an ENA with additional capabilities. Every EFA interface provides full ENA functionality plus OS-bypass. Similarly, ENA Express is an enhancement to ENA, not a separate adapter. Understanding these relationships prevents confusion on which features are available at each tier.

- **Focus on the constraints, not just the capabilities:** The exam-relevant and interview-relevant details are often the limitations: EFA traffic cannot cross Availability Zones, ENA Express requires both endpoints to be configured and in the same AZ, jumbo frames silently degrade to 1500 bytes for internet-bound traffic, and PMTUD requires ICMP to be allowed. These constraints drive real architectural decisions.

## Next Steps

After completing this study plan, consider exploring:

1. **VPC networking fundamentals** -- Understand how subnets, route tables, security groups, and NACLs interact with the ENI attributes covered in this plan, especially how source/destination checking affects NAT instances and routing appliances.

2. **EC2 placement groups and networking** -- Explore how cluster placement groups interact with enhanced networking and jumbo frames to achieve maximum throughput between tightly coupled instances.

3. **ENA Express configuration and monitoring** -- Follow the official guide at https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ena-express-configure.html to configure ENA Express and use CloudWatch metrics to monitor SRD utilization and fallback rates.

4. **Hands-on practice** -- Launch two instances in the same AZ, attach a secondary ENI, verify enhanced networking is enabled with `ethtool -i eth0`, test MTU with `ping -M do -s 8972`, and measure throughput differences between standard and jumbo frame configurations using `iperf3`.
