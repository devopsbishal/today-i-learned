# Route53 Routing Policies & Health Checks -- The Air Traffic Control Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), and fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager). But none of those services answer a question that comes before traffic even reaches your defenses: **which of your servers should a user talk to in the first place?**
>
> When a customer in Tokyo opens your application, should their request go to us-east-1 or ap-northeast-1? When your primary region catches fire, how does traffic automatically redirect to the disaster recovery site without anyone touching a keyboard? When you roll out a new version, how do you send 5% of traffic to the canary deployment while 95% stays on the stable release? That is the domain of Amazon Route53's routing policies -- a collection of seven distinct algorithms that transform DNS from a simple name-to-IP lookup into an intelligent, health-aware traffic routing layer that sits in front of your entire architecture.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| Route53 | The air traffic control tower at a global airport network | AWS's authoritative DNS service that resolves domain names to IP addresses and can make intelligent routing decisions based on health, location, latency, and weight |
| Simple routing | A phone book listing all branch locations for one business | Returns all values for a record in random order; no health check integration on the record itself, no intelligence -- just a flat lookup |
| Weighted routing | A restaurant host who seats 70% of diners in the main room and 30% in the patio | Distributes DNS responses across records by configurable percentages; perfect for canary deployments, A/B testing, and gradual migrations |
| Latency routing | An air traffic controller who routes each plane to the nearest airport with the shortest approach path | Returns the record associated with the AWS region that provides the lowest network latency for the requesting user, measured by Route53's global latency tables |
| Failover routing | A hospital's automatic transfer switch that cuts to the backup diesel generator the instant main power fails | Designates a primary and secondary record; DNS returns the primary as long as it passes health checks, and automatically switches to the secondary when the primary fails |
| Geolocation routing | A customs officer who routes travelers to different lines based on the country stamped in their passport | Returns different records based on the user's geographic location (continent, country, or US state); uses hard boundaries with no fallback between locations unless you configure a default record |
| Geoproximity routing | Each airport has a gravitational field you can dial up or down to expand or shrink its catchment area | Routes traffic based on the geographic distance between user and resource, with a bias knob (-99 to +99) that expands or shrinks a resource's effective catchment area |
| Multivalue answer routing | A phone book that lists eight branches but crosses out the ones that are closed today | Returns up to eight healthy IP addresses in random order; a simple form of DNS-level load balancing with health check filtering |
| Health check (endpoint) | A flight operations inspector who pings each airport every 30 seconds: "Are you open? Can you land planes?" | Route53 sends HTTP/HTTPS/TCP probes from global health checker nodes to your endpoint at configurable intervals (10s or 30s), marking it unhealthy after a failure threshold |
| Health check (calculated) | A regional aviation authority that declares a region "operational" only if at least 3 of its 5 airports are open | Aggregates the status of multiple child health checks using AND, OR, or threshold logic into a single parent health status |
| Health check (CloudWatch alarm) | A smoke detector that reports to the control tower without the inspector needing to visit in person | Monitors a CloudWatch alarm's data stream to determine health; essential for private resources that Route53's public health checkers cannot reach |
| Route53 ARC routing control | A manual override switch that lets the controller declare "reroute ALL traffic away from Airport East, effective immediately" | Simple on/off switches for manual or automated regional failover, operating on the data plane so they work even when the Route53 control plane is unavailable |
| Alias record | A direct internal radio frequency to another AWS resource -- no call forwarding, no extra charge | A Route53-specific record type that maps directly to AWS resources (ALB, CloudFront, S3, another Route53 record) with no additional DNS query charge and native health check propagation via Evaluate Target Health |
| Complex/nested routing | A decision tree of air traffic controllers: the first picks the continent, the second picks the airport, the third picks the runway | Layering routing policies via alias records to build multi-level routing trees (e.g., failover at top -> latency in middle -> weighted at bottom) |
| Data plane vs control plane | The radar system and radio frequencies (always on) vs the administrative office where you file paperwork (might close during a disaster) | Route53 DNS resolution and health checks run on the globally distributed data plane (highly available); the Route53 console and management APIs run on the control plane in us-east-1 (could be impaired during a regional event) |
| Active-active failover | Multiple airports all accepting flights simultaneously -- if one closes, the others absorb its traffic | All healthy endpoints serve traffic; use weighted, latency, geolocation, or multivalue routing with health checks to automatically remove unhealthy endpoints from DNS responses |
| Active-passive failover | One main airport handles all flights; the backup airport only opens if the main one shuts down | The failover routing policy specifically: primary record serves all traffic until it fails health checks, then secondary takes over |

---

## The Big Picture: DNS as the First Routing Decision

Think of your global infrastructure as a **network of airports**. Each AWS region is an airport. Each application endpoint (ALB, EC2, CloudFront) is a runway at that airport. When a user types your domain name into a browser, they are essentially asking: "I need to fly somewhere -- which airport should I go to?"

Route53 is the **air traffic control tower** that answers that question. But unlike a real tower that just assigns the nearest open runway, Route53 can apply seven different routing algorithms depending on what you care about most:

- **Speed?** Latency routing sends users to the region with the fastest network path.
- **Compliance?** Geolocation routing ensures EU users only hit EU servers.
- **Reliability?** Failover routing redirects all traffic to a standby region when the primary goes down.
- **Gradual rollout?** Weighted routing sends 5% to the new version and 95% to the old.
- **Fine-tuned control?** Geoproximity routing lets you stretch or shrink each region's "coverage area" with a bias dial.
- **Simple resilience?** Multivalue answer routing returns a pool of healthy IPs and lets the client choose.

The critical insight: **Route53 routing happens at the DNS layer, before any HTTP request is made.** The user's DNS resolver receives an IP address (or CNAME) from Route53, and all subsequent requests go directly to that endpoint. Route53 does not sit in the data path like a load balancer -- it makes a one-time routing decision that the client caches for the duration of the TTL. This means your TTL settings directly control how quickly traffic shifts when an endpoint fails or your routing policy changes.

```
HOW ROUTE53 FITS IN THE REQUEST PATH
═══════════════════════════════════════════════════════════════════════

  User (Tokyo)                                    Your Infrastructure
  ┌─────────┐                                     ┌──────────────────┐
  │ Browser  │                                     │  us-east-1 (ALB) │
  │          │                                     │  eu-west-1 (ALB) │
  │          │                                     │  ap-ne-1   (ALB) │
  └────┬─────┘                                     └────────▲─────────┘
       │                                                    │
       │ 1. DNS query: "app.example.com"                    │ 4. Direct HTTP
       │                                                    │    request to
       ▼                                                    │    chosen IP
  ┌──────────────────────────────────────────────┐          │
  │          Route53 (Air Traffic Control)        │          │
  │                                               │          │
  │  Hosted Zone: example.com                     │          │
  │  ┌──────────────────────────────────────────┐ │          │
  │  │ Routing Policy evaluates:                │ │          │
  │  │  - Health check status of each endpoint  │ │          │
  │  │  - User's location / latency / geo       │ │          │
  │  │  - Record weights / priority / bias      │ │          │
  │  └──────────────────┬───────────────────────┘ │          │
  │                     │                         │          │
  │  2. Decision: "ap-northeast-1 has lowest      │          │
  │     latency for this Tokyo user"              │          │
  │                     │                         │          │
  │  3. Returns: 13.112.xx.xx (ap-ne-1 ALB IP)   │          │
  └─────────────────────┼─────────────────────────┘          │
                        │                                    │
                        └────────────────────────────────────┘
                          Client caches for TTL seconds,
                          then sends HTTP directly to the IP.
                          Route53 is NOT in the data path.
```

