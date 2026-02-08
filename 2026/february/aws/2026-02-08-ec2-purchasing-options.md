# AWS EC2: Purchasing Options - The Airline Ticket Analogy

> Understanding EC2 Spot Instances, Reserved Instances, Savings Plans, and Capacity Reservations through a real-world analogy of buying airline tickets for business travel.

---

## TL;DR

| AWS EC2 Concept | Real-World Analogy | One-Liner |
|-----------------|-------------------|-----------|
| On-Demand Instance | Walk-up airline ticket | Full price, no commitment, fly whenever you want |
| Spot Instance | Standby/empty seat auction | Up to 90% off, but the airline can bump you with 2 minutes notice |
| Spot Capacity Pool | Flights on a specific route at a specific time | Each route+time combo has its own seat availability and price |
| Spot Interruption | Getting bumped from an oversold flight | Airline reclaims your seat; you get 2 min to grab your bags (unlike real airlines, there's no compensation -- you agreed to these terms for the discount) |
| Rebalance Recommendation | "Your flight is overbooked, consider rebooking" | Early warning before you actually get bumped |
| Price-Capacity-Optimized | Choosing flights with the most empty seats AND lowest fares | Best balance of availability and price |
| Reserved Instance (Standard) | Non-refundable annual airline pass for a specific route | Big discount, locked to that exact route for 1-3 years |
| Reserved Instance (Convertible) | Flexible annual pass - can change routes | Smaller discount, but you can switch airlines/routes |
| Zonal RI | Reserved seat on a specific flight | Guaranteed seat AND discount on that exact flight |
| Regional RI | Discount for any flight in a region | Discount applies anywhere in the region, but no guaranteed seat |
| RI Marketplace | Reselling your unused airline pass | Sell your remaining commitment to another traveler |
| Compute Savings Plan | Spending commitment across any airline/route | "I'll spend $X/hr on flights" - works on any airline, any route |
| EC2 Instance Savings Plan | Spending commitment on one airline's routes | Deeper discount, but locked to one airline (instance family) |
| On-Demand Capacity Reservation | Reserving a seat without a discount | Guarantees a seat exists for you, but costs full price unless paired with a pass |
| Combined Strategy | Travel department with mixed booking strategies | Mix of passes, standby, and reservations for different trip types |

---

## The Big Picture

Imagine you are the **travel manager for a large company** with hundreds of employees who fly constantly. Some trips are planned months in advance (production workloads), some are unpredictable (dev/test), and some could be cancelled or rescheduled without consequence (batch processing). Your job is to minimize travel costs while ensuring everyone gets where they need to go.

AWS gives you five ways to "book flights" (purchase compute):

```
EC2 PURCHASING OPTIONS — THE COMMITMENT vs SAVINGS SPECTRUM
=============================================================================

  Maximum Flexibility                                    Maximum Savings
  No Commitment                                          Full Commitment
  ◄──────────────────────────────────────────────────────────────────────►

  On-Demand         Capacity        Savings        Reserved
  (Walk-up)         Reservation     Plans          Instances
                    (Seat hold)     ($ commit)     (Route pass)

  0% savings        0% savings*     Up to 72%      Up to 72%
  No commitment     No term req.    1-3 year $     1-3 year
  Always available  Guarantees      Flexible       Less flexible
                    capacity        across         locked to
                                    services       config

  * Capacity Reservations provide no discount alone —
    pair with Savings Plans or Regional RIs for savings

  ┌─────────────────────────────────────────────────────────────────────┐
  │ SPOT INSTANCES — The Outlier (doesn't fit the linear spectrum)     │
  │                                                                     │
  │   Up to 90% savings AND no commitment — but with the risk of       │
  │   interruption. Spot is unique: highest savings, zero commitment,   │
  │   at the cost of reliability. Think of it as a different dimension  │
  │   entirely — you trade STABILITY (not commitment) for savings.      │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Spot Instances - Flying Standby at 90% Off

### The Analogy

You walk up to the gate and say, "I'll take any empty seat for 10 cents on the dollar." The airline loves this -- an empty seat earns nothing, so selling it cheap is pure profit. But there is a catch: **if a full-fare passenger shows up, you get bumped with 2 minutes to grab your bags.** You knew this when you bought the ticket. Smart standby travelers bring carry-on only, have backup plans, and book across multiple flights.

### The Technical Reality

Spot Instances let you use spare EC2 capacity at up to 90% off On-Demand pricing. AWS can reclaim these instances with a **2-minute interruption notice** when it needs the capacity back. The price fluctuates based on supply and demand but changes gradually -- AWS no longer uses the old auction-style bidding model.

### How They Connect

| Analogy Element | Technical Component |
|----------------|-------------------|
| Empty seats on a flight | Spare capacity in a Spot capacity pool |
| Specific route + time | Instance type + AZ = one capacity pool |
| Getting bumped | Spot interruption (2-minute warning via instance metadata) |
| "Your flight is overbooked, consider rebooking" | EC2 Rebalance Recommendation (early warning) |
| Booking across multiple flights | Diversifying across 10+ instance types and multiple AZs |
| Choosing flights with most empty seats | Price-capacity-optimized allocation strategy |

### Spot Capacity Pools

```
SPOT CAPACITY POOLS
=============================================================================

A capacity pool = one instance type in one AZ

  us-east-1a                us-east-1b                us-east-1c
  ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
  │ m5.large     │          │ m5.large     │          │ m5.large     │
  │ Price: $0.01 │          │ Price: $0.02 │          │ Price: $0.01 │
  │ Capacity: ██ │          │ Capacity: █  │          │ Capacity: ███│
  ├──────────────┤          ├──────────────┤          ├──────────────┤
  │ c5.large     │          │ c5.large     │          │ c5.large     │
  │ Price: $0.02 │          │ Price: $0.01 │          │ Price: $0.03 │
  │ Capacity: █  │          │ Capacity: ███│          │ Capacity: ██ │
  ├──────────────┤          ├──────────────┤          ├──────────────┤
  │ r5.large     │          │ r5.large     │          │ r5.large     │
  │ Price: $0.03 │          │ Price: $0.02 │          │ Price: $0.02 │
  │ Capacity: ███│          │ Capacity: ██ │          │ Capacity: █  │
  └──────────────┘          └──────────────┘          └──────────────┘

  Each pool has independent:
  ├── Price (fluctuates based on supply/demand)
  ├── Available capacity (changes as AWS provisions/reclaims)
  └── Interruption rate (varies significantly by pool — check the
      Spot Instance Advisor for current rates by instance type)

  KEY INSIGHT: More pools you can use = better availability + lower cost
  AWS recommends: Be flexible across at LEAST 10 instance types
```

### Spot Interruption Handling

```
SPOT INSTANCE LIFECYCLE
=============================================================================

  ┌───────────┐     ┌──────────────┐     ┌──────────────────────────┐
  │ Request   │────▶│ Running      │────▶│ Interrupted              │
  │ Spot      │     │ (discounted) │     │ (2-min warning)          │
  └───────────┘     └──────────────┘     └──────────────────────────┘
                          │                         │
                          │                         ├── Terminate (default)
                          │                         ├── Stop (can restart)
                          │                         └── Hibernate (resume)
                          │
                          ▼
                    ┌──────────────┐
                    │ Rebalance    │  ← EARLY WARNING (before interruption)
                    │ Recommend.   │     "Elevated risk of interruption"
                    └──────────────┘     Gives you time to proactively migrate


HOW TO DETECT INTERRUPTIONS:
=============================================================================

  Method 1: Instance Metadata (poll every 5 seconds)
  ─────────────────────────────────────────────────
  $ curl http://169.254.169.254/latest/meta-data/spot/instance-action
  # Returns: {"action": "terminate", "time": "2026-02-08T12:30:00Z"}

  Method 2: EventBridge Rule (RECOMMENDED - event-driven)
  ─────────────────────────────────────────────────
  Event pattern:
  {
    "source": ["aws.ec2"],
    "detail-type": [
      "EC2 Spot Instance Interruption Warning",
      "EC2 Instance Rebalance Recommendation"
    ]
  }

  Method 3: CloudWatch (monitoring)
  ─────────────────────────────────────────────────
  Monitor SpotInstanceInterruptions metric for trends
```

### Spot Allocation Strategies

```
ALLOCATION STRATEGIES
=============================================================================

  price-capacity-optimized (RECOMMENDED)
  ───────────────────────────────────────
  AWS picks pools with: most capacity available + lowest price
  Result: Low interruptions AND low cost
  Best for: Most workloads (containerized, batch, stateless)

  capacity-optimized
  ───────────────────────────────────────
  AWS picks pools with: most capacity available (ignores price)
  Result: Lowest interruptions, but may cost more
  Best for: Workloads with high interruption cost (long-running jobs)

  lowest-price (NOT RECOMMENDED)
  ───────────────────────────────────────
  AWS picks pools with: lowest price (ignores capacity)
  Result: Cheapest, but highest interruption rate
  Avoid: Concentrates instances in few pools = correlated interruptions

  diversified
  ───────────────────────────────────────
  AWS distributes evenly across all specified pools
  Result: Maximum spread, moderate price/interruption
  Best for: When you need predictable distribution
```

### Spot Fleet vs EC2 Fleet vs ASG

```
WHICH SPOT PROVISIONING METHOD TO USE?
=============================================================================

  Spot Fleet (LEGACY)
  ─────────────────────────────────────
  The original way to request Spot capacity. Manages its own fleet
  of instances. Still works, but AWS recommends migrating away.

  EC2 Fleet (NEWER)
  ─────────────────────────────────────
  Successor to Spot Fleet. Can launch a mix of On-Demand and Spot in
  a single API call. Useful for one-time batch launches.

  ASG with mixed_instances_policy (RECOMMENDED)
  ─────────────────────────────────────
  The modern approach. Combines Spot + On-Demand in a single ASG with
  auto-scaling, health checks, lifecycle hooks, and rolling updates.
  This is what you should use for most production workloads.

  This doc uses ASG because it's the current best practice.
```

### Terraform: Spot Instances with Auto Scaling

```hcl
# Spot Instances via Auto Scaling Group with mixed instances policy
resource "aws_autoscaling_group" "spot_workers" {
  name                = "batch-workers"
  desired_capacity    = 10
  min_size            = 0
  max_size            = 50
  vpc_zone_identifier = var.private_subnet_ids

  mixed_instances_policy {
    # On-Demand base capacity for stability
    instances_distribution {
      on_demand_base_capacity                  = 2    # 2 On-Demand as baseline
      on_demand_percentage_above_base_capacity = 0    # Rest is Spot
      spot_allocation_strategy                 = "price-capacity-optimized"
    }

    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.worker.id
        version            = "$Latest"
      }

      # Diversify across 10+ instance types (Spot best practice)
      override {
        instance_type = "m5.large"
      }
      override {
        instance_type = "m5a.large"
      }
      override {
        instance_type = "m5d.large"
      }
      override {
        instance_type = "m4.large"
      }
      override {
        instance_type = "m6i.large"
      }
      override {
        instance_type = "c5.large"
      }
      override {
        instance_type = "c5a.large"
      }
      override {
        instance_type = "c5d.large"
      }
      override {
        instance_type = "c4.large"
      }
      override {
        instance_type = "r5.large"
      }

      # ALTERNATIVE: Attribute-based instance type selection
      # Instead of manually listing 10+ types, define requirements
      # and let AWS pick matching instance types automatically.
      #
      # override {
      #   instance_requirements {
      #     memory_mib {
      #       min = 8192   # 8 GiB minimum
      #     }
      #     vcpu_count {
      #       min = 2
      #       max = 8
      #     }
      #     instance_generations = ["current"]
      #     burstable_performance = "excluded"
      #
      #     # Optional: restrict CPU manufacturers, architectures, etc.
      #     # cpu_manufacturers       = ["amazon-web-services", "intel", "amd"]
      #     # allowed_instance_types  = ["m5.*", "m6i.*", "c5.*", "r5.*"]
      #     # excluded_instance_types = ["m5.metal"]
      #   }
      # }
      #
      # This is especially useful when you want broad diversification
      # without maintaining a long manual list. AWS will select from
      # all instance types matching your constraints.
    }
  }

  tag {
    key                 = "Name"
    value               = "batch-worker"
    propagate_at_launch = true
  }
}

