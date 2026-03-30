# AWS Disaster Recovery Strategies -- The Hospital Emergency Preparedness Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock), equipped your instances with personal workbenches, shared libraries, and specialty studios (EBS, EFS, FSx), built the managed hotel chain with its revolutionary shared vault system (RDS and Aurora), organized the giant filing cabinet with drawer labels and folder tabs (DynamoDB with DAX and Global Tables), and installed the deli counter for lightning-fast repeat orders (ElastiCache with Global Datastore). All of these services are powerful individually. But here is a terrifying question: **what happens when an entire AWS Region goes down?** Not a single AZ -- an entire Region. A catastrophic failure, a natural disaster, a prolonged outage affecting every AZ in us-east-1 simultaneously. Your Multi-AZ database fails over beautifully within the Region, but if the Region itself is unreachable, Multi-AZ cannot save you. This is the domain of **Disaster Recovery (DR)** -- the discipline of planning, building, and testing the ability to recover your workloads in a completely different Region when the worst happens. AWS defines four DR strategies, arranged on a spectrum from cheap-and-slow to expensive-and-instant. Choosing the right one is not a technical decision alone -- it is a business decision that balances the cost of downtime against the cost of prevention. Think of it as hospital emergency preparedness: every hospital needs a plan for when disaster strikes, but the plan for a rural clinic looks very different from the plan for a Level 1 trauma center. The question is not "will you have a plan?" but "how fast must you recover, and how much can you afford to lose?"

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| RPO (Recovery Point Objective) | How many pages of the patient chart you can afford to lose -- "we can reconstruct up to 1 hour of notes, but not 24 hours" | The maximum acceptable amount of data loss measured in time; RPO of 1 hour means you can lose up to 1 hour of data |
| RTO (Recovery Time Objective) | How long patients can wait in the parking lot before the backup hospital is operational -- "we must be treating patients within 4 hours" | The maximum acceptable downtime; RTO of 4 hours means the system must be fully operational within 4 hours of a disaster |
| Backup & Restore | A rural clinic that keeps photocopies of patient charts in a fireproof safe at a sister clinic across town -- if the clinic burns down, they rent a new building, buy new equipment, and pull charts from the safe | Lowest cost: back up data to another Region, redeploy all infrastructure from IaC and restore data from backups during a disaster; RPO hours, RTO hours-to-days |
| Pilot Light | A hospital that keeps patient records continuously mirrored at a second campus with empty operating rooms -- the records are live, but the rooms need equipment wheeled in and staff called before surgeries can begin | Low-moderate cost: core data infrastructure (databases) actively replicating to DR Region, but compute and app servers are off; RPO minutes, RTO tens of minutes |
| Warm Standby | A hospital that maintains a small but fully operational satellite campus -- fewer beds and fewer staff than the main hospital, but it is already treating walk-in patients and just needs to call in extra staff and open more wings during a crisis | Moderate cost: a fully functional scaled-down copy of production is running and serving traffic in the DR Region; RPO seconds, RTO minutes |
| Multi-Site Active-Active | Two full-size hospitals in different cities, both treating patients every day, with patient records synchronized between them -- if one hospital closes, patients simply go to the other with no interruption in care | Highest cost: full production environments in multiple Regions all actively serving traffic simultaneously; RPO near-zero, RTO potentially zero |
| AWS Elastic Disaster Recovery (DRS) | A specialized medical transport service that continuously copies patient records to the backup campus and, when disaster strikes, rapidly sets up operating rooms and transfers staff in minutes -- getting pilot-light pricing with warm-standby speed | Managed service providing continuous data replication with automated compute recovery; achieves RPO seconds and RTO minutes at pilot-light cost |
| AWS Backup | The hospital's centralized records department that photocopies charts on a schedule, stores copies in the fireproof safe, and manages the retention policy for how long to keep each copy | Centralized backup service supporting 20+ AWS resource types with cross-region and cross-account copy rules, backup plans, and lifecycle policies |
| Failback | Reopening the original hospital after repairs, transferring patients back, and restoring it as the primary facility while the backup campus returns to standby mode | The process of returning operations to the original primary Region after a disaster is resolved, re-establishing the original DR topology |

---

## The Big Picture: DR as Hospital Emergency Preparedness

Every AWS service you have studied so far provides **high availability within a single Region** -- Multi-AZ deployments for RDS and Aurora, cross-AZ replication for ElastiCache, DynamoDB's automatic three-AZ replication, S3's 11 nines of durability across AZs. These mechanisms protect against AZ-level failures: a single data center losing power, a network partition within a Region, or hardware failures affecting one AZ.

But Regions can fail too. It is rare -- AWS Regions are designed with isolated infrastructure, independent power grids, and separate cooling systems -- but it happens. And even if the probability is low, the business impact can be catastrophic. A financial services company that cannot process transactions for 24 hours may face regulatory penalties. An e-commerce platform down during Black Friday loses millions in revenue per hour. A healthcare system offline during a crisis could endanger lives.

Disaster Recovery addresses one question: **when (not if) your primary Region becomes unavailable, how quickly can you resume operations in another Region, and how much data can you afford to lose?**

The answer to that question is expressed through two metrics that drive every DR decision.

### RPO and RTO -- The Two Numbers That Drive Everything

```
THE DR METRICS THAT DETERMINE YOUR STRATEGY
════════════════════════════════════════════════════════════════════════════

  Timeline of a disaster:

  ─── Normal operations ───┬──── Disaster ────┬──── Recovery ────┬── Back online ──▶
                           │                  │                  │
                      Last good               Disaster           System
                      backup/                 strikes            operational
                      replication                                again
                      point
                           │                  │                  │
                           ◀────── RPO ──────▶                   │
                           │  (data you lose) │                  │
                           │                  │                  │
                           │                  ◀────── RTO ──────▶
                           │                  │  (time you wait) │

  RPO = Recovery Point Objective
  ─────────────────────────────
  "How much data can we afford to lose?"
  ─ RPO of 24 hours: daily backups are sufficient
  ─ RPO of 1 hour: hourly backups or continuous replication with up to 1-hour lag
  ─ RPO of near-zero: continuous async replication with sub-second lag
    (synchronous replication exists within a Region for HA, but no
    AWS cross-Region replication is synchronous -- RPO=0 is a
    theoretical target for DR, never truly achievable)

  RTO = Recovery Time Objective
  ─────────────────────────────
  "How long can we be down?"
  ─ RTO of 24 hours: restore from backup at your own pace
  ─ RTO of 1 hour: infrastructure must be pre-provisioned or rapidly deployable
  ─ RTO of 0: active-active, traffic routes away instantly

  The business determines RPO/RTO. Engineering chooses the strategy to meet them.
  Lower RPO/RTO = higher cost. The art is finding the right balance.
```

**The critical insight**: RPO and RTO are not technical decisions. They are business decisions translated into technical requirements. The CTO does not say "use Pilot Light." The business says "we can tolerate 10 minutes of data loss and 30 minutes of downtime for our payment system." Engineering maps those constraints to the cheapest DR strategy that meets them.

---

## The Four DR Strategies -- From Rural Clinic to Trauma Center

AWS defines four DR strategies on a cost-recovery spectrum. Each builds on the previous one, adding more always-on infrastructure in the DR Region to buy faster recovery.

