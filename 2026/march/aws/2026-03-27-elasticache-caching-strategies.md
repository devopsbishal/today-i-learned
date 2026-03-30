# ElastiCache & Caching Strategies -- The Deli Counter Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), trained the air traffic control tower to route planes to the right airport (Route53), deployed the global newspaper delivery network (CloudFront with OAC), built the restaurant host, highway toll booth, and customs checkpoint (ALB, NLB, GLB), established the global Anycast expressway (Global Accelerator), organized the city archive system for objects (S3 with storage classes, lifecycle, replication, and Object Lock), equipped your instances with personal workbenches, shared libraries, and specialty studios (EBS, EFS, FSx), built the managed hotel chain with its revolutionary shared vault system (RDS and Aurora), and organized the giant filing cabinet with drawer labels and folder tabs for predictable access patterns (DynamoDB with DAX). But there is a persistent problem cutting across all of these systems: **databases are fast, but memory is faster.** A well-tuned Aurora query takes 5-10 ms. A DynamoDB GetItem takes 1-5 ms. But a Redis GET takes **under 1 ms** -- often under 100 microseconds. When your application serves millions of requests per second and even a few milliseconds of latency per database call compound into seconds of page-load time, you need a layer that short-circuits the database entirely for frequently accessed data. That layer is the **in-memory cache**, and Amazon ElastiCache is the managed service that runs it. Think of it as a deli counter -- the kitchen (your database) can prepare any dish from scratch, but the deli counter keeps the day's most popular items pre-made behind the glass. Customers who want today's special get served instantly from the counter. Customers who want something unusual wait for the kitchen. The art of caching is deciding what goes on the counter, when to refresh it, and what to do when the counter runs out of space.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| ElastiCache | A managed deli counter in front of the kitchen -- pre-made popular items served instantly, unusual orders still go to the kitchen | Fully managed in-memory data store (Redis/Valkey or Memcached) that sits between your application and databases, reducing latency from milliseconds to microseconds |
| Redis (Valkey) Engine | A deli counter with labeled sections, special trays, priority boards, and a PA system -- it can organize items by category, rank, and even broadcast announcements | In-memory data store supporting rich data structures (strings, lists, sets, sorted sets, hashes, streams, pub/sub), persistence, replication, and clustering |
| Memcached Engine | A simple deli counter with numbered tickets -- you store items and retrieve them by ticket number, nothing fancy, but it serves multiple customers simultaneously with impressive speed | Multi-threaded, pure key-value cache with no persistence, no replication, no data structures -- the simplest and fastest option for basic caching |
| Lazy Loading (Cache-Aside) | The deli counter only stocks an item after someone asks for it -- the first customer waits, but every subsequent customer gets instant service | Application checks cache first; on miss, fetches from DB, writes result to cache; only requested data is cached; stale data possible until TTL expires |
| Write-Through | Every time the kitchen finishes a dish, a copy immediately goes to the deli counter -- the counter always has the freshest version, but it fills up with items nobody ordered | Application writes to both cache and database on every write; cache is always current; wastes memory on unread data |
| Write-Behind (Write-Back) | Customers place orders at the deli counter, and the counter sends batches to the kitchen periodically -- faster service, but if the counter crashes before flushing to the kitchen, orders are lost | **Not an AWS-documented strategy.** General caching pattern: application writes to cache first; background process flushes to DB asynchronously; lowest write latency but risks data loss |
| TTL (Time-to-Live) | A freshness sticker on each deli item -- "discard after 2 hours" -- ensuring stale food is automatically cleared even if nobody explicitly removes it | Expiration time set on cache keys; the universal safety net that guarantees stale data is eventually evicted regardless of other strategies |
| Cluster Mode Disabled | A single deli counter (one prep station + assistants who can serve but not prep) -- all items come from one station, assistants help distribute to customers | Single shard: one primary node (reads + writes) with up to 5 read replicas; all data fits on one node; scale reads, not writes |
| Cluster Mode Enabled | Multiple deli counters across the food hall, each responsible for a different section of the menu, each with its own prep station and assistants | Data partitioned across up to 500 shards (16,384 hash slots); each shard has a primary + up to 5 replicas; horizontal read AND write scaling |
| ElastiCache Serverless | A catering service that shows up with exactly the right number of trays and staff for your event, scales up for the dinner rush, and scales down for the afternoon lull -- you just pay per plate served | Fully managed, auto-scaling cache with no node/shard management; pricing based on storage (GB-hours) and compute (ECPUs); always encrypted |
| Global Datastore | Opening branch deli counters in other cities that mirror the main counter's menu -- customers in Paris get the same items as New York, kept in sync within a second | Cross-region replication for Redis/Valkey: one primary region (read-write) with up to 2 secondary regions (read-only); sub-second replication lag |
| Redis Sorted Sets | A leaderboard on the deli wall that automatically re-ranks contestants as scores change -- you add a score, and the board instantly shows the new top 10 | Members with associated scores, automatically sorted; enables O(log N) ranked queries like "top 10 players" or "events in the last hour" |
| Redis Streams | An order ticket rail in the kitchen -- tickets arrive in order, multiple cooks can pull tickets from where they left off, and the rail keeps history | Append-only log data structure with consumer groups; similar to Kafka topics but built into Redis; at-least-once delivery with consumer group tracking (true exactly-once requires application-level idempotency) |
| Eviction Policies | The deli manager's rules for clearing counter space when it is full -- remove the oldest items? The least popular? The ones closest to expiry? | Eight Redis policies (allkeys-lru, volatile-ttl, noeviction, etc.) that control which keys are removed when memory is full |

---

## The Big Picture: ElastiCache as a Deli Counter in Front of the Database Kitchen

Every database service you have learned -- Aurora (the hotel vault), DynamoDB (the filing cabinet), even S3 (the city archive) -- stores data durably on disk or across a storage network. Even the fastest disk-based reads take single-digit milliseconds because of network hops, storage layer operations, and query parsing. That is fast enough for most applications, but not for all.

Consider an e-commerce product page viewed 100,000 times per minute. If each view queries Aurora for product details, reviews, and pricing:
- 100,000 queries/min x 5 ms each = **500 seconds of cumulative database time per minute**
- The database handles it, but most queries return identical results -- the product data changes maybe once per hour

Now place a cache in front:
- First request: cache miss, query Aurora (5 ms), store in cache
- Next 99,999 requests: cache hit, return from memory (0.1 ms each)
- Database receives **1 query per hour** instead of **6 million**

This is the fundamental value proposition: **trade memory for latency and database load reduction.** ElastiCache is the managed service that provides this memory layer using two battle-tested open-source engines.

```
THE CACHING LAYER IN YOUR ARCHITECTURE
════════════════════════════════════════════════════════════════════════════

  Without cache:                          With ElastiCache:
  ──────────────                          ─────────────────

  Application                             Application
      │                                       │
      │  Every request                        │  Check cache first
      │  hits database                        ▼
      │                                  ┌──────────────┐
      │                                  │ ElastiCache   │  ◀── 0.1 ms
      │                                  │ (in-memory)   │      response
      │                                  └──────┬───────┘
      │                                    hit? │ miss?
      │                                    │    │
      ▼                                    │    ▼
  ┌──────────────┐                         │  ┌──────────────┐
  │  Database     │  ◀── 5-10 ms           │  │  Database     │  ◀── only on
  │  (Aurora,     │      every time        │  │  (Aurora,     │     cache miss
  │   DynamoDB)   │                        │  │   DynamoDB)   │
  └──────────────┘                         │  └──────┬───────┘
                                           │         │
                                           │   populate cache
                                           │         │
                                           ◀─────────┘
                                    return result to app

  Key metrics:
  ─ Cache hit ratio of 95% = database load reduced by 95%
  ─ Response time: 0.1 ms (cache) vs 5-10 ms (database)
  ─ The remaining 5% cache misses are actually slower (check cache + query DB + write cache)
    but the aggregate system performance is dramatically better
```