# Launch template with Spot-friendly configuration
resource "aws_launch_template" "worker" {
  name_prefix   = "batch-worker-"
  image_id      = data.aws_ami.amazon_linux.id

  # NOTE: Do NOT set instance_market_options here when using mixed_instances_policy
  # on the ASG — the ASG's instances_distribution block controls Spot vs On-Demand
  # allocation. Setting both causes a conflict.

  # User data: graceful shutdown script for Spot interruptions
  user_data = base64encode(<<-EOF
    #!/bin/bash
    # Monitor for Spot interruption and drain gracefully
    while true; do
      RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
        http://169.254.169.254/latest/meta-data/spot/instance-action)
      if [ "$RESPONSE" == "200" ]; then
        echo "Spot interruption detected! Draining..."
        # Checkpoint current work
        # Deregister from load balancer
        # Signal ASG to launch replacement
        /opt/scripts/graceful-shutdown.sh
        break
      fi
      sleep 5
    done &
  EOF
  )
}

# EventBridge rule to capture Spot interruption warnings
resource "aws_cloudwatch_event_rule" "spot_interruption" {
  name        = "spot-interruption-warning"
  description = "Capture EC2 Spot Instance interruption warnings"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = [
      "EC2 Spot Instance Interruption Warning",
      "EC2 Instance Rebalance Recommendation"
    ]
  })
}

