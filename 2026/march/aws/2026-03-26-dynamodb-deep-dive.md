# DynamoDB Deep Dive -- The Filing Cabinet Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock), equipped your instances with personal workbenches, shared libraries, and specialty studios (EBS, EFS, FSx), and yesterday built the managed hotel chain with its revolutionary shared vault system for structured, queryable relational data (RDS and Aurora). But what happens when your application does not need the relational model at all? When your data does not have complex relationships requiring JOINs? When you need **single-digit millisecond latency at any scale** -- from 10 requests per second to 10 million -- without provisioning servers, managing storage, or planning capacity? When you know **exactly** how your application will query the data, and you want the database to be optimized for those specific access patterns rather than offering ad-hoc query flexibility? That is the domain of DynamoDB -- Amazon's fully managed NoSQL key-value and document database. And unlike the relational world where you design your schema around entities and relationships (normalization) and let SQL handle any query pattern you dream up later, DynamoDB inverts the process entirely: you identify your access patterns **first**, then design your keys and indexes to serve them. This is the single biggest mental shift from RDS/Aurora, and it is the reason this deep dive exists.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| DynamoDB Table | A giant filing cabinet with labeled dividers -- but instead of columns, each folder can contain whatever documents it wants | A schemaless collection of items where only the primary key attributes are required; every other attribute can differ per item -- unlike RDS where every row must conform to the table schema |
| Partition Key (PK) | The label on each filing cabinet drawer -- determines which drawer your folder goes into; the filing clerk hashes the label to pick the drawer | The attribute DynamoDB hashes to determine which physical partition stores the item; must be high-cardinality to ensure even distribution across partitions |
| Sort Key (SK) | The tab on each folder within a drawer -- folders sharing the same drawer are organized alphabetically by their tab, enabling you to grab a range of folders efficiently | The second component of a composite primary key; items sharing a partition key are stored sorted by sort key, enabling range queries (begins_with, between, >, <) |
| Global Secondary Index (GSI) | Building a completely separate filing cabinet with different drawer labels and folder tabs -- a full reorganization of the same documents for a different lookup pattern | An index with its own partition key and sort key (can differ from the base table), its own throughput, created or deleted anytime; supports only eventually consistent reads |
| Local Secondary Index (LSI) | Reorganizing folders within each existing drawer by a different tab -- same drawers, different sort order | An index that shares the base table's partition key but uses an alternate sort key; must be defined at table creation, shares base table throughput, 10 GB per-partition-key limit |
| On-Demand Capacity | A pay-per-use parking garage -- drive in, park, pay only for the time you actually park, no monthly reservation | Pay-per-request pricing with no capacity planning; DynamoDB automatically scales to handle any traffic level up to double the previous peak |
| Provisioned Capacity | A reserved monthly parking space -- cheaper per park if you use it daily, wasted if you park once a month | You specify read/write capacity units (RCU/WCU); supports auto-scaling; 25-40% cheaper than on-demand for steady workloads; reserved capacity saves up to 75% |
| DAX (DynamoDB Accelerator) | A reception desk with a photocopy of the most-requested files -- instead of walking to the filing cabinet every time, you check the desk first | Fully managed in-memory cache that sits between your app and DynamoDB; API-compatible (swap the SDK client); microsecond reads for cached items; write-through caching |
| DynamoDB Streams | A security camera recording every change to the filing cabinet -- every folder added, modified, or removed is logged in order, and you can trigger actions based on those recordings | Time-ordered sequence of item-level changes (insert/update/delete) retained for 24 hours; four view types (KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES); powers Lambda triggers and event-driven architectures |
| Global Tables | Opening identical filing cabinets in offices across multiple cities, with a courier service that synchronizes changes between them in under a second -- every office can add and modify folders | Multi-region, multi-active replication where every replica accepts reads and writes; last-writer-wins conflict resolution (MREC) or strong consistency across 3 regions (MRSC); 99.999% SLA |
| Single-Table Design | Storing customers, orders, and products in the same filing cabinet with a clever labeling scheme so you can pull a customer and all their orders in one trip to the drawer | Placing multiple entity types in one DynamoDB table with composite keys (e.g., PK=CUSTOMER#123, SK=ORDER#2024-01-15) to retrieve related items in a single query -- compensating for the lack of JOINs |

---

## The Big Picture: DynamoDB as a Filing Cabinet, RDS/Aurora as a Spreadsheet

The most effective way to internalize DynamoDB is to contrast it with the relational model you learned yesterday.

**RDS/Aurora is a spreadsheet.** Every row has the same columns. The schema is enforced -- you cannot insert a row that violates the column definitions. You write queries using SQL, and you can JOIN any tables together in ways the original designer never anticipated. The query optimizer figures out how to retrieve data efficiently regardless of how you ask for it. The trade-off: as your data grows into the billions of rows, the query optimizer struggles, JOINs become expensive, and vertical scaling (bigger instance) eventually hits a ceiling.

**DynamoDB is a filing cabinet.** Each table is a cabinet with multiple drawers. The **partition key** determines which drawer a folder goes into (DynamoDB hashes the key to pick the physical partition). Within a drawer, folders are organized by the **sort key** -- alphabetically sorted, so you can grab a range of folders efficiently. Each folder (item) can contain whatever documents (attributes) it wants -- one folder might have 5 pages, another 50. There is no schema enforcement beyond the primary key. The trade-off: you cannot open every drawer at once and cross-reference folders the way SQL JOINs can -- you must design your drawers and tabs (keys) around the specific lookups your application will perform.

```
THE FUNDAMENTAL PARADIGM SHIFT: RDS/AURORA vs DYNAMODB
════════════════════════════════════════════════════════════════════════════

  RDS/Aurora (Relational)                 DynamoDB (NoSQL Key-Value)
  ───────────────────────                 ──────────────────────────
  Design schema first,                    Design access patterns first,
  query any way later                     schema serves those patterns

  ┌──────────────────────────┐            ┌───────────────────────────┐
  │  Users Table             │            │  Single DynamoDB Table     │
  │  ┌────┬───────┬────────┐ │            │                           │
  │  │ id │ name  │ email  │ │            │  PK              SK       │
  │  ├────┼───────┼────────┤ │            │  ─────────────── ──────── │
  │  │ 1  │ Alice │ a@b.co │ │            │  USER#1          PROFILE  │
  │  │ 2  │ Bob   │ b@b.co │ │            │  USER#1          ORDER#01 │
  │  └────┴───────┴────────┘ │            │  USER#1          ORDER#02 │
  │                          │            │  USER#2          PROFILE  │
  │  Orders Table            │            │  USER#2          ORDER#01 │
  │  ┌────┬─────────┬───────┐│            │  PRODUCT#A       INFO     │
  │  │ id │ user_id │ total ││            │  PRODUCT#A       REVIEW#1 │
  │  ├────┼─────────┼───────┤│            │                           │
  │  │ 1  │ 1       │ $50   ││            │  Query: PK=USER#1         │
  │  │ 2  │ 1       │ $30   ││            │  Returns: profile + all   │
  │  └────┴─────────┴───────┘│            │  orders in ONE query      │
  │                          │            │  (no JOIN needed)          │
  │  SELECT * FROM users     │            │                           │
  │  JOIN orders             │            │  Trade-off: cannot easily  │
  │  ON users.id =           │            │  ask "all orders over $40" │
  │    orders.user_id;       │            │  unless you designed a     │
  │  (any query, anytime)    │            │  GSI for that pattern      │
  └──────────────────────────┘            └───────────────────────────┘

  Scales: Vertically                      Scales: Horizontally
  (bigger instance)                       (more partitions, unlimited)

  Latency: ~5-10ms typical               Latency: ~1-5ms typical
  (depends on query complexity)           (consistent at any scale)
```

---

## Part 1: Primary Keys -- The Drawer Labels and Folder Tabs

The primary key is the single most important design decision in DynamoDB. It determines how data is distributed across partitions, how it can be queried, and whether your table will perform well or suffer from throttling. With RDS/Aurora, the primary key is usually an auto-incrementing integer or UUID and has no impact on data distribution (the storage engine handles that transparently). In DynamoDB, the primary key **is** the distribution mechanism.

### Simple Primary Key (Partition Key Only)

**The analogy**: A filing cabinet where each drawer is labeled with a unique name. You store exactly one folder per drawer. To retrieve a folder, you tell the clerk the drawer label, and they hash it to find the right drawer instantly.

**Technical reality:**
- Each item is uniquely identified by its partition key value
- DynamoDB applies an internal hash function to the partition key to determine the physical partition
- Items with the same partition key value are **not allowed** -- the partition key must be unique per item
- Supports only **exact match** lookups (GetItem) -- you cannot query for a range of partition keys

```
SIMPLE PRIMARY KEY (PARTITION KEY ONLY)
════════════════════════════════════════════════════════════════

  Table: Users
  Partition Key: user_id

  ┌─────────────────────────────────────────────────┐
  │              DynamoDB Hash Function               │
  │   user_id ──▶ hash(user_id) ──▶ partition number  │
  └───────────┬───────────────┬───────────────┬───────┘
              ▼               ▼               ▼
  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
  │ Partition 0    │ │ Partition 1    │ │ Partition 2    │
  │                │ │                │ │                │
  │ user_id: U-42  │ │ user_id: U-7   │ │ user_id: U-99  │
  │ name: Alice    │ │ name: Bob      │ │ name: Carol    │
  │ email: a@b.co  │ │ email: b@c.co  │ │ plan: premium  │
  │ age: 30        │ │ age: 25        │ │ (no email attr)│
  │                │ │                │ │                │
  └───────────────┘ └───────────────┘ └───────────────┘

  Note: Each item can have different attributes (schemaless).
  Only user_id (the partition key) is required on every item.
```

### Composite Primary Key (Partition Key + Sort Key)

**The analogy**: A filing cabinet where each drawer is labeled with a category (partition key), and within each drawer, folders are organized by date tabs (sort key). You can open the "Customer-123" drawer and grab all folders, or just the folders with tabs between "2024-01-01" and "2024-06-30". Multiple folders can share the same drawer label as long as their tabs are different.

**Technical reality:**
- Items are uniquely identified by the **combination** of partition key + sort key
- Multiple items can share the same partition key (they live in the same partition, sorted by sort key)
- A group of items sharing the same partition key is called an **item collection**
- Supports **rich queries** on the sort key: `=`, `<`, `>`, `<=`, `>=`, `between`, `begins_with`
- The partition key is always an exact match in queries -- you **cannot** do a range query on the partition key

```
COMPOSITE PRIMARY KEY (PARTITION KEY + SORT KEY)
════════════════════════════════════════════════════════════════

  Table: OrderHistory
  Partition Key: customer_id
  Sort Key: order_date#order_id

  ┌─────────────────────────────────────────────────┐
  │              DynamoDB Hash Function               │
  │  customer_id ──▶ hash() ──▶ partition number      │
  └───────────┬───────────────────────────────┬───────┘
              ▼                               ▼
  ┌─────────────────────────────┐  ┌─────────────────────────────┐
  │ Partition for customer_id    │  │ Partition for customer_id    │
  │ = "CUST-123"                 │  │ = "CUST-456"                 │
  │                              │  │                              │
  │ Sorted by sort key:          │  │ Sorted by sort key:          │
  │ ┌──────────────────────────┐ │  │ ┌──────────────────────────┐ │
  │ │ SK: 2024-01-15#ORD-001  │ │  │ │ SK: 2024-03-01#ORD-050  │ │
  │ │ total: $50, status: done │ │  │ │ total: $200              │ │
  │ ├──────────────────────────┤ │  │ ├──────────────────────────┤ │
  │ │ SK: 2024-03-22#ORD-047  │ │  │ │ SK: 2024-07-18#ORD-088  │ │
  │ │ total: $30, items: 2     │ │  │ │ total: $75, status: new  │ │
  │ ├──────────────────────────┤ │  │ └──────────────────────────┘ │
  │ │ SK: 2024-06-10#ORD-102  │ │  │                              │
  │ │ total: $120              │ │  │                              │
  │ └──────────────────────────┘ │  │                              │
  └─────────────────────────────┘  └─────────────────────────────┘

  Query: PK = "CUST-123" AND SK between "2024-01-01" and "2024-04-01"
  Returns: ORD-001 and ORD-047 (efficient range scan within one partition)
```

### Partition Key Design: The Hot Partition Problem

This is where DynamoDB fundamentally diverges from RDS/Aurora. With Aurora, you never think about data distribution -- the storage layer splits data into 10 GB segments and spreads them across 3 AZs automatically. With DynamoDB, **you** are the architect of data distribution through your partition key choice.

**Each partition supports a hard ceiling:**
- **3,000 Read Capacity Units (RCU)** per second
- **1,000 Write Capacity Units (WCU)** per second

If 80% of your traffic targets a single partition key value ("hot partition"), that partition throttles even if your table's total provisioned capacity is far higher. This is like having a 50-drawer filing cabinet but 80% of employees only access drawer #7 -- the other 49 drawers sit idle while a queue forms at drawer #7.

**Worked example**: You provision 10,000 WCU across the table. Your table has 10 partitions, so each partition gets ~1,000 WCU capacity. If 8,000 of your 10,000 writes per second target a single partition key value, that partition receives 8,000 WCU of demand against its 1,000 WCU ceiling -- **7,000 writes/second are throttled**, even though the table as a whole has 9,000 WCU sitting idle on the other 9 partitions.

**Good partition keys (high cardinality, uniform access):**
- `user_id` -- millions of unique users, traffic spread across them
- `device_id` -- IoT devices, each sending data independently
- `session_id` -- web sessions, naturally distributed

**Bad partition keys (low cardinality, skewed access):**
- `status` -- only a few values ("active", "inactive", "pending"), most items hit "active"
- `date` -- all writes for a day hit the same partition key value
- `country_code` -- a few countries dominate traffic

**The write-sharding pattern (for unavoidably hot keys):**

When your access pattern inherently has a hot key (e.g., a viral product getting millions of votes), append a random suffix to distribute writes, then aggregate reads:

```
WRITE SHARDING -- SPREADING A HOT KEY ACROSS PARTITIONS
════════════════════════════════════════════════════════════════

  Problem: PK = "PRODUCT-viral" gets 50,000 writes/sec
           Single partition limit: 1,000 WCU -- throttled!

  Solution: Append random suffix (0-9)

  PK = "PRODUCT-viral#0"  ──▶ Partition A  (5,000 writes/sec)
  PK = "PRODUCT-viral#1"  ──▶ Partition B  (5,000 writes/sec)
  PK = "PRODUCT-viral#2"  ──▶ Partition C  (5,000 writes/sec)
  ...
  PK = "PRODUCT-viral#9"  ──▶ Partition J  (5,000 writes/sec)

  Write: randomly pick suffix 0-9, write to that partition key
  Read:  query all 10 suffixed keys, aggregate results in app

  Trade-off: reads become 10x more complex (10 queries + aggregation)
             but writes scale horizontally without throttling
```

**Adaptive capacity**: DynamoDB automatically detects hot partitions and isolates frequently accessed items onto dedicated partitions. This mitigates the problem but does not eliminate it -- adaptive capacity is a safety net, not a replacement for good key design.

---

## Part 2: Secondary Indexes -- Building Alternative Filing Cabinets

Your primary key design optimizes for your main access pattern. But applications typically have multiple query needs. In RDS/Aurora, you write a different SQL query with a different WHERE clause -- the query optimizer figures it out. In DynamoDB, if you need a different access pattern, you need a **secondary index**: a materialized view of your data reorganized by different keys.

### Global Secondary Index (GSI)

**The analogy**: Building a completely separate filing cabinet with different drawer labels and folder tabs. The original cabinet is organized by customer_id (drawers) and order_date (tabs). The GSI cabinet reorganizes the same documents by product_category (drawers) and total_amount (tabs). Both cabinets are maintained automatically -- whenever you add a folder to the original cabinet, a copy appears in the GSI cabinet. The GSI cabinet has its own staff (throughput) and its own capacity separate from the original.

**Technical reality:**
- **Own partition key and sort key** -- can be completely different from the base table
- **Created or deleted at any time** -- no need to decide at table creation
- **Own provisioned throughput** (separate RCU/WCU from the base table) or shares the table's on-demand capacity
- **No size limit** per partition key value
- **Supports only eventually consistent reads** -- this is the key limitation. If you write an item and immediately query the GSI, the GSI may not yet reflect the write. Typical replication lag is milliseconds but not guaranteed.
- **Up to 20 GSIs per table** (soft limit, can request increase)
- **Sparse index**: Items missing the GSI's key attributes are simply not included in the index -- this is a powerful filtering mechanism

```hcl
# Terraform: DynamoDB table with a GSI
resource "aws_dynamodb_table" "orders" {
  name         = "orders"
  billing_mode = "PAY_PER_REQUEST"  # On-demand capacity
  hash_key     = "customer_id"      # Partition key
  range_key    = "order_date"       # Sort key

  attribute {
    name = "customer_id"
    type = "S"  # String
  }

  attribute {
    name = "order_date"
    type = "S"
  }

  attribute {
    name = "product_category"
    type = "S"
  }

  attribute {
    name = "total_amount"
    type = "N"  # Number
  }

  # GSI: Query orders by product category, sorted by amount
  global_secondary_index {
    name            = "category-amount-index"
    hash_key        = "product_category"
    range_key       = "total_amount"
    projection_type = "INCLUDE"
    non_key_attributes = ["customer_id", "order_status"]
    # Only customer_id and order_status are projected into the index
    # (plus the key attributes). Accessing other attributes requires
    # a fetch back to the base table.
  }

  tags = {
    Name        = "orders"
    Environment = "production"
  }
}
```

### Local Secondary Index (LSI)

**The analogy**: Instead of building a separate filing cabinet, you reorganize the folders **within each existing drawer** by a different tab. The drawers stay the same (same partition key), but within each drawer, you can now look up folders by a different sort order. The original cabinet uses order_date as the tab; the LSI adds an alternative tab of order_status so you can find all "pending" orders for a customer within their drawer.

**Technical reality:**
- **Same partition key** as the base table, **different sort key** only
- **Must be defined at table creation time** -- cannot be added later. If you forget an LSI and realize you need one months later, you must recreate the entire table and migrate data. This is the most painful constraint.
- **Shares the base table's throughput** -- LSI reads/writes consume the base table's RCU/WCU, creating contention
- **10 GB item collection limit** per partition key value -- all items sharing a partition key (across the base table and all LSIs) cannot exceed 10 GB. This is a silent killer: your application works fine for months until a partition key's item collection hits 10 GB, then writes start failing with `ItemCollectionSizeLimitExceededException`. There is no auto-splitting like Aurora's 10 GB segments.
- **Supports strongly consistent reads** -- this is LSI's only advantage over GSI
- **Up to 5 LSIs per table** (hard limit, cannot be increased)

### GSI vs LSI Decision Framework

```
GSI vs LSI DECISION TREE
════════════════════════════════════════════════════════════════

  Do you need a different partition key than the base table?
       │
       ├── YES ──▶ GSI (only option -- LSI shares the base table's PK)
       │
       └── NO (same partition key, different sort key)
             │
             ├── Do you need strongly consistent reads on this index?
             │     │
             │     ├── YES ──▶ LSI (the only advantage of LSI)
             │     │
             │     └── NO ──▶ GSI (prefer GSI for flexibility)
             │
             ├── Has the table already been created?
             │     │
             │     ├── YES ──▶ GSI (LSI must be defined at creation)
             │     │
             │     └── NO ──▶ Consider LSI only if you need strong
             │                 consistency AND your item collections
             │                 will stay under 10 GB per partition key
             │
             └── Practical rule: DEFAULT TO GSI FOR EVERYTHING.
                   Use LSI only when you have a proven need for
                   strongly consistent reads on an alternate sort key
                   AND you are confident about the 10 GB limit.
```

| Feature | GSI | LSI |
|---|---|---|
| **Partition key** | Any attribute (different from base table) | Same as base table |
| **Sort key** | Any attribute | Different from base table |
| **When created** | Anytime (create/delete dynamically) | Table creation time only |
| **Throughput** | Own separate RCU/WCU | Shares base table's RCU/WCU |
| **Size limit** | None | 10 GB per partition key value |
| **Consistency** | Eventually consistent only | Eventually or strongly consistent |
| **Max per table** | 20 (soft limit) | 5 (hard limit) |
| **Projection types** | KEYS_ONLY, INCLUDE, ALL | KEYS_ONLY, INCLUDE, ALL |
| **Sparse index** | Yes (missing key attrs = item excluded) | Yes |

### Index Projections and Cost Implications

Every GSI and LSI stores a copy of projected attributes from the base table. The projection type controls what is copied:

- **KEYS_ONLY**: Only the base table and index key attributes. Smallest storage, lowest write cost, but any query needing other attributes triggers a **fetch** back to the base table (expensive).
- **INCLUDE**: Key attributes plus specific additional attributes you list. Good balance -- project only the attributes your queries need.
- **ALL**: Every attribute from the base table. Largest storage and highest write cost, but queries never need to fetch from the base table.

**Cost impact**: Every write to the base table that modifies a projected attribute triggers a write to the GSI. If you project ALL attributes into 5 GSIs, a single base table write creates 6 write operations total (1 base + 5 GSI). This is a multiplicative cost that surprises teams who casually add GSIs with ALL projections. Compare this with RDS where indexes are maintained by the engine transparently -- you still pay the overhead, but it is hidden in overall IOPS rather than explicitly billed per write.

---

## Part 3: Capacity Modes -- Parking Garage Pricing

DynamoDB offers two fundamentally different billing models, similar to the provisioned-vs-serverless choice you learned with Aurora (provisioned instances vs Serverless v2). The decision framework is similar: predictable traffic favors provisioned; unpredictable traffic favors on-demand.

### Read and Write Capacity Units

Before choosing a mode, understand the units:

| Unit | Definition | Consistency Impact |
|---|---|---|
| **1 RCU** | One strongly consistent read per second for an item up to 4 KB | Eventually consistent reads cost **half** (1 RCU = 2 eventually consistent reads) |
| **1 WCU** | One write per second for an item up to 1 KB | Transactional writes cost **double** (1 transactional write = 2 WCU) |

**Example calculation**: Reading a 9 KB item:
- Strongly consistent: ceiling(9 KB / 4 KB) = ceiling(2.25) = **3 RCU** per read
- Eventually consistent: Step 1: ceiling(9 KB / 4 KB) = 3. Step 2: 3 / 2 = 1.5, rounds up to **2 RCU** per read

### On-Demand Mode (Pay-Per-Request)

**The analogy**: A pay-per-use parking garage. You drive in, park, pay for exactly the time you used, and leave. No monthly commitment, no wasted capacity if you do not show up for a week. But each individual park is more expensive than a reserved space.

**Technical reality:**
- **Zero capacity planning** -- DynamoDB handles all scaling automatically
- Instantly accommodates up to **double the previous peak** traffic. If your table handled 10,000 WCU yesterday and today gets 20,000 WCU, DynamoDB scales automatically. But a sudden spike from 10,000 to 50,000 may trigger throttling until DynamoDB adjusts (typically minutes).
- **Pricing**: ~$1.25 per million write request units, ~$0.25 per million read request units (us-east-1)
- **Best for**: New tables with unknown traffic patterns, spiky/unpredictable workloads, applications where you prefer operational simplicity over cost optimization
- **Switching**: Can switch from on-demand to provisioned unlimited times. Can switch from provisioned to on-demand up to **4 times per 24 hours** (formerly once per day, relaxed in August 2025).

### Provisioned Mode (Reserved Capacity)

**The analogy**: A reserved monthly parking space. You pay a fixed monthly rate whether you park there every day or once. Cheaper per-park if you are a daily commuter, wasted money if you only drive on weekends. You can hire an attendant (auto-scaling) to reserve extra spaces when the lot gets full and release them when it empties.

**Technical reality:**
- You specify RCU and WCU per table (and per GSI for provisioned GSIs)
- **Auto-scaling** adjusts capacity between a minimum and maximum based on a target utilization percentage (typically 70%)
- Auto-scaling has a **reaction delay** of ~5-10 minutes -- traffic spikes faster than auto-scaling can react will cause throttling. Burst capacity (a 5-minute rolling window of unused capacity) provides a buffer, but it is not infinite.
- **Reserved capacity**: Commit to 1- or 3-year terms for up to **75% savings** over on-demand pricing
- **Free tier**: 25 WCU + 25 RCU of provisioned capacity permanently free (Always Free tier, not time-limited). This gives ~64.8 million writes/month (25 WCU × 86,400 sec/day × 30 days) and ~64.8 million strongly consistent reads/month (or ~129.6 million eventually consistent reads). The often-quoted "200 million requests/month" assumes a read-heavy workload with eventually consistent reads on items under 4 KB.

### Capacity Mode Decision Framework

```
ON-DEMAND vs PROVISIONED DECISION
════════════════════════════════════════════════════════════════

  Is traffic predictable (steady baseline with known peaks)?
       │
       ├── YES ──▶ Provisioned + Auto-Scaling
       │           (25-40% cheaper than on-demand)
       │           │
       │           ├── Traffic is extremely stable?
       │           │     ──▶ Consider Reserved Capacity (up to 75% off)
       │           │
       │           └── Peak-to-average ratio < 4x?
       │                 ──▶ Provisioned is almost certainly cheaper
       │
       └── NO ──▶ On-Demand
                  │
                  ├── New table with unknown traffic?     ──▶ On-Demand
                  ├── Peak-to-average ratio > 4x?         ──▶ On-Demand
                  ├── Traffic is rare but bursty?          ──▶ On-Demand
                  └── Operational simplicity > cost?       ──▶ On-Demand

  Comparison to Aurora:
  ┌────────────────────────────────────────────────────────┐
  │  Aurora Serverless v2    ←→    DynamoDB On-Demand      │
  │  (variable compute,           (variable throughput,    │
  │   pay per ACU-minute)          pay per request)        │
  │                                                        │
  │  Aurora Provisioned      ←→    DynamoDB Provisioned    │
  │  (fixed instance size,         (fixed RCU/WCU,         │
  │   cheaper for steady load)      cheaper for steady)    │
  └────────────────────────────────────────────────────────┘
```

### Terraform Example: Provisioned with Auto-Scaling

```hcl
# DynamoDB table with provisioned capacity and auto-scaling
resource "aws_dynamodb_table" "sessions" {
  name         = "user-sessions"
  billing_mode = "PROVISIONED"
  hash_key     = "session_id"
  range_key    = "created_at"

  read_capacity  = 100   # Baseline: 100 RCU
  write_capacity = 50    # Baseline: 50 WCU

  attribute {
    name = "session_id"
    type = "S"
  }

  attribute {
    name = "created_at"
    type = "S"
  }

  tags = {
    Name        = "user-sessions"
    Environment = "production"
  }
}

# Auto-scaling target for read capacity
resource "aws_appautoscaling_target" "read" {
  max_capacity       = 1000  # Scale up to 1000 RCU
  min_capacity       = 100   # Never drop below 100 RCU
  resource_id        = "table/${aws_dynamodb_table.sessions.name}"
  scalable_dimension = "dynamodb:table:ReadCapacityUnits"
  service_namespace  = "dynamodb"
}

# Auto-scaling policy: target 70% utilization
resource "aws_appautoscaling_policy" "read" {
  name               = "sessions-read-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.read.resource_id
  scalable_dimension = aws_appautoscaling_target.read.scalable_dimension
  service_namespace  = aws_appautoscaling_target.read.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBReadCapacityUtilization"
    }
    target_value       = 70.0  # Scale when utilization exceeds 70%
    scale_in_cooldown  = 60    # Wait 60s before scaling down
    scale_out_cooldown = 60    # Wait 60s before scaling up again
  }
}

# Auto-scaling target for write capacity
resource "aws_appautoscaling_target" "write" {
  max_capacity       = 500
  min_capacity       = 50
  resource_id        = "table/${aws_dynamodb_table.sessions.name}"
  scalable_dimension = "dynamodb:table:WriteCapacityUnits"
  service_namespace  = "dynamodb"
}

resource "aws_appautoscaling_policy" "write" {
  name               = "sessions-write-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.write.resource_id
  scalable_dimension = aws_appautoscaling_target.write.scalable_dimension
  service_namespace  = aws_appautoscaling_target.write.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBWriteCapacityUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 60
    scale_out_cooldown = 60
  }
}
```

---

## Part 4: DAX (DynamoDB Accelerator) -- The Reception Desk Cache

**The analogy**: Your filing cabinet handles thousands of requests per day, but you notice that 80% of the requests are for the same 200 files. Instead of sending clerks to the cabinet every time, you set up a **reception desk** at the entrance with photocopies of those 200 files. When someone asks for a file, the receptionist checks the desk first -- if the copy is there and recent enough (TTL), they hand it over instantly (microseconds). If not, they send a clerk to the cabinet (milliseconds), get the original, make a fresh copy for the desk, and hand it to the requester. Critically, when someone updates a file in the cabinet, the reception desk's copy is updated simultaneously (write-through), so the desk always has a fresh copy for subsequent reads.

**Technical reality:**
- **Fully managed in-memory cache** that sits between your application and DynamoDB
- **API-compatible**: Swap the DynamoDB SDK client for the DAX client SDK -- your existing `GetItem`, `Query`, `BatchGetItem`, and `Scan` calls work with zero application logic changes
- **Microsecond read latency** for cached items (vs single-digit millisecond for DynamoDB direct)
- **VPC-based**: DAX clusters run inside your VPC as EC2-like nodes (primary + read replicas)
- **Two separate caches**:
  - **Item cache**: Stores individual items from `GetItem`/`BatchGetItem` with configurable TTL (default 5 minutes) and LRU eviction
  - **Query cache**: Stores full result sets from `Query`/`Scan` operations, indexed by parameter values, with configurable TTL
- **Write-through**: Writes go to both DynamoDB and DAX simultaneously -- the item cache is updated on write, but the query cache is **not** invalidated (it relies on TTL expiration). This means a `Query` result may be stale even after a write, but a `GetItem` for the same item will return the fresh version.
- **Consistency limitation**: DAX only caches **eventually consistent reads**. Strongly consistent reads pass through DAX directly to DynamoDB, gaining no caching benefit. If your application requires strong consistency, DAX provides no value for those reads.

```
DAX ARCHITECTURE
════════════════════════════════════════════════════════════════

  Application (Lambda, ECS, EC2)
       │
       │  Uses DAX SDK client
       │  (drop-in replacement for DynamoDB SDK)
       │
       ▼
  ┌──────────────────────────────────────────────────┐
  │              DAX Cluster (in your VPC)            │
  │                                                   │
  │  ┌─────────────┐  ┌─────────────┐               │
  │  │ DAX Primary  │  │ DAX Replica  │ (up to 10    │
  │  │ Node         │  │ Node         │  replicas)   │
  │  │              │  │              │               │
  │  │ ┌─────────┐  │  │ ┌─────────┐  │               │
  │  │ │Item     │  │  │ │Item     │  │               │
  │  │ │Cache    │  │  │ │Cache    │  │               │
  │  │ │(5m TTL) │  │  │ │(5m TTL) │  │               │
  │  │ ├─────────┤  │  │ ├─────────┤  │               │
  │  │ │Query    │  │  │ │Query    │  │               │
  │  │ │Cache    │  │  │ │Cache    │  │               │
  │  │ │(5m TTL) │  │  │ │(5m TTL) │  │               │
  │  │ └─────────┘  │  │ └─────────┘  │               │
  │  └──────┬───────┘  └─────────────┘               │
  │         │                                         │
  └─────────┼─────────────────────────────────────────┘
            │
            │  Cache miss: GetItem/Query → DynamoDB
            │  Write-through: PutItem/UpdateItem → both DAX + DynamoDB
            │  Strongly consistent read: pass-through → DynamoDB directly
            │
            ▼
  ┌──────────────────────────────────┐
  │        DynamoDB Table             │
  │  (fully managed, no VPC needed)   │
  └──────────────────────────────────┘

  DAX vs Aurora Read Replicas:
  ┌────────────────────────────────────────────────────────┐
  │  DAX                         Aurora Read Replicas       │
  │  ─────                       ─────────────────────      │
  │  In-memory cache             Read from shared storage   │
  │  Data can be stale (TTL)     Sub-10ms lag (near-real)   │
  │  Microsecond reads           Low-millisecond reads      │
  │  Reduces DynamoDB RCU cost   No additional storage cost │
  │  Write-through pattern       No caching (direct reads)  │
  │  Requires VPC placement      Part of Aurora cluster     │
  └────────────────────────────────────────────────────────┘
```

### DAX vs ElastiCache: When to Use Which

| Dimension | DAX | ElastiCache (Redis/Memcached) |
|---|---|---|
| **Scope** | DynamoDB-only | Any data source (RDS, Aurora, APIs, computed results) |
| **Integration** | Drop-in SDK replacement, zero code changes to query logic | Application must implement caching logic (cache-aside, write-through, etc.) |
| **Protocol** | DynamoDB API | Redis/Memcached protocol |
| **Use case** | Accelerate DynamoDB reads with hot-key patterns | General-purpose caching, session stores, leaderboards, pub/sub |
| **Consistency** | Eventually consistent reads only | Application-managed consistency |
| **Data structures** | Items and query results only | Strings, lists, sets, sorted sets, hashes, streams (Redis) |

**The simple rule**: If you are caching DynamoDB reads and want zero code changes, use DAX. For everything else, use ElastiCache.

---

## Part 5: DynamoDB Streams -- The Security Camera System

**The analogy**: You install security cameras on every drawer of your filing cabinet. Every time a folder is added, modified, or removed, the camera records exactly what happened. The recordings are kept for 24 hours on a rolling basis. You can hire a staff member (Lambda function) to watch the recordings in real time and take action -- sending a notification when a VIP customer's folder is updated, archiving deleted folders to a vault (S3), or updating a report whenever a new order appears.

**Technical reality:**
- A **time-ordered, deduplicated sequence** of every item-level modification (insert, update, delete)
- Records are **retained for 24 hours** (for longer retention, use Kinesis Data Streams for DynamoDB: up to 365 days)
- **Guaranteed ordering per item** -- changes to the same item appear in the stream in the order they occurred. No global ordering across items.
- **Exactly-once delivery** to the stream (but Lambda may process a record more than once during retries -- design for idempotency)

### StreamViewType -- What the Camera Records

Set once when enabling the stream. Cannot be changed without disabling and re-enabling.

| StreamViewType | What Each Record Contains | Size | Use Case |
|---|---|---|---|
| **KEYS_ONLY** | Just the primary key of the modified item | Smallest | Trigger-only (know *what* changed, look up details if needed) |
| **NEW_IMAGE** | Complete item after modification | Medium | Replicate current state to another system |
| **OLD_IMAGE** | Complete item before modification | Medium | Audit trail of previous state, undo operations |
| **NEW_AND_OLD_IMAGES** | Both before and after | Largest | Compute diffs, implement complex business logic on changes |

### Stream Processing Patterns

```
DYNAMODB STREAMS INTEGRATION PATTERNS
════════════════════════════════════════════════════════════════

  Pattern 1: Lambda Trigger (most common)
  ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
  │  DynamoDB    │────▶│  DynamoDB    │────▶│  Lambda     │
  │  Table       │     │  Stream      │     │  Function   │
  │  (writes)    │     │  (24h retain)│     │  (process)  │
  └─────────────┘     └──────────────┘     └──────┬──────┘
                                                   │
                               ┌───────────────────┼───────────────┐
                               ▼                   ▼               ▼
                        ┌────────────┐     ┌────────────┐  ┌────────────┐
                        │ SNS/SES    │     │ Another    │  │ S3         │
                        │ (notify)   │     │ DynamoDB   │  │ (archive)  │
                        └────────────┘     │ Table      │  └────────────┘
                                           │ (aggregate)│
                                           └────────────┘

  Pattern 2: Kinesis Data Streams for DynamoDB (longer retention)
  ┌─────────────┐     ┌──────────────┐     ┌──────────────────┐
  │  DynamoDB    │────▶│  Kinesis     │────▶│  Multiple        │
  │  Table       │     │  Data Stream │     │  Consumers       │
  │              │     │  (365d max)  │     │  (KCL, Lambda,   │
  └─────────────┘     └──────────────┘     │   Firehose, etc.)│
                                            └──────────────────┘

  Key difference from RDS/Aurora:
  ┌────────────────────────────────────────────────────────────┐
  │  DynamoDB Streams           RDS/Aurora Equivalent           │
  │  ─────────────────          ─────────────────────           │
  │  First-class CDC with       Limited: RDS Event              │
  │  4 granularity levels,      Notifications (operational      │
  │  native Lambda triggers,    events only, not data changes), │
  │  24h retention, free        Aurora Activity Streams          │
  │  Lambda polling reads       (for auditing, not CDC)         │
  │                                                             │
  │  DynamoDB Streams is the backbone of serverless event-      │
  │  driven architectures. There is no equivalent in RDS/Aurora │
  │  for reacting to individual row-level data changes.         │
  └────────────────────────────────────────────────────────────┘
```

### Terraform Example: DynamoDB Table with Stream + Lambda Trigger

```hcl
# DynamoDB table with streams enabled
resource "aws_dynamodb_table" "orders" {
  name         = "orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  # Enable DynamoDB Streams with full before/after images
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  # Point-in-time recovery (continuous backups, 35-day retention)
  point_in_time_recovery {
    enabled = true
  }

  # Server-side encryption with AWS-owned key (default, free).
  # This is NOT the AWS-managed KMS key (aws/dynamodb) -- that requires
  # specifying kms_key_arn. For a customer-managed CMK, set kms_key_arn
  # to your own aws_kms_key resource.
  server_side_encryption {
    enabled = true
  }

  tags = {
    Name        = "orders"
    Environment = "production"
  }
}

# Lambda function triggered by DynamoDB Stream
resource "aws_lambda_event_source_mapping" "stream_trigger" {
  event_source_arn  = aws_dynamodb_table.orders.stream_arn
  function_name     = aws_lambda_function.process_orders.arn
  starting_position = "LATEST"  # Process only new records (TRIM_HORIZON for all)

  # Batching configuration
  batch_size                         = 100   # Up to 100 records per invocation
  maximum_batching_window_in_seconds = 5     # Wait up to 5s to fill the batch
  parallelization_factor             = 10    # Up to 10 concurrent batches per shard

  # Error handling
  maximum_retry_attempts             = 3     # Retry failed batches 3 times
  maximum_record_age_in_seconds      = 3600  # Skip records older than 1 hour
  bisect_batch_on_function_error     = true  # Split batch on failure to isolate bad record

  # Dead-letter queue for failed records
  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.dlq.arn
    }
  }
}
```

---

## Part 6: Global Tables -- Multi-Region Filing Cabinet Network

**The analogy**: Your company opens offices in New York, London, and Tokyo. Each office gets an identical filing cabinet. A courier service synchronizes every change between all three cabinets in under a second -- when the New York office adds a folder, it appears in London and Tokyo within a second. Critically, **every office can add and modify folders** (true active-active, unlike Aurora Global Database where only the headquarters accepts deposits and other offices must forward write requests). If New York and London modify the same folder simultaneously, the courier service uses a simple rule: **the last change wins** (based on timestamps), and the losing change is silently overwritten.

**This is the sharpest contrast with Aurora Global Database:**

| Dimension | DynamoDB Global Tables | Aurora Global Database |
|---|---|---|
| **Write model** | Active-active (any region writes) | Single-writer (one primary region; secondaries use write forwarding) |
| **Conflict resolution** | Last-writer-wins (LWW) by internal timestamp | No conflicts (single writer) |
| **Replication mechanism** | DynamoDB Streams (application-level) | Storage-level redo log replication |
| **Replication lag** | Typically <1 second | Typically <1 second |
| **Consistency model** | Eventually consistent (MREC) or strong (MRSC, 3 regions only) | Eventually consistent in secondary regions; strong in primary |
| **SLA** | 99.999% (five nines) | 99.99% (four nines) |
| **Failover** | No failover needed (all regions are active) | Managed switchover or unplanned failover required |
| **Use case** | Global apps needing local-latency writes in every region | Global apps tolerating single write region + local reads |

### MREC vs MRSC Consistency Modes

- **MREC (Multi-Region Eventual Consistency)** -- the default. Writes replicate asynchronously with last-writer-wins. Reads in any region may not immediately reflect writes from another region. This is fine for most applications (user profiles, session data, content) where stale-by-one-second is acceptable.

- **MRSC (Multi-Region Strong Consistency)** -- introduced June 2025. Writes synchronously replicate across exactly **three regions** (the minimum for a write quorum of 2/3) before being acknowledged. RPO is zero (no data loss). The trade-off is higher write latency (cross-region round trip) and a 3-region limit. Use for financial transactions, inventory counts, or any data where even sub-second inconsistency is unacceptable.

### Global Tables Architecture

```
DYNAMODB GLOBAL TABLES -- ACTIVE-ACTIVE MULTI-REGION
════════════════════════════════════════════════════════════════

  ┌─── us-east-1 ────────────────┐
  │                               │
  │  ┌──────────────────────┐    │
  │  │ DynamoDB Replica     │    │
  │  │ Table (read + write) │    │      ┌─── eu-west-1 ────────────┐
  │  │                      │◀──▶│◀────▶│                           │
  │  │ Orders table         │    │      │  ┌──────────────────────┐ │
  │  │ PK: customer_id      │    │      │  │ DynamoDB Replica     │ │
  │  │ SK: order_date        │    │      │  │ Table (read + write) │ │
  │  └──────────────────────┘    │      │  │                      │ │
  │                               │      │  │ Identical schema,    │ │
  │  DAX Cluster (local cache)   │      │  │ local DAX cache      │ │
  │  Streams → local Lambda      │      │  └──────────────────────┘ │
  │                               │      │                           │
  └───────────────┬───────────────┘      └──────────┬────────────────┘
                  │                                  │
                  │     Async replication (<1s)       │
                  │     via DynamoDB Streams           │
                  │     Last-Writer-Wins (MREC)        │
                  │                                  │
                  └────────────┬─────────────────────┘
                               │
                  ┌────────────▼────────────────────┐
                  │       ap-northeast-1             │
                  │                                  │
                  │  ┌──────────────────────┐       │
                  │  │ DynamoDB Replica     │       │
                  │  │ Table (read + write) │       │
                  │  │                      │       │
                  │  │ Local reads + writes  │       │
                  │  │ for APAC users        │       │
                  │  └──────────────────────┘       │
                  │                                  │
                  └──────────────────────────────────┘

  Key behaviors:
  ─ Every replica accepts reads AND writes (true active-active)
  ─ Conflict resolution: last-writer-wins by internal timestamp
  ─ DAX caches are local per region (do NOT replicate)
  ─ TTL deletes replicate across regions
  ─ Transactions are isolated per region (cannot span regions)
  ─ Streams are per-replica (each region has its own stream)
  ─ On-demand or provisioned mode syncs globally
  ─ Auto-scaling settings are per-replica
```

### Global Tables Requirements

- All replicas must share the same table name, key schema, and DynamoDB Streams configuration
- **MREC mode**: Streams must be enabled with `NEW_AND_OLD_IMAGES` view type (MRSC uses a different internal replication mechanism and does not require DynamoDB Streams)
- Must use on-demand or provisioned with auto-scaling (static provisioned capacity without auto-scaling is not recommended because replicas may have different traffic levels)
- **MREC mode**: Tables can have existing data when adding a replica -- DynamoDB backfills automatically with no performance impact. **MRSC mode**: Tables must be empty when creating the global table (no backfill support)

---

## Part 7: Single-Table Design -- The Paradigm Shift

This is the concept that most confuses developers coming from relational databases, and it is arguably the most interview-worthy DynamoDB topic.

**The analogy**: In the relational world (RDS/Aurora), you have a separate filing cabinet for each type of document -- one for customers, one for orders, one for products. To answer "show me Customer #123 and all their orders," you open two cabinets and cross-reference folders by customer_id (a JOIN). This works beautifully because SQL can JOIN anything.

In the DynamoDB world, you have **one filing cabinet** with a clever labeling scheme. Customer #123's profile is in the drawer labeled "CUSTOMER#123" with a folder tab "PROFILE". Their orders are in the same drawer with tabs "ORDER#2024-01-15", "ORDER#2024-03-22", etc. To get the customer and all their orders, you open **one drawer** and grab everything -- a single Query operation. No JOIN needed because the data is **pre-joined** by key design.

**Why single-table design exists**: DynamoDB has **no JOIN operation**. Period. If your data is spread across multiple tables, assembling related data requires multiple round-trips from your application -- each adding latency and consuming separate capacity. Single-table design eliminates this by storing heterogeneous entity types in one table with composite keys that group related items into the same partition.

### Single-Table Design Pattern

```
SINGLE-TABLE DESIGN -- PRE-JOINING DATA BY KEY DESIGN
════════════════════════════════════════════════════════════════

  Table: ECommerce
  PK (Partition Key) | SK (Sort Key)           | Attributes
  ──────────────────────────────────────────────────────────────
  CUSTOMER#123       | PROFILE                 | name=Alice, email=a@b.co
  CUSTOMER#123       | ORDER#2024-01-15#001    | total=$50, status=shipped
  CUSTOMER#123       | ORDER#2024-03-22#047    | total=$30, status=pending
  CUSTOMER#123       | ORDER#2024-06-10#102    | total=$120, status=delivered
  CUSTOMER#456       | PROFILE                 | name=Bob, plan=premium
  CUSTOMER#456       | ORDER#2024-03-01#050    | total=$200, status=shipped
  PRODUCT#A          | INFO                    | name=Widget, price=$25
  PRODUCT#A          | REVIEW#2024-02-01#R1    | stars=5, author=Alice
  PRODUCT#A          | REVIEW#2024-05-15#R2    | stars=3, author=Bob

  Access Patterns Served:
  ─────────────────────────────────────────────────────────────
  1. Get customer profile:
     Query PK = "CUSTOMER#123" AND SK = "PROFILE"

  2. Get customer + all orders:
     Query PK = "CUSTOMER#123" AND SK begins_with("ORDER#")

  3. Get customer + orders in date range:
     Query PK = "CUSTOMER#123" AND SK between "ORDER#2024-01" and "ORDER#2024-04"

  4. Get product + all reviews:
     Query PK = "PRODUCT#A" AND SK begins_with("REVIEW#")

  5. Get everything about customer #123:
     Query PK = "CUSTOMER#123" (no SK condition)
     Returns: profile + all orders in one query

  Each of these is a SINGLE DynamoDB Query operation.
  In RDS, patterns 2-5 would each require a JOIN.
```

### When to Use Single-Table vs Multi-Table

Single-table design is **not always the right choice**. The AWS Database Blog explicitly acknowledges this:

| Approach | When to Use | Trade-offs |
|---|---|---|
| **Single-table** | Well-defined, stable access patterns; performance-critical workloads; serverless applications needing minimal round-trips | Steep learning curve; inflexible when new access patterns emerge; difficult for analytics (Athena/Redshift expect normalized data); complex to maintain |
| **Multi-table** | Evolving access patterns; teams unfamiliar with DynamoDB; data needs to be queried by analytics tools; simpler to reason about | Multiple round-trips for related data; higher latency for compound queries; more capacity consumed per operation |

**The practical reality**: Many production DynamoDB deployments use a hybrid approach -- single-table design for the core transactional data where access patterns are well-known and performance-critical, and separate tables for ancillary data (audit logs, analytics staging, configuration).

### Overloaded GSI Pattern

Single-table design often requires **overloaded GSIs** -- a GSI where the key attributes have different meanings depending on the entity type:

```
OVERLOADED GSI PATTERN
════════════════════════════════════════════════════════════════

  Base Table:
  PK              | SK                      | GSI1PK          | GSI1SK
  ─────────────────────────────────────────────────────────────────────
  CUSTOMER#123    | PROFILE                 | alice@b.co      | CUSTOMER#123
  CUSTOMER#123    | ORDER#2024-01-15#001    | SHIPPED          | 2024-01-15
  CUSTOMER#123    | ORDER#2024-03-22#047    | PENDING          | 2024-03-22
  PRODUCT#A       | INFO                    | WIDGET           | $25
  PRODUCT#A       | REVIEW#2024-02-01#R1    | CUSTOMER#123    | 2024-02-01

  GSI1 enables these additional access patterns:
  ─────────────────────────────────────────────────────────────────────
  1. Look up customer by email:
     GSI1 Query: GSI1PK = "alice@b.co"

  2. Get all shipped orders (across all customers):
     GSI1 Query: GSI1PK = "SHIPPED" AND GSI1SK between "2024-01" and "2024-12"

  3. Get all reviews by a specific customer:
     GSI1 Query: GSI1PK = "CUSTOMER#123" AND GSI1SK begins_with("2024")
     (where the item type is REVIEW)

  The same GSI1PK attribute holds:
  ─ email (for PROFILE items)
  ─ order status (for ORDER items)
  ─ product name (for INFO items)
  ─ reviewer customer_id (for REVIEW items)
```

---

## Part 8: DynamoDB Transactions, TTL, and Operational Features

### Transactions

DynamoDB supports **ACID transactions** across up to 100 items in one or more DynamoDB tables within the same AWS account and region. The aggregate size of all items in a transaction cannot exceed **4 MB**. This was a major addition (2018) that bridged a significant gap with relational databases.

- **TransactWriteItems**: Atomic write of up to 100 actions (Put, Update, Delete, ConditionCheck)
- **TransactGetItems**: Atomic read of up to 100 items
- **Cost**: Transactions consume **2x the WCU** (write) and **2x the RCU** (read) of their non-transactional equivalents
- **Scope**: **Single region only** -- transactions cannot span regions in Global Tables

### Time to Live (TTL)

- Specify a TTL attribute containing a Unix epoch timestamp
- DynamoDB automatically deletes items when the timestamp expires (within 48 hours of expiration -- not instant). **Gotcha**: Expired items still appear in Query and Scan results until physically deleted -- applications should filter on the TTL attribute client-side to exclude expired items.
- **Zero cost** -- TTL deletions do not consume WCU
- TTL deletes appear in DynamoDB Streams (so downstream consumers can react to expirations)
- TTL deletes **replicate across regions** in Global Tables
- Use case: Session expiration, temporary data cleanup, GDPR data retention compliance

### Point-in-Time Recovery (PITR)

- Continuous backups with **35-day retention** and per-second granularity
- Restore creates a **new table** (cannot restore in-place, similar to how RDS PITR restores to a new instance)
- Restores include GSIs, LSIs, streams configuration, encryption settings, and TTL settings
- **Must be explicitly enabled** (not on by default, unlike Aurora's continuous backups)

### On-Demand Backup and Restore

- Manual snapshots retained indefinitely (like RDS manual snapshots)
- Can restore to the same or different region
- Can restore with or without GSIs (useful for creating a lighter copy)

### Encryption

- **All DynamoDB tables are encrypted at rest** -- you cannot create an unencrypted table (unlike RDS where encryption is optional at creation). This is a simpler security posture.
- Three key options: AWS-owned key (default, free), AWS-managed KMS key (`aws/dynamodb`), or customer-managed KMS key (CMK)
- Encryption key can be changed on an existing table (unlike RDS where encryption must be set at creation and requires snapshot migration to change)

### Conditional Writes and Optimistic Locking

Without foreign keys and constraints, DynamoDB uses **ConditionExpression** on writes to enforce business rules at the database level:

- **ConditionExpression**: Attach a condition to any Put, Update, or Delete -- the operation fails if the condition is not met. Example: `PutItem` with `attribute_not_exists(PK)` prevents overwriting an existing item (an upsert guard).
- **Optimistic locking pattern**: Add a `version` attribute to each item. On every update, include `ConditionExpression: version = :expected_version` and increment the version. If another writer modified the item between your read and write, the version will not match and the write fails -- your application retries with the latest version. This is analogous to `SELECT ... FOR UPDATE` in RDS, but without the lock overhead.
- Conditional writes are **idempotent** when combined with a client token, making them safe for retries in distributed systems.

### Batch Operations (BatchWriteItem / BatchGetItem)

Distinct from transactions -- batches have **no atomicity guarantee** but are cheaper (no 2x cost multiplier):

- **BatchWriteItem**: Up to 25 Put or Delete operations in a single call (no Update). Individual items can target different tables. Some items may succeed while others fail (partial failure) -- check `UnprocessedItems` in the response and retry.
- **BatchGetItem**: Up to 100 items across one or more tables in a single call. Returns items in parallel for lower latency than individual GetItem calls.
- **When to use**: Bulk loading data, batch reads for displaying dashboards, or any operation where all-or-nothing atomicity is not required. Use transactions when you need atomicity.

### PartiQL Support

DynamoDB supports **PartiQL**, a SQL-compatible query language that lets you use familiar `SELECT`, `INSERT`, `UPDATE`, and `DELETE` syntax:

```sql
-- PartiQL examples (run via AWS CLI, SDK, or DynamoDB console)
SELECT * FROM "orders" WHERE "PK" = 'CUSTOMER#123' AND begins_with("SK", 'ORDER#')
INSERT INTO "orders" VALUE {'PK': 'CUSTOMER#789', 'SK': 'PROFILE', 'name': 'Carol'}
UPDATE "orders" SET "status" = 'shipped' WHERE "PK" = 'CUSTOMER#123' AND "SK" = 'ORDER#2024-01-15#001'
```

**Interview-critical point**: "Can you use SQL with DynamoDB?" -- Yes, through PartiQL. But PartiQL does **not** add JOIN capability or change the underlying data model. It is syntactic sugar over the DynamoDB API -- a `SELECT` still maps to a Query or Scan operation, and you still need proper key design for performance. PartiQL also supports transactions via `ExecuteTransaction`.

### Query vs Scan -- The Cost Difference

- **Query**: Targets a **single partition** by specifying an exact partition key value. Optionally filters by sort key conditions. Reads only the items that match -- efficient and predictable. This is the operation you should use 99% of the time.
- **Scan**: Reads **every item in the entire table** (or index), then applies filters client-side. Consumes massive RCU proportional to the full table size, regardless of how many items match the filter. A Scan on a 100 GB table reads all 100 GB even if only 10 items match.
- **When Scan is acceptable**: One-time data migrations, very small tables (under a few thousand items), or tables where you genuinely need to process every item. For recurring workloads, always design a GSI to turn a Scan into a Query.
- **Parallel Scan**: Splits the table into segments scanned concurrently by multiple workers -- faster but consumes even more RCU. Use only for one-time bulk operations.

### Export to S3

DynamoDB supports **direct table export to S3** without consuming read capacity:

- Exports in DynamoDB JSON or Amazon Ion format
- Uses PITR data (must have PITR enabled) -- exports a snapshot at any point within the 35-day PITR window
- **Zero impact on table performance** -- does not consume RCU or affect live traffic
- Use for: populating a data lake, feeding Athena/Redshift analytics, creating backups in a portable format
- **Format note**: Native export produces DynamoDB JSON or Amazon Ion, not Parquet directly. For optimal Athena performance, add a Glue ETL job to convert to Parquet columnar format.
- This is simpler than the Streams → Firehose → S3 pipeline for full-table exports. **Decision**: Use Export to S3 for batch/nightly analytics (full snapshots). Use Streams → Firehose → S3 for near-real-time analytics (incremental changes updated continuously).

### Table Classes (Standard vs Standard-IA)

Similar to S3 storage classes, DynamoDB offers two table classes:

| Table Class | Storage Cost | Read/Write Cost | Use Case |
|---|---|---|---|
| **Standard** (default) | Higher | Lower | Frequently accessed data (most workloads) |
| **Standard-IA** (Infrequent Access) | ~60% lower storage | ~25% higher per-request | Tables where storage dominates cost and access is infrequent (logs, audit trails, old records) |

- Can switch between classes with no downtime or performance impact
- Evaluate using CloudWatch metrics: if storage cost is >50% of total table cost and the table is infrequently accessed, Standard-IA may save money

---

## Part 9: DynamoDB vs RDS/Aurora Decision Framework

```
DYNAMODB vs RDS/AURORA DECISION TREE
════════════════════════════════════════════════════════════════════════════

  Does your data have complex relationships requiring JOINs?
       │
       ├── YES ──▶ RDS/Aurora
       │           (SQL JOINs, foreign keys, referential integrity)
       │
       └── NO or MINIMAL
             │
             ├── Do you know your access patterns upfront?
             │     │
             │     ├── YES ──▶ DynamoDB (design keys around patterns)
             │     │
             │     └── NO (ad-hoc queries needed)
             │           ──▶ RDS/Aurora (SQL handles any query pattern)
             │
             ├── Do you need single-digit ms latency at any scale?
             │     │
             │     ├── YES ──▶ DynamoDB (consistent latency, horizontal scaling)
             │     │
             │     └── NO ──▶ Either works (evaluate other factors)
             │
             ├── Is the workload serverless (Lambda-heavy)?
             │     │
             │     ├── YES ──▶ DynamoDB (no connections to manage,
             │     │           no RDS Proxy needed, no max_connections limit)
             │     │
             │     └── NO ──▶ Either works
             │
             ├── Do you need ACID transactions across many entities?
             │     │
             │     ├── COMPLEX (multi-table, hundreds of rows)
             │     │     ──▶ RDS/Aurora (SQL transactions, no 100-item limit)
             │     │
             │     └── SIMPLE (up to 100 items, under 4 MB)
             │           ──▶ DynamoDB transactions may suffice
             │
             └── Do you need zero operational overhead?
                   │
                   ├── YES ──▶ DynamoDB (no instances, no patching,
                   │           no storage management, no backups to configure)
                   │
                   └── NO ──▶ RDS/Aurora (managed but still has instances,
                               parameter groups, maintenance windows)
```

### Quick Comparison Table

| Feature | DynamoDB | RDS/Aurora |
|---|---|---|
| **Data model** | Key-value / document (schemaless) | Relational (schema-enforced) |
| **Query language** | API-based (GetItem, Query, Scan) | SQL (SELECT, JOIN, subqueries) |
| **Scaling** | Horizontal (automatic partitioning) | Vertical (bigger instance) + read replicas |
| **Latency** | Single-digit ms, consistent at any scale | Varies with query complexity and data size |
| **Max item/row size** | 400 KB per item | No practical per-row limit (constrained by page size) |
| **Transactions** | Up to 100 items, 2x cost, single region | Unlimited rows, native, cross-table JOINs |
| **JOINs** | Not supported (use single-table design) | Native SQL JOINs |
| **Connections** | HTTP API (no connection limit) | TCP connections (max_connections limit) |
| **Serverless** | Always serverless (no instances to manage) | Aurora Serverless v2 (still has instances, just auto-scaled) |
| **Encryption** | Always encrypted (mandatory) | Optional at creation (cannot add later) |
| **Multi-region writes** | Global Tables (active-active) | Aurora Global DB (single writer + write forwarding) |
| **Backup** | PITR (must enable), on-demand snapshots | Automated daily + PITR (Aurora: continuous) |
| **Caching** | DAX (native, API-compatible) | ElastiCache (external, manual integration) |
| **Cost model** | Per-request or per-capacity-unit | Per-instance-hour + storage + I/O |
| **Best for** | Known access patterns, high scale, serverless | Complex queries, ad-hoc analytics, relationships |

---

## Production Architecture: E-Commerce Platform with DynamoDB

```
PRODUCTION DYNAMODB ARCHITECTURE -- E-COMMERCE PLATFORM
════════════════════════════════════════════════════════════════════════════

  ┌──── us-east-1 (PRIMARY) ───────────────────────────────────────────┐
  │                                                                     │
  │  ┌─── Application Layer ─────────────────────────────────────────┐  │
  │  │                                                               │  │
  │  │  API Gateway ──▶ Lambda Functions ──▶ DAX Cluster ──▶ DynamoDB│  │
  │  │  (REST API)      (business logic)     (microsecond   (orders   │  │
  │  │                                        read cache)    table)   │  │
  │  │                                                               │  │
  │  │  CloudFront ──▶ ALB ──▶ ECS Services ──────────────▶ DynamoDB │  │
  │  │  (static +      (container   │                      (product   │  │
  │  │   dynamic)       routing)    │                       catalog)  │  │
  │  │                              │                                │  │
  │  │                    ┌─────────┴─────────┐                      │  │
  │  │                    │  Aurora PostgreSQL │                      │  │
  │  │                    │  (analytics, ad-hoc│                      │  │
  │  │                    │   queries, reports)│                      │  │
  │  │                    └───────────────────┘                      │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌─── DynamoDB Table: orders (single-table design) ──────────────┐  │
  │  │                                                                │  │
  │  │  PK: CUSTOMER#{id}     SK: PROFILE / ORDER#{date}#{id}       │  │
  │  │  Billing: On-Demand    Encryption: CMK                        │  │
  │  │  PITR: Enabled         Streams: NEW_AND_OLD_IMAGES            │  │
  │  │  GSI1: status-date     GSI2: product-date                     │  │
  │  │                                                                │  │
  │  │  Stream ──▶ Lambda ──▶ EventBridge ──▶ SNS (notifications)    │  │
  │  │                    │                                           │  │
  │  │                    └──▶ Kinesis Firehose ──▶ S3 ──▶ Athena    │  │
  │  │                         (stream to data lake for analytics)    │  │
  │  └────────────────────────────────────┬───────────────────────────┘  │
  │                                       │                              │
  │                          Global Tables replication                   │
  │                          (async, <1s, LWW conflict resolution)       │
  │                                       │                              │
  └───────────────────────────────────────┼──────────────────────────────┘
                                          │
                      ┌───────────────────┴───────────────────┐
                      ▼                                       ▼
  ┌──── eu-west-1 ──────────────────┐   ┌──── ap-northeast-1 ──────────┐
  │                                  │   │                               │
  │  Lambda ──▶ DAX ──▶ DynamoDB    │   │  Lambda ──▶ DAX ──▶ DynamoDB │
  │  (EU users: local reads+writes) │   │  (APAC users: local r+w)     │
  │                                  │   │                               │
  │  DAX cache is LOCAL (not synced) │   │  Streams ──▶ local Lambda    │
  │  Streams ──▶ local Lambda        │   │  (region-specific processing)│
  └──────────────────────────────────┘   └───────────────────────────────┘

  Architecture decisions:
  ─ DynamoDB for transactional data (orders, customers): known access
    patterns, single-digit ms latency, serverless, Global Tables
  ─ Aurora for analytics: ad-hoc SQL queries, JOINs, complex reports
  ─ DynamoDB Streams → Firehose → S3 → Athena: bridge DynamoDB to
    the analytics world without impacting transactional performance
  ─ DAX per region: microsecond reads for hot items (popular products)
  ─ Global Tables: local-latency reads AND writes in every region
```

---

## Key Takeaways

- **DynamoDB inverts the relational design process.** With RDS/Aurora, you normalize your schema first and query any way later using SQL. With DynamoDB, you identify your access patterns first and design your partition key, sort key, and GSIs to serve those exact patterns. If a new access pattern emerges after deployment, you must add a GSI or restructure data -- there is no ad-hoc query flexibility.

- **The partition key is the distribution mechanism -- bad key design causes throttling regardless of total capacity.** Each partition supports 3,000 RCU and 1,000 WCU. A hot partition key causes throttling even if your table has 100,000 WCU provisioned. High-cardinality, uniformly accessed partition keys are essential. This is fundamentally different from RDS/Aurora where the storage layer handles distribution transparently.

- **Default to GSI over LSI for almost everything.** GSIs can have different partition and sort keys, can be created anytime, have no size limit, and have separate throughput. LSIs must be defined at table creation, share base table throughput, and enforce a 10 GB per-partition-key limit. LSI's only advantage is supporting strongly consistent reads on the alternate sort key.

- **On-demand vs provisioned follows the same logic as Aurora Serverless v2 vs provisioned.** Unpredictable traffic, new tables, and peak-to-average ratios above 4x favor on-demand. Steady workloads with predictable traffic favor provisioned with auto-scaling (25-40% cheaper). Reserved capacity adds up to 75% savings for committed workloads.

- **DAX is a zero-code-change DynamoDB cache; ElastiCache is general-purpose.** DAX is a drop-in SDK replacement providing microsecond reads for cached items. It only works with DynamoDB and only caches eventually consistent reads. Use ElastiCache (Redis/Memcached) for caching from any data source with richer data structures and full control over caching logic.

- **DynamoDB Streams is the backbone of event-driven serverless architectures.** Four view types (KEYS_ONLY through NEW_AND_OLD_IMAGES), 24-hour retention, native Lambda integration with free polling reads, and guaranteed per-item ordering. There is no equivalent in RDS/Aurora for reacting to individual row-level data changes -- this is a defining advantage of DynamoDB for event-driven designs.

- **Global Tables provides true active-active multi-region writes with last-writer-wins conflict resolution.** Every replica reads and writes locally with sub-second replication. Compare with Aurora Global Database which has a single write region (secondaries can only forward writes at higher latency). MRSC mode (3 regions, strong consistency) was added in June 2025 for zero-data-loss requirements. Global Tables delivers 99.999% SLA versus Aurora Global Database's 99.99%.

- **Single-table design compensates for DynamoDB's lack of JOINs by pre-joining data through key design.** Storing multiple entity types in one table with composite keys (PK=CUSTOMER#123, SK=ORDER#2024-01-15) enables retrieving related items in a single Query. But it adds complexity and inflexibility -- use it for performance-critical, well-defined access patterns, not as a universal rule.

- **DynamoDB is always encrypted and always serverless.** Unlike RDS where encryption must be enabled at creation time (and requires snapshot migration if forgotten) and instances must be sized, DynamoDB encrypts all tables automatically and has no instances to manage. This simpler operational model is why DynamoDB is the default database choice for serverless (Lambda-based) architectures.

- **Query, don't Scan.** Query targets a single partition (efficient). Scan reads every item in the table (expensive). A Scan on a 100 GB table consumes RCU for all 100 GB even if only 10 items match your filter. If you find yourself needing a Scan for a recurring workload, add a GSI to turn it into a Query.

- **Bridge DynamoDB to the analytics world using Streams or Export to S3, not Scan.** For real-time incremental changes, use DynamoDB Streams → Firehose → S3 → Athena. For full-table exports, use the native Export to S3 feature (zero RCU impact, uses PITR data). Never run analytical queries directly on DynamoDB. Keep transactional and analytical workloads separated -- DynamoDB for OLTP, Aurora/Athena/Redshift for OLAP.

- **Use conditional writes to enforce business rules without JOINs or foreign keys.** ConditionExpression on writes is DynamoDB's substitute for relational constraints. The optimistic locking pattern (version attribute + condition check) prevents concurrent overwrites without explicit locks.

- **Batch operations are not transactions.** BatchWriteItem (25 items) and BatchGetItem (100 items) are cheaper than transactions (no 2x cost multiplier) but offer no atomicity -- individual items can fail independently. Use transactions when all-or-nothing is required; use batches for bulk operations where partial failure is acceptable.

---

## Further Reading

- [DynamoDB Core Components (official docs)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html) -- Primary keys, attributes, items, tables
- [Best Practices for Partition Key Design (official docs)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html) -- High-cardinality keys, write sharding, adaptive capacity
- [Secondary Indexes Best Practices (official docs)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-indexes-general.html) -- GSI vs LSI, projection strategies, sparse indexes
- [DAX Architecture (official docs)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.concepts.html) -- Item cache, query cache, write-through behavior
- [DynamoDB Streams (official docs)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html) -- StreamViewType, Lambda triggers, Kinesis integration
- [Global Tables (official docs)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html) -- MREC, MRSC, replica management
- [Single-Table vs Multi-Table Design (AWS blog)](https://aws.amazon.com/blogs/database/single-table-vs-multi-table-design-in-amazon-dynamodb/) -- When to use each approach
- [Alex DeBrie's DynamoDB Single-Table Design](https://www.alexdebrie.com/posts/dynamodb-single-table/) -- Deeper single-table patterns and trade-offs