---

## Alias Records vs Non-Alias Records: The Foundation

Before diving into routing policies, you need to understand the record type that makes Route53 powerful: the **alias record**. This is not a standard DNS record type (like A, AAAA, or CNAME) -- it is a Route53-specific extension. An alias record is configured like a CNAME (you specify a target DNS name) but resolved like an A record (Route53 returns the target's IP address directly to the client). This is why it works at the zone apex where CNAMEs are forbidden by the DNS spec.

**The analogy**: A non-alias CNAME record is like **call forwarding** -- your phone rings, you forward it to another number, and the caller pays for two calls. An alias record is like a **direct internal radio frequency** -- the tower communicates directly with the airport over an internal channel, no forwarding involved, no extra charge.

| Feature | Alias Record | Non-Alias (Standard) Record |
|---------|-------------|---------------------------|
| Zone apex (example.com) | Yes -- can create alias A/AAAA at the apex | No -- CNAME records cannot exist at the zone apex (DNS spec violation) |
| DNS query charge | No additional charge for alias queries to AWS resources | Standard Route53 query charges apply |
| Targets | AWS resources only: ALB, NLB, CloudFront, S3 website, Elastic Beanstalk, VPC Interface Endpoint, API Gateway, Global Accelerator, another Route53 record (same or different hosted zone) | Any IP address or hostname |
| Health checking | Built-in via "Evaluate Target Health" -- automatically inherits the target's health without creating a separate health check | Must create an explicit Route53 health check and associate it with the record |
| TTL | Inherited from the target resource (you cannot set it). When aliasing to another Route53 record, the TTL is the TTL of the target record -- a commonly tested point | You set the TTL explicitly |

**Why this matters for routing policies**: When you build complex routing trees (layering policies on top of each other), alias records are the glue. An alias record at the top level can point to another Route53 record that uses a different routing policy, and the "Evaluate Target Health" flag propagates health status up through the tree without you creating a separate health check for every intermediate level.

---

## The Seven Routing Policies -- Deep Dive

### 1. Simple Routing -- The Phone Book

**The analogy**: A phone book with one listing per business name. You look up "Pizza Palace" and get all their locations' phone numbers. You pick one at random and call.

**The technical reality**: Simple routing is the default. You create a single record (e.g., an A record for `app.example.com`) and provide one or more IP addresses. Route53 returns all values in random order. The client's resolver typically picks the first one.

**Key characteristics**:
- Cannot attach a Route53 health check to a simple routing record (but alias records with simple routing still benefit from Evaluate Target Health)
- Can specify multiple values in a single record
- No `set_identifier` parameter (it is the only policy that does not require one)
- Use case: Single-resource configurations, dev environments, or when you just need basic DNS

```
Simple Routing — All IPs Returned, Client Picks One
════════════════════════════════════════════════════

  DNS Query: app.example.com
            │
            ▼
  ┌─────────────────────────────┐
  │  Route53 Simple Record      │
  │                             │
  │  Values:                    │
  │    10.0.1.10                │
  │    10.0.2.20                │     No health checks.
  │    10.0.3.30                │     No intelligence.
  │                             │     Random order.
  └─────────┬───────────────────┘
            │
            ▼
  Response: [10.0.2.20, 10.0.3.30, 10.0.1.10]
            (shuffled randomly each time)
```

```hcl
# Terraform -- Simple routing (no set_identifier needed)
resource "aws_route53_record" "simple" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  ttl     = 300

  records = [
    "10.0.1.10",
    "10.0.2.20",
    "10.0.3.30",
  ]
}
```

### 2. Weighted Routing -- The Restaurant Host

**The analogy**: A restaurant host who seats diners across two rooms: 90% go to the main dining room (stable version) and 10% go to the new patio (canary release). If the patio catches fire and is condemned (fails health check), 100% of diners go to the main room.

**The technical reality**: You create multiple records with the same name and type, each with a `weight` value. Route53 calculates the probability of returning each record as `weight / sum_of_all_weights`. A record with weight 70 alongside a record with weight 30 gets 70% of DNS responses.

**Key characteristics**:
- Weights range from 0 to 255
- A weight of 0 means "never return this record" in normal operation. The fallback cascade is: (1) Route53 returns only healthy nonzero-weight records; (2) if all nonzero records are unhealthy, it promotes healthy zero-weight records; (3) if ALL records are unhealthy, Route53 treats them all as if they were healthy and falls back to the **original weights** -- meaning zero-weight records are still excluded (0/total = 0%). Additionally, if ALL records have weight 0 and no health checks, traffic is distributed equally across all of them. This multi-tier fallback is a common exam gotcha
- Each record requires a unique `set_identifier`
- Health checks are optional but strongly recommended -- without them, Route53 returns unhealthy endpoints
- Use cases: canary deployments, blue/green, A/B testing, gradual region migration

```
Weighted Routing — Percentage-Based Traffic Distribution
════════════════════════════════════════════════════════

  DNS Query: app.example.com
            │
            ▼
  ┌──────────────────────────────────────────────┐
  │  Route53 evaluates weights:                  │
  │                                              │
  │  Record A: us-east-1 ALB   weight=70  (70%) │ ◄── Health check ✓
  │  Record B: eu-west-1 ALB   weight=20  (20%) │ ◄── Health check ✓
  │  Record C: ap-ne-1 ALB     weight=10  (10%) │ ◄── Health check ✗ UNHEALTHY
  │                                              │
  │  Record C is unhealthy → removed from pool   │
  │  Recalculated: A=70/90≈78%, B=20/90≈22%     │
  └──────────────────────────────────────────────┘
```

```hcl
# Terraform -- Weighted routing for canary deployment
resource "aws_route53_record" "stable" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "stable"

  alias {
    name                   = aws_lb.stable.dns_name
    zone_id                = aws_lb.stable.zone_id
    evaluate_target_health = true
  }

  weighted_routing_policy {
    weight = 90
  }
}

resource "aws_route53_record" "canary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "canary"

  alias {
    name                   = aws_lb.canary.dns_name
    zone_id                = aws_lb.canary.zone_id
    evaluate_target_health = true
  }

  weighted_routing_policy {
    weight = 10
  }
}
```

**The zero-weight fallback cascade**: If you set Record C's weight to 0, Route53 will never return it in normal operation. But if Records A and B both fail their health checks, Route53 promotes healthy zero-weight records -- so Record C activates. If Record C is also unhealthy, Route53 treats all records as if they were healthy and falls back to the **original weights** -- A gets 70/90, B gets 20/90, and C (weight 0) still gets nothing. Think of it as a three-tier safety net: healthy nonzero -> healthy zero-weight -> "pretend everyone is healthy and use original weights."

### 3. Latency Routing -- The Shortest Flight Path

**The analogy**: An air traffic controller who checks the real-time approach times for every airport in the network and routes each incoming plane to the airport with the shortest approach path. The controller does not care about geographic distance -- a plane over the Atlantic might have lower latency to a European airport than a nearby one that is congested.

**The technical reality**: Route53 maintains a global latency table that maps DNS resolver IP ranges to AWS regions based on network latency measurements. When a DNS query arrives, Route53 looks up the resolver's IP, finds which AWS region has the lowest latency for that resolver, and returns the record associated with that region.

**Key characteristics**:
- Latency is measured to **AWS regions**, not to individual endpoints -- you specify the region when creating each record
- Latency data is based on Route53's measurements, not yours -- you cannot override the latency values
- Latency changes over time; Route53 periodically updates its tables
- Health checks are strongly recommended -- if the lowest-latency region is unhealthy, Route53 returns the next-lowest
- Use case: Multi-region applications where performance matters most

```
Latency Routing — Best Performance for Each User
═════════════════════════════════════════════════

  User in Tokyo          User in London          User in New York
       │                      │                       │
       ▼                      ▼                       ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Route53 Latency Table Lookup                            │
  │                                                          │
  │  Tokyo resolver:                                         │
  │    → ap-northeast-1: 12ms ◄── LOWEST, return this       │
  │    → us-east-1:      180ms                               │
  │    → eu-west-1:      260ms                               │
  │                                                          │
  │  London resolver:                                        │
  │    → eu-west-1:      15ms  ◄── LOWEST, return this      │
  │    → us-east-1:      85ms                                │
  │    → ap-northeast-1: 250ms                               │
  │                                                          │
  │  New York resolver:                                      │
  │    → us-east-1:      8ms   ◄── LOWEST, return this      │
  │    → eu-west-1:      75ms                                │
  │    → ap-northeast-1: 170ms                               │
  └──────────────────────────────────────────────────────────┘
```

```hcl
# Terraform -- Latency routing across three regions
resource "aws_route53_record" "us_east" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "us-east-1"

  alias {
    name                   = aws_lb.us_east.dns_name
    zone_id                = aws_lb.us_east.zone_id
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = "us-east-1"
  }
}

resource "aws_route53_record" "eu_west" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "eu-west-1"

  alias {
    name                   = aws_lb.eu_west.dns_name
    zone_id                = aws_lb.eu_west.zone_id
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = "eu-west-1"
  }
}

resource "aws_route53_record" "ap_ne" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "ap-northeast-1"

  alias {
    name                   = aws_lb.ap_ne.dns_name
    zone_id                = aws_lb.ap_ne.zone_id
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = "ap-northeast-1"
  }
}
```

### 4. Failover Routing -- The Backup Generator

**The analogy**: A hospital has a main power supply and a backup diesel generator. The main power runs everything normally. A sensor (health check) monitors the main power continuously. The instant the main power fails, an automatic transfer switch (failover routing) cuts over to the diesel generator. Patients never know it happened -- the lights never go off.

**The technical reality**: You create exactly two records: one designated `PRIMARY` and one designated `SECONDARY`. Route53 returns the primary as long as it passes its health check. When the primary fails, Route53 returns the secondary. The primary record **must** have a health check. The secondary record's health check is optional -- if you attach one and the secondary also fails, Route53 returns the primary anyway (last resort).

**Key characteristics**:
- Exactly one primary and one secondary per failover group
- Health check on primary is **mandatory**
- Health check on secondary is optional but recommended
- The secondary can be a static S3 website hosting a "we're experiencing issues" page
- This is the only routing policy that is **explicitly for active-passive failover** -- all other policies are active-active by nature
- Use case: DR failover, maintenance pages, active-passive multi-region

```
Failover Routing — Automatic Disaster Recovery
═══════════════════════════════════════════════

  NORMAL STATE                         FAILOVER STATE
  ════════════                         ══════════════

  DNS Query                            DNS Query
      │                                    │
      ▼                                    ▼
  ┌──────────────┐                    ┌──────────────┐
  │ PRIMARY      │                    │ PRIMARY      │
  │ us-east-1    │ ◄── HC: ✓ PASS    │ us-east-1    │ ◄── HC: ✗ FAIL
  │ Returns this │                    │ SKIPPED      │
  └──────────────┘                    └──────────────┘
                                           │
  ┌──────────────┐                    ┌────▼─────────┐
  │ SECONDARY    │                    │ SECONDARY    │
  │ eu-west-1    │  (standby,        │ eu-west-1    │ ◄── Returns this
  │ NOT returned │   not served)     │ NOW ACTIVE   │
  └──────────────┘                    └──────────────┘

  Failover time: ~70-150 seconds typical (see Failover Timing
  section below for the three-stage breakdown).
```

```hcl
# Terraform -- Failover routing with health check
resource "aws_route53_health_check" "primary" {
  fqdn              = aws_lb.primary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10  # Fast interval: 10s (costs more than 30s)

  tags = {
    Name = "primary-alb-health-check"
  }
}

resource "aws_route53_record" "primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "primary"

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true  # Propagate ALB health too
  }

  failover_routing_policy {
    type = "PRIMARY"
  }

  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "secondary"

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }

  failover_routing_policy {
    type = "SECONDARY"
  }
}
```

### 5. Geolocation Routing -- The Customs Officer

**The analogy**: An international airport's customs hall with different lines based on the country stamped in your passport. German passport holders go to Line A (which routes them to services in the EU), US passport holders go to Line B (US servers), Japanese passport holders go to Line C (APAC servers). If someone arrives with a passport from a country that does not have a dedicated line, they go to the "All Others" default line -- and if there is no default line, they get a "no answer" (NODATA response).

**The technical reality**: You create records tagged with a location: continent (e.g., Europe), country (e.g., DE for Germany), or US state (e.g., California). Route53 maps the requesting resolver's IP to a geographic location using a GeoIP database, then returns the most specific matching record: state > country > continent > default.

**Key characteristics**:
- Hard boundaries -- there is no "fuzzy" overlap like geoproximity. A user in Germany gets the Europe/DE record, period
- **You MUST create a "default" record** as a catch-all, otherwise users from unmatched locations receive a NODATA response (no DNS answer at all)
- The most specific match wins: a user in California hits the "US-CA" record before "US" before "North America" before "Default"
- Does NOT optimize for latency or performance -- purely location-based
- Use case: GDPR compliance (EU data stays in EU), content localization, legal restrictions (block certain countries)
- **Geolocation vs Latency**: Geolocation guarantees which server handles a region's traffic (compliance). Latency optimizes performance without guarantees about location. Use geolocation when you MUST control where traffic goes; use latency when you want the fastest response.

```
Geolocation Routing — Compliance-Driven Traffic Steering
════════════════════════════════════════════════════════

  User from Germany          User from Japan          User from Brazil
       │                          │                        │
       ▼                          ▼                        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Route53 GeoIP Database Lookup                               │
  │                                                              │
  │  Germany → Match: Continent=Europe                           │
  │           → Return: eu-west-1 ALB IP                         │
  │                                                              │
  │  Japan   → Match: Country=JP                                 │
  │           → Return: ap-northeast-1 ALB IP                    │
  │                                                              │
  │  Brazil  → Match: Continent=South America                    │
  │           → No South America record → Fall to Default        │
  │           → Return: us-east-1 ALB IP (default)               │
  └──────────────────────────────────────────────────────────────┘

  SPECIFICITY CHAIN:  State → Country → Continent → Default → NODATA
```

```hcl
# Terraform -- Geolocation routing for GDPR compliance
resource "aws_route53_record" "europe" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "europe"

  alias {
    name                   = aws_lb.eu.dns_name
    zone_id                = aws_lb.eu.zone_id
    evaluate_target_health = true
  }

  geolocation_routing_policy {
    continent = "EU"
  }
}

resource "aws_route53_record" "asia" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "asia"

  alias {
    name                   = aws_lb.ap.dns_name
    zone_id                = aws_lb.ap.zone_id
    evaluate_target_health = true
  }

  geolocation_routing_policy {
    country = "JP"
  }
}

# CRITICAL: Always create a default record for unmatched locations
resource "aws_route53_record" "default" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"

  set_identifier = "default"

  alias {
    name                   = aws_lb.us.dns_name
    zone_id                = aws_lb.us.zone_id
    evaluate_target_health = true
  }

  geolocation_routing_policy {
    country = "*"  # Default / catch-all
  }
}
```

### 6. Geoproximity Routing -- The Adjustable Radar Range

**The analogy**: Imagine each airport has a **gravitational field** that you can dial up or down. By default, each airport's gravity covers the area closest to it -- planes fall into the orbit of the nearest airport. But you can **turn up the gravity** on one airport (positive bias), and users who previously fell into a neighboring airport's orbit now get pulled toward this one instead. Or you can **dial down the gravity** (negative bias) to shrink its pull and push planes toward other airports. A bias of +99 on the us-east-1 airport is like making it a black hole -- it pulls in traffic from both North America and Europe.

**The technical reality**: Geoproximity routing calculates the geographic distance between the user and each resource, applies a bias adjustment (-99 to +99), and routes to the "closest" resource after bias. Unlike geolocation (hard boundaries), geoproximity creates **soft, overlapping boundaries** that you can fine-tune.

**Key characteristics**:
- Requires **Route53 Traffic Flow** -- a visual traffic policy editor where you define routing rules as a versioned traffic policy document, then create a traffic policy instance that applies it to a hosted zone record. This means geoproximity records are defined inside a traffic policy document, not as individual `aws_route53_record` resources. In Terraform, you use `aws_route53_traffic_policy` and `aws_route53_traffic_policy_instance` instead
- Bias range: -99 to +99. A positive bias expands the effective catchment area around the resource (pulls traffic toward it), and a negative bias shrinks it (pushes traffic away). The effect is nonlinear -- small bias values (e.g., +10) make subtle adjustments, while values near +/-99 create dramatic shifts in boundary placement
- Supports **non-AWS resources** by specifying latitude and longitude coordinates
- AWS resources are specified by region -- Route53 uses the region's geographic center
- Use case: Gradually shifting traffic to a new region, load balancing across unevenly distributed resources, fine-tuning after a latency-based setup

**Geolocation vs Geoproximity decision rule**: If you need a **guarantee** that users from Country X always hit Region Y (compliance), use geolocation. If you need **distance-based routing with adjustable boundaries** (performance optimization with control), use geoproximity.

> **Interview tip**: "When would you use geolocation vs geoproximity?" is one of the most common Route53 interview questions. The one-liner: geolocation = **compliance** (hard boundaries, passport-based), geoproximity = **optimization** (soft boundaries, distance-based with a dial). If the word "must" or "regulatory" appears in the scenario, it is geolocation.

```
Geoproximity — Bias Shifts the Boundary Between Regions
═══════════════════════════════════════════════════════

  NO BIAS (default)                    WITH BIAS (+25 on us-west-2)
  ═════════════════                    ════════════════════════════

  ┌─────────────────────────┐          ┌─────────────────────────┐
  │ North America           │          │ North America           │
  │                         │          │                         │
  │  us-west-2    │ us-east-1│         │  us-west-2      │us-e-1│
  │    ◉          │    ◉    │          │    ◉            │  ◉   │
  │               │         │          │                 │      │
  │  ◄── 50% ──► │ ◄─ 50% ─►│         │  ◄─── 70% ───► │◄30%─►│
  │               │         │          │                 │      │
  └─────────────────────────┘          └─────────────────────────┘
                                         Boundary shifted east →
   Users in middle go to nearest.        us-west-2's "catchment area"
                                         expanded by positive bias.
```

```hcl
# Terraform -- Geoproximity routing (requires Traffic Flow, not standard records)
resource "aws_route53_traffic_policy" "geoproximity" {
  name     = "geoproximity-routing"
  comment  = "Route based on geographic proximity with bias"

  document = jsonencode({
    AWSPolicyFormatVersion = "2015-10-01"
    RecordType             = "A"
    StartRule              = "geopx_rule"
    Rules = {
      geopx_rule = {
        RuleType = "geoproximity"
        GeoproximityLocations = [
          {
            EndpointReference = "us_east_ep"
            Region            = "aws:route53:us-east-1"
            Bias              = 0
          },
          {
            EndpointReference = "us_west_ep"
            Region            = "aws:route53:us-west-2"
            Bias              = 25  # Expand us-west-2's catchment area
          }
        ]
      }
    }
    Endpoints = {
      us_east_ep = {
        Type  = "elastic-load-balancer"
        Value = "dualstack.my-alb-us-east.us-east-1.elb.amazonaws.com"
      }
      us_west_ep = {
        Type  = "elastic-load-balancer"
        Value = "dualstack.my-alb-us-west.us-west-2.elb.amazonaws.com"
      }
    }
  })
}

resource "aws_route53_traffic_policy_instance" "geoproximity" {
  name                   = "app.example.com"
  traffic_policy_id      = aws_route53_traffic_policy.geoproximity.id
  traffic_policy_version = aws_route53_traffic_policy.geoproximity.version
  hosted_zone_id         = aws_route53_zone.main.zone_id
  ttl                    = 60
}
```

### 7. Multivalue Answer Routing -- The Filtered Phone Book

**The analogy**: A phone book that lists up to eight branch offices for a business, but it crosses out any branch that is currently closed (failed health check). When you look up the business, you get all the open branches -- you pick one. It is not fancy routing, but at least you never call a closed branch.

**The technical reality**: Multivalue answer returns up to eight healthy records in response to a DNS query. Each record can have its own health check. Unhealthy records are excluded from the response. If no records are healthy, Route53 returns all records regardless of health status, ensuring clients still receive a DNS response rather than nothing.

**Key characteristics**:
- Returns up to 8 records per query (DNS limitation)
- Not a replacement for a load balancer -- it provides DNS-level health filtering, not connection-level load balancing
- Each record needs a unique `set_identifier` and its own health check
- Think of it as "simple routing with health checks" -- the client still picks randomly from the returned set

> **Interview tip**: "What is the difference between simple routing and multivalue answer routing?" Simple routing returns all IPs with no health filtering and no `set_identifier`. Multivalue answer returns up to 8 *healthy* IPs, each with its own health check and `set_identifier`. If health-aware DNS without a load balancer is the requirement, multivalue is the answer.
- Use case: Simple stateless services where you want DNS-level health filtering without a load balancer

```hcl
# Terraform -- Multivalue answer routing
resource "aws_route53_record" "instance_1" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  ttl     = 60

  set_identifier = "instance-1"

  records = ["10.0.1.10"]

  multivalue_answer_routing_policy = true

  health_check_id = aws_route53_health_check.instance_1.id
}

resource "aws_route53_record" "instance_2" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  ttl     = 60

  set_identifier = "instance-2"

  records = ["10.0.2.20"]

  multivalue_answer_routing_policy = true

  health_check_id = aws_route53_health_check.instance_2.id
}
```

---

## Routing Policy Comparison Table

| Policy | Use Case | Health Checks | set_identifier | Key Parameter | Active/Passive |
|--------|----------|---------------|----------------|---------------|----------------|
| Simple | Single resource, basic DNS | Not supported on the record (alias ETH only) | Not required | N/A | N/A |
| Weighted | Canary, A/B, gradual migration | Optional (recommended) | Required | `weight` (0-255) | Active-active |
| Latency | Multi-region performance | Optional (recommended) | Required | `region` | Active-active |
| Failover | DR, maintenance page | **Mandatory on primary** | Required | `type` (PRIMARY/SECONDARY) | **Active-passive** |
| Geolocation | Compliance, localization | Optional (recommended) | Required | `continent`, `country`, or `subdivision` | Active-active |
| Geoproximity | Distance-based with tuning | Optional (recommended) | Required (via Traffic Flow) | `bias` (-99 to +99) | Active-active |
| Multivalue | Simple health-filtered DNS | Optional (recommended) | Required | N/A | Active-active |

---

## Health Checks -- The Inspection System

Health checks are the nervous system that makes routing policies intelligent. Without health checks, Route53 blindly returns records for endpoints that might be down. With health checks, Route53 removes unhealthy endpoints from DNS responses before they reach users.

### Four Health Check Types

#### 1. Endpoint Health Checks -- The Flight Inspector

Route53's global network of health checker nodes sends HTTP, HTTPS, or TCP probes directly to your endpoint at regular intervals.

```
Endpoint Health Check Architecture
═══════════════════════════════════

  Route53 Health Checker Nodes (global)
  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
  │ US-E   │  │ US-W   │  │ EU-W   │  │ AP-NE  │  ... (15+ locations)
  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘
      │           │           │           │
      └───────────┴───────────┴───────────┘
                      │
                      ▼  HTTP/HTTPS/TCP probe
              ┌──────────────┐
              │ Your Endpoint │
              │ (ALB, EC2,   │
              │  on-prem)    │
              └──────────────┘

  Configuration:
  - Protocol: HTTP | HTTPS | TCP
  - Interval: 10s (fast, higher cost) or 30s (standard)
  - Failure threshold: 1-10 consecutive failures
  - String matching: Optional -- check that response body contains a specific string
  - Regions: Choose which health checker regions to use (or all)

  Decision:
  - Healthy if ≥18% of health checkers report healthy (for standard interval)
  - Unhealthy otherwise
```

**Critical gotcha**: Route53 health checkers send probes from **public IP addresses**. If your endpoint is behind a security group, you must allow inbound traffic from the [Route53 health checker IP ranges](https://ip-ranges.amazonaws.com/ip-ranges.json) (filter for `ROUTE53_HEALTHCHECKS` service). This is the number one reason health checks show "unhealthy" when the endpoint is actually running fine.

```hcl
# Terraform -- Endpoint health check with string matching
resource "aws_route53_health_check" "web" {
  fqdn              = "app.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  # Optional: response body must contain this string (first 5120 bytes only)
  search_string     = "\"status\":\"healthy\""

  # Minimum 3 regions required; only certain regions have health checker nodes
  regions = [
    "us-east-1",
    "eu-west-1",
    "ap-southeast-1",
  ]

  tags = {
    Name        = "web-app-health-check"
    Environment = "production"
  }
}
```

#### 2. Calculated Health Checks -- The Regional Aviation Authority

**The analogy**: A regional aviation authority declares an entire region "operational" based on a quorum of its airports. If the rule is "at least 3 of 5 airports must be open," then 2 airports can close and the region still counts as healthy. But if a third closes, the entire region goes unhealthy.

Calculated health checks aggregate multiple child health checks into a single parent status using three models:

| Model | Logic | Use Case |
|-------|-------|----------|
| AND | Healthy only if ALL children are healthy | Strict -- entire stack must be up |
| OR | Healthy if ANY child is healthy | Lenient -- any working component is enough |
| Threshold (N of M) | Healthy if at least N of M children are healthy | Quorum -- tolerates partial failures |

```hcl
# Terraform -- Calculated health check (2 of 3 must be healthy)
resource "aws_route53_health_check" "web_1" {
  fqdn              = "web-1.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_health_check" "web_2" {
  fqdn              = "web-2.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_health_check" "web_3" {
  fqdn              = "web-3.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_health_check" "calculated" {
  type                   = "CALCULATED"
  child_health_threshold = 2  # Healthy if at least 2 of 3 children healthy

  child_healthchecks = [
    aws_route53_health_check.web_1.id,
    aws_route53_health_check.web_2.id,
    aws_route53_health_check.web_3.id,
  ]

  tags = {
    Name = "web-stack-calculated-health"
  }
}
```

#### 3. CloudWatch Alarm Health Checks -- The Smoke Detector

**The analogy**: Instead of sending a human inspector to check the building (endpoint health check), you install a smoke detector (CloudWatch alarm) inside the building that reports its status to the control tower via radio. The inspector never visits -- the building reports its own health.

This is essential for **private resources** that Route53's public health checkers cannot reach (e.g., instances in private subnets, DynamoDB tables, internal microservices). You create a CloudWatch alarm that monitors a relevant metric (e.g., `TargetResponseTime`, `UnHealthyHostCount`, or a custom application metric), then create a Route53 health check that watches that alarm's data stream.

**Important nuance**: Route53 monitors the CloudWatch metric data stream, not the alarm state. This means Route53 can react to the metric crossing a threshold before the CloudWatch alarm itself transitions to ALARM state (because CloudWatch alarms have their own evaluation periods and datapoints-to-alarm settings). The health check watches the data, not the alarm's state machine.

```hcl
# Terraform -- CloudWatch alarm-based health check for private resource
resource "aws_cloudwatch_metric_alarm" "db_cpu" {
  alarm_name          = "rds-primary-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 60
  statistic           = "Average"
  threshold           = 90
  alarm_description   = "RDS primary CPU above 90% for 3 minutes"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.primary.id
  }
}

resource "aws_route53_health_check" "db_health" {
  type                            = "CLOUDWATCH_METRIC"
  cloudwatch_alarm_name           = aws_cloudwatch_metric_alarm.db_cpu.alarm_name
  cloudwatch_alarm_region         = data.aws_region.current.name  # Must match the region where the CloudWatch alarm was created
  insufficient_data_health_status = "LastKnownStatus"

  tags = {
    Name = "rds-cloudwatch-health-check"
  }
}
```

#### 4. Route53 Application Recovery Controller (ARC) Routing Controls -- The Manual Override Switch

**The analogy**: Every airport has an automated system that decides if it is open based on weather sensors and runway conditions (health checks). But ARC routing controls add a **manual override switch** in the control tower. Regardless of what the sensors say, the controller can flip the switch to "CLOSED" and instantly reroute all traffic. This is used for planned maintenance, pre-emptive failover before a predicted storm, or when the automated sensors are giving false positives.

ARC routing controls are simple on/off switches:
- `ON` = healthy (Route53 routes traffic to this endpoint)
- `OFF` = unhealthy (Route53 stops routing traffic to this endpoint)

**The critical architectural detail**: ARC routing controls operate on the Route53 **data plane**, which is globally distributed and designed for extreme availability. This means you can flip the switch and trigger failover even during an event that impairs the Route53 control plane (console, API) in us-east-1. This is the exact pattern the AWS DR blog post recommends: never depend on the control plane for failover.

```
Route53 ARC Architecture
═════════════════════════

  ┌──────────────────────────────────────────────────────┐
  │  Route53 Application Recovery Controller             │
  │                                                      │
  │  Cluster (5 AZs across 5 regions)                    │
  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────┐ ┌────┐    │
  │  │us-e-1  │ │us-w-2  │ │eu-w-1  │ │ap-1│ │ap-2│    │
  │  └────────┘ └────────┘ └────────┘ └────┘ └────┘    │
  │                                                      │
  │  Routing Controls:                                   │
  │  ┌──────────────────┐  ┌──────────────────┐         │
  │  │ us-east-1: ON  ✓ │  │ eu-west-1: OFF ✗ │         │
  │  └────────┬─────────┘  └────────┬─────────┘         │
  │           │                     │                    │
  │  Safety Rules:                                       │
  │  "At least 1 routing control must be ON"             │
  │  (prevents accidentally turning off everything)      │
  └───────────┼─────────────────────┼────────────────────┘
              │                     │
              ▼                     ▼
  ┌───────────────────┐  ┌───────────────────┐
  │ Route53 Health    │  │ Route53 Health    │
  │ Check: HEALTHY    │  │ Check: UNHEALTHY  │
  │ (because ON)      │  │ (because OFF)     │
  └───────────────────┘  └───────────────────┘
              │                     │
              ▼                     ▼
  Route53 Failover Record      Route53 Failover Record
  PRIMARY: returns this        SECONDARY: skipped
```

---

## Complex/Nested Routing -- The Decision Tree

This is where Route53 becomes a true traffic management platform. By using alias records that point to other Route53 records, you can **layer routing policies into a decision tree** where each level applies a different algorithm.

**The analogy**: Think of a multi-tier air traffic control system. The first controller (Level 1) decides which continent the plane should go to based on proximity. The second controller (Level 2) decides which specific airport within that continent based on whether airports are healthy. The third controller (Level 3) distributes planes across runways at the chosen airport by weight (traffic percentage).

### Example: Failover -> Latency -> Weighted Tree

The most common production pattern: failover at the top (active-passive between primary and DR clusters), latency in the middle (pick the best region within the active cluster), and weighted at the bottom (distribute across instances within each region).

```
COMPLEX ROUTING TREE — PRODUCTION MULTI-REGION DR ARCHITECTURE
══════════════════════════════════════════════════════════════

                    app.example.com
                          │
                          ▼
              ┌───────────────────────┐
              │    LEVEL 1: FAILOVER  │
              │    Active / Passive   │
              └─────┬───────────┬─────┘
                    │           │
              PRIMARY│     SECONDARY
              (alias)│     (alias)
                    │           │
         ┌──────────▼──┐  ┌────▼──────────┐
         │ LEVEL 2:    │  │ LEVEL 2:      │
         │ LATENCY     │  │ STATIC S3     │
         │ (pick best  │  │ "Maintenance" │
         │  region)    │  │  page         │
         └──┬──────┬───┘  └───────────────┘
            │      │
    us-east-1│  eu-west-1
     (alias) │   (alias)
            │      │
   ┌────────▼──┐ ┌─▼──────────┐
   │ LEVEL 3:  │ │ LEVEL 3:   │
   │ WEIGHTED  │ │ WEIGHTED   │
   │           │ │            │
   │ v2: 90%  │ │ v2: 90%   │
   │ v3: 10%  │ │ v3: 10%   │
   └───┬───┬──┘ └──┬────┬───┘
       │   │       │    │
       ▼   ▼       ▼    ▼
    ALB  ALB     ALB   ALB
   (v2) (v3)   (v2)  (v3)


  HEALTH PROPAGATION (bottom-up):
  ═══════════════════════════════
  1. ALB has its own target group health checks (application-level)
  2. Weighted alias records use "Evaluate Target Health" → inherits ALB health
  3. Latency alias records use "Evaluate Target Health" → if ALL weighted records
     in a region are unhealthy, the latency record for that region becomes unhealthy
  4. Route53 backs up to the next-best-latency region
  5. If ALL latency records are unhealthy, the failover PRIMARY becomes unhealthy
  6. Route53 serves the SECONDARY (S3 maintenance page)
```

**How health propagates up the tree**: This is the most important concept in complex Route53 architectures. When you set `evaluate_target_health = true` on an alias record, the health of the alias target propagates upward automatically. You do not need to create separate health checks for each intermediate alias -- the health status cascades from the bottom (actual endpoints) to the top (the record the user queries).

The cascade works like this:
1. An individual ALB target group reports unhealthy targets
2. The ALB itself becomes unhealthy (no healthy targets)
3. The weighted alias record for that ALB inherits the unhealthy status via Evaluate Target Health
4. If ALL weighted records in a region are unhealthy, the latency alias record for that region becomes unhealthy
5. Route53 backs up to the next-best-latency region
6. If ALL latency records are unhealthy, the failover PRIMARY is unhealthy
7. Route53 returns the SECONDARY

---

## Active-Active vs Active-Passive Failover

These are the two fundamental failover models in Route53, and every routing policy maps to one of them.

### Active-Active: All Airports Open

**Every healthy endpoint serves traffic simultaneously.** If one fails, the remaining endpoints absorb its traffic. You implement active-active failover using **any routing policy except failover** (weighted, latency, geolocation, multivalue) combined with health checks.

```
Active-Active with Latency Routing
═══════════════════════════════════

  NORMAL: Both regions serve traffic based on latency
  ┌─────────────────────────────────────────────────┐
  │  us-east-1    ◄──── US users (lower latency)    │
  │  eu-west-1    ◄──── EU users (lower latency)    │
  └─────────────────────────────────────────────────┘

  PARTIAL FAILURE: us-east-1 unhealthy → all traffic to eu-west-1
  ┌─────────────────────────────────────────────────┐
  │  us-east-1    ✗ UNHEALTHY (removed from DNS)    │
  │  eu-west-1    ◄──── ALL users (only healthy)    │
  └─────────────────────────────────────────────────┘

  RECOVERY: us-east-1 recovers → traffic rebalances
  ┌─────────────────────────────────────────────────┐
  │  us-east-1    ◄──── US users (restored)         │
  │  eu-west-1    ◄──── EU users (restored)         │
  └─────────────────────────────────────────────────┘
```

### Active-Passive: One Airport on Standby

**Only the primary endpoint serves traffic.** The secondary sits idle (cold or warm standby) until the primary fails. You implement active-passive using the **failover routing policy** specifically.

Active-passive is appropriate when:
- The DR region is a scaled-down, cost-optimized standby (not fully provisioned)
- The application cannot tolerate split-brain or data conflicts across regions
- You want predictable, explicit failover behavior
- The secondary is a static "sorry" page (S3 website)

### Three Active-Passive Patterns

```
Pattern 1: Simple Primary/Secondary
════════════════════════════════════
  PRIMARY (us-east-1 ALB) ──HC──→ SECONDARY (eu-west-1 ALB)


Pattern 2: Multiple Primary Resources Behind Weighted
═══════════════════════════════════════════════════════
  FAILOVER PRIMARY (alias) ──→ Weighted group:
                                 ├── Instance A (weight=50)
                                 └── Instance B (weight=50)
  FAILOVER SECONDARY (alias) ──→ eu-west-1 ALB


Pattern 3: ARC Routing Controls for Deliberate Failover
═══════════════════════════════════════════════════════
  FAILOVER PRIMARY (alias)
       │
       └──→ HC linked to ARC routing control (ON/OFF switch)
            Flip to OFF → instant failover, no DNS record changes
  FAILOVER SECONDARY (alias)
       │
       └──→ HC linked to ARC routing control (ON/OFF switch)
```

---

## Data Plane vs Control Plane -- The Critical Reliability Distinction

This is the most architecturally important concept in Route53 for disaster recovery design.

**The analogy**: An airport has two systems. The **radar and radio system** (data plane) runs 24/7 on distributed, hardened infrastructure -- it can route planes even during a natural disaster. The **administrative office** (control plane) is where you file flight plans, update records, and manage configurations -- it is in one building and could be knocked offline by a local power outage. During a crisis, you need the radar and radios working. You do NOT want your failover plan to depend on walking into the admin office to file paperwork.

| Component | Plane | Location | Availability |
|-----------|-------|----------|--------------|
| DNS resolution (answering queries) | Data plane | Globally distributed across 200+ edge locations | Designed for 100% availability |
| Health checks | Data plane | Globally distributed health checker nodes | Designed for 100% availability |
| ARC routing controls | Data plane | Distributed across 5 regions in a cluster | Designed for extreme availability |
| Route53 console | Control plane | us-east-1 | Could be impaired during regional events |
| Route53 API (ChangeResourceRecordSets) | Control plane | us-east-1 | Could be impaired during regional events |
| Traffic Flow policy editor | Control plane | us-east-1 | Could be impaired during regional events |

> **Interview tip**: If an interviewer asks "how would you implement automated DR failover for Route53?", the answer must mention health checks or ARC routing controls (data plane). If your answer involves the Route53 API or console, you have described a broken DR plan. This distinction between data plane and control plane is what separates senior-level answers.

**The rule**: Your failover mechanism must NEVER depend on the control plane being available during the event that triggers failover. If your runbook says "during an outage, log into the Route53 console and change the DNS record," you have a broken DR plan. Health checks and ARC routing controls operate on the data plane -- they work even when us-east-1 is impaired.

```
DATA PLANE vs CONTROL PLANE IN A us-east-1 OUTAGE SCENARIO
═══════════════════════════════════════════════════════════

                    us-east-1 IMPAIRED
                    ┌────────────────────────────────┐
                    │  Route53 Console      ✗ DOWN   │
                    │  Route53 API          ✗ DOWN   │
                    │  Your primary app     ✗ DOWN   │
                    └────────────────────────────────┘

  ✓ GOOD PATTERN: Automated failover via health checks (data plane)
  ────────────────────────────────────────────────────────────────
  Route53 Health Check (data plane, running globally)
       │
       └──→ Detects: primary endpoint in us-east-1 is down
       └──→ Action: automatically returns SECONDARY record (eu-west-1)
       └──→ NO DEPENDENCY on us-east-1 control plane

  ✓ GOOD PATTERN: Manual failover via ARC routing controls (data plane)
  ────────────────────────────────────────────────────────────────────
  ARC Routing Control (data plane, 5-region cluster)
       │
       └──→ Operator flips switch: us-east-1 → OFF
       └──→ Route53 health check linked to ARC → unhealthy
       └──→ Failover record returns SECONDARY (eu-west-1)
       └──→ NO DEPENDENCY on us-east-1 control plane

  ✗ BAD PATTERN: Manual DNS change via console/API (control plane)
  ────────────────────────────────────────────────────────────────
  Operator tries to log into Route53 console
       │
       └──→ Console is in us-east-1 → UNAVAILABLE
       └──→ API is in us-east-1 → UNAVAILABLE
       └──→ Failover BLOCKED. Users stuck hitting dead endpoint
            until us-east-1 recovers.
```

**Defense in depth on the data plane**: A robust DR plan needs **two independent failover paths, both on the data plane**: (1) automated health checks as the first line of defense, and (2) ARC routing controls (or an equivalent data-plane manual override) as the second line. The structural test for any DR plan: "If every automated mechanism fails to trigger, does the manual escape hatch also depend on the thing that is broken?" If yes, you have a single point of failure in your recovery mechanism -- which is arguably worse than a single point of failure in your application, because it means you cannot recover.

---

## Failover Timing -- How Fast Does It Actually Switch?

The total failover time from endpoint failure to all users hitting the healthy endpoint is the sum of three stages:

```
FAILOVER TIMELINE
═════════════════

  Endpoint fails
       │
       ├──── Stage 1: Health check detection ────────────────────────┐
       │     Interval (10s or 30s) x Failure Threshold (1-10)       │
       │     Best case:  10s x 1  = 10 seconds                      │
       │     Default:    30s x 3  = 90 seconds                      │
       │     Worst case: 30s x 10 = 300 seconds                     │
       │                                                             │
       ├──── Stage 2: DNS propagation ───────────────────────────────┤
       │     Route53 updates the record set (near-instant)           │
       │                                                             │
       ├──── Stage 3: TTL expiration ────────────────────────────────┤
       │     Clients and resolvers cache the old answer              │
       │     until the TTL expires.                                  │
       │     Low TTL (60s) → fast failover                           │
       │     High TTL (300s) → slow failover                         │
       │                                                             │
       ▼                                                             │
  All users hitting healthy endpoint                                 │
                                                                     │
  TOTAL BEST CASE:  10s + ~0s + 60s  = ~70 seconds                  │
  TOTAL TYPICAL:    90s + ~0s + 60s  = ~150 seconds                  │
  TOTAL WORST CASE: 300s + ~0s + 300s = ~600 seconds (10 min)       │
  ───────────────────────────────────────────────────────────────────┘

  TO MINIMIZE FAILOVER TIME:
  - Use fast-interval health checks (10s instead of 30s) ← costs more
  - Set failure threshold to 1 or 2 (aggressive, risk of false positives)
  - Use low TTL on DNS records (60s)
  - Use alias records with Evaluate Target Health (no separate HC needed)
```

---

## Cost Considerations

| Component | Pricing | Key Detail |
|-----------|---------|------------|
| Hosted zone | $0.50/month per hosted zone | First 25 hosted zones are $0.50/month each |
| Standard queries | $0.40 per million queries (first 1B) | Alias queries to AWS resources: **free** |
| Latency queries | $0.60 per million | Premium over standard queries |
| Geolocation/Geoproximity queries | $0.70 per million | Most expensive query type |
| Health checks (endpoint, standard) | $0.50/month per health check | Interval: 30s |
| Health checks (endpoint, fast) | $1.00/month per health check | Interval: 10s |
| Health checks (string matching) | +$1.00/month per health check | On top of base cost |
| Health checks (HTTPS) | +$0.75/month per health check | On top of base cost |
| Calculated health checks | $0.50/month per health check | Same as basic endpoint |
| ARC routing controls | $2.50/hour per cluster (~$1,825/month) | Significant cost -- justified only for critical DR |

**Cost optimization tip**: Use alias records pointing to AWS resources (ALB, CloudFront, S3) whenever possible. Alias queries are free, and Evaluate Target Health eliminates the need for separate health checks on intermediate records. A complex routing tree with 10 alias records costs $0 in query charges and $0 in health check charges for the alias layers -- only the bottom-level endpoint health checks cost money.

---

## Key Takeaways

- **Route53 routing happens at the DNS layer, not the data path.** Route53 returns an IP address and gets out of the way. It does not proxy or forward traffic. This means TTL directly controls how quickly clients pick up routing changes -- low TTL (60s) for faster failover, higher TTL (300s) for less DNS query cost.

- **Every routing policy except Simple requires a `set_identifier`.** This is the unique label that distinguishes records with the same name and type. Forget it and Terraform will error. Choose descriptive identifiers like the region name or deployment version.

- **Failover routing is the ONLY active-passive policy.** All other policies (weighted, latency, geolocation, geoproximity, multivalue) are active-active by nature -- all healthy endpoints serve traffic simultaneously. If you want active-passive, use the failover routing policy specifically.

- **Always create a default record for geolocation routing.** Without a default catch-all, users from unmatched locations receive NODATA (no DNS response at all). This is a silent failure that is hard to debug in production.

- **Geolocation vs Geoproximity -- know the difference cold.** Geolocation uses hard boundaries based on GeoIP mapping (continent/country/state) and is for compliance ("EU users MUST hit EU servers"). Geoproximity uses soft, distance-based boundaries with a tunable bias dial and is for performance optimization ("shift 20% more traffic to us-west-2"). Mixing these up is the most common interview mistake.

- **Health checks from Route53 come from public IPs.** If your security groups do not allow inbound traffic from the Route53 health checker IP ranges, the health check will report unhealthy even though the endpoint works fine. Allow the `ROUTE53_HEALTHCHECKS` IP ranges in your security groups.

- **Use CloudWatch alarm health checks for private resources.** Route53 health checkers cannot reach instances in private subnets, internal NLBs, DynamoDB tables, or RDS instances. For these, create a CloudWatch alarm on a relevant metric and link a Route53 health check to that alarm.

- **Failover must use the data plane, not the control plane.** Health checks and ARC routing controls operate on Route53's globally distributed data plane. The Route53 console and API operate on the control plane in us-east-1. Your DR plan must never depend on the control plane being available during the event that triggered failover. If your runbook says "change the DNS record," your DR plan is broken. A robust DR plan needs two independent data-plane failover paths: automated health checks as the first line, and ARC routing controls as the manual override second line.

- **Evaluate Target Health on alias records eliminates intermediate health checks.** When you build a complex routing tree with alias records at every level, setting `evaluate_target_health = true` lets health status cascade from the bottom (endpoint health checks) to the top (the record users query) without creating separate health checks for each alias layer. This saves money and reduces configuration complexity.

- **Weighted routing has a zero-weight fallback rule.** Records with weight 0 are never returned in normal operation. But if ALL non-zero-weight records are unhealthy, Route53 promotes healthy zero-weight records. If ALL records are unhealthy, Route53 treats them as healthy and uses original weights -- meaning zero-weight records are still excluded even in the all-unhealthy scenario.

- **Minimize failover time with three levers.** Fast-interval health checks (10s), low failure threshold (1-2), and low DNS TTL (60s). Best case is approximately 70 seconds from endpoint failure to all users hitting the healthy endpoint. Default settings give you approximately 150 seconds.

- **ARC routing controls cost $2.50/hour per cluster (~$1,825/month).** This is significant and only justified for business-critical applications where you need deliberate, data-plane-based failover capability. For most applications, health-check-based automatic failover is sufficient and far cheaper.

---

## Further Reading

- [Choosing a Routing Policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html) -- official overview and navigation hub for all routing policies
- [Active-Active and Active-Passive Failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html) -- the two failover models with configuration patterns
- [Health Checks in Complex Configurations](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-complex-configs.html) -- how to build multi-level routing trees with Evaluate Target Health
- [Route53 Health Check Types](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-types.html) -- endpoint, calculated, CloudWatch alarm, and ARC routing controls
- [Creating DR Mechanisms Using Route53](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/) -- data plane vs control plane reliability for failover
- [Route53 ARC Developer Guide](https://docs.aws.amazon.com/r53recovery/latest/dg/what-is-route53-recovery.html) -- clusters, routing controls, and safety rules
- [Route53 IP Ranges for Health Checkers](https://ip-ranges.amazonaws.com/ip-ranges.json) -- filter by `ROUTE53_HEALTHCHECKS` service for security group rules