resource "aws_cloudwatch_event_target" "spot_sns" {
  rule      = aws_cloudwatch_event_rule.spot_interruption.name
  target_id = "spot-interruption-sns"
  arn       = aws_sns_topic.alerts.arn
}
```

---

## Part 2: Reserved Instances - The Annual Airline Pass

### The Analogy

Your company knows that certain employees fly the **same route every single week** -- New York to Chicago, Monday morning, same airline. For these predictable trips, you buy an **annual pass** at a steep discount. Standard passes are locked to that exact route and airline but save the most money. Convertible passes cost a bit more but let you switch routes or airlines mid-year. Zonal passes come with a **guaranteed seat** on a specific flight; regional passes give you a discount on **any flight in the region** but no guaranteed seat.

### The Technical Reality

Reserved Instances provide up to 72% discount over On-Demand pricing in exchange for a 1- or 3-year commitment to a specific instance configuration. They come in two offering classes and two scopes, and offer three payment options that affect the discount level.

### Offering Classes

```
STANDARD vs CONVERTIBLE RESERVED INSTANCES
=============================================================================

STANDARD RI — "Non-refundable route-locked pass"
─────────────────────────────────────────────────
  Discount:     Up to 72% off On-Demand
  Locked to:    Instance family, OS, tenancy, region
  Size flex:    YES (Regional scope) — discount applies across sizes in family
                NO (Zonal scope) — must match exact instance size
  Can change:   AZ (within region), network type
  Can sell:     YES — on RI Marketplace
  Best for:     Stable, well-understood workloads that won't change