---

## Part 1: Redis vs Memcached -- The Swiss Army Knife vs the Single-Blade Knife

### The Analogy

**Redis (and its successor Valkey) is a Swiss Army knife.** It can slice (key-value), uncork (pub/sub messaging), file (sorted sets for leaderboards), saw (streams for event processing), and tweeze (geospatial queries). It has a built-in flashlight (persistence to disk), a compass (replication for HA), and a magnifying glass (Lua scripting). It is bigger and more complex, but it handles almost any task you throw at it.

**Memcached is a single-blade knife.** It does one thing -- key-value caching -- with maximum simplicity and multi-threaded speed. No persistence, no replication, no data structures. If the node dies, the cache is gone, and you start fresh. But for pure caching workloads where the database is the source of truth and the cache is entirely disposable, that simplicity is a virtue.

### The Valkey Context

Before diving into the comparison, an important context: in March 2024, Redis Ltd. changed the Redis license from BSD to a dual AGPLv3/SSPL model, restricting cloud providers from offering Redis as a managed service. In response, the Linux Foundation created **Valkey** -- a BSD-licensed fork of Redis 7.2.4 maintained by AWS, Google, Oracle, and others. AWS now defaults to Valkey for new ElastiCache clusters, and Valkey is **20-33% cheaper** on ElastiCache than Redis OSS. Valkey is API-compatible with Redis -- your application code, clients, and commands work unchanged. Throughout this document, "Redis" and "Valkey" are used interchangeably unless a distinction matters.

### Feature Comparison

| Feature | Redis / Valkey | Memcached |
|---------|---------------|-----------|
| **Data structures** | Strings, Lists, Sets, Sorted Sets, Hashes, Streams, Pub/Sub, Bitmaps, HyperLogLog, Geospatial | Strings only (key-value blobs) |
| **Threading** | Single-threaded command execution (I/O threads for network in Redis 6+) | Multi-threaded (leverages multiple CPU cores natively) |
| **Persistence** | RDB snapshots + AOF append-only file | None -- purely volatile |
| **Replication** | Primary-replica with automatic failover (Multi-AZ) | None -- no built-in replication |
| **Cluster mode** | Cluster Mode Enabled (up to 500 shards) | Auto-discovery with consistent hashing (up to 60 nodes) |
| **Transactions** | MULTI/EXEC atomic command batches | None |
| **Pub/Sub** | Built-in publish/subscribe messaging | None |
| **Lua scripting** | Server-side Lua scripts for atomic operations | None |
| **Snapshots/backups** | Automated daily backups, manual snapshots | None |
| **Encryption** | At rest + in transit (TLS) | At rest (1.6.12+) + in transit (TLS); Serverless always encrypts both |
| **Global Datastore** | Yes (cross-region replication) | No |
| **ElastiCache Serverless** | Yes | Yes |
| **Max item size** | 512 MB per value | 1 MB per value (default, configurable to 128 MB) |
| **Data tiering** | Yes (r6gd nodes: SSD + memory) | No |

### The Decision Rule

**Choose Memcached only when** ALL of these are true:
1. You need the simplest possible key-value caching
2. Your cache is purely disposable (losing it causes a temporary performance hit, not data loss)
3. You want multi-threaded performance on a single node
4. You do not need persistence, replication, pub/sub, sorted sets, or any advanced features

**Choose Redis/Valkey for everything else.** In practice, Redis/Valkey is the default choice for 90%+ of ElastiCache deployments because the feature set is dramatically richer at comparable performance, and the operational safety of replication and persistence makes it the pragmatic choice even for simple caching.

---

## Part 2: Caching Strategies -- The Three AWS Approaches to Stocking the Deli Counter

Caching strategies determine **how** data flows between your application, the cache, and the database. This is not an ElastiCache-specific concept -- it applies to any caching layer -- but it is the intellectual core of using ElastiCache effectively.

### Strategy 1: Lazy Loading (Cache-Aside)

**The analogy**: The deli counter starts empty each morning. When the first customer asks for a turkey sandwich, there is none on the counter. The staff goes to the kitchen (database), makes the sandwich, puts a copy on the counter, and serves the customer. When the second customer asks for a turkey sandwich, it is already on the counter -- served instantly. The rye bread sandwich nobody asks for is never made and never wastes counter space.

**How it works:**

```python
def get_product(product_id):
    # Step 1: Check the deli counter (cache)
    cached = redis.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)  # Cache HIT -- sub-millisecond

    # Step 2: Cache MISS -- go to the kitchen (database)
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)

    # Step 3: Stock the counter for next time, with a freshness sticker (TTL)
    redis.setex(
        f"product:{product_id}",
        3600,  # TTL: 1 hour freshness sticker
        json.dumps(product)
    )

    return product
```

**Strengths:**
- **Memory-efficient**: Only requested data enters the cache -- no wasted memory on unread items
- **Resilient to cache failure**: If ElastiCache goes down, the application falls back to the database (slower but functional)
- **Simple to implement**: The application owns the caching logic

**Weaknesses:**
- **Cache miss penalty**: First request is slower (three trips: check cache, query DB, write cache) vs one trip without caching
- **Stale data**: If the product price changes in the database, the cache still serves the old price until the TTL expires or the key is explicitly invalidated
- **Thundering herd**: When a popular key expires, hundreds of concurrent requests simultaneously miss the cache and hit the database

### Strategy 2: Write-Through

**The analogy**: Every time the kitchen finishes a new dish or updates a recipe, a copy immediately goes to the deli counter. The counter always shows the freshest version of every item the kitchen has produced. The downside: the counter fills up with items nobody has ordered -- the kitchen made 50 different dishes today, but customers only asked for 12 of them.

**How it works:**

```python
def update_product(product_id, new_data):
    # Step 1: Update the kitchen (database)
    db.execute("UPDATE products SET ... WHERE id = %s", product_id, new_data)

    # Step 2: Immediately update the counter (cache)
    redis.setex(
        f"product:{product_id}",
        3600,  # Still use TTL as a safety net
        json.dumps(new_data)
    )
```

**Strengths:**
- **Cache is always fresh**: Data in the cache matches the database immediately after every write
- **No stale-data window**: Eliminates the lazy loading gap between a database write and the next cache miss

**Weaknesses:**
- **Write penalty**: Every write does two operations (DB + cache), adding latency to the write path
- **Wasted memory**: Data that is written but never read still occupies cache space
- **Useless without reads**: If the cache key is never read between writes, the write-through work is pure waste

### The Recommended Combination

In production, the strategies are **complementary, not competing**:

```
THE PRODUCTION CACHING STRATEGY STACK
════════════════════════════════════════════════════════════════════════════

  1. LAZY LOADING as the universal baseline
     ─ Every read path uses cache-aside
     ─ Handles cache misses, node failures, cold starts gracefully

  2. WRITE-THROUGH added selectively for hot data
     ─ Shopping cart updates: write-through (must be fresh immediately)
     ─ User profile changes: write-through (user expects to see changes)
     ─ Product catalog: lazy loading only (changes infrequently)

  3. TTL on EVERYTHING as the safety net
     ─ Write-through data: short TTL (5-15 min) -- data is refreshed by
       writes anyway, TTL catches edge cases where write-through fails
     ─ Lazy-loaded data: longer TTL (1-24 hours) depending on staleness
       tolerance
     ─ Session data: TTL = session timeout (30 min)

  ┌─────────────────────────────────────────────────────────────────┐
  │                     Application Logic                           │
  │                                                                 │
  │  READ PATH:                    WRITE PATH:                      │
  │  ┌──────────┐                  ┌──────────────────┐             │
  │  │ Check    │──hit──▶ return   │ Write to DB      │             │
  │  │ cache    │                  │                  │             │
  │  │          │──miss─┐          │ Is this hot data? │             │
  │  └──────────┘       │          │ YES: also write   │             │
  │                     ▼          │      to cache     │             │
  │              ┌──────────┐      │ NO:  let lazy     │             │
  │              │ Query DB │      │      loading      │             │
  │              │          │      │      handle it    │             │
  │              └────┬─────┘      └──────────────────┘             │
  │                   │                                             │
  │              Write to cache                                     │
  │              with TTL                                           │
  │                   │                                             │
  │              Return result                                      │
  └─────────────────────────────────────────────────────────────────┘

  This is exactly what DAX does natively for DynamoDB:
  ─ Item cache = write-through (fresh after PutItem/UpdateItem)
  ─ Query cache = lazy loading (populated on Query, NOT invalidated on writes)
  ─ Both caches = TTL-based expiration (default 5 minutes)

  With ElastiCache, YOU implement this logic. With DAX, it is built in.
  That is DAX's zero-code-change advantage -- and ElastiCache's flexibility advantage.
```

### Cache Invalidation: The Hardest Problem in Computer Science

Phil Karlton's famous quote -- "There are only two hard things in Computer Science: cache invalidation and naming things" -- exists because invalidation is genuinely difficult. The fundamental tension: you want fresh data (invalidate aggressively) but also want high cache hit rates (invalidate conservatively). Strategies for managing this:

**TTL-based expiration**: The simplest approach. Set a TTL and accept that data may be stale for up to that duration. Works for data where eventual consistency is acceptable (product listings, blog posts, weather data).

**Explicit invalidation**: On every write, explicitly delete or update the cache key. Works for data that must be immediately consistent (shopping carts, user settings). The gotcha: if the database write succeeds but the cache invalidation fails (network glitch), the cache serves stale data indefinitely until TTL catches it -- this is why you always set TTL even with explicit invalidation.

**Event-driven invalidation**: Use DynamoDB Streams or Aurora event notifications to trigger Lambda functions that invalidate cache keys. Decouples the write path from cache management but adds complexity and a small delay.

**The thundering herd problem**: When a popular cache key expires and hundreds of concurrent requests simultaneously hit the database.

```
THE THUNDERING HERD
════════════════════════════════════════════════════════════════════════════

  Time 0: Cache key "hot-product:42" exists, TTL = 3600s
          100 requests/sec served from cache ──▶ 0 DB queries

  Time 3600: TTL expires, key is evicted

  Time 3600.001: 100 concurrent requests arrive, ALL miss cache
                 ALL 100 send identical query to database
                 Database receives 100x spike for identical data

  ┌──────────┐     100 simultaneous     ┌──────────┐
  │  App      │ ──────── misses ───────▶ │  Database │  ◀── 100x spike!
  │ (100 req) │     cache empty          │           │
  └──────────┘                           └──────────┘

  Mitigations:
  1. Cache-lock (mutex): First request acquires a lock, fetches from DB,
     populates cache. Other 99 requests wait or serve stale data.
  2. Pre-warming: Proactively refresh keys before TTL expires
     (e.g., at 80% of TTL, a background job refreshes the key)
  3. Staggered TTLs: Add random jitter to TTL values so keys do not
     all expire simultaneously: TTL = base_ttl + random(0, 300)
  4. "Never expire" + background refresh: No TTL on the key; a
     separate worker periodically updates it from the database
```

### Beyond AWS: Write-Behind (Write-Back)

> **Note:** Write-Behind is **not covered in the AWS ElastiCache documentation** and is **not a native ElastiCache feature**. The three AWS-documented strategies are Lazy Loading, Write-Through, and Adding TTL. Write-Behind is included here as a general caching concept you may encounter in system design interviews.

**The analogy**: Customers place orders at the deli counter itself. The counter staff serves the customer immediately and adds the order to a batch sheet. Every 5 minutes, the batch sheet is sent to the kitchen for permanent preparation. Instant service -- but if the counter catches fire before the batch sheet reaches the kitchen, those orders are lost forever.

**How it works:** The application writes to the cache first. A background process asynchronously flushes writes to the database in batches. You must implement this yourself with application logic (e.g., Redis Streams + a worker).

**Strengths:** Lowest write latency (microseconds vs milliseconds); batch efficiency reduces DB load.

**Weaknesses:** Risk of data loss if the cache node fails before flushing; requires a reliable background worker; database is temporarily behind the cache.

**When it comes up:** System design interviews discussing write-heavy workloads (e.g., IoT telemetry ingestion, analytics event pipelines) where you can tolerate potential data loss in exchange for write speed.

---

## Part 3: Cluster Mode Disabled vs Enabled -- One Counter vs a Food Hall

This is the most important architectural decision after engine selection. It determines how your data is distributed, how you scale, and what features are available.

### Cluster Mode Disabled (Single Shard)

**The analogy**: One deli counter with a single prep station and up to five serving assistants. The prep chef (primary node) makes all the food and manages the menu. The assistants (read replicas) can serve pre-made items to customers but cannot prep new items or change the menu. If the prep chef calls in sick, one assistant is promoted to prep chef (automatic failover). All items must fit on this one counter -- if the counter is not big enough, you need a bigger counter (vertical scaling).

**Technical reality:**
- **One shard**: One primary node for reads and writes, up to 5 read replicas
- **All data on one node**: Maximum data size = node's available memory (e.g., cache.r7g.16xlarge = ~400 GB usable)
- **Read scaling**: Add replicas to distribute read traffic (up to 5)
- **No write scaling**: All writes go to the single primary
- **Multi-AZ**: Automatic failover to a replica in a different AZ
- **Simpler operations**: No hash slot management, no cross-shard operations

```
CLUSTER MODE DISABLED (SINGLE SHARD)
════════════════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────────────┐
  │  Single Shard (Node Group)                                       │
  │                                                                  │
  │  ┌──────────────┐    async repl    ┌──────────────┐              │
  │  │ Primary Node  │ ──────────────▶ │ Replica 1     │              │
  │  │ (read+write)  │                 │ (read-only)   │   AZ-b      │
  │  │               │                 └──────────────┘              │
  │  │  cache.r7g.   │    async repl    ┌──────────────┐              │
  │  │  xlarge       │ ──────────────▶ │ Replica 2     │              │
  │  │               │                 │ (read-only)   │   AZ-c      │
  │  │  AZ-a         │                 └──────────────┘              │
  │  └──────────────┘                                                │
  │                                                                  │
  │  All data resides on the primary (replicated to replicas).       │
  │  Max data = single node's memory.                                │
  │  Reads: scale horizontally with replicas.                        │
  │  Writes: limited to primary node's capacity.                     │
  └──────────────────────────────────────────────────────────────────┘

  Parallel: This is like Aurora with Cluster Mode Disabled --
  one writer, multiple readers sharing the same data.
  (Except Aurora readers share storage; Redis replicas hold full copies.)
```