```
THE DR STRATEGY SPECTRUM
════════════════════════════════════════════════════════════════════════════

  COST ──────────────────────────────────────────────────────────────▶ $$$

  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
  │                │ │                │ │                │ │                │
  │   BACKUP &     │ │   PILOT        │ │   WARM         │ │  MULTI-SITE    │
  │   RESTORE      │ │   LIGHT        │ │   STANDBY      │ │  ACTIVE-ACTIVE │
  │                │ │                │ │                │ │                │
  │  RPO: hours    │ │  RPO: minutes  │ │  RPO: seconds  │ │  RPO: ~zero    │
  │  RTO: hours    │ │  RTO: 10s min  │ │  RTO: minutes  │ │  RTO: ~zero    │
  │                │ │                │ │                │ │                │
  │  DR Region:    │ │  DR Region:    │ │  DR Region:    │ │  DR Region:    │
  │  backups only  │ │  data live,    │ │  scaled-down   │ │  full prod,    │
  │  nothing runs  │ │  compute off   │ │  everything on │ │  serving live  │
  │                │ │                │ │                │ │  traffic       │
  └────────────────┘ └────────────────┘ └────────────────┘ └────────────────┘

  ◀──────────────────────────────────────────────────────────────────────▶
  RECOVERY TIME ◀── slower                                  faster ──▶
```

---

## Strategy 1: Backup & Restore -- The Rural Clinic with a Fireproof Safe

### The Analogy

Imagine a small rural clinic. Every night after closing, the office manager photocopies the day's patient charts and drives them to a fireproof safe at a sister clinic 50 miles away. If the clinic burns down tomorrow, the staff rents a new building, orders new equipment, pulls the chart copies from the safe, and reopens. It takes days. Patients treated between the last photocopying run and the fire have no records -- those are lost. But for a small clinic where a few days of downtime is survivable and the charts can mostly be reconstructed from memory, this level of preparedness is proportionate to the risk.

### The Technical Reality

In the Backup & Restore strategy, the DR Region contains **nothing running**. No databases, no servers, no load balancers. The only thing in the DR Region is backed-up data -- EBS snapshots, RDS snapshots, S3 objects replicated via CRR, DynamoDB PITR exports, and infrastructure-as-code templates stored in a repository.

When disaster strikes, the recovery team:
1. Deploys infrastructure in the DR Region using CloudFormation or Terraform
2. Restores data from backups (snapshots, S3 objects, database restores)
3. Deploys application code via the CI/CD pipeline
4. Updates DNS to route traffic to the new Region

**RPO**: Determined by backup frequency. Daily backups = RPO of up to 24 hours. Hourly snapshots = RPO of up to 1 hour. S3 CRR replicates objects within minutes (you already know this from the S3 deep dive), giving near-zero RPO for object data.

**RTO**: Hours to days. Deploying infrastructure from scratch, restoring terabytes of database snapshots, and validating application health all take time. Large RDS snapshots can take hours to restore. EBS snapshots load lazily -- you can mount them immediately, but first-access reads hit S3 and are slow until the data is fully loaded (this is where Fast Snapshot Restore (FSR) helps, as you learned in the Block Storage deep dive).

### AWS Services Involved

```
BACKUP & RESTORE -- WHAT LIVES WHERE
════════════════════════════════════════════════════════════════════════════

  ┌──── PRIMARY REGION (us-east-1) ──────────────────────────────────────┐
  │                                                                       │
  │  ┌── Active Production Stack ─────────────────────────────────────┐  │
  │  │  Route53 ──▶ ALB ──▶ ECS/EKS ──▶ Aurora + DynamoDB + S3      │  │
  │  │  (all actively serving traffic)                                │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                       │
  │  ┌── Backup Mechanisms ───────────────────────────────────────────┐  │
  │  │  AWS Backup:                                                   │  │
  │  │    ─ Aurora automated snapshots (every 5 min via PITR)         │  │
  │  │    ─ EBS snapshots via DLM (hourly/daily)                     │  │
  │  │    ─ DynamoDB PITR (continuous, 35-day window)                 │  │
  │  │    ─ EFS backups                                               │  │
  │  │  S3 CRR ──────────────────────────────────────────────────┐   │  │
  │  │  Cross-region snapshot copies ─────────────────────────┐  │   │  │
  │  └────────────────────────────────────────────────────────┼──┼───┘  │
  └───────────────────────────────────────────────────────────┼──┼──────┘
                                                              │  │
                        Snapshots copied to DR ───────────────┘  │
                        S3 objects replicated ───────────────────┘
                                                              │  │
  ┌──── DR REGION (us-west-2) ────────────────────────────────┼──┼──────┐
  │                                                           ▼  ▼      │
  │  ┌── Only Backups Exist ──────────────────────────────────────────┐ │
  │  │  S3 bucket (CRR replica)                                       │ │
  │  │  Aurora snapshot copies                                        │ │
  │  │  EBS snapshot copies                                           │ │
  │  │  DynamoDB table export (or on-demand backup copy)              │ │
  │  │  CloudFormation/Terraform templates (in S3 or Git)             │ │
  │  │                                                                │ │
  │  │  NO compute. NO databases running. NO load balancers.          │ │
  │  │  Just dormant data waiting to be restored.                     │ │
  │  └────────────────────────────────────────────────────────────────┘ │
  │                                                                     │
  │  ON DISASTER: deploy everything from scratch + restore data         │
  └─────────────────────────────────────────────────────────────────────┘
```

| Service | DR Role |
|---------|---------|
| **AWS Backup** | Centralized backup orchestration with cross-region copy rules; supports Aurora, RDS, EBS, EFS, DynamoDB, FSx, and more |
| **S3 CRR** | Continuous object replication to DR Region (already studied -- near-zero RPO for S3 data) |
| **EBS Snapshots + DLM** | Automated snapshot scheduling with cross-region copy (already studied -- incremental, FSR for fast restore) |
| **Aurora/RDS Snapshots** | Automated backups with cross-region copy; PITR provides 5-minute granularity for Aurora |
| **DynamoDB PITR** | Continuous backup with 35-day window; Export to S3 for cross-region archiving (already studied) |
| **CloudFormation StackSets** | Deploy infrastructure across Regions from a single template |
| **Route53** | Update DNS records to point to the newly deployed DR stack after recovery |

### When to Use Backup & Restore

- Non-critical workloads where hours of downtime are tolerable
- Development and staging environments
- Workloads where the cost of running any infrastructure in a second Region cannot be justified
- As the **baseline layer under every other strategy** -- even Multi-Site Active-Active needs backups to protect against data corruption that replicates across Regions

---

## Strategy 2: Pilot Light -- The Hospital with Mirrored Records and Empty Operating Rooms

### The Analogy

Now imagine a larger hospital that cannot afford to lose patient data. It builds a second campus across town with a fiber-optic link that mirrors every patient record in real time. The medical records system at the second campus is live -- every chart update at the main hospital appears at the backup campus within seconds. But the operating rooms at the second campus are dark. The surgical equipment is in storage. The on-call list exists but no surgeons are staffed there. If the main hospital goes down, the backup campus has all the patient data, but it needs to wheel equipment into operating rooms, call in staff, and boot up monitoring systems before it can treat a single patient. This takes maybe 30 minutes -- far faster than building a hospital from scratch, but not instant.

### The Technical Reality

Pilot Light keeps the **data tier** actively running and replicating in the DR Region. Databases, object stores, and critical data services are live and continuously synchronized. But the **compute and application tiers** are not running -- no EC2 instances, no ECS tasks, no Lambda functions deployed, no load balancers accepting traffic. The "pilot light" (like the small flame in a gas furnace that is always burning, ready to ignite the main burner) is the data replication.