CONVERTIBLE RI — "Flexible pass - can change routes"
─────────────────────────────────────────────────
  Discount:     Up to 66% off On-Demand
  Locked to:    Only the term commitment
  Can change:   Instance family, size, OS, tenancy (via exchange)
  Can sell:     NO — cannot sell on RI Marketplace
  Best for:     When you want savings but need flexibility to evolve

  EXCHANGE RULE: New RI must be of EQUAL OR GREATER value
  ├── You can "trade up" but not "trade down"
  └── Example: Convert 1x r5.xlarge to 2x m5.large (if equal/greater value)
```

### Scope: Regional vs Zonal

```
REGIONAL vs ZONAL RESERVED INSTANCES
=============================================================================

REGIONAL RI — "Discount for any flight in the region"
─────────────────────────────────────────────────
  Scope:              Entire AWS Region
  Capacity Reserved:  NO — discount only, no guaranteed capacity
  Size Flexibility:   YES — discount applies across sizes in same family
  Queuing:            YES — can queue for future start date
  Applied to:         Any matching instance in any AZ in that region

  Example: Regional r5 RI in us-east-1
  ├── r5.large in us-east-1a  → discount applies ✅
  ├── r5.xlarge in us-east-1b → discount applies ✅ (size flex)
  └── r5.large in us-east-1c  → discount applies ✅


ZONAL RI — "Reserved seat on a specific flight"
─────────────────────────────────────────────────
  Scope:              Specific Availability Zone
  Capacity Reserved:  YES — guarantees capacity in that AZ
  Size Flexibility:   NO — must match exact instance size
  Queuing:            NO — starts immediately
  Applied to:         Only matching instance in that exact AZ

  Example: Zonal r5.large RI in us-east-1a
  ├── r5.large in us-east-1a  → discount + capacity ✅
  ├── r5.large in us-east-1b  → neither ❌
  └── r5.xlarge in us-east-1a → neither ❌ (no size flex)


DECISION:
─────────────────────────────────────────────────
  Need guaranteed capacity in a specific AZ?
  │
  ├── YES → Zonal RI (or consider On-Demand Capacity Reservation)
  │
  └── NO → Regional RI (more flexible, size flexibility)
```

### RI Size Flexibility: Normalization Factors

Regional RIs apply discounts across different sizes within the same instance family using a **normalization factor** system. Each instance size has a factor relative to `small` (factor = 1):

```
INSTANCE SIZE NORMALIZATION FACTORS
=============================================================================

  Size          Factor      Equivalent smalls
  ──────────    ──────      ─────────────────
  nano          0.25        1/4 of a small
  micro         0.5         1/2 of a small
  small         1           baseline
  medium        2           2 smalls
  large         4           4 smalls
  xlarge        8           8 smalls
  2xlarge       16          16 smalls
  4xlarge       32          32 smalls
  8xlarge       64          64 smalls

  EXAMPLE: You buy one Regional r5.xlarge RI (factor = 8)
  ─────────────────────────────────────────────────────────
  That single RI can cover ANY of these:
  ├── 1x r5.xlarge     (8 = 8)  ✅
  ├── 2x r5.large      (4+4 = 8)  ✅
  ├── 4x r5.medium     (2+2+2+2 = 8)  ✅
  ├── 1x r5.large + 2x r5.medium  (4+2+2 = 8)  ✅
  └── Any combo that sums to factor 8

  NOTE: Normalization only applies to Regional RIs within the SAME family.
  An r5.xlarge RI cannot cover m5 instances.
