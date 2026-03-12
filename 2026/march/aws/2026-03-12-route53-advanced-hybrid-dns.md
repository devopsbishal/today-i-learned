# Route53 Advanced -- Hybrid DNS -- The Corporate Phone System Analogy

> Yesterday you learned how Route53 acts as the air traffic control tower -- routing users to the right airport based on latency, location, weight, and health. But all of that assumed a single, public DNS namespace visible to the entire internet. In the real world, companies run **two parallel universes of DNS**: the public internet knows `api.example.com` resolves to `54.231.xx.xx` (a public ALB), while inside the corporate network, the same name resolves to `10.0.5.42` (a private NLB sitting in a VPC). On-premises data centers run their own DNS servers for legacy domains like `corp.internal.acme.com` that AWS workloads need to reach. AWS VPCs run private hosted zones for domains that on-premises servers need to resolve. The two worlds must talk to each other -- and neither can see the other's DNS by default.
>
> This is the domain of **hybrid DNS**: Private Hosted Zones, split-view DNS, Resolver endpoints, forwarding rules, and the centralized DNS architectures that stitch together multi-account AWS environments with on-premises data centers into a single, coherent name resolution fabric. If Route53 routing policies are the air traffic control algorithms, today's topic is the **entire telephone and radio infrastructure** that connects every control tower, runway, and ground station so they can actually communicate.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Private Hosted Zone (PHZ) | The internal phone directory that only works on company desk phones | A DNS zone whose records are only resolvable from VPCs you explicitly associate with it -- invisible to the public internet |
| VPC association | Plugging a building's phone system into the internal switchboard | Linking a VPC to a private hosted zone so the VPC's DNS resolver (CIDR+2) can answer queries from that zone's records |
| Cross-account PHZ sharing | Letting a satellite office use headquarters' internal phone directory | Using `aws_route53_vpc_association_authorization` + `aws_route53_zone_association` to associate a VPC in Account B with a PHZ owned by Account A |
| Split-view DNS | Two phone directories with the same cover but different numbers inside -- one for employees, one for the public | Creating a public and private hosted zone for the same domain name so internal queries resolve to private IPs while external queries resolve to public IPs |
| NXDOMAIN trap | The internal directory says "no listing found" and hangs up instead of transferring you to the public operator | If a PHZ exists for a domain but has no matching record, the VPC Resolver returns NXDOMAIN instead of falling through to the public hosted zone |
| DNSSEC signing | A wax seal on every letter proving it came from the real sender | Cryptographically signing every DNS response in a public hosted zone so resolvers can verify the answer was not tampered with in transit |
| Key Signing Key (KSK) | Your personal signet ring that authenticates the wax seal stamp | An asymmetric KMS key (ECC_NIST_P256) you own that signs the DNSKEY record set; you manage it, but it only signs other keys, not individual records |
| Zone Signing Key (ZSK) | The wax seal stamp used on every outgoing letter, authenticated by the signet ring | A key fully managed by Route53 that signs every individual record in the zone; rotated automatically |
| DS record (chain of trust) | Registering your signet ring's imprint with the royal seal authority | A Delegation Signer record added at the parent zone (registrar) that tells resolvers "this child zone's signatures are legitimate" -- completes the chain of trust |
| DNSSEC validation | Checking that the wax seal on a received letter matches the registered imprint | Configuring the VPC Resolver to verify DNSSEC signatures on responses it receives from other domains -- independent from signing your own zone |
| Route53 Resolver (VPC+2) | The building's internal switchboard operator who knows all internal extensions and can call outside | The built-in DNS resolver at the VPC's CIDR base + 2 address; handles queries for private hosted zones, VPC internal hostnames, and forwards everything else to public DNS |
| Inbound Resolver endpoint | A direct phone line from the outside world into the internal switchboard | ENIs in your VPC that receive DNS queries from on-premises DNS servers over Direct Connect or VPN, allowing on-prem to resolve your private hosted zone records |
| Outbound Resolver endpoint | A dedicated outside line the switchboard uses to call external numbers | ENIs that the VPC Resolver uses to forward DNS queries to on-premises DNS servers based on domain-matching forwarding rules |
| Forwarding rule | A routing slip that says "for any call to *.corp.acme.com, dial the on-prem switchboard at 10.1.1.53" | A domain-level rule that tells the Resolver to forward matching queries to specified target IP addresses (usually on-premises DNS servers) instead of resolving them via public DNS |
| System rule | A routing slip that overrides forwarding and says "for this domain, use the internal directory" | A rule type that forces the Resolver to use the default behavior (private hosted zone or public DNS) for a domain, overriding any forwarding rules |
| Rule sharing via RAM | Distributing copies of the routing slips to every switchboard operator in the company | Sharing Resolver forwarding rules across accounts using AWS Resource Access Manager so every VPC in every account forwards the same domains to the same on-prem DNS servers |
| Route 53 Profiles | A pre-packaged phone system configuration kit: plug it in and the building gets all the right directories and routing slips at once | A container that bundles PHZ associations, forwarding rules, and Resolver DNS Firewall rule groups into a single shareable unit for simplified multi-account DNS management |
| Centralized DNS VPC | The headquarters switchboard room that handles all external calls for every building in the company | A shared-services VPC in the Networking account hosting inbound and outbound Resolver endpoints, connected to spoke VPCs via Transit Gateway |

---