### Cluster Mode Enabled (Multiple Shards)

**The analogy**: A food hall with multiple deli counters, each responsible for a section of the menu. Counter A handles sandwiches (hash slots 0-5460), Counter B handles salads (hash slots 5461-10922), Counter C handles hot dishes (hash slots 10923-16383). Each counter has its own prep chef and assistants. When a customer orders a sandwich, they go to Counter A. When they order a salad, Counter B. The food hall can handle vastly more customers than a single counter because the work is distributed. Need to serve more menu items? Add more counters (resharding). Need to move "wraps" from Counter A to Counter B because Counter A is overwhelmed? Online resharding handles that without closing the food hall.

**Technical reality:**
- **Multiple shards**: Up to 500 shards, each with its own primary + up to 5 replicas
- **16,384 hash slots**: Redis hashes each key to one of 16,384 slots, distributed across shards
- **Horizontal write scaling**: Each shard has its own primary, so total write capacity = sum of all shard primaries
- **Larger datasets**: Total memory = sum of all shard memories (e.g., 500 shards x 400 GB = 200 TB theoretical max)
- **Online resharding**: Add/remove shards and rebalance hash slots without downtime
- **Required for Global Datastore**: You cannot use Global Datastore with Cluster Mode Disabled

```
CLUSTER MODE ENABLED (MULTIPLE SHARDS)
════════════════════════════════════════════════════════════════════════════

  ┌──── Shard 1 (slots 0-5460) ────┐  ┌──── Shard 2 (slots 5461-10922) ─┐
  │                                 │  │                                   │
  │  ┌──────────┐   ┌──────────┐   │  │  ┌──────────┐   ┌──────────┐     │
  │  │ Primary   │──▶│ Replica  │   │  │  │ Primary   │──▶│ Replica  │     │
  │  │ (R+W)    │   │ (R only) │   │  │  │ (R+W)    │   │ (R only) │     │
  │  │  AZ-a    │   │  AZ-b    │   │  │  │  AZ-b    │   │  AZ-c    │     │
  │  └──────────┘   └──────────┘   │  │  └──────────┘   └──────────┘     │
  └─────────────────────────────────┘  └───────────────────────────────────┘

  ┌──── Shard 3 (slots 10923-16383) ┐
  │                                   │
  │  ┌──────────┐   ┌──────────┐     │
  │  │ Primary   │──▶│ Replica  │     │
  │  │ (R+W)    │   │ (R only) │     │
  │  │  AZ-c    │   │  AZ-a    │     │
  │  └──────────┘   └──────────┘     │
  └───────────────────────────────────┘

  Key routing:
  ─ Application sends: SET product:42 "data"
  ─ Redis hashes "product:42" → slot 9783 → Shard 2's primary
  ─ Application sends: GET user:session:abc → slot 1204 → Shard 1's primary/replica

  Parallel to DynamoDB:
  ─ Hash slots ≈ DynamoDB partitions
  ─ Key hashing to determine shard ≈ Partition key hashing
  ─ Hot slot problem ≈ Hot partition problem
  ─ Online resharding ≈ DynamoDB adaptive capacity + partition splitting
```

### The Decision Rule

| Criteria | Cluster Mode Disabled | Cluster Mode Enabled |
|----------|----------------------|---------------------|
| **Dataset size** | Fits in single node memory (up to ~400 GB) | Exceeds single node memory |
| **Write pattern** | Write-light, read-heavy | Write-heavy, needs horizontal write scaling |
| **Operations** | Simpler, no shard management | More complex, but supports online resharding |
| **Global Datastore** | Not supported | Required |
| **Multi-key operations** | All keys on same shard (no restrictions) | Keys must be on same shard -- use **hash tags**: Redis hashes only the portion inside `{}` to determine the slot, so `{user:123}:cart` and `{user:123}:profile` both hash on `user:123` and land on the same shard, enabling MGET/transactions/Lua across them |
| **Typical use case** | Session store, small-medium caches | Large-scale caching, leaderboards, high-write workloads |

---

## Part 4: ElastiCache Serverless -- The Catering Service

### The Analogy

Running a node-based ElastiCache cluster is like owning a deli with a fixed counter size. You choose the counter dimensions (node type), the number of counters (shards), and the number of staff (replicas). If you overestimate, you pay for empty counter space. If you underestimate, customers queue up and some leave (throttling). Resizing means closing the deli for renovations.

**ElastiCache Serverless is a catering service.** You tell it what you need served, and it shows up with exactly the right equipment. Dinner rush? More trays and staff appear automatically. Quiet afternoon? Equipment is quietly packed away. You pay per plate served (ECPU) and per shelf of food stored (GB-hour), with no fixed infrastructure commitment.

### How It Works

- **Single endpoint**: No choosing node types, shard counts, or replica counts -- one DNS endpoint handles everything
- **Auto-scaling**: Scales compute and memory independently based on workload, with no downtime
- **Pricing model**:
  - **Storage**: Billed per GB-hour. Minimum billing: 1 GB for Redis OSS and Memcached engines; 100 MB for Valkey engine
  - **Compute**: Measured in **ElastiCache Processing Units (ECPUs)** -- 1 ECPU = one simple GET/SET of up to 1 KB; complex data structure operations consume proportionally more ECPUs
- **Mandatory encryption**: Always encrypted at rest and in transit (not optional like node-based)
- **Supported engines**: Valkey, Redis OSS, and Memcached

### The Familiar Pattern

ElastiCache Serverless follows the exact same pattern you have seen twice already:

```
THE SERVERLESS vs PROVISIONED PATTERN ACROSS AWS
════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────┬──────────────────────┬──────────────────────┐
  │ Service                │ Serverless Mode       │ Provisioned Mode     │
  ├────────────────────────┼──────────────────────┼──────────────────────┤
  │ DynamoDB               │ On-Demand             │ Provisioned RCU/WCU  │
  │  (Mar 26 deep dive)    │ (pay per request)     │ (auto-scaling)       │
  │                        │                       │ + Reserved Capacity  │
  ├────────────────────────┼──────────────────────┼──────────────────────┤
  │ Aurora                 │ Serverless v2         │ Provisioned instances│
  │  (Mar 25 deep dive)    │ (0-256 ACU)           │ (db.r7g.xlarge etc.) │
  │                        │ (scale to zero)       │ + Reserved Instances │
  ├────────────────────────┼──────────────────────┼──────────────────────┤
  │ ElastiCache            │ Serverless            │ Node-based clusters  │
  │  (today)               │ (ECPUs + GB-hours)    │ (cache.r7g.xlarge)   │
  │                        │                       │ + Reserved Nodes     │
  └────────────────────────┴──────────────────────┴──────────────────────┘

  Universal trade-off:
  ─ Serverless: Zero capacity planning, instant scaling, higher per-unit cost
  ─ Provisioned: Requires sizing, manual/auto scaling, lower per-unit cost
  ─ Rule of thumb: Start serverless for new/variable workloads;
    migrate to provisioned + reserved for steady, predictable workloads
```

### When to Use Serverless vs Node-Based

| Scenario | Recommendation | Reason |
|----------|---------------|--------|
| New application, unknown traffic | Serverless | No capacity planning needed |
| Dev/test environments | Serverless | Scales to near-zero during idle |
| Highly variable traffic (10x spikes) | Serverless | Instant auto-scaling |
| Steady-state production, predictable load | Node-based + Reserved Nodes | Up to 55% savings with reserved pricing |
| Need Global Datastore | Node-based | Global Datastore not supported on Serverless (as of March 2026) |
| Need precise control over topology | Node-based | Choose exact node types, shard layout, replica placement |