```

### Payment Options

```
PAYMENT OPTIONS — DISCOUNT vs CASH FLOW
=============================================================================

              ┌────────────────┬──────────────┬──────────────┐
              │  All Upfront   │ Partial Up.  │  No Upfront  │
              ├────────────────┼──────────────┼──────────────┤
  Discount    │  Highest       │  Medium      │  Lowest      │
              │  (~72% Std)    │  (~60% Std)  │  (~30-45%*)  │
              ├────────────────┼──────────────┼──────────────┤
  Cash flow   │  Large upfront │  Split cost  │  No upfront  │
              │  payment       │  upfront +   │  monthly     │
              │                │  monthly     │  payments    │
              ├────────────────┼──────────────┼──────────────┤
  Risk        │  Highest       │  Medium      │  Lowest      │
              │  (money locked │              │  (but still  │
              │  for term)     │              │  committed)  │
              └────────────────┴──────────────┴──────────────┘

  * No Upfront discounts vary significantly by instance type, region,
    and term length. Always check current pricing for your specific config.

  1-YEAR vs 3-YEAR TERM:
  ├── 1-year: Lower commitment, lower discount
  ├── 3-year: Higher commitment, highest discount
  └── Longer term = bigger discount, but more risk if needs change
```

### RI Marketplace

```
RI MARKETPLACE — Reselling Your Unused Pass
=============================================================================

THE ANALOGY:
You bought a 3-year annual pass for the NYC-Chicago route.
After 18 months, your company closes the Chicago office.
You can sell the remaining 18 months on the marketplace to
another traveler who wants that route.

RULES:
├── Only STANDARD RIs can be sold (not Convertible)
├── Must have been active for at least 30 days
├── Seller sets the upfront price (can take a loss to sell faster)
├── Remaining term appears as-is to buyer
├── AWS charges 12% service fee on upfront price
└── Buyer gets the RI at potentially shorter term and lower price

BUYER TIP:
├── Can find 6-month or 9-month RIs at discounts
├── Good for: Known short-term projects
└── Check marketplace before buying full-term directly from AWS
```

---

## Part 3: Savings Plans - The Corporate Travel Budget

### The Analogy

Instead of buying passes for specific routes, your company commits to **spending a fixed dollar amount per hour on travel** -- any airline, any route, any class. You tell the travel agency, "We will spend at least $50/hour on flights for the next year." In return, you get a discount on every ticket. The more restrictive your commitment (only one airline), the deeper the discount. But the beauty is you never have to specify which flights -- the discount just applies automatically.

### The Technical Reality

Savings Plans are a commitment to a **consistent amount of compute usage** (measured in dollars per hour) for a 1- or 3-year term. Unlike RIs which commit to a specific instance configuration, Savings Plans commit to a spending level and apply discounts automatically to qualifying usage.

### Savings Plan Types

```
SAVINGS PLAN TYPES
=============================================================================

COMPUTE SAVINGS PLAN — "Any airline, any route"
─────────────────────────────────────────────────
  Discount:    Up to 66% off On-Demand
  Flexibility: Maximum
  ├── Any instance family (m5, c5, r5, etc.)
  ├── Any instance size (large, xlarge, 2xlarge, etc.)
  ├── Any Region (us-east-1, eu-west-1, etc.)
  ├── Any OS (Linux, Windows, etc.)
  ├── Any tenancy (shared, dedicated)
  ├── Also covers: AWS Fargate and AWS Lambda
  │   (Like a general transportation budget that covers trains and ride-shares too)
  └── Discount applied AUTOMATICALLY to highest-discount usage first

  Best for: Organizations with diverse, evolving compute needs


EC2 INSTANCE SAVINGS PLAN — "One airline, any route"
─────────────────────────────────────────────────
  Discount:    Up to 72% off On-Demand
  Flexibility: Moderate
  ├── Locked to: One instance FAMILY in one REGION (e.g., m5 in us-east-1)
  ├── Flexible: Instance size, OS, tenancy
  ├── Does NOT cover: Fargate, Lambda, other families, other regions
  └── Discount applied AUTOMATICALLY

  Best for: Stable workloads within a known instance family and region


SAGEMAKER SAVINGS PLAN (not covered here)
─────────────────────────────────────────────────
  Applies to SageMaker usage only
```

### Savings Plans vs Reserved Instances

```
HEAD-TO-HEAD COMPARISON
=============================================================================

                    Compute SP    EC2 Instance SP   Convertible RI   Standard RI
                    ──────────    ───────────────   ──────────────   ───────────