## The Big Picture: Two Worlds, One Name

Think of a large corporation with two communication systems that evolved independently. The **public phone system** is the regular telephone network -- anyone in the world can dial the company's main number and reach the front desk. The **internal phone system** is a private PBX (Private Branch Exchange) with four-digit extensions -- employees can dial `4201` to reach the server room, but that extension means nothing to the outside world. These two systems share a common trunk (the internet and VPN connections), but they maintain separate directories, separate routing rules, and separate trust boundaries.

AWS DNS works exactly the same way:

- **Public Hosted Zones** are the public phone book -- anyone on the internet can query them and get answers. This is what you learned yesterday with routing policies.
- **Private Hosted Zones** are the internal PBX directory -- only VPCs explicitly plugged into the zone can resolve its records.
- **The VPC Resolver (CIDR+2)** is the internal switchboard operator that sits in every VPC. It checks the internal directory first (private hosted zones), then dials outside (public DNS) if no match is found.
- **Resolver endpoints** are the physical phone lines that bridge the internal system to the on-premises PBX across the corporate WAN (Direct Connect or VPN).
- **Forwarding rules** are the routing slips that tell the switchboard "for calls to these extensions, patch them through to the on-prem switchboard instead of looking them up locally."

The challenge is making these two worlds work seamlessly: an EC2 instance in AWS needs to resolve `db.corp.internal.acme.com` (hosted on-prem), while an on-prem application server needs to resolve `api.internal.acme.com` (hosted in a private VPC). Neither world can see the other's DNS by default. Hybrid DNS is the bridge.

```
THE TWO-WORLD DNS PROBLEM
══════════════════════════════════════════════════════════════════════════════

  ON-PREMISES DATA CENTER                     AWS CLOUD
  ┌──────────────────────────┐                ┌──────────────────────────────┐
  │                          │                │                              │
  │  Corporate DNS Server    │                │  VPC Resolver (CIDR+2)       │
  │  (10.1.1.53)             │                │  (10.0.0.2)                  │
  │  ┌────────────────────┐  │                │  ┌──────────────────────┐    │
  │  │ corp.acme.com      │  │   Can't reach  │  │ Private Hosted Zone  │    │
  │  │ legacy.acme.com    │  │◄──── each ────▶│  │ internal.acme.com   │    │
  │  │ db.internal.acme   │  │     other!      │  │ api.internal.acme   │    │
  │  └────────────────────┘  │                │  └──────────────────────┘    │
  │                          │                │                              │
  │  App Server needs to     │                │  EC2 needs to resolve        │
  │  resolve:                │                │  resolve:                    │
  │  api.internal.acme.com   │                │  db.corp.acme.com            │
  │  (lives in AWS VPC)      │                │  (lives on-prem)             │
  │                          │                │                              │
  └──────────┬───────────────┘                └──────────────┬───────────────┘
             │                                               │
             │           Direct Connect / VPN                │
             └───────────────────────────────────────────────┘
                     Network connectivity exists,
                     but DNS resolution does NOT cross
                     this boundary by default
```

---

## Private Hosted Zones: The Internal Phone Directory

### The Analogy

A Private Hosted Zone is your company's **internal phone directory** -- a book of names and extension numbers that only works from company desk phones. If an outsider picks up a payphone and dials extension `4201`, nothing happens. The extension only means something inside the company's private phone system. And a desk phone can only look up extensions from directories it has been given -- if Building A has the Engineering directory and Building B has the Finance directory, an engineer in Building A cannot call a Finance extension unless someone gives Building A a copy of the Finance directory.

### The Technical Reality

A Private Hosted Zone (PHZ) is a DNS zone container that responds to queries **only** from VPCs you explicitly associate with it. The critical VPC settings that must be enabled:

- **enableDnsSupport** = `true` -- turns on the VPC Resolver at CIDR+2
- **enableDnsHostnames** = `true` -- assigns DNS hostnames to EC2 instances

When you create a PHZ, you associate one or more VPCs with it. The VPC Resolver (the built-in DNS server at CIDR base + 2) then knows to check that zone's records when queries arrive for matching domain names.

### Key Behaviors

**Precedence rule**: When a VPC is associated with a private hosted zone for `example.com`, the Resolver checks the PHZ **before** looking at public DNS. This means the private zone effectively "shadows" the public zone for that domain within the VPC.

**The NXDOMAIN trap**: This is the single most important gotcha in PHZ architecture. If you create a PHZ for `example.com` and associate it with a VPC, then **all** queries for `*.example.com` from that VPC hit the private zone first. If the PHZ contains a record for `api.example.com` but not for `www.example.com`, the query for `www.example.com` returns **NXDOMAIN** -- it does **not** fall through to the public hosted zone. The internal directory says "no listing found" and hangs up instead of transferring you to the public operator.

**Mitigation**: For split-view DNS, you must mirror every public record you need into the private hosted zone. If a record should resolve the same way internally as it does externally, copy it into the PHZ.

**Resolution precedence**: The VPC Resolver uses **most-specific domain match first** across both forwarding rules and PHZ associations. If you have a forwarding rule for `acme.com` and a PHZ for `api.acme.com`, a query for `api.acme.com` hits the PHZ (more specific match). At **equal specificity**, forwarding rules beat PHZs -- so a forwarding rule for `example.com` wins over a PHZ for `example.com`.

### Cross-Account PHZ Sharing