---

## Part 5: Global Datastore -- Opening Branch Counters in Other Cities

### The Analogy

Your New York deli is a massive success, but European customers suffer 150 ms round-trip latency ordering from across the Atlantic. You open branch deli counters in Paris and Tokyo. The New York counter (primary) is the only one that can create new recipes (writes), but every new recipe is replicated to Paris and Tokyo within a second. Customers in Paris read from their local counter at local-latency speed. If New York has a catastrophic event, you can promote the Paris counter to become the new primary in under a minute.

### Technical Reality

- **One primary region** (read-write) + **up to 2 secondary regions** (read-only)
- **Sub-second replication lag** (typically)
- **Failover promotion**: Secondary promoted to primary in under a minute
- **Requires Cluster Mode Enabled** -- cannot be used with Cluster Mode Disabled
- **Node-based only** -- not available with ElastiCache Serverless (as of March 2026)

### Cross-Region Replication Comparison

You now know three cross-region replication patterns. Here is how they compare:

```
THREE CROSS-REGION REPLICATION MODELS
════════════════════════════════════════════════════════════════════════════

  ┌────────────────────┬──────────────────┬──────────────────┬──────────────────┐
  │                    │ Aurora Global     │ DynamoDB Global  │ ElastiCache      │
  │                    │ Database          │ Tables           │ Global Datastore │
  ├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
  │ Write model        │ Single writer    │ Active-active    │ Single writer    │
  │                    │ (+ write fwd)    │ (every region)   │                  │
  ├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
  │ Secondary regions  │ Up to 5          │ No published     │ Up to 2          │
  │                    │                  │ limit (all       │                  │
  │                    │                  │ available regions│                  │
  │                    │                  │ eligible)        │                  │
  ├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
  │ Replication        │ Storage-level    │ DynamoDB Streams  │ Async Redis      │
  │ mechanism          │ (sub-1s lag)     │ (sub-1s lag)     │ replication      │
  │                    │                  │                  │ (sub-1s lag)     │
  ├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
  │ Conflict           │ N/A (single      │ Last-writer-wins │ N/A (single      │
  │ resolution         │ writer)          │ (MREC) or strong │ writer)          │
  │                    │                  │ consistency      │                  │
  │                    │                  │ (MRSC, 3 regions)│                  │
  ├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
  │ Planned failover   │ Managed          │ N/A (all active) │ Manual promotion │
  │                    │ switchover       │                  │ (under 1 min)    │
  │                    │ (zero data loss) │                  │                  │
  ├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
  │ SLA                │ 99.99%           │ 99.999%          │ 99.99%           │
  ├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
  │ Data type          │ Relational       │ Key-value/doc    │ In-memory cache  │
  └────────────────────┴──────────────────┴──────────────────┴──────────────────┘

  Pattern: Aurora and ElastiCache follow the "single writer with passive
  secondaries" model. DynamoDB is the only AWS service offering true
  active-active multi-region writes with automatic conflict resolution.

  In a multi-region system design, you would typically use ALL THREE:
  ─ Aurora Global Database for relational data (analytics, complex queries)
  ─ DynamoDB Global Tables for key-value access patterns (orders, sessions)
  ─ ElastiCache Global Datastore for the caching layer (session cache, hot data)
```

---

## Part 6: Redis Data Structures Beyond Key-Value -- Why Redis Wins

This is what separates "I know ElastiCache is a cache" from "I can design a leaderboard, a rate limiter, and a session store using ElastiCache." Redis is not just a key-value store -- it is a data structure server. Each structure has use cases that go far beyond caching.

### Sorted Sets -- The Leaderboard

**The analogy**: A scoreboard on the deli wall that automatically re-ranks entries as scores change. You add a player with a score, and the board instantly shows where they rank. You can ask "top 10 players" or "all players with scores between 100 and 200" in O(log N) time.

```
# Add scores to a gaming leaderboard
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2200 "player:bob"
ZADD leaderboard 1800 "player:carol"

# Top 3 players (highest first) -- using modern ZRANGE syntax (Redis 6.2+)
ZRANGE leaderboard 0 2 REV WITHSCORES
# Returns: bob (2200), carol (1800), alice (1500)
# Note: ZREVRANGE is deprecated since Redis 6.2; use ZRANGE ... REV instead

# Alice's rank (0-indexed, highest = rank 0)
ZREVRANK leaderboard "player:alice"
# Returns: 2

# Players with scores between 1600 and 2000 -- using modern ZRANGE syntax
ZRANGE leaderboard 1600 2000 BYSCORE WITHSCORES
# Returns: carol (1800)
# Note: ZRANGEBYSCORE is deprecated since Redis 6.2; use ZRANGE ... BYSCORE instead
```

**Use cases**: Gaming leaderboards, trending products, priority queues, sliding-window rate limiters (score = timestamp).

### Streams -- The Event Log

**The analogy**: An order ticket rail in the kitchen. New tickets are pinned to the end of the rail. Multiple cook stations (consumer groups) can read from the rail independently, each tracking where they left off. If a cook crashes mid-order, the ticket is reassigned to another cook. The rail keeps history, so you can replay past tickets.

```
# Producer: add an event to the order stream
XADD orders * customer "alice" item "sandwich" total "12.50"
# Returns: 1680000000001-0 (auto-generated ID based on timestamp)

# Consumer group: create a group to process orders
XGROUP CREATE orders kitchen-processors $ MKSTREAM

# Consumer: read next unprocessed event
XREADGROUP GROUP kitchen-processors cook-1 COUNT 1 STREAMS orders >
# Returns the next unprocessed order for cook-1

# Acknowledge processing complete
XACK orders kitchen-processors 1680000000001-0
```

**Use cases**: Event sourcing, activity feeds, IoT telemetry ingestion, chat messaging, audit logs. A lighter-weight alternative to Kafka when you do not need Kafka's scale or persistence guarantees.

### Pub/Sub -- The PA System

Fire-and-forget messaging. A publisher broadcasts to a channel, and all current subscribers receive the message. No persistence -- if nobody is listening, the message is lost. Unlike Streams, there is no replay or consumer groups.

**Use cases**: Real-time notifications, chat rooms, live sports scores, cache invalidation broadcasts across application servers.

### Hashes -- Object Storage Without Serialization

```
# Store a user profile as a hash (no JSON serialization needed)
HSET user:123 name "Alice" email "alice@example.com" plan "premium"

# Read a single field (no need to deserialize the entire object)
HGET user:123 plan
# Returns: "premium"

# Increment a numeric field atomically
HINCRBY user:123 login_count 1
```

**Use cases**: User profiles, session data, object caching where you need to read/update individual fields without fetching the entire blob.

### Geospatial Indexes

```
# Add store locations
GEOADD stores -73.985428 40.748817 "store:nyc-midtown"
GEOADD stores -118.243685 34.052234 "store:la-downtown"

# Find stores within 10 km of a point
GEOSEARCH stores FROMLONLAT -73.990000 40.750000 BYRADIUS 10 km
```

**Use cases**: Nearest-store finders, ride-sharing driver matching, location-based recommendations.

---

## Part 7: ElastiCache vs DAX -- The Decision Framework

You already learned DAX in depth yesterday (the reception desk cache for the filing cabinet). Here is the concise comparison for choosing between them:

| Dimension | DAX | ElastiCache (Redis/Valkey) |
|-----------|-----|---------------------------|
| **What it caches** | DynamoDB reads only | Anything: RDS, Aurora, DynamoDB, APIs, computed results |
| **Code change required** | Swap SDK client, zero query logic changes | Application implements caching logic (cache-aside, write-through, etc.) |
| **Protocol** | DynamoDB API | Redis protocol |
| **Caching strategy** | Built-in write-through (item cache) + lazy loading (query cache) + TTL | You choose and implement: lazy loading, write-through, write-behind, TTL |
| **Consistency** | Eventually consistent reads only (strong consistent reads pass through) | Application-managed |
| **Data structures** | Items and query results | Strings, lists, sets, sorted sets, hashes, streams, pub/sub, geo |
| **Serverless option** | No (node-based only) | Yes (ElastiCache Serverless) |
| **Cross-region** | No (per-region only) | Yes (Global Datastore) |
| **Best for** | DynamoDB read acceleration with zero effort | General-purpose caching, sessions, leaderboards, pub/sub, multi-source caching |

**The three scenarios where you use ElastiCache even with DynamoDB:**
1. **Aggregated results**: DAX caches individual queries; if you compute aggregations from multiple DynamoDB queries, cache the aggregated result in ElastiCache
2. **Cross-service caching**: The same cache serves data from both DynamoDB and Aurora in a hybrid architecture (like the e-commerce pattern from your Mar 26 deep dive)
3. **Rich data structures**: You need sorted sets for leaderboards, pub/sub for real-time notifications, or streams for event processing -- capabilities DAX does not offer

---

## Part 8: Security -- Locking Down the Deli Counter

ElastiCache's security model follows the same patterns you learned with RDS/Aurora, with a critical "at creation time" constraint.

### Four Security Layers

**1. Network isolation (VPC + Security Groups)**
- ElastiCache clusters deploy inside your VPC, in private subnets
- Security groups control which IP ranges/security groups can reach port 6379 (Redis) or 11211 (Memcached)
- No public accessibility by default (unlike RDS, which has a public accessibility toggle -- ElastiCache does not even offer this)

**2. Encryption in transit (TLS)**
- TLS 1.2+ between clients and cache nodes, and between nodes
- **Mandatory for Serverless**, optional for node-based
- **Can be added after creation on Redis 7.0+/Valkey 7.2+** via a two-phase migration: set TLS mode to `preferred` (accepts both TLS and plaintext), migrate all clients to TLS, then set to `required`. Earlier engine versions require creating a new cluster.

**3. Encryption at rest**
- AES-256 for on-disk data during sync and backup operations
- **Mandatory for Serverless**, optional for node-based
- **Cannot be added after creation** on node-based clusters (same immutable constraint as RDS -- must create a new cluster and migrate)

**4. Authentication and authorization**

| Method | Requirements | How It Works |
|--------|-------------|-------------|
| **Redis AUTH token** | Redis 3.2+ | Shared password string; clients must send AUTH command on connect |
| **IAM authentication** | Redis 7.0+ / Valkey 7.2+ | Short-lived 15-minute IAM tokens; no password management |
| **RBAC (Role-Based Access Control)** | Redis 6.0+ | Per-user access strings defining allowed commands and key patterns |

RBAC example: Allow a read-only user to GET keys matching `session:*` but deny all write commands:
```
# User access string (order matters -- rules are applied left-to-right)
on ~session:* -@all +get +mget
# "on" = user is active
# "~session:*" = can access keys matching this pattern
# "-@all" = deny all commands first (baseline)
# "+get +mget" = then allow specific read commands
# IMPORTANT: If you put -@all LAST, it overrides the earlier +get +mget and denies everything
```

### The Security Configuration Pattern

```
ELASTICACHE SECURITY -- SAME PATTERN AS RDS
════════════════════════════════════════════════════════════════════════════

  ┌───────────────────────────────────────────────────────────────┐
  │ VPC (network boundary)                                        │
  │                                                               │
  │  ┌─── Private Subnets ──────────────────────────────────────┐ │
  │  │                                                           │ │
  │  │  ┌─── ElastiCache Subnet Group ──────────────────────┐   │ │
  │  │  │                                                    │   │ │
  │  │  │  ┌──────────────┐  ┌──────────────┐               │   │ │
  │  │  │  │ Primary Node  │  │ Replica Node  │               │   │ │
  │  │  │  │ (AZ-a)       │  │ (AZ-b)       │               │   │ │
  │  │  │  └──────┬───────┘  └──────────────┘               │   │ │
  │  │  │         │                                          │   │ │
  │  │  │    Security Group: allow port 6379                 │   │ │
  │  │  │    from app-server SG only                         │   │ │
  │  │  │                                                    │   │ │
  │  │  │    Encryption: at rest (AES-256) + in transit (TLS)│   │ │
  │  │  │    Auth: IAM authentication or RBAC                │   │ │
  │  │  └────────────────────────────────────────────────────┘   │ │
  │  │                                                           │ │
  │  └───────────────────────────────────────────────────────────┘ │
  │                                                               │
  │  Key constraint: At-rest encryption CANNOT be enabled after    │
  │  creation (same as RDS). In-transit encryption CAN be added   │
  │  after creation on Redis 7+/Valkey via two-phase TLS migration│
  └───────────────────────────────────────────────────────────────┘
```

---

## Part 9: Eviction Policies -- What to Remove When the Counter Is Full

When ElastiCache runs out of memory, the eviction policy determines which keys are removed to make room. Choosing the wrong policy can cause subtle, hard-to-debug issues.

### The Eight Redis Eviction Policies

| Policy | Scope | Strategy | Best For |
|--------|-------|----------|----------|
| **allkeys-lru** | All keys | Evict least recently used | General-purpose caching (recommended default) |
| **allkeys-lfu** | All keys | Evict least frequently used | Workloads with frequency-based access patterns |
| **allkeys-random** | All keys | Evict random keys | When all keys have equal access probability |
| **volatile-lru** | Keys with TTL | Evict least recently used with TTL set | Mixed use: cache (with TTL) + persistent data (no TTL) |
| **volatile-lfu** | Keys with TTL | Evict least frequently used with TTL set | Same as above but frequency-based |
| **volatile-random** | Keys with TTL | Evict random keys with TTL set | Random selection among expirable keys |
| **volatile-ttl** | Keys with TTL | Evict keys closest to expiration | When shortest-lived data is least valuable |
| **noeviction** | None | Return error on write when full | When data loss is unacceptable (use as a database, not cache) |

**Critical gotcha with volatile-* policies**: If all your keys have no TTL set, volatile-* policies behave like **noeviction** -- they have no eligible keys to evict, so writes fail when memory is full. This is the default trap: the Redis default policy is `volatile-lru`, which means keys without a TTL are never evicted. If you forget to set TTLs on your cache keys, the cache fills up and starts returning errors.

**Recommendation**: Use `allkeys-lru` as the default for pure caching workloads. Use `volatile-lru` only if you have a mix of expirable cache data (with TTL) and persistent data (without TTL) that must never be evicted.

---

## Part 10: Monitoring, Scaling, and Cost Optimization

### Key Metrics to Watch

| Metric | Warning Threshold | Action |
|--------|-------------------|--------|
| **CPUUtilization** | >90% sustained | Scale up node type or add shards |
| **EngineCPUUtilization** | >90% (Redis-specific) | Redis is single-threaded; this saturates before CPUUtilization |
| **DatabaseMemoryUsagePercentage** | >80% | Risk of evictions; scale up memory or add shards |
| **CacheHitRate** | <80% | Review TTLs, pre-warm cache, check key design |
| **Evictions** | Rising trend | Memory pressure; scale up or review TTL strategy |
| **CurrConnections** | Near max-clients | Connection pooling needed; consider increasing max-clients |
| **ReplicationLag** | >1s sustained | Replica overloaded; scale up replica or reduce write volume |
| **SwapUsage** | >0 | Instance memory exhausted; immediate scale-up needed |