Max Discount        Up to 66%     Up to 72%         Up to 66%        Up to 72%

Commitment Type     $/hour        $/hour            Instance config  Instance config

Any Family          ✅            ❌ (locked)        ❌ (exchange)     ❌
Any Size            ✅            ✅                 ⚠️ Regional*     ⚠️ Regional*
Any OS/Tenancy      ✅            ✅                 ❌ (exchange)     ❌
Any Region          ✅            ❌ (locked)        ❌               ❌
Fargate/Lambda      ✅            ❌                 ❌               ❌

Capacity Reserve    ❌            ❌                 ❌**             ✅ (Zonal)
Sell on Marketplace ❌            ❌                 ❌               ✅
Auto-applied        ✅            ✅                 ✅               ✅

* Regional RIs (both Standard and Convertible) have size flexibility within the
  same instance family. Zonal RIs do NOT have size flexibility.
** Convertible RIs can be zonal, but exchanging may break capacity reservation

RECOMMENDATION FOR MOST ORGANIZATIONS:
─────────────────────────────────────────────────
  1. Start with Compute Savings Plans for maximum flexibility
  2. Layer EC2 Instance Savings Plans for stable, known workloads
  3. Use Standard RIs only if you NEED zonal capacity reservation
     or want to resell on the Marketplace
  4. Savings Plans are generally preferred over RIs due to simpler
     management and automatic application across services
```

### How Savings Plans Apply

```
SAVINGS PLAN APPLICATION ORDER
=============================================================================

You commit: $10/hour Compute Savings Plan

NOTE: The prices below are illustrative examples — actual SP rates and
discounts vary by instance type, region, OS, and commitment term. Check
the AWS Savings Plans pricing page for current rates.

Your running instances at a given hour:
┌─────────────────────┬───────────────┬─────────────────┬───────────────┐
│ Instance            │ On-Demand $/hr│ SP Rate $/hr    │ Savings       │
├─────────────────────┼───────────────┼─────────────────┼───────────────┤
│ m5.xlarge (Linux)   │ $0.192        │ $0.120          │ 37.5%         │
│ c5.2xlarge (Linux)  │ $0.340        │ $0.210          │ 38.2%         │
│ r5.large (Windows)  │ $0.226        │ $0.170          │ 24.8%         │
│ Lambda (1M invoc.)  │ $0.200        │ $0.130          │ 35.0%         │
└─────────────────────┴───────────────┴─────────────────┴───────────────┘

AWS applies your $10/hr commitment to usage automatically:
  Step 1: SP rate covers c5.2xlarge   → $0.210 of commitment used
  Step 2: SP rate covers m5.xlarge    → $0.120 of commitment used
  Step 3: SP rate covers Lambda       → $0.130 of commitment used
  Step 4: SP rate covers r5.large     → $0.170 of commitment used
  ...continues until $10/hr fully applied
  Remaining usage → billed at On-Demand rates

KEY: AWS applies to HIGHEST SAVINGS PERCENTAGE first to maximize your benefit
```

---

## Part 4: On-Demand Capacity Reservations - Reserving a Seat Without a Discount

### The Analogy

You call the airline and say, "Hold me a seat on the 8 AM flight to Chicago every day. I'll pay full fare -- I just need to guarantee the seat exists." This is separate from any discount program. You can then **combine** this seat hold with your annual pass or spending commitment to get both the guaranteed seat AND a discount. Without the pass, you still get the seat -- you just pay full price for it.

### The Technical Reality

On-Demand Capacity Reservations (ODCRs) guarantee that EC2 capacity is available for you in a specific Availability Zone. They do NOT provide a billing discount on their own. You are billed at On-Demand rates whether you run instances in the reserved capacity or not. The power comes from **combining** them with Savings Plans or Regional RIs.

```
CAPACITY RESERVATIONS — KEY FACTS
=============================================================================

  What they do:     Guarantee instance capacity in a specific AZ
  What they don't:  Provide any billing discount on their own
  Billing:          On-Demand rate, EVEN IF the capacity is UNUSED
  Term:             No long-term commitment — cancel anytime
  Scope:            Specific instance type + AZ + tenancy + platform

  CRITICAL DISTINCTION:
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │   Savings Plans / Regional RIs  =  DISCOUNT without CAPACITY  │
  │   Capacity Reservations         =  CAPACITY without DISCOUNT  │
  │   Combine them                  =  CAPACITY + DISCOUNT        │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

### Combining for Full Benefit