In a multi-account AWS environment, a private hosted zone typically lives in a shared-services or networking account, but VPCs in spoke accounts need to resolve its records. This requires a two-step authorization dance:

1. **Account A** (PHZ owner) creates an authorization: "I authorize VPC `vpc-abc123` in Account B to associate with my zone."
2. **Account B** (VPC owner) creates the association: "I want my VPC to use Account A's zone."

Both sides must act -- the PHZ owner grants permission, and the VPC owner accepts it. This mirrors the two-sided pattern you learned in cross-account IAM (trust policy + identity policy) and cross-account KMS (key policy + IAM policy).

```hcl
# Account A (PHZ owner) -- authorize the cross-account association
resource "aws_route53_vpc_association_authorization" "spoke" {
  zone_id = aws_route53_zone.internal.zone_id
  vpc_id  = "vpc-abc123"  # VPC in Account B

  # If the VPC is in a different region, specify it
  vpc_region = "us-east-1"
}

# Account B (VPC owner) -- accept the association
# This resource must be created with the Account B provider
resource "aws_route53_zone_association" "shared_zone" {
  zone_id = "Z1234567890ABC"  # PHZ ID from Account A
  vpc_id  = aws_vpc.spoke.id
}
```

---

## Split-View DNS: Same Cover, Different Numbers

### The Analogy

Imagine your company publishes two phone directories with **identical covers** -- both say "ACME Corp Directory" on the front. The one distributed to the public lobby lists the main reception number: `555-0100`. The one on every employee's desk lists the direct internal line: `ext 4201`. Same company name, same directory title, completely different numbers. A visitor calling from outside always reaches reception. An employee calling from inside always reaches the server room directly.

### The Technical Reality

Split-view (or split-horizon) DNS means creating a **public hosted zone** and a **private hosted zone** for the same domain name (e.g., `api.example.com`). The VPC Resolver checks the private zone first for queries originating inside associated VPCs. The public Route53 nameservers answer queries from everywhere else.

```
SPLIT-VIEW DNS IN ACTION
══════════════════════════════════════════════════════════════════

  EXTERNAL USER (internet)            INTERNAL EC2 (VPC)
  ┌──────────────┐                    ┌──────────────┐
  │ Laptop       │                    │ EC2 Instance │
  │ resolves:    │                    │ resolves:    │
  │ api.example  │                    │ api.example  │
  │   .com       │                    │   .com       │
  └──────┬───────┘                    └──────┬───────┘
         │                                   │
         │ Queries public DNS                │ Queries VPC Resolver
         ▼                                   ▼
  ┌──────────────────┐             ┌──────────────────┐
  │ PUBLIC Hosted    │             │ PRIVATE Hosted   │
  │ Zone             │             │ Zone              │
  │ example.com      │             │ example.com       │
  │                  │             │                   │
  │ api.example.com  │             │ api.example.com   │
  │ → 54.231.10.50   │             │ → 10.0.5.42       │
  │   (public ALB)   │             │   (private NLB)   │
  └──────────────────┘             └──────────────────┘

  Same domain, different answers.
  External traffic hits the public ALB (with WAF, SSL termination).
  Internal traffic goes directly to the private NLB (lower latency, no NAT).
```

### The NXDOMAIN Trap in Split-View

This is where the phone directory analogy earns its keep. Suppose your public zone has 50 records, but your private zone only has 5 records (the ones that need internal overrides). When an EC2 instance queries one of the 45 records that only exists in the public zone:

1. VPC Resolver checks the private zone first -- no match found
2. Returns **NXDOMAIN** -- "this domain does not exist"
3. Does **NOT** fall through to the public zone

The fix: mirror every public record you need into the private zone, even if it points to the same public IP. The private zone is a complete override -- not a selective overlay.

---

## DNSSEC: The Wax Seal on Every Letter

### The Analogy

In the 18th century, important letters were sealed with wax imprinted by the sender's **signet ring**. When the recipient received the letter, they could verify: (1) the seal was unbroken (the letter was not tampered with in transit), and (2) the imprint matched the known seal of the sender (the letter actually came from who it claims). But a signet ring alone is not enough -- you need a **registry of known seal imprints** so you can verify a seal you have never seen before. That registry is the chain of trust.

DNSSEC works the same way. Without it, DNS responses are like unsigned postcards -- anyone along the delivery route could swap out the postcard for a forgery (DNS spoofing, cache poisoning). DNSSEC puts a wax seal on every response.

### The Two-Key Model

DNSSEC uses two keys in a hierarchical signing structure:

| Key | Full Name | Who Manages It | What It Signs | Rotation | Analogy |
|-----|-----------|---------------|---------------|----------|---------|
| **KSK** | Key Signing Key | You (via KMS) | The DNSKEY record set (which contains the ZSK's public key) | You control -- never auto-rotated by Route53 | Your personal signet ring -- authenticates the stamp |
| **ZSK** | Zone Signing Key | Route53 (fully managed) | Every individual DNS record (A, AAAA, CNAME, MX, etc.) | Automatic (Route53 rotates it) | The wax seal stamp used on every letter |

The KSK signs the ZSK. The ZSK signs the actual records. This separation means Route53 can rotate the ZSK frequently without you needing to update the chain of trust at the registrar.

### Chain of Trust

The chain of trust connects your zone's DNSSEC signatures to the global DNS root:

```
DNSSEC CHAIN OF TRUST
═══════════════════════════════════════════════════════

  Root Zone (.)                     "I vouch for .com"
  ┌─────────────┐                   DS record for .com
  │  . (root)   │──────────────────────────┐
  └─────────────┘                          │
                                           ▼
  TLD Zone (.com)                   "I vouch for example.com"
  ┌─────────────┐                   DS record for example.com
  │   .com      │──────────────────────────┐
  └─────────────┘                          │
                                           ▼
  Your Zone (example.com)
  ┌──────────────────────────────────────────────────┐
  │  DNSKEY record: contains KSK public key          │
  │                 + ZSK public key                  │
  │                                                  │
  │  KSK signs → DNSKEY record set (RRSIG)           │
  │  ZSK signs → every other record (RRSIG)          │
  │                                                  │
  │  A record: api.example.com → 54.231.10.50        │
  │  RRSIG:    [ZSK signature of the A record]       │
  └──────────────────────────────────────────────────┘

  Resolver validation path:
  1. Get DNSKEY for example.com → extract KSK public key
  2. Check DS record at .com → confirms KSK is legitimate
  3. KSK signature validates the DNSKEY record (which has ZSK)
  4. ZSK signature validates the A record
  5. Chain is complete → answer is trustworthy
```

### KMS Integration

The KSK is backed by an **asymmetric customer-managed KMS key** with specific requirements:

- **Key spec**: ECC_NIST_P256 (the only supported spec for DNSSEC KSK)
- **Key usage**: SIGN_VERIFY
- **Region**: Must be in **us-east-1** (Route53 is a global service operating from us-east-1)
- **Key policy**: Must grant Route53 (`dnssec-route53.amazonaws.com`) permission to use the key

This connects directly to your KMS knowledge from March 9 -- the KSK is an asymmetric customer-managed key, meaning you own the key policy, control deletion, and can audit every signing operation in CloudTrail.

### Signing vs Validation -- Two Independent Features

| Feature | What It Does | Scope | You Must Do |
|---------|-------------|-------|-------------|
| **DNSSEC signing** | Signs your zone's DNS responses with cryptographic signatures | Your public hosted zones | Create KSK (KMS key), enable signing, add DS record at registrar |
| **DNSSEC validation** | Verifies signatures on DNS responses your VPC Resolver receives from other domains | Your VPC Resolver | Enable validation in Resolver DNSSEC config (per-VPC setting) |

You can sign your zone without validating others. You can validate others without signing your zone. They are completely independent. Most organizations should do both.

---

## Route53 Resolver Endpoints: The Phone Lines Between Worlds

### The Analogy

Your company headquarters (AWS VPCs) has an internal switchboard (VPC Resolver at CIDR+2). Your branch offices (on-premises data centers) have their own switchboards (corporate DNS servers like Active Directory DNS). These switchboards can route calls internally just fine, but they cannot call each other -- there is no phone line connecting them. Resolver endpoints are those **physical phone lines**:

- An **inbound endpoint** is a **direct-dial number** that the branch office switchboard can call to reach headquarters' internal directory. The branch office configures a conditional forwarder: "for anything ending in `.internal.acme.com`, dial these numbers."
- An **outbound endpoint** is a **dedicated outside line** that headquarters' switchboard uses to call the branch office. The switchboard has routing slips (forwarding rules): "for anything ending in `.corp.acme.com`, call the branch office switchboard at these numbers."

The asymmetry is critical: **inbound endpoints are destinations** (on-prem DNS servers forward TO them). **Outbound endpoints are sources** (the VPC Resolver sends queries FROM them).

### How They Work

Each Resolver endpoint consists of **2+ ENIs** deployed across at least two Availability Zones for high availability. Each ENI gets a private IP address in the subnet you specify.

```
RESOLVER ENDPOINTS -- INBOUND AND OUTBOUND FLOWS
══════════════════════════════════════════════════════════════════════════

  ON-PREMISES                                AWS VPC
  ┌──────────────────────┐                   ┌──────────────────────────────┐
  │                      │                   │                              │
  │  Corporate DNS       │    INBOUND FLOW   │  Inbound Endpoint            │
  │  (Active Directory)  │   (on-prem → AWS) │  ┌─────────────────────┐     │
  │                      │                   │  │ ENI: 10.0.1.53 (AZa)│     │
  │  Conditional         │──────────────────▶│  │ ENI: 10.0.2.53 (AZb)│     │
  │  forwarder:          │   Direct Connect  │  └──────────┬──────────┘     │
  │  *.internal.acme.com │   or VPN          │             │                │
  │  → 10.0.1.53         │                   │             ▼                │
  │    10.0.2.53         │                   │  VPC Resolver (10.0.0.2)     │
  │                      │                   │             │                │
  │                      │                   │             ▼                │
  │                      │                   │  Private Hosted Zone         │
  │                      │                   │  internal.acme.com           │
  │                      │                   │  → 10.0.5.42                 │
  │                      │                   │                              │
  │                      │                   │                              │
  │  Corporate DNS       │   OUTBOUND FLOW   │  Outbound Endpoint           │
  │                      │   (AWS → on-prem) │  ┌─────────────────────┐     │
  │  Listens on:         │                   │  │ ENI: 10.0.1.54 (AZa)│     │
  │  10.1.1.53           │◀──────────────────│  │ ENI: 10.0.2.54 (AZb)│     │
  │  10.1.2.53           │   Direct Connect  │  └──────────▲──────────┘     │
  │                      │   or VPN          │             │                │
  │                      │                   │  Forwarding Rule:            │
  │  Returns:            │                   │  *.corp.acme.com             │
  │  db.corp.acme.com    │                   │  → 10.1.1.53, 10.1.2.53     │
  │  → 10.1.5.100        │                   │             ▲                │
  │                      │                   │             │                │
  │                      │                   │  VPC Resolver (10.0.0.2)     │
  │                      │                   │             ▲                │
  │                      │                   │             │                │
  │                      │                   │  EC2 Instance queries:       │
  │                      │                   │  db.corp.acme.com            │
  └──────────────────────┘                   └──────────────────────────────┘
```

### Scaling and Throughput

- Each ENI handles **10,000 queries per second (QPS)**
- Maximum **6 ENIs per endpoint** (~60K QPS ceiling per endpoint)
- **Throughput caveat**: Performance can degrade to as low as **1,500 QPS per ENI** when security group connection tracking is heavily utilized or when queries are routed through a Network Load Balancer -- plan capacity accordingly
- Add more ENIs to an endpoint to scale beyond 10K QPS
- You can specify private IP addresses when creating ENIs -- **do this in production** so you can recreate endpoints with the same IPs without updating every on-prem conditional forwarder
- Monitor with CloudWatch: `InboundQueryVolume` and `OutboundQueryVolume`

### Forwarding Rules

Forwarding rules are the routing slips that tell the VPC Resolver where to send queries for specific domains:

| Rule Type | Behavior | Use Case |
|-----------|----------|----------|
| **Forward** | Send matching queries to specified target IPs via outbound endpoint | Route `corp.acme.com` queries to on-premises DNS servers |
| **System** | Use the default Resolver behavior (PHZ or public DNS) | Override a parent forward rule for a specific subdomain |
| **Recursive** | Forward to the VPC Resolver itself (auto-created for some AWS internal domains) | Internal AWS use -- you do not create these |

**Rule precedence**: When multiple rules match, the most specific (longest domain match) wins. If you have a forward rule for `acme.com` and a system rule for `aws.acme.com`, queries for `api.aws.acme.com` use the system rule.

```hcl
# Outbound Resolver endpoint
resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "outbound-to-onprem"
  direction = "OUTBOUND"

  security_group_ids = [aws_security_group.resolver_outbound.id]

  # At least 2 AZs for high availability
  ip_address {
    subnet_id = aws_subnet.resolver_az_a.id
    ip        = "10.0.1.54"  # Static IP -- critical for stability
  }
  ip_address {
    subnet_id = aws_subnet.resolver_az_b.id
    ip        = "10.0.2.54"
  }
}

# Forwarding rule: send *.corp.acme.com to on-prem DNS
resource "aws_route53_resolver_rule" "to_onprem" {
  domain_name          = "corp.acme.com"
  name                 = "forward-to-onprem"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id

  target_ip {
    ip   = "10.1.1.53"
    port = 53
  }
  target_ip {
    ip   = "10.1.2.53"
    port = 53
  }
}

# Associate the rule with a VPC
resource "aws_route53_resolver_rule_association" "main" {
  resolver_rule_id = aws_route53_resolver_rule.to_onprem.id
  vpc_id           = aws_vpc.main.id
}

# Share the rule via RAM to other accounts
resource "aws_ram_resource_share" "dns_rules" {
  name                      = "shared-dns-forwarding-rules"
  allow_external_principals = false  # Organization only
}

resource "aws_ram_resource_association" "dns_rule" {
  resource_arn       = aws_route53_resolver_rule.to_onprem.arn
  resource_share_arn = aws_ram_resource_share.dns_rules.arn
}

resource "aws_ram_principal_association" "org" {
  principal          = "arn:aws:organizations::111111111111:organization/o-abc123"
  resource_share_arn = aws_ram_resource_share.dns_rules.arn
}
```

---

## Centralized Hybrid DNS Architecture

### The Analogy

Imagine a corporate campus with 20 buildings (spoke VPCs in different accounts), each with its own switchboard. Rather than installing an outside phone line (Resolver endpoints) in every single building -- which is expensive and creates 20 different numbers the branch offices need to know about -- the company builds a **central telephone exchange** (shared-services VPC) in the main administration building (Networking account). Every building connects to the central exchange via internal corridors (Transit Gateway). The branch offices (on-premises) only need one set of phone numbers -- the central exchange's lines.

### The Architecture

This is the production-grade pattern recommended by AWS for multi-account hybrid DNS:

```
CENTRALIZED HYBRID DNS WITH TRANSIT GATEWAY
══════════════════════════════════════════════════════════════════════════════

  ON-PREMISES                    NETWORKING ACCOUNT              SPOKE ACCOUNTS
  ┌──────────────┐               ┌──────────────────────────┐
  │              │               │  Shared-Services VPC      │   ┌───────────┐
  │  Corp DNS    │               │                          │   │ Prod VPC  │
  │  10.1.1.53   │               │  ┌─────────────────────┐ │   │ Account B │
  │              │  Direct       │  │ Inbound Endpoint    │ │   │           │
  │  Conditional │  Connect      │  │ 10.0.1.53 (AZa)    │ │   │ Queries   │
  │  forwarders: │  or VPN       │  │ 10.0.2.53 (AZb)    │ │   │ for corp  │
  │  *.internal  │──────────────▶│  └──────────┬──────────┘ │   │ .acme.com │
  │  .acme.com   │               │             │            │   │     │     │
  │  → 10.0.1.53 │               │             ▼            │   └─────┼─────┘
  │    10.0.2.53 │               │  VPC Resolver            │         │
  │              │               │             │            │         │ TGW
  │              │               │             ▼            │         │
  │              │               │  Private Hosted Zones    │         │
  │              │               │  internal.acme.com       │   ┌─────┼─────┐
  │              │               │                          │   │ Dev VPC   │
  │              │               │  ┌─────────────────────┐ │   │ Account C │
  │              │               │  │ Outbound Endpoint   │ │   │           │
  │              │◀──────────────│  │ 10.0.1.54 (AZa)    │ │   │ Queries   │
  │              │  Direct       │  │ 10.0.2.54 (AZb)    │ │   │ for corp  │
  │  Returns:    │  Connect      │  └──────────▲──────────┘ │   │ .acme.com │
  │  answers for │  or VPN       │             │            │   │     │     │
  │  corp.acme   │               │  Forwarding Rule:        │   └─────┼─────┘
  │  .com        │               │  *.corp.acme.com         │         │
  │              │               │  → 10.1.1.53, 10.1.2.53  │         │ TGW
  │              │               │             ▲            │         │
  └──────────────┘               │             │            │         │
                                 └─────────────┼────────────┘         │
                                               │                      │
                                    ┌──────────┴──────────────────────┘
                                    │
                              ┌─────┴─────┐
                              │  Transit   │
                              │  Gateway   │
                              │            │
                              │ Route table│
                              │ propagation│
                              └────────────┘

  HOW IT WORKS (OUTBOUND -- AWS → on-prem):
  1. Spoke VPC (Account B) EC2 queries db.corp.acme.com
  2. Spoke VPC Resolver checks forwarding rules (shared via RAM)
  3. Rule matches *.corp.acme.com → Resolver service internally
     routes the query to the outbound endpoint ENIs in shared-services VPC
     (this is a control-plane operation -- NOT a data-plane packet through TGW)
  4. Outbound endpoint forwards query through DX/VPN to on-prem DNS
  5. On-prem DNS returns answer back through the same path

  INBOUND (on-prem → AWS):
  1. On-prem app queries api.internal.acme.com
  2. On-prem DNS conditional forwarder sends to inbound endpoint IPs
  3. Query arrives at inbound endpoint ENIs in shared-services VPC
  4. VPC Resolver resolves against private hosted zone
  5. Answer returns to on-prem via DX/VPN
```

### Why Centralize?

| Approach | Resolver Endpoints | On-Prem Config | Cost | Operational Overhead |
|----------|-------------------|----------------|------|---------------------|
| **Decentralized** (endpoints in every VPC) | N pairs of endpoints across N VPCs | N sets of inbound IPs to configure as forwarders | High -- each endpoint ENI costs per hour + per query | High -- manage endpoints in every account |
| **Centralized** (endpoints in shared-services VPC) | 1 pair of endpoints | 1 set of inbound IPs | Low -- single pair of endpoints serves all VPCs | Low -- manage in one place, share rules via RAM |

The centralized pattern works because the **Route 53 Resolver service internally routes outbound forwarding queries** from spoke VPCs to the outbound endpoint ENIs in the shared-services VPC. This is a control-plane operation managed by the Resolver service itself -- the DNS query packets do **not** traverse Transit Gateway as data-plane traffic. Spoke VPCs do not need their own endpoints -- they just need the forwarding rules (shared via RAM) that reference the outbound endpoint in the shared-services VPC.

Transit Gateway **is** needed for **inbound** resolution: on-premises DNS servers forward queries to the inbound endpoint ENI IPs, and those packets must be routable over DX/VPN → TGW → shared-services VPC. TGW also carries general workload traffic between VPCs -- just not the outbound DNS forwarding path.

> **Anti-pattern warning**: Do not use Resolver endpoints to centralize VPC-to-VPC DNS resolution (routing queries from spoke VPCs through a central inbound endpoint to resolve PHZs). For VPC-to-VPC resolution, use **cross-account PHZ association** or **Route 53 Profiles** instead. Centralizing all DNS through endpoints introduces unnecessary AZ dependency, cost, and latency for pure AWS-to-AWS resolution. Resolver endpoints are primarily for **hybrid** (AWS ↔ on-prem) traffic.

For **inbound** resolution, on-premises DNS servers only need to know the inbound endpoint IPs in the shared-services VPC. The private hosted zones can be associated with the shared-services VPC (and optionally with spoke VPCs too if they need direct resolution).

---

## Route 53 Profiles: The Pre-Packaged Phone System Kit

### The Analogy

Without Profiles, setting up DNS for a new building (VPC/account) is like manually configuring a phone system from scratch: associate the right internal directories, install the routing slips, configure the DNS firewall rules -- and repeat for every new building. Route 53 Profiles is a **pre-packaged phone system configuration kit**: you define the kit once (which directories to include, which routing slips, which firewall rules), and then plug it into any new building. The building instantly gets the full DNS configuration.

### What Profiles Bundle

A Route 53 Profile is a container that packages together:

1. **Private Hosted Zone associations** -- which PHZs the VPC should resolve
2. **Resolver DNS Firewall rule group associations** -- which DNS queries to block or allow
3. **Resolver forwarding rules** -- which domains to forward to on-prem or other DNS servers
4. **DNSSEC validation settings** -- enable/disable DNSSEC validation for associated VPCs
5. **DNS Firewall failure mode** -- whether to fail open or closed when Firewall is unavailable
6. **Resolver reverse DNS configuration** -- reverse lookup behavior

Items 4-6 are **configuration overrides** that the Profile pushes to associated VPCs, making Profiles a governance tool -- not just a convenience wrapper for bundling resources.

You create a Profile, add these DNS configuration items to it, then **share it via RAM** to other accounts. When a VPC in a spoke account associates with the Profile, it inherits all the DNS configurations in one step.

### Profiles vs Manual RAM Sharing

| Capability | Manual (RAM + individual associations) | Route 53 Profiles |
|------------|---------------------------------------|-------------------|
| Share forwarding rules | Yes -- share each rule individually via RAM | Yes -- bundle rules into a Profile |
| Share PHZ associations | Requires authorization + association per zone per VPC | Bundle into Profile -- one association |
| Share DNS Firewall rules | Share each rule group individually via RAM | Bundle into Profile |
| Add new DNS config to all VPCs | Must update every VPC individually | Update the Profile -- all associated VPCs inherit changes |
| Scale | Manageable for <10 accounts | Designed for 100+ account Organizations |

Profiles are the **newer, recommended approach** for large-scale multi-account DNS management. They do not replace Resolver endpoints -- you still need inbound/outbound endpoints for hybrid resolution. Profiles simplify the distribution of DNS configuration to spoke accounts.

---

## Multi-VPC DNS Patterns

### Pattern 1: Decentralized (Simple, Small Scale)

Each VPC has its own private hosted zone. No cross-VPC resolution. Suitable for isolated workloads.

```
  VPC A                    VPC B                    VPC C
  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
  │ PHZ:         │         │ PHZ:         │         │ PHZ:         │
  │ app-a.int    │         │ app-b.int    │         │ app-c.int    │
  │              │         │              │         │              │
  │ No cross-VPC │         │ No cross-VPC │         │ No cross-VPC │
  │ resolution   │         │ resolution   │         │ resolution   │
  └──────────────┘         └──────────────┘         └──────────────┘
```

### Pattern 2: Shared PHZ with Multi-VPC Association

One private hosted zone associated with multiple VPCs. All VPCs can resolve the same records. No Resolver endpoints needed for VPC-to-VPC DNS.

```
  ┌──────────────────────────────────────────────────┐
  │            Private Hosted Zone                    │
  │            internal.acme.com                      │
  │            (owned by Networking Account)          │
  └──────────┬──────────────┬──────────────┬──────────┘
             │              │              │
        associated     associated     associated
             │              │              │
         ┌───┴───┐     ┌───┴───┐     ┌───┴───┐
         │ VPC A │     │ VPC B │     │ VPC C │
         │ Acct A│     │ Acct B│     │ Acct C│
         └───────┘     └───────┘     └───────┘
         All three VPCs can resolve *.internal.acme.com
```

### Pattern 3: Centralized DNS Hub (Production-Grade)

The full pattern described in the architecture section above. Shared-services VPC hosts Resolver endpoints, PHZs, and forwarding rules. Spoke VPCs connect via Transit Gateway. On-premises connects via Direct Connect or VPN.

This is the pattern you would implement in a real landing zone (connecting to your Control Tower and Organizations knowledge from March 2-3).

---

## The Complete DNS Query Resolution Order

Understanding the resolution order inside a VPC is critical for debugging:

```
DNS QUERY RESOLUTION ORDER INSIDE A VPC
══════════════════════════════════════════════════════════════════

  EC2 queries: api.example.com
       │
       ▼
  ┌─────────────────────────────────────────────────────┐
  │ Step 1: Find the MOST SPECIFIC domain match          │
  │         across BOTH forwarding rules AND PHZs        │
  │                                                       │
  │  Example: Forwarding rule for acme.com               │
  │           PHZ for api.acme.com                        │
  │           Query for api.acme.com → PHZ wins           │
  │           (more specific match, regardless of type)   │
  │                                                       │
  │  At EQUAL specificity:                                │
  │    Forwarding rule beats PHZ                          │
  │    (e.g., both for example.com → rule wins)           │
  └──────────────────────┬──────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │ Step 2: If the winner is a Forwarding Rule           │
  │         → forward to target IPs via                   │
  │           outbound endpoint                           │
  │           (STOP -- done)                              │
  │                                                       │
  │         If the winner is a PHZ                        │
  │           record found → return the answer            │
  │             (STOP -- done)                            │
  │           NO matching record → NXDOMAIN !!            │
  │             (STOP -- does NOT fall through)            │
  │                                                       │
  │         If NO match (no rule, no PHZ)                 │
  │           → continue to Step 3                        │
  └──────────────────────┬──────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │ Step 3: Public DNS (recursive resolution)            │
  │         VPC Resolver queries public DNS              │
  │         (Route53 public hosted zone or               │
  │          any authoritative nameserver)                │
  │                                                       │
  │         Returns whatever public DNS returns           │
  └─────────────────────────────────────────────────────┘
```

**The key insight**: The Resolver uses **most-specific domain match first** across both forwarding rules and PHZs. At equal specificity, forwarding rules beat PHZs, and PHZs beat public DNS. But PHZs are an **all-or-nothing shadow** -- once a PHZ exists for a domain, the public zone is completely hidden from that VPC for all subdomains of that domain, even if the PHZ does not have a matching record.

---

## Resolver Query Logging

Route 53 Resolver Query Logging lets you log **every DNS query** originating from your VPCs. This is essential for debugging DNS resolution issues, auditing DNS activity, and detecting DNS exfiltration attempts.

**Destinations** (pick one per logging configuration):
- **CloudWatch Logs** -- best for real-time analysis and metric filters
- **S3** -- best for long-term archival and cost-effective storage
- **Kinesis Data Firehose** -- best for streaming to third-party SIEM tools

Each log entry includes: the query name, query type, response code, the VPC that originated the query, and the Resolver endpoint used (if any). You can share query logging configurations across accounts via RAM -- aligning with the centralized DNS architecture pattern.

**When to use it**: Always enable in production. It is the primary tool for answering "why can't my EC2 instance resolve `db.corp.acme.com`?" -- you can see whether the query was forwarded, which endpoint handled it, and what response came back.

---

## Security Considerations

### Resolver Endpoint Security Groups

Resolver endpoint ENIs are regular ENIs -- they need security groups. For inbound endpoints, you must allow DNS traffic (TCP/UDP port 53) from on-premises DNS server IP ranges. For outbound endpoints, you must allow DNS traffic to on-premises DNS server IPs.

```hcl
# Security group for inbound Resolver endpoint
resource "aws_security_group" "resolver_inbound" {
  name_prefix = "resolver-inbound-"
  vpc_id      = aws_vpc.shared_services.id

  ingress {
    description = "DNS from on-premises"
    from_port   = 53
    to_port     = 53
    protocol    = "tcp"
    cidr_blocks = ["10.1.0.0/16"]  # On-prem CIDR
  }

  ingress {
    description = "DNS from on-premises"
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = ["10.1.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### DNS Firewall (Route53 Resolver DNS Firewall)

A separate service (not covered in depth here) that lets you filter DNS queries from your VPCs -- blocking queries to known malicious domains or restricting DNS exfiltration. DNS Firewall rule groups can be bundled into Route 53 Profiles for centralized distribution.

### DNSSEC Operational Safety

- **Do not delete the KMS key** backing your KSK without first disabling DNSSEC signing and removing the DS record at the registrar. Deleting the KMS key while DNSSEC is active will make your entire zone unresolvable by validating resolvers.
- Set the KMS key deletion **waiting period to the maximum (30 days)** for DNSSEC KSKs as a safety net.
- DNSSEC is only supported on **public** hosted zones -- private hosted zones do not support DNSSEC signing.

---

## Key Takeaways

- **Private Hosted Zones are VPC-scoped**: A PHZ only responds to queries from associated VPCs. Two VPC settings must be enabled: `enableDnsSupport` and `enableDnsHostnames`. Cross-account association requires a two-step authorization and acceptance flow.

- **The NXDOMAIN trap is the most common PHZ gotcha**: If a PHZ exists for a domain, the VPC Resolver returns NXDOMAIN for any subdomain not found in the PHZ -- it does NOT fall through to public DNS. For split-view DNS, mirror every needed public record into the private zone.

- **Most-specific match wins, then forwarding rules beat PHZs**: The Resolver finds the most specific domain match across both forwarding rules and PHZ associations. At equal specificity, forwarding rules win. A PHZ for `api.acme.com` beats a forwarding rule for `acme.com` (more specific), but a forwarding rule for `example.com` beats a PHZ for `example.com` (equal specificity).

- **Resolver endpoints are directional**: Inbound endpoints are destinations (on-prem forwards TO them). Outbound endpoints are sources (VPC Resolver forwards FROM them). Each ENI handles 10,000 QPS; scale by adding ENIs.

- **Centralize endpoints in a shared-services VPC**: Deploy Resolver endpoints once in a Networking account VPC, connect spoke VPCs via Transit Gateway, and share forwarding rules via RAM. This avoids deploying endpoints in every VPC and gives on-prem a single set of IPs to target.

- **Route 53 Profiles simplify multi-account DNS**: Bundle PHZ associations, forwarding rules, and DNS Firewall rules into a Profile, share via RAM, and associate with spoke VPCs. One Profile update propagates to all associated VPCs. Recommended for Organizations with 100+ accounts.

- **DNSSEC uses a two-key model**: KSK (your asymmetric KMS key, ECC_NIST_P256, must be in us-east-1) signs the DNSKEY record. ZSK (fully managed by Route53) signs every other record. The DS record at the registrar completes the chain of trust. Signing your zone and validating others' zones are independent features.

- **Static IPs for Resolver endpoints**: Always specify private IPs when creating endpoint ENIs. If you ever need to recreate the endpoint, you can reuse the same IPs without updating every on-prem conditional forwarder.

- **Split-view DNS is a pattern, not a feature**: You implement it by creating a public and private hosted zone with the same domain name. The VPC Resolver automatically prefers the private zone for associated VPCs. But you must manage record parity to avoid the NXDOMAIN trap.

- **Security groups matter for Resolver endpoints**: Endpoint ENIs are regular ENIs that need security groups allowing DNS traffic (TCP/UDP 53) from/to the appropriate CIDR ranges.