### Scaling Patterns

**Vertical scaling (bigger nodes)**: Change node type (e.g., cache.r7g.large to cache.r7g.xlarge). Requires a brief disruption for node-based clusters (Multi-AZ failover minimizes downtime).

**Horizontal read scaling**: Add read replicas (Cluster Mode Disabled: up to 5; Cluster Mode Enabled: up to 5 per shard).

**Horizontal write scaling**: Add shards (Cluster Mode Enabled only). Online resharding redistributes hash slots without downtime.

**Data tiering (r6gd nodes)**: Extends memory with SSD storage. Infrequently accessed data moves to SSD automatically; frequently accessed data stays in memory. Up to 5x more data at 60% lower cost versus pure-memory nodes. The data-tiering model: Redis moves least-recently-used values (not keys -- keys always stay in memory) to SSD, fetching them back to memory on access.

### Cost Optimization Strategies

1. **Reserved Nodes**: Up to 55% savings for 1-year or 3-year commitments on node-based clusters (same model as RDS Reserved Instances and DynamoDB Reserved Capacity)

2. **Valkey over Redis**: 20-33% cheaper on ElastiCache with API compatibility; default choice for new clusters

3. **Right-sizing**: Monitor DatabaseMemoryUsagePercentage; if consistently below 50%, you are over-provisioned. Use smaller nodes or fewer shards.

4. **Data tiering (r6gd)**: For datasets where 20%+ of data is infrequently accessed, r6gd nodes store cold data on SSD at 60% lower cost than equivalent pure-memory nodes

5. **Serverless for variable workloads**: Avoid paying for peak capacity during off-hours; Serverless scales to near-zero

6. **TTL discipline**: Short TTLs on low-value data prevent memory bloat; longer TTLs on expensive-to-compute data maximize cache hit rates

7. **Appropriate eviction policy**: `allkeys-lru` ensures the cache self-manages memory pressure; `noeviction` causes write failures that may trigger costly error-handling paths

---

## Production Architecture: E-Commerce Platform with ElastiCache

```
PRODUCTION ELASTICACHE ARCHITECTURE -- E-COMMERCE PLATFORM
════════════════════════════════════════════════════════════════════════════

  ┌──── us-east-1 (PRIMARY) ───────────────────────────────────────────┐
  │                                                                     │
  │  ┌─── Application Layer ─────────────────────────────────────────┐  │
  │  │                                                               │  │
  │  │  CloudFront ──▶ ALB ──▶ ECS Services                        │  │
  │  │  (CDN edge)     (route)   │                                  │  │
  │  │                           │                                  │  │
  │  │           ┌───────────────┼───────────────┐                  │  │
  │  │           ▼               ▼               ▼                  │  │
  │  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │  │
  │  │  │ Product      │ │ Session      │ │ Leaderboard  │         │  │
  │  │  │ Service      │ │ Service      │ │ Service      │         │  │
  │  │  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘         │  │
  │  │         │                │                │                  │  │
  │  └─────────┼────────────────┼────────────────┼──────────────────┘  │
  │            │                │                │                     │
  │            ▼                ▼                ▼                     │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │  ElastiCache Redis/Valkey (Cluster Mode Enabled, 3 shards)   │  │
  │  │                                                              │  │
  │  │  │  Shard 1 (Primary + 2 Replicas across AZs)                  │  │
  │  │  Shard 2 (Primary + 2 Replicas across AZs)                  │  │
  │  │  Shard 3 (Primary + 2 Replicas across AZs)                  │  │
  │  │                                                              │  │
  │  │  Keys distributed by hash slot (16,384 slots across shards):│  │
  │  │    ─ Product catalog cache (lazy loading, TTL = 1 hour)     │  │
  │  │    ─ User sessions (write-through, TTL = 30 min)            │  │
  │  │    ─ Shopping carts (write-through, TTL = 24 hours)         │  │
  │  │    ─ Leaderboard sorted sets (no TTL, direct Redis writes)  │  │
  │  │    ─ Rate limiting counters (TTL = sliding window)          │  │
  │  │    ─ API response cache (lazy loading, TTL = 5 min)         │  │
  │  │  Note: You don't control which shard a key lands on (hash   │  │
  │  │  slot routing). Use hash tags to co-locate related keys.    │  │
  │  │                                                              │  │
  │  │  Encryption: at rest + in transit                            │  │
  │  │  Auth: IAM authentication                                    │  │
  │  │  Eviction: allkeys-lru                                       │  │
  │  └───────────────────────────┬──────────────────────────────────┘  │
  │                              │                                     │
  │              Global Datastore replication (async, <1s)             │
  │                              │                                     │
  │  ┌───────────────────────────┼──────────────────────────────────┐  │
  │  │  Database Layer           │                                  │  │
  │  │                           │                                  │  │
  │  │  Aurora PostgreSQL   DynamoDB         (same architecture     │  │
  │  │  (product catalog,   (orders, sessions from Mar 26 deep     │  │
  │  │   analytics, ad-hoc  with DAX for     dive)                 │  │
  │  │   queries)           DynamoDB-only                           │  │
  │  │                      read acceleration)                      │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  └────────────────────────────────┬────────────────────────────────────┘
                                   │
                 Global Datastore  │
                                   │
            ┌──────────────────────┴──────────────────────┐
            ▼                                             ▼
  ┌──── eu-west-1 ──────────────────┐  ┌──── ap-northeast-1 ─────────────┐
  │                                  │  │                                  │
  │  ElastiCache (read-only)        │  │  ElastiCache (read-only)         │
  │  ─ Local reads for EU users     │  │  ─ Local reads for APAC users    │
  │  ─ Sessions read locally        │  │  ─ Leaderboard reads locally     │
  │  ─ Writes routed to us-east-1   │  │  ─ Writes routed to us-east-1   │
  │                                  │  │                                  │
  │  Aurora Global DB (read-only)   │  │  DynamoDB Global Tables          │
  │  DynamoDB Global Tables         │  │  (active-active writes)          │
  │  (active-active writes)         │  │                                  │
  └──────────────────────────────────┘  └──────────────────────────────────┘

  Architecture decisions:
  ─ ElastiCache for cross-service caching: fronts BOTH Aurora and DynamoDB
  ─ DAX for DynamoDB-specific acceleration (zero code change on order lookups)
  ─ ElastiCache for leaderboards/rate-limiting (sorted sets, not available in DAX)
  ─ Global Datastore for multi-region session reads (single-writer, but reads local)
  ─ DynamoDB Global Tables for order data (active-active writes in every region)
  ─ Aurora Global Database for analytics (single writer, read-only secondaries)
  ─ Note: Global Datastore secondaries are READ-ONLY (like Aurora, unlike DynamoDB)
```

### Terraform Example: ElastiCache Cluster Mode Enabled with Encryption