```
COMBINING CAPACITY RESERVATIONS WITH DISCOUNTS
=============================================================================

  SCENARIO: You need guaranteed m5.xlarge capacity in us-east-1a
            AND you want a discount

  OPTION A: Capacity Reservation + Compute Savings Plan
  ─────────────────────────────────────────────────────
  ├── ODCR guarantees the capacity in us-east-1a
  ├── Savings Plan discount auto-applies to usage
  ├── Result: Guaranteed capacity + up to 66% off
  └── Most flexible (can change instance family later)

  OPTION B: Capacity Reservation + Regional RI
  ─────────────────────────────────────────────────────
  ├── ODCR guarantees the capacity in us-east-1a
  ├── Regional RI discount auto-applies to matching usage
  ├── Result: Guaranteed capacity + up to 72% off
  └── Less flexible (locked to instance family)

  OPTION C: Zonal RI (does both)
  ─────────────────────────────────────────────────────
  ├── Single purchase provides both capacity + discount
  ├── Result: Guaranteed capacity + up to 72% off
  ├── BUT: 1-3 year term commitment required
  └── Cannot cancel the capacity reservation separately

  OPTION A is RECOMMENDED for most use cases:
  ├── Decouple capacity from discount decisions
  ├── Cancel ODCR anytime if capacity needs change
  └── Savings Plan continues to apply elsewhere
```

### Terraform: Capacity Reservation

```hcl
# On-Demand Capacity Reservation
resource "aws_ec2_capacity_reservation" "critical_app" {
  instance_type           = "m5.xlarge"
  instance_platform       = "Linux/UNIX"
  availability_zone       = "us-east-1a"
  instance_count          = 5
  instance_match_criteria = "targeted"  # Only specific instances use it

  # No end date — cancel manually when no longer needed
  end_date_type = "unlimited"

  tags = {
    Name        = "critical-app-capacity"
    Environment = "prod"
  }
}

# Instance that targets the capacity reservation
resource "aws_instance" "critical_app" {
  count         = 5
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "m5.xlarge"
  subnet_id     = var.subnet_us_east_1a_id

  capacity_reservation_specification {
    capacity_reservation_target {
      capacity_reservation_id = aws_ec2_capacity_reservation.critical_app.id
    }
  }

  tags = {
    Name = "critical-app-${count.index}"
  }
}

# Capacity Reservation with a future end date (for planned events)
resource "aws_ec2_capacity_reservation" "black_friday" {
  instance_type     = "c5.2xlarge"
  instance_platform = "Linux/UNIX"
  availability_zone = "us-east-1a"
  instance_count    = 20

  # Reserve from Nov 25 to Dec 2 for Black Friday
  end_date_type = "limited"
  end_date      = "2026-12-02T00:00:00Z"

  instance_match_criteria = "open"  # Any matching instance can use it

  tags = {
    Name  = "black-friday-capacity"
    Event = "black-friday-2026"
  }
}
```

---

## Part 5: Building a Cost Optimization Strategy

### The Travel Department Strategy

A smart travel manager does not use one booking method for everything. Instead, they layer strategies:

```
LAYERED COST OPTIMIZATION STRATEGY
=============================================================================

  ┌─────────────────────────────────────────────────────────────────────┐
  │                        YOUR EC2 WORKLOADS                           │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ LAYER 1: SPOT INSTANCES (up to 90% off)                      │  │
  │  │ Interruptible workloads: batch, CI/CD, dev/test, data proc.  │  │
  │  │ ├── Use price-capacity-optimized strategy                    │  │
  │  │ ├── Diversify across 10+ instance types                     │  │
  │  │ └── Handle interruptions via EventBridge + graceful shutdown │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ LAYER 2: SAVINGS PLANS (up to 72% off)                       │  │
  │  │ Stable baseline: production servers, always-on services      │  │
  │  │ ├── Compute SP for flexibility across families/regions       │  │
  │  │ ├── EC2 Instance SP for known, stable instance families      │  │
  │  │ └── Use AWS Cost Explorer recommendations for sizing         │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ LAYER 3: CAPACITY RESERVATIONS (no discount alone)           │  │
  │  │ Mission-critical: databases, primary app servers             │  │
  │  │ ├── Guarantee capacity in specific AZ                        │  │
  │  │ ├── Combine with Savings Plans for discount + capacity       │  │
  │  │ └── Use for planned events (Black Friday, product launches)  │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ LAYER 4: ON-DEMAND (full price)                              │  │
  │  │ Unpredictable spikes: traffic bursts, emergency scaling      │  │
  │  │ ├── Absorbs what Savings Plans don't cover                   │  │
  │  │ └── Target: minimize this layer as much as possible          │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘


SIZING YOUR COMMITMENT:
=============================================================================

  Monthly compute spend profile (y-axis = monthly spend,
  but Savings Plan commitments are hourly — $20k/mo ≈ ~$27/hr):
                                                        ← On-Demand
  $50k ┤                    ╭─╮                           (unpredictable
       │              ╭─────╯ ╰────╮                       spikes)
  $40k ┤         ╭────╯            ╰───╮
       │    ╭────╯                     ╰────╮
  $30k ┤────╯                               ╰─── ← Spot + On-Demand
       │═══════════════════════════════════════ ← Savings Plan commitment
  $20k ┤                                           ($20k/hr baseline)
       │
  $10k ┤
       │
    $0 ┼────┬────┬────┬────┬────┬────┬────┬────
       Jan  Feb  Mar  Apr  May  Jun  Jul  Aug

  RULE: Commit Savings Plans to your MINIMUM consistent usage
  ├── If your floor is $20k/hr → commit $20k/hr in Savings Plans
  ├── Variable usage above that → Spot or On-Demand
  └── Never over-commit — you pay the commitment even if you use less

  HOW TO FIND YOUR FLOOR — AWS Cost Explorer Recommendations:
  ─────────────────────────────────────────────────────────────
  1. Open AWS Cost Explorer → Left menu → "Savings Plans" → "Recommendations"
  2. Choose plan type (Compute SP or EC2 Instance SP) and term (1yr or 3yr)
  3. Cost Explorer analyzes your past 7/30/60 days of usage and recommends
     an hourly commitment level to maximize savings without over-committing
  4. Start conservatively — choose the recommendation based on your lowest
     usage period, then layer additional commitments as you gain confidence

  This turns "commit to your floor" from abstract advice into a concrete
  number you can act on.
```