When disaster strikes:
1. Deploy compute infrastructure (EC2 ASGs, ECS services, Lambda functions) from IaC
2. Scale out to production capacity
3. Promote read replicas to primary (for Aurora/RDS -- see critical distinction below)
4. Route traffic to DR Region via Route53 or Global Accelerator

**RPO**: Minutes. Continuous replication means data loss is limited to the replication lag (Aurora Global Database: sub-1 second; DynamoDB Global Tables: sub-second; RDS cross-region read replicas: seconds to minutes of async lag).

**RTO**: Tens of minutes. Infrastructure must be provisioned, which involves launching instances, waiting for health checks, deploying application code, and scaling to production capacity. This is significantly faster than Backup & Restore because the data is already there -- no snapshot restore time.

### AWS Services Involved

```
PILOT LIGHT -- WHAT LIVES WHERE
════════════════════════════════════════════════════════════════════════════

  ┌──── PRIMARY REGION (us-east-1) ──────────────────────────────────────┐
  │                                                                       │
  │  Route53 ──▶ ALB ──▶ ECS/EKS ──▶ Aurora Writer + DynamoDB           │
  │  (all active, serving production traffic)                             │
  │                                                                       │
  │  Aurora Global Database ──── storage-level replication ────────┐     │
  │  DynamoDB Global Tables ──── active replication ───────────────┤     │
  │  S3 CRR ──── object replication ──────────────────────────────┤     │
  │  ElastiCache Global Datastore ──── async replication ─────────┤     │
  │  EBS snapshots ──── cross-region copy (periodic) ─────────────┤     │
  └───────────────────────────────────────────────────────────────┤─────┘
                                                                  │
                        Continuous data replication ───────────────┘
                                                                  │
  ┌──── DR REGION (us-west-2) ────────────────────────────────────┼─────┐
  │                                                               ▼     │
  │  ┌── DATA TIER: LIVE AND REPLICATING ────────────────────────────┐ │
  │  │  Aurora Global DB secondary cluster (read-only, sub-1s lag)   │ │
  │  │  DynamoDB Global Table replica (read-write capable)           │ │
  │  │  S3 CRR replica bucket (objects replicated in minutes)        │ │
  │  │  ElastiCache Global Datastore secondary (read-only)           │ │
  │  └───────────────────────────────────────────────────────────────┘ │
  │                                                                     │
  │  ┌── COMPUTE TIER: OFF ──────────────────────────────────────────┐ │
  │  │  AMIs available (copied from primary)                         │ │
  │  │  ECS task definitions registered                              │ │
  │  │  Lambda functions NOT deployed or at zero concurrency         │ │
  │  │  ALB NOT created (or created but no targets)                  │ │
  │  │  ASG desired capacity = 0                                     │ │
  │  │                                                               │ │
  │  │  ON DISASTER:                                                 │ │
  │  │    1. CloudFormation/Terraform deploys compute resources      │ │
  │  │    2. ASG scales to production capacity                       │ │
  │  │    3. Aurora secondary promoted to writer (detaches from      │ │
  │  │       global cluster -- becomes independent)                  │ │
  │  │    4. Route53/Global Accelerator shifts traffic here          │ │
  │  └───────────────────────────────────────────────────────────────┘ │
  └─────────────────────────────────────────────────────────────────────┘
```

### When to Promote a Read Replica vs. Failover to a Standby

This is a critical distinction that connects directly to what you already know from the RDS/Aurora deep dive.

**Within a Region (Multi-AZ failover)**: RDS Multi-AZ uses **synchronous** replication to a standby that serves zero traffic. Failover is automatic, DNS-based, takes 60-120 seconds (RDS classic) or under 35 seconds (Multi-AZ DB Cluster). Zero data loss. Aurora uses its shared storage volume with 6 copies across 3 AZs -- failover promotes a read replica to writer, typically under 30 seconds. This is **not DR** -- this is high availability within a Region.

**Cross-Region (DR failover)**: Aurora Global Database uses **asynchronous** storage-level replication with sub-1 second lag. During DR, you have two options:

| Action | Mechanism | Data Loss | Use Case |
|--------|-----------|-----------|----------|
| **Managed planned switchover** | Graceful promotion; ensures zero data loss by completing replication before switching | Zero | Planned DR drills, Region evacuation |
| **Unplanned failover (detach + promote)** | Detach secondary from global cluster, promote to standalone writer | RPO ~1 second (replication lag) | Primary Region down unexpectedly |

For **RDS cross-region read replicas** (non-Aurora), the replica uses asynchronous engine-level replication. Promoting the replica to a standalone instance is a manual action that breaks the replication link permanently. The RPO equals the replication lag at the moment of promotion -- typically seconds to minutes, but can be longer under heavy write load.