```hcl
# ElastiCache Redis/Valkey Cluster with Cluster Mode Enabled
resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "ecommerce-cache"
  description          = "E-commerce caching layer - Cluster Mode Enabled"

  # Engine configuration
  engine         = "valkey"        # AWS default; 20-33% cheaper than redis
  engine_version = "7.2"
  node_type      = "cache.r7g.large"  # Graviton3, memory-optimized
  port           = 6379

  # Cluster Mode Enabled configuration
  num_node_groups         = 3     # 3 shards
  replicas_per_node_group = 2     # 2 replicas per shard (across AZs)

  # High availability
  automatic_failover_enabled = true
  multi_az_enabled           = true

  # Security
  at_rest_encryption_enabled = true   # MUST be set at creation -- cannot add later
  transit_encryption_enabled = true   # Can be added later on Redis 7+/Valkey via TLS migration, but best to enable at creation
  auth_token                 = var.redis_auth_token  # Or use IAM auth

  # Network
  subnet_group_name  = aws_elasticache_subnet_group.private.name
  security_group_ids = [aws_security_group.elasticache.id]

  # Maintenance and backups
  maintenance_window       = "sun:05:00-sun:06:00"
  snapshot_window           = "03:00-04:00"
  snapshot_retention_limit  = 7    # Keep 7 daily snapshots
  auto_minor_version_upgrade = true

  # Parameters
  parameter_group_name = aws_elasticache_parameter_group.custom.name

  tags = {
    Environment = "production"
    Service     = "ecommerce-cache"
  }
}

# Custom parameter group for tuning
resource "aws_elasticache_parameter_group" "custom" {
  family = "valkey7"
  name   = "ecommerce-cache-params"

  # Set eviction policy for pure caching workload
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  # Enable keyspace notifications for cache invalidation events
  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"  # Notify on expired keys
  }
}

# Subnet group: deploy across private subnets in 3 AZs
resource "aws_elasticache_subnet_group" "private" {
  name       = "ecommerce-cache-subnets"
  subnet_ids = var.private_subnet_ids  # From your VPC module
}

# Security group: allow Redis traffic from application servers only
resource "aws_security_group" "elasticache" {
  name_prefix = "elasticache-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [var.app_server_sg_id]  # Only app servers can reach cache
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ElastiCache Serverless (for comparison -- simpler configuration)
resource "aws_elasticache_serverless_cache" "serverless" {
  engine = "valkey"
  name   = "ecommerce-cache-serverless"

  # No node type, shard count, or replica count needed
  # Auto-scales compute and memory automatically

  cache_usage_limits {
    data_storage {
      maximum = 10  # Maximum 10 GB (scales down automatically)
      unit    = "GB"
    }
    ecpu_per_second {
      maximum = 10000  # Maximum 10,000 ECPUs/second
    }
  }

  # Security (mandatory -- always encrypted)
  subnet_ids         = var.private_subnet_ids
  security_group_ids = [aws_security_group.elasticache.id]

  # Note: No auth_token needed for Serverless with IAM auth
  # No snapshot configuration -- handled automatically
}
```

---

## Key Takeaways

- **ElastiCache is the general-purpose in-memory caching layer that sits in front of any database or data source.** Unlike DAX (DynamoDB-only, zero-code-change), ElastiCache requires you to implement caching logic but caches anything: Aurora queries, DynamoDB results, API responses, computed aggregations, session data, and more. The two services are complementary, not competing.

- **Default to Redis/Valkey over Memcached.** Memcached is simpler and multi-threaded, but Redis/Valkey offers persistence, replication, data structures, pub/sub, clustering, Global Datastore, and data tiering. The only reason to choose Memcached is if you need the absolute simplest key-value cache and explicitly do not need any of those features.

- **Use Valkey for new clusters.** Valkey is the Linux Foundation's BSD-licensed fork of Redis 7.2.4, API-compatible, and 20-33% cheaper on ElastiCache. AWS defaults to it. There is no technical reason to choose Redis OSS over Valkey for new deployments.

- **Combine lazy loading + selective write-through + TTL everywhere.** Lazy loading (cache-aside) is your universal baseline -- memory-efficient, resilient to failures. Add write-through for hot data that must be immediately fresh (sessions, carts). Set TTL on everything as the universal safety net against stale data and edge cases. This is exactly the pattern DAX implements natively for DynamoDB.

- **Cache invalidation is genuinely hard -- TTL is your best friend.** Explicit invalidation is fragile (what if the delete fails?). Event-driven invalidation adds complexity. TTL guarantees that even in the worst case, stale data expires. Use staggered TTLs (base + random jitter) to prevent thundering herds.

- **Cluster Mode Enabled for production, Cluster Mode Disabled for simplicity.** Cluster Mode Enabled provides horizontal write scaling, larger datasets, online resharding, and is required for Global Datastore. Cluster Mode Disabled is simpler but limits you to one node's memory and one primary's write throughput.

- **ElastiCache Serverless follows the same pattern as Aurora Serverless v2 and DynamoDB on-demand.** Zero capacity planning, instant scaling, higher per-unit cost. Use for variable/new workloads. Migrate to node-based + Reserved Nodes for steady, predictable workloads where the up to 55% savings justify the operational overhead.

- **At-rest encryption must be enabled at creation time on node-based clusters** (same constraint as RDS). In-transit encryption can be added retroactively on Redis 7+/Valkey via a two-phase TLS migration (`preferred` → `required`), but at-rest requires creating a new cluster and migrating. Serverless always encrypts both -- one less decision to get wrong.

- **Redis data structures unlock use cases far beyond caching.** Sorted sets for leaderboards and rate limiters, streams for event processing, pub/sub for real-time notifications, hashes for object storage, geospatial for location queries. Mentioning these in interviews demonstrates depth beyond "ElastiCache is a cache."

- **The `allkeys-lru` eviction policy should be your default for caching.** The Redis default `volatile-lru` only evicts keys with a TTL set -- if your keys lack TTLs, the cache fills up and starts returning errors. Switch to `allkeys-lru` for pure caching workloads.

- **Global Datastore follows the same single-writer model as Aurora Global Database, not the active-active model of DynamoDB Global Tables.** One primary region handles writes, up to two secondary regions serve read-only replicas with sub-second lag. Requires Cluster Mode Enabled and node-based clusters (not Serverless).

- **Monitor EngineCPUUtilization, not just CPUUtilization.** Redis is single-threaded for command processing. EngineCPUUtilization can hit 100% (saturating the main thread) while overall CPUUtilization shows 25% on a 4-core node. This is the Redis-specific metric that catches performance bottlenecks.

---

## Further Reading

- [What Is Amazon ElastiCache? (official docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html) -- Service overview, engine comparison, deployment models
- [Caching Strategies (official docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html) -- Lazy loading, write-through, TTL with pseudocode examples
- [Caching Best Practices (AWS)](https://aws.amazon.com/caching/best-practices/) -- Thundering herd, eviction policies, Russian doll caching
- [Redis vs Memcached Engine Selection (official docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/SelectEngine.html) -- Feature comparison table
- [Cluster Mode Disabled vs Enabled (official docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Replication.Redis-RedisCluster.html) -- Shard topology, hash slots, scaling
- [ElastiCache Serverless (AWS blog)](https://aws.amazon.com/blogs/aws/amazon-elasticache-serverless-for-redis-and-memcached-now-generally-available/) -- Architecture, ECPU pricing, use cases
- [Global Datastore (official docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Redis-Global-Datastore.html) -- Cross-region replication, failover
- [ElastiCache Security (official docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/encryption.html) -- Encryption, AUTH, RBAC, IAM authentication
- [Redis Data Types (redis.io)](https://redis.io/docs/latest/develop/data-types/) -- Sorted sets, streams, pub/sub, geospatial, and more
- [DAX vs ElastiCache (Mar 26 DynamoDB deep dive)](2026-03-26-dynamodb-deep-dive.md) -- Detailed DAX internals and comparison table