### Decision Framework

```
PURCHASING OPTION DECISION TREE
=============================================================================

  Is the workload interruptible / fault-tolerant?
  │
  ├── YES → Can it handle 2-minute interruption notices?
  │         │
  │         ├── YES → SPOT INSTANCES (up to 90% off)
  │         │         Use ASG with mixed instances policy
  │         │
  │         └── NO → ON-DEMAND (with Savings Plans discount)
  │
  └── NO → Is usage predictable for 1-3 years?
            │
            ├── YES → Do you need capacity guarantee in specific AZ?
            │         │
            │         ├── YES → CAPACITY RESERVATION + SAVINGS PLAN
            │         │         (or Zonal RI if you want single purchase)
            │         │
            │         └── NO → SAVINGS PLAN
            │                  ├── Known family+region → EC2 Instance SP (72%)
            │                  └── Variable/multi-service → Compute SP (66%)
            │
            └── NO → ON-DEMAND
                      (minimize via right-sizing and auto-scaling)
```

---

## Key Takeaways

### Spot Instances
1. **Up to 90% off, but interruptible** -- AWS can reclaim with 2-minute notice; design for graceful degradation
2. **Diversify across 10+ instance types** -- More capacity pools = better availability and lower interruption rates
3. **Use price-capacity-optimized allocation** -- Best balance of low price and high availability; recommended for most Spot workloads
4. **Use EventBridge for interruption handling** -- Event-driven beats polling; set up rules for both interruption warnings and rebalance recommendations
5. **Never set a max price below On-Demand** -- Setting no max price (default) caps at On-Demand rate and maximizes your chances of keeping the instance

### Reserved Instances
6. **Standard RI = highest discount, lowest flexibility** -- Up to 72% off, but locked to family/OS/tenancy; can sell on RI Marketplace
7. **Convertible RI = moderate discount, exchangeable** -- Up to 66% off, can exchange to different family/size/OS; cannot sell
8. **Regional RI = discount across AZs with size flexibility** -- No capacity guarantee, but flexible across AZs and sizes within a family
9. **Zonal RI = discount + capacity guarantee** -- Locks you to a specific AZ and exact size, but guarantees the instance will launch

### Savings Plans
10. **Savings Plans are generally preferred over RIs** -- Simpler management, automatic application, and Compute SPs cover Fargate and Lambda too
11. **Commit to your floor, not your ceiling** -- Analyze minimum consistent usage via Cost Explorer; you pay the commitment whether you use it or not
12. **EC2 Instance SP for known workloads, Compute SP for everything else** -- 72% vs 66% discount; the flexibility tradeoff mirrors Standard vs Convertible RIs

### Capacity Reservations
13. **Capacity without discount, discount without capacity** -- ODCRs guarantee seats; Savings Plans/RIs provide discounts; combine them for both
14. **Great for planned events** -- Reserve capacity for Black Friday, product launches, or migration windows without a long-term commitment
15. **Decouple capacity from billing** -- Cancel the ODCR anytime without affecting your Savings Plan; maximum operational flexibility

---

*Written on February 8, 2026*