For **DynamoDB Global Tables**, there is no promotion step. Every replica is already read-write (active-active). If a Region fails, the replicas in other Regions continue serving reads and writes immediately. This is why DynamoDB Global Tables naturally fits the Multi-Site Active-Active strategy, while Aurora Global Database fits Pilot Light and Warm Standby more naturally (we revisit this distinction in Strategy 4's write strategies section below).

---

## Strategy 3: Warm Standby -- The Satellite Campus Already Treating Patients

### The Analogy

The hospital now operates a permanent satellite campus. It is smaller -- maybe 20% of the beds, a handful of doctors, one operating room instead of ten -- but it is fully operational. Walk-in patients can be seen. Minor surgeries happen. The medical records system is fully synchronized. The pharmacy is stocked. If the main hospital goes down, the satellite campus does not need to build anything or deploy anything -- it just needs to **scale up**: call in more doctors, open the unused wings, increase the pharmacy orders. The infrastructure is already there, just running at reduced capacity. This scaling takes minutes, not the tens of minutes that Pilot Light requires for deploying new infrastructure.

### The Technical Reality

Warm Standby runs a **complete, functional copy of your production environment** in the DR Region, but at reduced scale. Everything is deployed and running -- load balancers, compute instances, databases, caches -- just with smaller instance types or fewer instances. The environment can handle traffic at reduced capacity **immediately** without any deployment step.

The key distinction from Pilot Light: **Warm Standby can process requests without additional action.** Pilot Light cannot -- it requires deploying compute resources first. This single sentence is the most commonly tested distinction on AWS exams.

When disaster strikes:
1. Scale out compute resources to production capacity (increase ASG desired count, scale ECS service, increase Lambda provisioned concurrency)
2. Scale up database resources if needed (though Aurora auto-scales readers)
3. Route production traffic to DR Region via Route53 or Global Accelerator

**RPO**: Seconds. Same continuous replication mechanisms as Pilot Light (Aurora Global Database, DynamoDB Global Tables, S3 CRR). The data-tier RPO is identical between Pilot Light and Warm Standby because both use the same replication services. The practical RPO difference comes from Warm Standby having active compute that may hold in-flight transaction state closer to the moment of failure, while Pilot Light's dormant compute has no in-flight state to preserve.

**RTO**: Minutes. No infrastructure deployment needed -- just scaling existing resources. Auto Scaling can begin adding capacity immediately. The bottleneck is how fast instances pass health checks and begin serving traffic.

### AWS Services Involved

```
WARM STANDBY -- WHAT LIVES WHERE
════════════════════════════════════════════════════════════════════════════

  ┌──── PRIMARY REGION (us-east-1) ──────────────────────────────────────┐
  │                                                                       │
  │  Route53 ──▶ ALB ──▶ ECS (10 tasks) ──▶ Aurora (r6g.2xl writer)    │
  │  (100% of production traffic)            DynamoDB (on-demand)        │
  │                                          ElastiCache (3 shards)      │
  │                                                                       │
  └──────────────────────────────────┬───────────────────────────────────┘
                                     │  Continuous replication
                                     │  (all data services)
  ┌──── DR REGION (us-west-2) ──────┼───────────────────────────────────┐
  │                                  ▼                                   │
  │  ┌── SCALED-DOWN BUT FULLY OPERATIONAL ──────────────────────────┐  │
  │  │                                                                │  │
  │  │  ALB ──▶ ECS (2 tasks)  ──▶ Aurora Global DB secondary       │  │
  │  │  (active, health-checked)    (read-only, can be promoted)     │  │
  │  │                              DynamoDB Global Table replica     │  │
  │  │                              ElastiCache Global Datastore      │  │
  │  │                              (read-only secondary)             │  │
  │  │                                                                │  │
  │  │  ASG: min=2, desired=2, max=20  (production is desired=10)    │  │
  │  │  Lambda: provisioned concurrency = 10 (production = 100)      │  │
  │  │                                                                │  │
  │  │  CAN SERVE TRAFFIC NOW at reduced capacity.                   │  │
  │  │  Route53 health checks are passing.                           │  │
  │  │                                                                │  │
  │  │  ON DISASTER:                                                 │  │
  │  │    1. Scale out: ASG desired = 10, Lambda concurrency = 100  │  │
  │  │    2. Promote Aurora secondary to writer                      │  │
  │  │    3. Route53 failover routes 100% traffic here               │  │
  │  │    (Steps 1-3 happen in minutes, not tens of minutes)         │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                       │
  │  Key advantage over Pilot Light:                                      │
  │  ─ ALB is already running and health-checked                          │
  │  ─ ECS tasks are already running and passing health checks            │
  │  ─ No "cold start" delay for compute provisioning                     │
  │  ─ Continuous testing is easier (traffic can be routed here anytime)   │
  └───────────────────────────────────────────────────────────────────────┘
```

### Warm Standby vs. Hot Standby

When you scale the Warm Standby environment to **full production capacity** (not just a fraction), it becomes a **Hot Standby**. The entire production stack is running at full scale in both Regions, but only one Region serves traffic. This eliminates even the scaling step -- failover is just a DNS change. The trade-off is cost: you are paying for two full production environments while only one serves traffic. Hot Standby is the bridge between Warm Standby and Multi-Site Active-Active.

**Important**: Hot Standby is not a separate AWS-defined DR strategy -- it is an industry term for a Warm Standby scaled to 100% of production capacity. AWS documentation only defines four strategies. Some third-party materials treat Hot Standby as a fifth tier, but on AWS exams, it falls under Warm Standby.

---

## Strategy 4: Multi-Site Active-Active -- Two Full Hospitals Treating Patients Every Day

### The Analogy

The hospital system now operates two full-size hospitals in different cities. Both hospitals treat patients every day. Both have full emergency departments, full surgical suites, full ICUs. Patient records are synchronized between them in real time. If one hospital closes for any reason, patients simply go to the other hospital. There is no "failover" in the traditional sense -- both were already operational. The challenge is not recovery; it is **data consistency**. If a patient is admitted to Hospital A and simultaneously updates their insurance at Hospital B, which record wins? This is the same data consistency challenge you studied with DynamoDB Global Tables (last-writer-wins with MREC) and Aurora Global Database (single-writer with read-only secondaries).

### The Technical Reality

Multi-Site Active-Active runs **full production environments** in two or more Regions, with all Regions actively serving live traffic simultaneously. There is no primary and secondary -- every Region is a peer. Traffic is distributed using Route53 latency-based or geolocation routing (already studied), or Global Accelerator endpoint groups.

This is the only strategy where there is no traditional failover step. If a Region fails, Route53 health checks detect the failure and stop routing traffic there. The remaining Regions absorb the full load. RTO is effectively zero for the routing layer, though the surviving Regions may need to scale up to handle the additional traffic.

**RPO**: Near-zero for infrastructure failures. The critical caveat: **data corruption is the exception.** If a bug corrupts data in one Region, that corruption replicates to all Regions instantly via Global Tables or S3 CRR. This is why even Active-Active architectures need backups -- you cannot replicate your way out of corrupted data. Point-in-time recovery (Aurora PITR, DynamoDB PITR, S3 versioning) is the safety net.

**RTO**: Potentially zero. Traffic routes away from the failed Region automatically.

### The Data Consistency Challenge -- Three Write Strategies

The hardest problem in Active-Active is not failover -- it is deciding **where writes happen**. AWS documents three patterns:

```
ACTIVE-ACTIVE WRITE STRATEGIES
════════════════════════════════════════════════════════════════════════════

  Strategy 1: WRITE GLOBAL (single writer)
  ─────────────────────────────────────────
  One Region accepts all writes. Other Regions read locally.
  Writes from non-primary Regions are forwarded to the primary.

  ┌────── us-east-1 ──────┐     ┌────── eu-west-1 ──────┐
  │  Aurora Global DB      │     │  Aurora Global DB      │
  │  PRIMARY (read+write)  │◀───▶│  SECONDARY (read-only) │
  │                        │     │  Write forwarding ──▶  │
  └────────────────────────┘     └────────────────────────┘

  Service: Aurora Global Database (you studied the write-forwarding
  mechanism -- writes from secondary are forwarded at the SQL layer
  to the primary, not the storage layer)
  Trade-off: No write conflicts. But write latency from remote Regions
  includes cross-Region round-trip. Managed planned switchover completes
  in under 30 seconds (Aurora MySQL 3.09+, PostgreSQL 16.8+/15.12+).
  Unplanned failover (detach + promote) typically takes 1-2 minutes.


  Strategy 2: WRITE LOCAL (writes to nearest Region)
  ───────────────────────────────────────────────────
  Every Region accepts writes. Conflicts resolved automatically.

  ┌────── us-east-1 ──────┐     ┌────── eu-west-1 ──────┐
  │  DynamoDB Global Table │     │  DynamoDB Global Table │
  │  REPLICA (read+write)  │◀───▶│  REPLICA (read+write)  │
  │  Last-writer-wins      │     │  Last-writer-wins      │
  └────────────────────────┘     └────────────────────────┘

  Service: DynamoDB Global Tables (MREC -- last-writer-wins based on
  timestamp, as you studied; or MRSC for strong consistency across
  3 Regions but with quorum latency)
  Trade-off: Lowest write latency (local writes). But last-writer-wins
  can silently overwrite concurrent updates. Application must be
  designed to tolerate this, or use conditional writes.


  Strategy 3: WRITE PARTITIONED (each Region owns a subset of data)
  ────────────────────────────────────────────────────────────────
  Writes are partitioned by key so each Region "owns" certain data.

  ┌────── us-east-1 ──────┐     ┌────── eu-west-1 ──────┐
  │  S3 Bucket             │     │  S3 Bucket             │
  │  Owns: US customer     │◀───▶│  Owns: EU customer     │
  │  uploads               │     │  uploads               │
  │  Bidirectional CRR     │     │  Bidirectional CRR     │
  └────────────────────────┘     └────────────────────────┘

  Service: S3 with bidirectional CRR (enable replica modification
  sync + delete marker replication for true bidirectional)
  Trade-off: No conflicts (each Region writes different keys).
  But requires application-level routing logic to direct writes
  to the correct Region.
```

### AWS Services Involved in Active-Active

| Service Layer | AWS Service | Write Strategy |
|---------------|-------------|----------------|
| **Traffic routing** | Route53 (latency/geolocation), Global Accelerator (endpoint groups with traffic dials) | N/A |
| **Relational data** | Aurora Global Database with write forwarding | Write Global (single writer) |
| **NoSQL data** | DynamoDB Global Tables | Write Local (all replicas read-write) |
| **Object storage** | S3 bidirectional CRR | Write Partitioned or Write Local |
| **Caching** | ElastiCache Global Datastore | Write Global (single primary Region for writes) |
| **Infrastructure** | CloudFormation StackSets, Terraform workspaces | Deploy identical stacks to all Regions |
| **DNS failover** | Route53 health checks + failover routing | Automatic traffic rerouting |

---

## AWS Elastic Disaster Recovery (DRS) -- The Modern Alternative

AWS Elastic Disaster Recovery is a managed service that blurs the line between Pilot Light and Warm Standby. It provides **continuous data replication** (like Pilot Light) with **automated compute recovery** (reducing the manual work of Pilot Light) at a cost closer to Pilot Light than Warm Standby.

**How it works:**
1. Install the DRS replication agent on source servers (EC2, on-premises, or other clouds)
2. DRS continuously replicates block-level data to the DR Region (RPO: seconds)
3. In the DR Region, only lightweight staging resources exist (low cost)
4. On failover, DRS automatically provisions right-sized EC2 instances and attaches the replicated data
5. RTO: minutes (automated orchestration vs. manual Pilot Light deployment)

**Key advantage**: DRS achieves Warm Standby RPO/RTO targets at Pilot Light cost because it does not keep compute running in the DR Region -- it only keeps data replicated and orchestrates compute provisioning automatically during failover.

**When to use DRS vs. the traditional strategies:**
- DRS is ideal for **lift-and-shift workloads** (EC2-based applications) where you want automated failover without running duplicate compute
- DRS does **not** replace Aurora Global Database, DynamoDB Global Tables, or S3 CRR for data-tier replication -- those services have their own replication mechanisms
- DRS is best thought of as the **compute-tier** DR solution that complements database-tier and storage-tier replication services you already know

---

## Strategy Comparison -- The Complete Picture

| Dimension | Backup & Restore | Pilot Light | Warm Standby | Multi-Site Active-Active |
|-----------|-----------------|-------------|--------------|--------------------------|
| **RPO** | Hours (backup frequency) | Minutes (replication lag) | Seconds (continuous replication) | Near-zero |
| **RTO** | Hours to days | Tens of minutes | Minutes | Potentially zero |
| **Cost** | $ (lowest) | $$ (data tier running) | $$$ (full stack, scaled down) | $$$$ (full prod in all Regions) |
| **DR Region: Data tier** | Snapshots/backups only | Live, replicating | Live, replicating | Live, serving traffic |
| **DR Region: Compute tier** | Nothing | Off (AMIs ready, ASG at 0) | Running at reduced scale | Full production scale |
| **DR Region: Network tier** | Nothing | Nothing or minimal | ALB/NLB active, health-checked | ALB/NLB active, serving traffic |
| **Failover action** | Deploy all + restore data | Deploy compute + scale + promote DBs | Scale up + promote DBs + route traffic | Route traffic away from failed Region |
| **Testing ease** | Hard (must deploy everything) | Moderate (deploy compute, test, tear down) | Easy (already running, route test traffic) | Easiest (already serving traffic) |
| **Failback complexity** | Moderate | Moderate-High (re-establish replication) | Moderate-High | Lowest (if both Regions survive and no data corruption -- corrupted data replicates everywhere, making failback extremely complex) |
| **Best for** | Non-critical workloads, dev/staging | Core business apps, moderate RTO tolerance | Revenue-critical apps, low RTO needed | Mission-critical, zero-downtime required |

### Cost Estimation Framework

The cost of each strategy is driven by what runs in the DR Region 24/7:

```
WHAT YOU PAY FOR IN EACH DR STRATEGY (DR REGION COSTS)
════════════════════════════════════════════════════════════════════════════

  Backup & Restore:
    ─ S3 storage for snapshots and replicated objects
    ─ Cross-region data transfer for backups
    ─ ~5-10% of primary Region cost

  Pilot Light:
    ─ Aurora Global DB secondary cluster (reader instances)
    ─ DynamoDB Global Table replica (on-demand: pay per replicated write)
    ─ ElastiCache Global Datastore secondary nodes
    ─ S3 CRR storage + transfer
    ─ ~15-25% of primary Region cost

  Warm Standby:
    ─ Everything in Pilot Light, PLUS:
    ─ ALB/NLB (hourly + LCU charges even with minimal traffic)
    ─ ECS/EKS tasks or EC2 instances (scaled down)
    ─ Lambda provisioned concurrency
    ─ NAT Gateways, VPC endpoints, etc.
    ─ ~25-50% of primary Region cost

  Multi-Site Active-Active:
    ─ Full production infrastructure in every Region
    ─ ~100% of primary Region cost PER ADDITIONAL REGION
    ─ But: serving traffic from multiple Regions improves global latency,
      so the cost is not purely DR overhead -- it is also a UX investment
```

---

## Mapping AWS Services to DR Roles

You have already studied every service in this table. This maps them to their specific DR function:

| DR Function | AWS Service | Strategy Applicability | What You Already Know |
|-------------|-------------|------------------------|-----------------------|
| **Centralized backup** | AWS Backup | All (baseline for every strategy) | Cross-region copy rules, 15+ resource types |
| **Object replication** | S3 CRR | All | Near-zero RPO, bidirectional for Active-Active |
| **Block storage backup** | EBS Snapshots + DLM | Backup & Restore, Pilot Light | Incremental snapshots, FSR for fast restore, Archive tier |
| **Relational DB replication** | Aurora Global Database | Pilot Light, Warm, Active-Active | Sub-1s storage-level replication, managed switchover vs unplanned failover |
| **Relational DB replication** | RDS Cross-Region Read Replica | Pilot Light, Warm | Async engine-level replication, manual promote to standalone |
| **NoSQL replication** | DynamoDB Global Tables | Pilot Light, Warm, Active-Active | Active-active, MREC last-writer-wins, MRSC for strong consistency |
| **Cache replication** | ElastiCache Global Datastore | Pilot Light, Warm, Active-Active | Single-writer primary + 2 read-only secondaries, sub-1s lag |
| **File storage replication** | EFS Cross-Region Replication | Pilot Light, Warm, Active-Active | Automatic async replication to another Region |
| **Traffic routing** | Route53 (failover routing) | All (during recovery) | Health checks, failover routing policy, 60-second TTL |
| **Traffic routing** | Global Accelerator | Warm, Active-Active | Anycast IPs, endpoint groups, traffic dials, instant failover |
| **Compute recovery** | AWS Elastic Disaster Recovery | Pilot Light alternative | Continuous block replication, automated compute provisioning |
| **Infrastructure deploy** | CloudFormation StackSets | All | Multi-Region, multi-account deployments from single template |
| **Readiness checking** | AWS Application Recovery Controller (ARC) | Warm, Active-Active | Readiness checks, routing controls for manual failover via data-plane API; Region Switch for orchestrated multi-service recovery |
| **Resilience validation** | AWS Resilience Hub | All | Continuously assesses whether your architecture meets stated RPO/RTO targets; generates alarms when compliance drifts; produces resiliency recommendations per Well-Architected best practices |
| **Chaos engineering** | AWS Fault Injection Service (FIS) | All | Simulates AZ failures, Region connectivity disruption, and service impairments in a controlled way; integrates with ARC Region Switch for end-to-end DR validation |

---

## The Failover Decision Tree

```
DISASTER DETECTED -- WHAT HAPPENS NEXT?
════════════════════════════════════════════════════════════════════════════

  Is the entire primary Region unavailable?
  │
  ├── NO: Single AZ failure
  │   └── Multi-AZ handles this automatically (within-Region HA)
  │       ─ RDS: automatic DNS failover to standby (60-120s)
  │       ─ Aurora: promote a reader to writer (~30s)
  │       ─ DynamoDB: transparent (3-AZ by default)
  │       ─ ElastiCache Multi-AZ: automatic replica promotion
  │       ─ ALB: routes to healthy AZs automatically
  │       *** This is NOT DR. This is HA. No cross-Region action needed. ***
  │
  └── YES: Region-level failure → DR strategy activates
      │
      ├── BACKUP & RESTORE
      │   1. Team assembles (human decision to invoke DR)
      │   2. Deploy infrastructure via IaC in DR Region
      │   3. Restore data from most recent backups/snapshots
      │   4. Deploy application code via CI/CD pipeline
      │   5. Validate: smoke tests, data integrity checks
      │   6. Update Route53 to point to DR Region
      │   Timeline: hours to a full day
      │
      ├── PILOT LIGHT
      │   1. Data is already live in DR Region (no restore needed)
      │   2. Deploy compute: launch EC2/ECS, deploy Lambda
      │   3. Scale to production capacity
      │   4. Promote Aurora secondary to writer (detach from global cluster)
      │      ─ DynamoDB Global Tables: no promotion needed (already read-write)
      │   5. Route53 health check fails → automatic failover (if configured)
      │      ─ Or: ARC routing control for manual failover via data-plane API
      │   Timeline: 10-30 minutes
      │
      ├── WARM STANDBY
      │   1. Data is live, compute is running at reduced scale
      │   2. Scale out: increase ASG desired, ECS service count, Lambda concurrency
      │   3. Promote Aurora secondary to writer
      │   4. Route53 health check fails → automatic failover
      │   Timeline: 1-10 minutes
      │
      └── MULTI-SITE ACTIVE-ACTIVE
          1. Route53 health check detects Region failure
          2. Traffic automatically stops routing to failed Region
             ─ CAVEAT: DNS caching in resolvers, ISPs, and browsers means
               some users continue hitting the dead Region until cached
               entries expire (TTL-dependent, seconds to minutes)
             ─ MITIGATION: Global Accelerator avoids this entirely --
               Anycast IPs route at the network layer, not DNS layer
          3. Surviving Regions may need to scale up to absorb additional load
          4. DynamoDB Global Tables: replicas continue serving (no action)
             ─ In-flight writes acknowledged by failed Region: not lost;
               DynamoDB resumes replication when Region recovers
             ─ In-flight writes never acknowledged: lost; client must retry
          5. Aurora Global DB: promote secondary to writer (if using write-global)
          Timeline: seconds (routing) + minutes (scaling)
```

---

## Production Architecture: Multi-Region E-Commerce Platform

```
PRODUCTION DR ARCHITECTURE -- E-COMMERCE (WARM STANDBY)
════════════════════════════════════════════════════════════════════════════

  ┌──── us-east-1 (PRIMARY) ─────────────────────────────────────────────┐
  │                                                                       │
  │  Route53 (failover routing, primary)                                  │
  │      │                                                                │
  │      ▼                                                                │
  │  ALB ──▶ ECS Fargate (10 tasks)                                      │
  │      │         │                                                      │
  │      │    ┌────┴────────────────────────────────────────────┐         │
  │      │    │                                                  │         │
  │      │    ▼                                                  ▼         │
  │      │  Aurora PostgreSQL (Global DB)    DynamoDB Global     │         │
  │      │  Writer: db.r6g.2xlarge          Tables (orders,     │         │
  │      │  + 2 readers (same Region)        sessions)           │         │
  │      │    │                                │                 │         │
  │      │    │  ElastiCache Valkey            │                 │         │
  │      │    │  (3 shards, CME)               │                 │         │
  │      │    │  Global Datastore primary      │                 │         │
  │      │    │                                │                 │         │
  │  AWS Backup: cross-region copy rules       │                 │         │
  │    ─ Aurora snapshots → us-west-2          │                 │         │
  │    ─ EBS snapshots → us-west-2             │                 │         │
  │    ─ EFS backups → us-west-2               │                 │         │
  │                                                                       │
  └──────────┬──────────────────────────────────────┬────────────────────┘
             │                                      │
             │  Aurora Global DB: storage-level      │  DynamoDB Global
             │  replication (<1s lag)                 │  Tables: active
             │                                      │  replication
             │  ElastiCache Global Datastore:        │
             │  async replication (<1s lag)           │
             │                                      │
  ┌──────────▼──────────────────────────────────────▼────────────────────┐
  │                                                                       │
  │  ┌──── us-west-2 (DR -- WARM STANDBY) ──────────────────────────────┐│
  │  │                                                                   ││
  │  │  Route53 (failover routing, secondary)                            ││
  │  │      │                                                            ││
  │  │      ▼                                                            ││
  │  │  ALB ──▶ ECS Fargate (2 tasks)    ◀── health checks passing     ││
  │  │      │         │                                                  ││
  │  │      │    ┌────┴────────────────────────────────────────┐         ││
  │  │      │    │                                              │         ││
  │  │      │    ▼                                              ▼         ││
  │  │      │  Aurora Global DB secondary    DynamoDB Global   │         ││
  │  │      │  Reader: db.r6g.large          Table replica     │         ││
  │  │      │  (read-only, promotable)       (read-write)      │         ││
  │  │      │    │                                              │         ││
  │  │      │    │  ElastiCache Valkey (1 shard, read-only)    │         ││
  │  │      │    │  Global Datastore secondary                  │         ││
  │  │      │                                                            ││
  │  │  ON FAILOVER:                                                     ││
  │  │    1. Route53 detects primary ALB health check failure             ││
  │  │    2. DNS failover routes traffic to us-west-2 ALB               ││
  │  │    3. ECS ASG scales from 2 → 10 tasks (auto-scaling or manual)  ││
  │  │    4. Aurora secondary promoted to independent writer             ││
  │  │    5. ElastiCache Global Datastore: promote secondary to primary ││
  │  │    6. Application fully operational in minutes                    ││
  │  │                                                                   ││
  │  │  BACKUPS IN DR REGION (protects against data corruption):         ││
  │  │    ─ Aurora snapshot copies (from AWS Backup cross-region rules)  ││
  │  │    ─ DynamoDB PITR enabled on replica table                       ││
  │  │    ─ S3 versioning on CRR replica bucket                          ││
  │  └───────────────────────────────────────────────────────────────────┘│
  └───────────────────────────────────────────────────────────────────────┘

  RPO: < 1 second (Aurora Global DB) / near-zero (DynamoDB Global Tables)
  RTO: ~5-10 minutes (Route53 failover TTL + ECS scaling + Aurora promotion)
  Cost: ~35% of primary Region (smaller instances, fewer tasks, no write traffic)
```

### Terraform Example: Warm Standby DR with Aurora Global Database

```hcl
# ═══════════════════════════════════════════════════════════════════════
# PRIMARY REGION (us-east-1)
# ═══════════════════════════════════════════════════════════════════════

provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

# Aurora Global Cluster (the global umbrella)
resource "aws_rds_global_cluster" "ecommerce" {
  global_cluster_identifier = "ecommerce-global"
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"  # 16.8+ supports managed switchover in under 30 seconds
  database_name             = "ecommerce"
  storage_encrypted         = true
}

# Primary Region: Aurora cluster (writer)
resource "aws_rds_cluster" "primary" {
  provider = aws.primary

  cluster_identifier        = "ecommerce-primary"
  global_cluster_identifier = aws_rds_global_cluster.ecommerce.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"  # 16.8+ supports managed switchover in under 30 seconds
  master_username           = "admin"
  master_password           = var.db_password
  db_subnet_group_name      = aws_db_subnet_group.primary.name
  vpc_security_group_ids    = [aws_security_group.aurora_primary.id]

  # Backups (critical even with Global DB -- protects against data corruption)
  backup_retention_period   = 35  # Maximum retention for PITR
  preferred_backup_window   = "03:00-04:00"
  copy_tags_to_snapshot     = true

  # Enable deletion protection in production
  deletion_protection = true
}

resource "aws_rds_cluster_instance" "primary_writer" {
  provider = aws.primary

  identifier         = "ecommerce-primary-writer"
  cluster_identifier = aws_rds_cluster.primary.id
  instance_class     = "db.r6g.2xlarge"  # Full production capacity
  engine             = "aurora-postgresql"
}

resource "aws_rds_cluster_instance" "primary_reader" {
  provider = aws.primary
  count    = 2  # Two readers for within-Region read scaling

  identifier         = "ecommerce-primary-reader-${count.index}"
  cluster_identifier = aws_rds_cluster.primary.id
  instance_class     = "db.r6g.xlarge"
  engine             = "aurora-postgresql"
}

# ═══════════════════════════════════════════════════════════════════════
# DR REGION (us-west-2) -- WARM STANDBY
# ═══════════════════════════════════════════════════════════════════════

# DR Region: Aurora secondary cluster (read-only, promotable)
resource "aws_rds_cluster" "dr" {
  provider = aws.dr

  cluster_identifier        = "ecommerce-dr"
  global_cluster_identifier = aws_rds_global_cluster.ecommerce.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.8"  # 16.8+ supports managed switchover in under 30 seconds
  db_subnet_group_name      = aws_db_subnet_group.dr.name
  vpc_security_group_ids    = [aws_security_group.aurora_dr.id]

  # DR cluster gets its own backup retention (independent PITR in DR Region)
  backup_retention_period = 35

  # No master_username/password needed -- inherited from global cluster
  # No database_name needed -- inherited from global cluster

  depends_on = [aws_rds_cluster.primary]
}

resource "aws_rds_cluster_instance" "dr_reader" {
  provider = aws.dr

  identifier         = "ecommerce-dr-reader"
  cluster_identifier = aws_rds_cluster.dr.id
  instance_class     = "db.r6g.large"  # Smaller than primary (warm standby)
  engine             = "aurora-postgresql"

  # Promotion tier 0 = first to be promoted to writer on failover
  promotion_tier = 0
}

# ═══════════════════════════════════════════════════════════════════════
# ROUTE53 FAILOVER ROUTING
# ═══════════════════════════════════════════════════════════════════════

resource "aws_route53_health_check" "primary_alb" {
  fqdn              = aws_lb.primary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10  # Check every 10 seconds

  tags = {
    Name = "primary-region-health-check"
  }
}

resource "aws_route53_record" "app_primary" {
  zone_id = var.hosted_zone_id
  name    = "app.example.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary_alb.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_dr" {
  zone_id = var.hosted_zone_id
  name    = "app.example.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "dr"

  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}

# ═══════════════════════════════════════════════════════════════════════
# AWS BACKUP -- Cross-Region Copy (baseline for all strategies)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_backup_plan" "cross_region" {
  name = "ecommerce-cross-region-backup"

  rule {
    rule_name         = "daily-cross-region"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 3 * * ? *)"  # Daily at 3 AM UTC

    lifecycle {
      delete_after = 35  # 35-day retention
    }

    # Copy to DR Region (critical for data corruption protection)
    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn

      lifecycle {
        delete_after = 35
      }
    }
  }
}

# Backup vault in primary Region
resource "aws_backup_vault" "primary" {
  provider = aws.primary
  name     = "ecommerce-backup-vault"
}

# Backup vault in DR Region
resource "aws_backup_vault" "dr" {
  provider = aws.dr
  name     = "ecommerce-backup-vault-dr"
}
```

---

## Testing and Automating DR -- The Part Everyone Skips

### AWS Fault Injection Service (FIS)

A DR plan that has never been tested is a hope, not a plan. AWS Fault Injection Service (FIS) is the managed chaos engineering tool for validating DR readiness. FIS can:

- **Simulate AZ failures**: Terminate instances in a specific AZ, degrade EBS I/O, or deny AZ network traffic
- **Simulate Region connectivity disruption**: Block cross-Region replication to test how your application behaves when replication lag spikes
- **Inject service impairments**: Throttle API calls, add latency to network paths, stress-test CPU/memory
- **Integrate with ARC Region Switch**: Run an end-to-end DR drill that fails over traffic, validates application health, and fails back -- all in a controlled experiment with automatic rollback if safety conditions are breached

In an interview, saying "We use FIS to simulate Region-level faults and validate our failover runbooks quarterly" signals mature operational practice.

### Automating Failover with Runbooks

The failover decision tree above implies a human-driven process, but production DR should be automated as much as possible. AWS provides two key mechanisms:

- **AWS Systems Manager Automation documents (runbooks)**: Pre-built and custom runbooks that execute failover steps in sequence -- promote Aurora secondary, scale ASGs, update Route53 records. AWS provides reference runbooks for common DR actions. These are the "playbook" that oncall engineers execute (or that triggers automatically from CloudWatch alarms).
- **Step Functions workflows**: For complex multi-step failover orchestration with error handling, parallel execution, and human approval gates. Example: Step Function detects Region failure → promotes Aurora → scales ECS → runs smoke tests → if tests pass, updates Route53 → if tests fail, alerts oncall and halts.
- **ARC routing controls**: Data-plane-only operations (not dependent on control-plane APIs that may be degraded during a Region failure) that shift traffic between Regions. These are the most resilient failover mechanism because they operate on the data plane.

The automation maturity ladder: manual runbook (documented steps) → SSM Automation (scripted steps) → Step Functions (orchestrated with gates) → ARC + FIS (tested and validated regularly).

---

## Critical Gotchas and Interview Traps

**1. "Replication is not backup."**
This is the most important DR principle. Replication (Aurora Global Database, DynamoDB Global Tables, S3 CRR) protects against infrastructure failures -- a Region going down. But if a developer's buggy deployment corrupts data in the primary, that corruption replicates to all secondaries within seconds. Replication faithfully copies your mistakes. Only point-in-time backups (PITR) let you roll back to before the corruption. Every DR strategy needs backups as the foundational layer.

**2. "Pilot Light cannot serve traffic; Warm Standby can."**
This is the single most tested distinction. If the DR environment requires deploying compute resources before it can handle requests, it is Pilot Light. If it can handle requests (even at reduced capacity) immediately, it is Warm Standby.

**3. "Multi-AZ is not DR."**
Multi-AZ (RDS failover, Aurora replica promotion, cross-AZ ALB routing) is high availability within a single Region. It protects against AZ failures. DR protects against Region failures. They are complementary -- you need both, and they operate at different levels.

**4. "Active-Active does not eliminate the need for failover logic."**
Even in Active-Active with DynamoDB Global Tables, you still need Route53 health checks to stop routing traffic to a failed Region. Aurora Global Database still requires promoting a secondary to writer. ElastiCache Global Datastore still requires promoting a secondary to primary. The "zero-downtime" promise applies to the traffic routing layer, not necessarily to every component.

**5. "Auto Scaling during DR is a control-plane operation."**
Scaling ASGs, ECS services, or Lambda concurrency during failover depends on the AWS control plane. During a major Region failure, control-plane API calls may be degraded. The mitigation is to provision full capacity in the DR Region (making it a Hot Standby) or use data-plane-only operations like AWS Application Recovery Controller (ARC) routing controls, which operate on the more resilient data plane.

**6. "Service quotas in the DR Region must match production."**
If your primary Region has a service quota of 500 EC2 instances and your DR Region has the default quota of 20, your Auto Scaling will fail during failover. Request quota increases in the DR Region proactively. This is a commonly overlooked preparation step.

**7. "Multi-Region KMS keys are essential for encrypted DR."**
This connects directly to your KMS deep dive (Mar 9). Encrypted EBS snapshots, Aurora clusters, and S3 objects encrypted with a single-Region KMS key **cannot be decrypted in another Region** -- the key simply does not exist there. You have two options: (a) use **multi-Region KMS keys** (a primary key in us-east-1 with a replica key in us-west-2 that share the same key material) so encrypted resources are automatically decryptable in the DR Region, or (b) re-encrypt snapshots with a DR-Region key during the cross-region copy. Option (a) is simpler and is the recommended pattern for DR. Without this, your first failover drill will fail at the "restore encrypted snapshot" step.

**8. "A DR plan that has never been tested is a hope, not a plan."**
Use AWS Fault Injection Service (FIS) to run controlled DR drills. Common testing pitfalls: (a) testing failover but never testing failback, (b) testing with synthetic traffic but never with production-like load, (c) testing individual components but never the full end-to-end flow, (d) not validating that service quotas, IAM roles, security groups, and KMS keys are all correctly configured in the DR Region. Schedule quarterly DR drills at minimum.

---

## Key Takeaways

- **RPO and RTO are business decisions, not technical ones.** The business determines how much data loss and downtime are acceptable. Engineering selects the cheapest DR strategy that meets those targets. A strategy that exceeds requirements wastes money; one that falls short creates unacceptable risk.

- **The four strategies form an additive spectrum.** Backup & Restore is the baseline that every other strategy includes. Pilot Light adds live data replication. Warm Standby adds running compute. Active-Active adds traffic serving. Each layer increases cost and reduces RTO/RPO.

- **Pilot Light vs. Warm Standby: the key is compute state.** Pilot Light has live data but no running compute -- it must deploy and scale before serving traffic (RTO: tens of minutes). Warm Standby has live data AND running compute at reduced scale -- it only needs to scale up (RTO: minutes). This distinction is the most commonly tested concept.

- **Even Active-Active needs backups.** Replication protects against infrastructure failure. Backups protect against data corruption, accidental deletion, and malicious changes. Data corruption replicates instantly across Regions -- you cannot replicate your way out of bad data. Enable PITR, versioning, and cross-region backup copies in every strategy.

- **DynamoDB Global Tables is naturally Active-Active; Aurora Global Database is naturally Active-Passive.** Global Tables replicas are all read-write (write-local with last-writer-wins). Aurora Global Database has one writer and read-only secondaries (write-global with optional write forwarding). Choose your database's replication model based on your application's write pattern.

- **AWS Elastic Disaster Recovery (DRS) achieves Warm Standby RTO/RPO at Pilot Light cost** by continuously replicating block data without running compute in the DR Region, then automatically orchestrating compute provisioning during failover. Best for EC2-based workloads.

- **Multi-AZ is high availability, not disaster recovery.** Multi-AZ protects against AZ failures within a Region. DR protects against Region failures. You need both. They are different layers of resilience.

- **Test your DR plan regularly with AWS Fault Injection Service (FIS).** A DR plan that has never been tested is a hope, not a plan. FIS can simulate AZ failures, Region connectivity disruption, and service impairments in controlled experiments with automatic rollback. Warm Standby and Active-Active are easier to test because infrastructure is already running. Pilot Light requires deploying compute for testing (and tearing it down afterward). Backup & Restore requires a full end-to-end rehearsal. Automate failover steps with SSM Automation runbooks or Step Functions workflows.

- **Account for service quotas, IAM roles, and encryption keys in the DR Region.** These are commonly overlooked. Service quotas must be pre-increased. IAM roles must exist. KMS keys must be available -- use **multi-Region KMS keys** so encrypted snapshots, Aurora clusters, and S3 objects are decryptable in the DR Region without re-encryption (as studied in the KMS deep dive, single-Region keys cannot decrypt data in another Region). Security groups and VPC configurations must mirror production.

- **Failback is often harder than failover.** During failover, the DR Region was pre-prepared. During failback, the original primary just recovered with stale data and broken replication links. **Aurora Global Database failback**: (1) delete the old stale cluster in the original Region (it cannot be reconnected after unplanned failover broke the topology), (2) add the original Region as a new secondary to the current primary's Global Database, (3) wait for replication lag to reach near-zero, (4) use managed planned switchover to swap roles cleanly (preserves topology, unlike unplanned failover). **DynamoDB Global Tables failback**: effortless -- Global Tables is multi-master, so there was no promotion to break. When the original Region recovers, DynamoDB automatically resumes replication. No manual steps needed.

---

## Further Reading

- [AWS Reliability Pillar -- DR Strategies (REL13-BP02)](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html) -- The authoritative source for all four strategies with architecture diagrams
- [AWS Whitepaper -- Disaster Recovery Options in the Cloud](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html) -- Deep service mappings for each strategy
- [DR Architecture Blog Part I -- Strategies Overview](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-i-strategies-for-recovery-in-the-cloud/) -- High-level comparison framework
- [DR Architecture Blog Part III -- Pilot Light and Warm Standby](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iii-pilot-light-and-warm-standby/) -- Deep dive on the most confused pair
- [DR Architecture Blog Part IV -- Multi-Site Active-Active](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/) -- Active-Active write strategies
- [Establishing RPO and RTO Targets](https://aws.amazon.com/blogs/mt/establishing-rpo-and-rto-targets-for-cloud-applications/) -- Business-side of DR target setting
- [AWS Elastic Disaster Recovery](https://aws.amazon.com/disaster-recovery/) -- DRS service overview
- [Aurora Global Database (Mar 25 deep dive)](2026-03-25-rds-aurora-deep-dive.md) -- Managed switchover, unplanned failover, write forwarding
- [DynamoDB Global Tables (Mar 26 deep dive)](2026-03-26-dynamodb-deep-dive.md) -- MREC last-writer-wins, MRSC strong consistency
- [ElastiCache Global Datastore (Mar 27 deep dive)](2026-03-27-elasticache-caching-strategies.md) -- Single-writer cross-region replication
- [S3 CRR (Mar 23 deep dive)](2026-03-23-s3-advanced-deep-dive.md) -- Bidirectional replication, replication metrics
