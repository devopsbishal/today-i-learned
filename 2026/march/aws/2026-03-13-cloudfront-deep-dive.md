# AWS CloudFront Deep Dive -- The Global Newspaper Delivery Network Analogy

> You have built the corporate conglomerate (Organizations), automated the building management platform (Control Tower), wired up the airport security checkpoints (IAM), connected the surveillance cameras and inspectors (GuardDuty, Config, Inspector, Security Hub), installed the locksmith shop (KMS), fortified the castle walls (WAF, Shield, Network Firewall, Firewall Manager), and trained the air traffic control tower to route planes to the right airport (Route53). But none of those services answer the question that comes after the user arrives at the right endpoint: **how do you deliver content to millions of users worldwide without making each of them wait for a round trip to your origin data center?** When a user in Sydney requests an image hosted in us-east-1, should that request fly 16,000 kilometers to Virginia and back? When ten thousand users in Frankfurt request the same CSS file within the same minute, should your origin server handle ten thousand identical responses? That is the domain of Amazon CloudFront -- a global content delivery network that places copies of your content at 750+ edge locations around the world, so users receive responses from a server that might be across the street instead of across the ocean.

---

## TL;DR -- Concepts at a Glance

| AWS Concept | Real-World Analogy | One-Liner |
|-------------|-------------------|-----------|
| CloudFront | A global newspaper delivery network with local newsstands in every neighborhood | AWS's CDN that caches and delivers content from 750+ edge locations worldwide, reducing latency by serving users from the nearest point of presence |
| Edge location (POP) | A neighborhood newsstand that stocks today's most popular papers | One of 750+ points of presence where CloudFront caches content closest to viewers; handles the final delivery to users |
| Regional edge cache (REC) | A regional distribution warehouse that stocks a wider catalog than any single newsstand | 15 larger cache facilities between edge locations and origins; holds more content for longer, absorbing cache misses from multiple POPs before going to the origin |
| Origin Shield | The central national warehouse that every regional warehouse orders from | An optional single-point cache layer in one AWS region that collapses all cache misses into one request to the origin, maximizing cache hit ratio and reducing origin load |
| Distribution | A delivery contract specifying what you publish, where it comes from, and how it gets delivered | The top-level CloudFront resource that ties together origins, cache behaviors, SSL certificates, logging, and WAF associations |
| Origin | The printing press where the original content is produced | The source server CloudFront fetches content from on cache miss: S3, ALB, NLB, custom HTTP server, or a VPC origin |
| Origin group | A backup printing press that takes over when the primary press breaks down | A pair of origins (primary + secondary) for automatic failover when the primary returns 5xx errors or times out |
| Cache behavior | Routing rules that say "sports section goes to Press A, comics go to Press B" | Path-pattern-based rules that map URL patterns to specific origins with their own caching, protocol, and edge compute settings |
| Cache policy | A newsstand manager's checklist: "Consider articles different if they have different headlines, authors, or editions" | Defines what goes into the cache key (which headers, cookies, query strings make objects unique) and TTL settings |
| Origin request policy | A note stapled to the order form: "Also tell the press what paper size and ink color we need, but do not create separate stock for each" | Controls additional headers, cookies, and query strings forwarded to the origin without adding them to the cache key |
| OAC (Origin Access Control) | A secure courier badge that only CloudFront delivery trucks can wear | The recommended method to restrict S3 origin access to CloudFront only, using SigV4 short-term credentials; supports SSE-KMS, all regions, and dynamic requests |
| OAI (Origin Access Identity) | The old courier ID card -- still works, but has been retired for new hires | Legacy mechanism for S3 origin restriction; does not support SSE-KMS, opt-in regions, or write operations; deprecated in favor of OAC |
| VPC Origins | A private underground tunnel from the delivery truck directly to the printing press, with no public loading dock | CloudFront connects to private ALBs/NLBs inside your VPC over the AWS backbone, eliminating the need for public endpoints |
| Lambda@Edge | A regional editor who can rewrite articles, check press credentials, or reroute deliveries at the warehouse level | Node.js/Python functions running at 15 regional edge caches with up to 5s (viewer) / 30s (origin) execution, network access, and four trigger points (viewer request/response, origin request/response) |
| CloudFront Functions | A newsstand clerk who stamps, re-labels, or redirects papers in under a millisecond right at the counter | Lightweight JavaScript functions running at 750+ edge locations with sub-millisecond execution; viewer request/response triggers only, no network access |
| Signed URL | A ticket with an expiration stamp for picking up a specific reserved newspaper | A URL containing a signature, expiration, and optionally IP restriction that grants temporary access to a specific private object |
| Signed cookie | A membership card that grants access to the entire premium section | A set of cookies granting access to multiple restricted files without changing URLs -- ideal for streaming video segments or gated content areas |
| Cache invalidation | Sending a recall notice to every newsstand: "Pull yesterday's front page off the rack" | Removes objects from all edge caches before TTL expiration; first 1,000 paths/month free, then $0.005/path; prefer versioned file names instead |
| Versioned file names | Printing a new edition number on each paper so newsstands naturally replace old stock | Embedding a hash or version in the file name (style.abc123.css) so new deploys are new cache keys; instant rollback, no invalidation needed, browsers fetch fresh copies automatically |

---

## The Big Picture: Content Delivery as a Newspaper Network

Think of your web application as a **newspaper publishing operation**. Your origin server (S3 bucket, ALB, custom server) is the **printing press** -- it produces the authoritative, original content. Without a CDN, every reader in every city must send a messenger to the press, wait for the paper to be printed, and carry it back. For a press in Virginia, a reader in Sydney waits 200+ milliseconds for a single round trip -- and if a million readers want the same front page, the press handles a million identical print jobs.

CloudFront transforms this into a **global newspaper delivery network**:

1. **The Printing Press** (Origin) -- Your S3 bucket, ALB, or custom server where the original content lives. It only prints when someone actually needs a fresh copy.

2. **The Central National Warehouse** (Origin Shield, optional) -- A single warehouse in one region that consolidates all orders. When five regional warehouses each need the same article, only one request goes to the press. The warehouse stocks the result and fulfills the other four from its inventory.

3. **Regional Distribution Warehouses** (Regional Edge Caches) -- 15 large warehouses positioned around the world. They hold a broader catalog than any single newsstand and keep stock longer. When a newsstand runs out, it checks the regional warehouse first before bothering the national warehouse or the press.

4. **Neighborhood Newsstands** (Edge Locations / POPs) -- 750+ small outlets in cities worldwide, as close to readers as possible. They stock the most popular papers. When a reader asks for today's front page, the newsstand either hands it over instantly (cache hit) or sends an order up the chain (cache miss).

The critical insight: **every cache hit at any layer stops the request from traveling further toward the origin.** A cache hit at the newsstand (POP) means sub-10ms delivery. A cache hit at the regional warehouse (REC) avoids the long trip to the origin. A cache hit at Origin Shield means only one request ever reaches your press, even if 15 regional warehouses needed the same content simultaneously.

```
CLOUDFRONT FOUR-TIER CACHE HIERARCHY
═══════════════════════════════════════════════════════════════════════════

   VIEWERS (browsers, mobile apps, API clients)
   ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
   │  Sydney   │ │  Tokyo    │ │  Frankfurt│ │  New York │
   └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
  ╔═══════════════════════════════════════════════════════════════════╗
  ║  TIER 1: EDGE LOCATIONS (POPs) -- 750+ worldwide               ║
  ║  "Neighborhood Newsstands"                                      ║
  ║                                                                 ║
  ║  Cache HIT  ──▶ Return content immediately (1-10ms)             ║
  ║  Cache MISS ──▶ Forward to regional edge cache                  ║
  ║                                                                 ║
  ║  Also runs: CloudFront Functions (viewer request/response)      ║
  ║            Lambda@Edge (viewer request/response triggers)      ║
  ╚═══════════════════════════════╤═══════════════════════════════════╝
                                  │ cache miss
                                  ▼
  ╔═══════════════════════════════════════════════════════════════════╗
  ║  TIER 2: REGIONAL EDGE CACHES (RECs) -- 15 locations            ║
  ║  "Regional Distribution Warehouses"                             ║
  ║                                                                 ║
  ║  Larger cache capacity, longer retention than POPs              ║
  ║  Serves multiple POPs in the geographic area                    ║
  ║                                                                 ║
  ║  Cache HIT  ──▶ Return to POP (POP caches it too)              ║
  ║  Cache MISS ──▶ Forward to Origin Shield (or direct to origin)  ║
  ║                                                                 ║
  ║  Also runs: Lambda@Edge (origin request/response triggers)      ║
  ║                                                                 ║
  ║  NOTE: Proxy methods (PUT/POST/PATCH/DELETE) BYPASS RECs        ║
  ║        and go straight to origin -- dynamic writes don't cache  ║
  ╚═══════════════════════════════╤═══════════════════════════════════╝
                                  │ cache miss
                                  ▼
  ╔═══════════════════════════════════════════════════════════════════╗
  ║  TIER 3: ORIGIN SHIELD (optional, one AWS region)               ║
  ║  "Central National Warehouse"                                   ║
  ║                                                                 ║
  ║  Single additional cache layer that collapses duplicate          ║
  ║  requests from multiple RECs into one origin fetch              ║
  ║                                                                 ║
  ║  Best for: high-traffic sites, expensive origin operations,     ║
  ║  origins outside AWS where you pay per request                  ║
  ╚═══════════════════════════════╤═══════════════════════════════════╝
                                  │ cache miss (only request that reaches origin)
                                  ▼
  ╔═══════════════════════════════════════════════════════════════════╗
  ║  TIER 4: ORIGIN                                                 ║
  ║  "The Printing Press"                                           ║
  ║                                                                 ║
  ║  S3 Bucket │ ALB/NLB │ Custom HTTP Server │ VPC Origin          ║
  ║                                                                 ║
  ║  Processes the request and returns the authoritative response   ║
  ║  with Cache-Control headers that influence TTL at all tiers     ║
  ╚═══════════════════════════════════════════════════════════════════╝
```

---

## Part 1: Distributions and Origins -- The Publishing Contract and Printing Presses

### Distributions

A CloudFront **distribution** is your publishing contract -- the top-level resource that defines everything about how content gets delivered. Think of it as the agreement between you (the publisher) and the delivery network (CloudFront) that specifies:

- Where the original content comes from (origins)
- How different types of content should be handled (cache behaviors)
- What domain names viewers use to access it (aliases/CNAMEs)
- Which SSL certificate to present (ACM certificate, must be in us-east-1)
- What security measures to apply (WAF Web ACL, geo-restriction)
- Where to send delivery logs (S3, CloudWatch, Firehose)

Each distribution gets a unique domain name like `d111111abcdef8.cloudfront.net`. You typically create a Route53 alias record (as you learned on [Mar 11](2026-03-11-route53-routing-policies.md)) pointing your custom domain to this distribution domain -- alias records to CloudFront are free, instant, and work at the zone apex.

**Key constraint**: CloudFront is a **global service** with its control plane in **us-east-1**. This means:
- ACM certificates for CloudFront **must be in us-east-1**
- WAF Web ACLs for CloudFront **must be created in us-east-1** (as you learned on [Mar 10](2026-03-10-aws-network-application-protection.md))
- CloudFront API calls go to us-east-1 regardless of where you call from
- **HTTP/3 (QUIC) support**: CloudFront supports HTTP/3 with QUIC for improved performance on lossy networks and faster connection establishment. Enable per-distribution.

### Price Classes -- Controlling Your Delivery Footprint

Not every application needs global delivery. Price classes let you limit which edge locations serve your content to reduce costs:

| Price Class | Edge Locations Included | Cost |
|-------------|------------------------|------|
| **PriceClass_100** | North America + Europe only | Lowest |
| **PriceClass_200** | PriceClass_100 + Asia, Middle East, Africa | Medium |
| **PriceClass_All** | All edge locations worldwide | Highest (default) |

If a viewer in a region you excluded makes a request, CloudFront still serves it -- but from the nearest included edge location, which may add latency. Content is never blocked, just potentially slower.

### Origins -- Your Printing Presses

An origin is any server that CloudFront fetches content from on a cache miss. CloudFront supports four origin types, each with distinct configuration:

#### S3 Origins

The most common origin type. CloudFront fetches objects directly from an S3 bucket using the S3 REST API.

**S3 REST API origin vs S3 website endpoint origin** -- an important distinction:
- **S3 REST API origin** (recommended): Use the bucket domain (`my-bucket.s3.us-east-1.amazonaws.com`). Supports OAC/OAI so the bucket stays private. Supports SSE-KMS. Does not support S3 redirect rules or index documents.
- **S3 website endpoint origin** (custom origin): Use the website endpoint domain (`my-bucket.s3-website-us-east-1.amazonaws.com`). Treated as a custom HTTP origin -- no OAC/OAI support, so the bucket must be public. Supports S3 redirect rules and index documents. Use this only when you need S3's static website hosting features (redirects, custom error pages, index.html routing).

```
S3 Origin Configuration:
┌─────────────────────────────────────────────────────┐
│  Origin domain: my-bucket.s3.us-east-1.amazonaws.com│
│  Origin path:   /assets  (optional prefix)          │
│  Origin access:  OAC (recommended) or OAI (legacy)  │
│                                                     │
│  Viewer requests /images/logo.png                   │
│  CloudFront fetches s3://my-bucket/assets/images/   │
│                              logo.png               │
│                  (origin path + URI path)            │
└─────────────────────────────────────────────────────┘
```

#### ALB/NLB Origins (Public)

For dynamic content served by your application. CloudFront connects to the load balancer over the public internet. The ALB/NLB must be internet-facing (public) unless you use VPC Origins (covered in Part 4).

- **ALB**: Supports HTTP/HTTPS. CloudFront can terminate TLS and re-encrypt to the ALB, or connect over HTTP to the ALB within the AWS network.
- **NLB**: Supports HTTP/HTTPS/TCP. Useful when you need TCP-level load balancing behind CloudFront.

With a public ALB origin, you historically needed a **custom header validation** pattern to prevent users from bypassing CloudFront and hitting the ALB directly: CloudFront sends a secret custom header (`X-Custom-Header: s3cr3t-v4lu3`), and the ALB's WAF Web ACL or listener rules reject requests missing that header. VPC Origins (Part 4) eliminates this workaround entirely.

#### Custom HTTP Origins

Any HTTP/HTTPS endpoint accessible over the internet: an on-premises server, an EC2 instance with a public IP, a third-party API. You specify the domain name, port, and protocol policy.

#### Origin Groups -- Backup Printing Presses

An origin group pairs a **primary origin** and a **secondary origin** for automatic failover. If CloudFront receives a 5xx error, a 4xx error (configurable), or a connection timeout from the primary, it automatically retries the request against the secondary.

```
ORIGIN GROUP FAILOVER FLOW
════════════════════════════════════════════════

  Cache behavior (path: /api/*)
         │
         ▼
  ┌──────────────────────┐
  │    Origin Group       │
  │  ┌────────────────┐  │
  │  │ Primary: ALB-1 │──┼──── Request ──▶ ALB-1 returns 503
  │  └────────────────┘  │
  │  ┌────────────────┐  │         │
  │  │ Secondary: ALB-2│──┼────────┘ Automatic retry ──▶ ALB-2
  │  └────────────────┘  │                               returns 200
  └──────────────────────┘
                                    Response ──▶ Viewer
```

The failover is **per-request** and transparent to the viewer. The failover status codes are configurable -- by default, CloudFront fails over on 500, 502, 503, and 504. You can also include 400, 403, 404, and 416.

**Common pattern**: Primary origin is an S3 bucket, secondary is another S3 bucket in a different region (cross-region replication). This gives you automatic static content failover without Route53 health checks.

---

## Part 2: Cache Behaviors -- The Routing Desk

### The Analogy

Cache behaviors are the **routing desk** at the distribution warehouse. When a delivery order arrives, the routing desk looks at the item name (URL path) and decides: "Sports section goes to Press A, comics go to Press B, everything else goes to Press C." Each routing rule specifies not just which press to use, but also how to handle that type of content -- how long to keep it in stock, whether to require ID, what delivery speed to use.

### How Path Pattern Matching Works

Every distribution has a **default behavior** (path pattern `*`) that catches all requests not matched by any other behavior. You then add additional behaviors with specific path patterns. The rules:

1. **Path patterns use simple wildcards**: `*` matches zero or more characters, `?` matches exactly one character
2. **Evaluation order matters**: behaviors are evaluated in the order you define them (precedence). The **first match wins**. The default behavior (`*`) is always evaluated last
3. **Each behavior maps to exactly one origin** (or origin group)
4. **Each behavior has its own** cache policy, origin request policy, viewer protocol policy, allowed HTTP methods, edge function associations, and TTL settings

```
CACHE BEHAVIOR PATH PATTERN MATCHING
════════════════════════════════════════════════════════════════

  Incoming request: https://cdn.example.com/api/v2/users?page=1

  Behavior 1:  /static/*       ──▶ S3 Origin        (no match)
  Behavior 2:  /api/*          ──▶ ALB Origin        ✓ MATCH (first match wins)
  Behavior 3:  /images/*.jpg   ──▶ S3 Origin        (not evaluated)
  Default:     *               ──▶ S3 Origin        (fallback, not reached)

  Result: Request goes to ALB Origin with Behavior 2's settings
```

### Behavior Settings That Matter

| Setting | What It Controls | Common Values |
|---------|-----------------|---------------|
| **Viewer protocol policy** | Whether to allow HTTP, redirect HTTP to HTTPS, or require HTTPS | "Redirect HTTP to HTTPS" for most use cases |
| **Allowed HTTP methods** | Which HTTP methods CloudFront forwards to the origin | GET/HEAD for static content; GET/HEAD/OPTIONS/PUT/POST/PATCH/DELETE for APIs |
| **Cache policy** | What goes into the cache key + TTL settings | CachingOptimized for S3, CachingDisabled for dynamic APIs |
| **Origin request policy** | What additional data to forward to the origin (not in cache key) | AllViewerExceptHostHeader for ALB origins |
| **Response headers policy** | Security headers CloudFront adds to responses | SecurityHeadersPolicy adds HSTS, X-Content-Type-Options, etc. |
| **Compress objects automatically** | Whether to gzip/Brotli compress responses | Yes for text-based content (HTML, CSS, JS, JSON) |
| **Function associations** | CloudFront Functions or Lambda@Edge at trigger points | URL rewriting, header manipulation, auth checks |

### The Multi-Origin, Multi-Behavior Pattern

This is the pattern that makes CloudFront work as both a CDN and a reverse proxy under a single domain:

```
SINGLE DOMAIN, MULTIPLE ORIGINS VIA CACHE BEHAVIORS
════════════════════════════════════════════════════════════════════════

  https://app.example.com
         │
    CloudFront Distribution
         │
         ├── /static/*     ──▶ S3 Bucket (OAC)
         │                     Cache policy: CachingOptimized (long TTL)
         │                     Compress: Yes
         │
         ├── /api/*        ──▶ ALB (VPC Origin or public + custom header)
         │                     Cache policy: CachingDisabled
         │                     Allowed methods: ALL (GET/HEAD/OPTIONS/PUT/POST/PATCH/DELETE)
         │                     Origin request policy: AllViewerExceptHostHeader
         │
         ├── /media/*      ──▶ S3 Bucket (OAC) or MediaStore
         │                     Cache policy: CachingOptimized
         │                     Signed URLs/cookies required
         │
         └── * (default)   ──▶ S3 Bucket (OAC)
                               Cache policy: CachingOptimized
                               (serves index.html for SPA routing)
```

---

## Part 3: Cache Policies vs Origin Request Policies -- The Inventory System vs the Order Form

### The Analogy

This is the most nuanced concept in CloudFront, and getting it wrong tanks your cache hit ratio.

Think of the regional warehouse (REC) manager maintaining an **inventory system**. The cache policy is the **inventory labeling system** -- it defines what makes two items "the same" or "different" in the warehouse. If you label items only by product name, then all "Widget A" boxes are interchangeable regardless of who ordered them. But if you also label by customer name, then "Widget A for Alice" and "Widget A for Bob" are different inventory items, even if the actual product is identical. The more labels you add, the more fragmented your inventory becomes, and the lower your "already in stock" rate (cache hit ratio).

The origin request policy is the **order form** you send to the printing press when you do need a fresh copy. The press might need to know the customer's language preference, their account tier, or the requested format -- but none of that information needs to fragment your inventory. You write it on the order form (forwarded to the origin) without adding it to the inventory label (cache key).

**The cardinal rule**: Put as little as possible in the cache key (cache policy). Forward as much as needed to the origin (origin request policy). Everything in the cache policy is automatically forwarded to the origin, so you never duplicate values between the two.

**Before optimizing the cache key, ask whether caching makes sense at all.** If an API endpoint returns personalized content per user (based on their auth token), the right answer is often `CachingDisabled` with `AllViewerExceptHostHeader` -- not trying to restructure what goes in the cache key. A `CachingDisabled` policy is not a failure; it is the correct choice for truly dynamic, per-user content.

### Cache Policy Structure

A cache policy has three components:

```
CACHE POLICY ANATOMY
════════════════════════════════════════════════

  Cache Policy: "MyAppCachePolicy"
  ┌─────────────────────────────────────────┐
  │  TTL Settings                           │
  │  ├── Minimum TTL:    0 seconds          │
  │  ├── Maximum TTL:    86400 (1 day)      │
  │  └── Default TTL:    3600 (1 hour)      │
  │                                         │
  │  Cache Key Settings                     │
  │  ├── Headers:  Accept-Language           │  ← makes objects unique per language
  │  ├── Cookies:  (none)                   │  ← all cookies ignored for caching
  │  ├── Query Strings: page, sort          │  ← only these QS params matter
  │  └── Compression: gzip, br normalized   │  ← Accept-Encoding collapsed to gzip/br
  │                                         │
  │  Result: Cache key =                    │
  │    domain + path + Accept-Language       │
  │    + ?page=X&sort=Y + encoding          │
  └─────────────────────────────────────────┘
```

**TTL interaction with origin headers**:

```
Origin sends Cache-Control: max-age=600?
  ├── Is 600 > Maximum TTL?  ──▶ Use Maximum TTL
  ├── Is 600 < Minimum TTL?  ──▶ Use Minimum TTL
  ├── Otherwise              ──▶ Use 600 (origin wins)
  └── No cache header at all? ──▶ Use Default TTL
```

### Origin Request Policy Structure

```
ORIGIN REQUEST POLICY ANATOMY
════════════════════════════════════════════════

  Origin Request Policy: "MyAppOriginPolicy"
  ┌─────────────────────────────────────────┐
  │  Additional Headers to Forward:          │
  │  ├── X-Forwarded-For (viewer IP)        │
  │  ├── CloudFront-Viewer-Country          │
  │  └── CloudFront-Is-Mobile-Viewer        │
  │                                         │
  │  Additional Cookies to Forward:          │
  │  └── session_id                         │
  │                                         │
  │  Additional Query Strings to Forward:    │
  │  └── (All)                              │
  │                                         │
  │  These values are sent to the origin    │
  │  but do NOT fragment the cache          │
  └─────────────────────────────────────────┘
```

### AWS Managed Policies You Should Know

| Policy | Type | What It Does | Use With |
|--------|------|-------------|----------|
| **CachingOptimized** | Cache | No headers, no cookies, no query strings in key; gzip+Brotli compression; 1-day default TTL | S3 static assets |
| **CachingDisabled** | Cache | Minimum/Default/Maximum TTL all set to 0; nothing cached | Dynamic API content via ALB |
| **CachingOptimizedForUncompressedObjects** | Cache | Same as CachingOptimized but no compression normalization | Already-compressed content (images, video) |
| **AllViewerExceptHostHeader** | Origin Request | Forwards all headers, cookies, and query strings EXCEPT the Host header | ALB origins (ALB needs its own Host header, not the CloudFront domain) |
| **UserAgentRefererHeaders** | Origin Request | Forwards only User-Agent and Referer headers | Origins that need browser info for analytics without cache fragmentation |
| **CORS-S3Origin** | Origin Request | Forwards Origin, Access-Control-Request-Method, and Access-Control-Request-Headers | S3 origins with CORS requirements |

### The Venn Diagram Mental Model

```
    ┌─────────────────────────────────────────────────────┐
    │         ORIGIN REQUEST (what the origin sees)        │
    │                                                     │
    │    ┌───────────────────────────────────┐             │
    │    │     CACHE KEY                     │             │
    │    │     (cache policy)                │             │
    │    │                                   │             │
    │    │  • Headers in cache policy        │             │
    │    │  • Cookies in cache policy        │  Additional │
    │    │  • Query strings in cache policy  │  forwarded: │
    │    │  • URL path (always in key)       │             │
    │    │  • Domain (always in key)         │  • Headers  │
    │    │                                   │  • Cookies  │
    │    │  (automatically forwarded)        │  • QStrings │
    │    └───────────────────────────────────┘  (from ORP) │
    │                                                     │
    └─────────────────────────────────────────────────────┘

  Cache key values are ALWAYS forwarded.
  Origin request policy adds MORE values without expanding the cache key.
  Never duplicate values between the two policies.
```

---

## Part 4: Securing Origins -- OAC, OAI, and VPC Origins

### Origin Access Control (OAC) -- The Secure Courier Badge

#### The Analogy

Think of your S3 bucket as a **warehouse with a loading dock**. Without access control, anyone who knows the warehouse address can walk up to the dock and take inventory. OAC is a **secure courier badge system** -- only delivery trucks wearing an authorized CloudFront badge can access the loading dock. The badge uses short-term, frequently rotated credentials (SigV4 signing), and the warehouse security guard (S3 bucket policy) verifies the badge before releasing any inventory.

#### How OAC Works

1. You create an OAC resource in CloudFront and associate it with your S3 origin
2. CloudFront signs every request to S3 using SigV4 with its own service principal
3. You add a bucket policy on S3 that allows `s3:GetObject` only from the CloudFront service principal for your specific distribution

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipalReadOnly",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-bucket/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/EDFDVBD6EXAMPLE"
                }
            }
        }
    ]
}
```

#### OAC vs OAI -- Why OAI Is Deprecated

| Capability | OAC (Recommended) | OAI (Legacy) |
|-----------|-------------------|--------------|
| **SSE-KMS encrypted objects** | Yes -- OAC signs requests with SigV4, so KMS can verify the caller and decrypt (connect this to the [KMS envelope encryption flow](2026-03-09-aws-kms-encryption-deep-dive.md) you studied on Mar 9) | No -- OAI cannot present the required `kms:Decrypt` authorization |
| **All AWS regions** | Yes, including opt-in regions launched after December 2022 | No -- does not work in newer opt-in regions |
| **Dynamic requests (PUT/POST/DELETE)** | Yes -- supports upload-through-CloudFront patterns | No -- read-only |
| **Credential rotation** | Short-term credentials rotated automatically | Static identity, no rotation |
| **Granular per-distribution scoping** | Yes -- bucket policy uses `AWS:SourceArn` to scope access to a specific distribution | Weak -- OAI identity can be shared across distributions |

**KMS integration detail**: When your S3 bucket uses SSE-KMS, you must also add a statement to the KMS key policy granting `kms:Decrypt` and `kms:GenerateDataKey` to the `cloudfront.amazonaws.com` service principal, scoped to your distribution ARN. This is the same two-policy pattern (resource policy + key policy) you learned in the [KMS cross-account access section](2026-03-09-aws-kms-encryption-deep-dive.md).

### VPC Origins -- The Private Underground Tunnel

#### The Analogy

Before VPC Origins, using an ALB as a CloudFront origin was like having a printing press with a **public loading dock on the street**. Even though you only wanted CloudFront delivery trucks to pick up papers, the dock was visible and accessible to anyone. You had to post a guard who checked for a secret passphrase (custom header validation) to prevent unauthorized pickups.

VPC Origins changes this entirely. It builds a **private underground tunnel** from the CloudFront delivery network directly into your private warehouse. The loading dock no longer faces the street at all -- it is completely inside the building, accessible only through the tunnel. No secret passphrase needed because there is no public entrance to protect.

#### How VPC Origins Works

1. You create a VPC Origin resource, specifying your private ALB, NLB, or EC2 instance
2. CloudFront creates a service-managed ENI in your VPC to reach the private origin
3. You update your security group to allow inbound traffic from the CloudFront-managed security group
4. The ALB/NLB can be in a **private subnet with no public IP** -- CloudFront connects over the AWS backbone

```
VPC ORIGINS ARCHITECTURE
════════════════════════════════════════════════════════════════

  Internet                   AWS Backbone (private)
  ┌──────────┐              ┌────────────────────────────────────────┐
  │          │              │                                        │
  │ Viewers  │──── HTTPS ──▶│  CloudFront    ═══ VPC Origin ═══▶  VPC │
  │          │              │  Edge Location    (AWS backbone)   │     │
  │          │              │                                   │     │
  └──────────┘              │                          ┌────────▼───┐│
                            │                          │ Private ALB ││
  No public                 │                          │ (no public  ││
  ALB endpoint              │                          │  IP at all) ││
  exists                    │                          └─────────────┘│
                            └────────────────────────────────────────┘

  BEFORE VPC Origins:
  ALB must be internet-facing → custom header trick for security

  AFTER VPC Origins:
  ALB is internal/private → CloudFront is the ONLY way in
```

**Current limitations to know**: VPC Origins does not support WebSocket connections, gRPC, or Lambda@Edge origin request/response triggers. These restrictions may change as the feature matures.

---

## Part 5: Edge Compute -- Lambda@Edge vs CloudFront Functions

### The Analogy

Your newspaper delivery network needs people who can modify papers in transit -- stamp a regional edition label, translate a headline, check a subscriber's membership card, or even rewrite an article for a different audience. CloudFront offers two types of workers:

**CloudFront Functions** are the **newsstand clerks** -- they work right at the counter (750+ edge locations), handle simple tasks in under a millisecond (stamping, labeling, redirecting), but they cannot make phone calls (no network access), cannot lift heavy boxes (2MB memory), and only interact with customers at the point of pickup (viewer request/response triggers only).

**Lambda@Edge** functions are the **regional editors** -- they work at the distribution warehouses (15 RECs), can make phone calls to verify information (network access, AWS SDK), have a larger desk to work at (128 MB for viewer triggers, up to 3,008 MB for origin triggers), and can modify content at every stage of the delivery pipeline (all four trigger points). But they take longer (seconds, not sub-milliseconds) and cost more.

### The Four Trigger Points

**Critical location detail**: CloudFront Functions execute at the **POP** (750+ locations). Lambda@Edge executes at the **REC** (15 locations) -- even for viewer request/response triggers. This means a viewer request Lambda@Edge trigger requires a POP-to-REC hop before the function runs, adding latency on every request including cache hits. When both a CloudFront Function and a Lambda@Edge function are on the same viewer request trigger, the CloudFront Function runs first (at the POP), and its output becomes the input to the Lambda@Edge function (at the REC).

```
EDGE COMPUTE TRIGGER POINTS (with execution locations)
════════════════════════════════════════════════════════════════════════

  Viewer        POP (750+)              REC (15)             Origin
  ┌──────┐      ┌──────────┐          ┌──────────────┐      ┌──────┐
  │      │      │          │          │              │      │      │
  │      │─────▶│ CF Func  │─────────▶│ L@E VIEWER   │      │      │
  │      │      │ (viewer  │ POP→REC  │ REQUEST      │      │      │
  │      │      │ request) │  hop     │              │      │      │
  │      │      │          │          │  ┌─────────┐ │      │      │
  │      │      └──────────┘          │  │  Cache  │ │      │      │
  │      │                            │  │  lookup │ │      │      │
  │      │                            │  └────┬────┘ │      │      │
  │      │                            │  HIT──┤──MISS│      │      │
  │      │                            │  │    │    │ │      │      │
  │      │                            │  │    │    ▼ │      │      │
  │      │                            │  │    │ L@E  │      │      │
  │      │                            │  │    │ ORIGIN├─────▶│      │
  │      │                            │  │    │ REQ  │      │      │
  │      │                            │  │    │    │ │      │      │
  │      │                            │  │    │ L@E  │      │      │
  │      │                            │  │    │ ORIGIN│◀─────│      │
  │      │                            │  │    │ RESP │      │      │
  │      │                            │  │    │    │ │      │      │
  │      │      ┌──────────┐          │  ▼    ▼    ▼ │      │      │
  │      │◀─────│ CF Func  │◀─────────│ L@E VIEWER   │      │      │
  │      │      │ (viewer  │          │ RESPONSE     │      │      │
  │      │      │ response)│          │              │      │      │
  └──────┘      └──────────┘          └──────────────┘      └──────┘
```

### Decision Framework

| Dimension | CloudFront Functions | Lambda@Edge |
|-----------|---------------------|-------------|
| **Location** | 750+ edge locations (POPs) | 15 regional edge caches |
| **Triggers** | Viewer request, viewer response only | All four: viewer request/response, origin request/response |
| **Runtime** | JavaScript (`cloudfront-js-2.0` runtime, ES 5.1 base with select ES 6+ features) | Node.js or Python |
| **Execution time** | Sub-millisecond (hard limit ~1ms) | Up to 5s (viewer triggers), 30s (origin triggers) |
| **Memory** | 2 MB | 128 MB (viewer triggers), 128 MB - 3,008 MB (origin triggers) |
| **Package size** | 10 KB | 1 MB (viewer), 50 MB (origin) |
| **Network access** | No | Yes (HTTP calls, AWS SDK, DynamoDB, etc.) |
| **Request body access** | No | Yes (origin request/response only) |
| **Pricing** | ~$0.10 per million invocations (much cheaper) | ~$0.60 per million + duration charges |
| **KeyValueStore** | Yes (low-latency key-value reads at the edge) | No |

### Use Case Mapping

**CloudFront Functions** (the newsstand clerk):
- URL rewrites and redirects (`/about` to `/about/index.html`)
- Adding/modifying response headers (HSTS, CSP, X-Frame-Options)
- Cache key normalization (lowercase URLs, strip tracking query params)
- Simple JWT/token validation using KeyValueStore
- A/B testing via cookie inspection (redirect, don't generate content)
- Geo-based redirects using CloudFront-provided headers

**Lambda@Edge** (the regional editor):
- Authentication against external IdPs (viewer request trigger)
- Dynamic image resizing/transformation (origin response trigger)
- SEO-friendly server-side rendering (origin request trigger)
- Generating HTTP responses without reaching the origin (viewer request)
- Modifying origin response headers before caching (origin response)
- A/B testing that requires database lookups (viewer request trigger)

### CloudFront Functions KeyValueStore

CloudFront Functions can read from a **KeyValueStore** -- a low-latency, globally replicated key-value data store at the edge. This enables data-driven logic without network calls:

- **Redirect maps**: Store URL redirect rules (`/old-path` -> `/new-path`) and look them up in the function
- **Feature flags**: Toggle features per-user or per-region without redeploying
- **A/B test configurations**: Store test percentages and variant mappings
- **Geo-based routing rules**: Map countries to specific origins or content versions

The KeyValueStore is eventually consistent (updates propagate to all POPs within seconds) and supports up to 5 MB of data per store. You associate a KeyValueStore with a CloudFront Function, and the function reads keys synchronously with sub-millisecond latency. This is what makes CloudFront Functions viable for use cases that would otherwise require Lambda@Edge just for a database lookup.

---

## Part 6: Content Security -- Signed URLs, Signed Cookies, and Geo-Restriction

### Signed URLs -- The Reserved Pickup Ticket

**The analogy**: A signed URL is a **ticket for a specific reserved newspaper**. The ticket has the customer's name (optional IP restriction), the paper they can pick up (resource path), and an expiration time. Anyone who presents a valid, unexpired ticket gets the paper. But the ticket only works for one specific paper, and it expires.

**When to use signed URLs**:
- Restricting access to **individual files** (a downloadable PDF, a software installer)
- Clients that **do not support cookies** (mobile API clients, curl-based downloads)


**Signed URL structure**:
```
https://d111111abcdef8.cloudfront.net/premium/report.pdf
  ?Expires=1710460800
  &Signature=Abc123...
  &Key-Pair-Id=K2JCJMDEHXQW5F
```

### Signed Cookies -- The Membership Card

**The analogy**: A signed cookie is a **membership card** for the premium section of the newsstand. Flash the card and you can browse and pick up anything in the premium section -- sports, comics, crosswords -- without needing a separate ticket for each paper. The card has an expiration and optionally restricts which section you can access.

**When to use signed cookies**:
- Granting access to **multiple restricted files** without changing URLs (HLS video streams with dozens of segment files, a subscriber-only content area)
- Situations where you **cannot or do not want to change URLs** (existing links, SEO considerations)

**Note**: Signed cookies require cookie-capable clients (browsers). Native mobile apps with custom HTTP stacks that lack a cookie jar may need signed URLs instead, even for multi-file use cases.

### Canned Policy vs Custom Policy

| Feature | Canned Policy | Custom Policy |
|---------|--------------|---------------|
| **Resource** | Exactly one file (no wildcards) | Wildcard path patterns (e.g., `/premium/*`) |
| **Expiration** | Yes (end time only) | Yes (end time) + optional start time (activate-after) |
| **IP restriction** | No | Yes (restrict to specific IP or CIDR range) |
| **URL length** | Shorter (no policy statement in URL) | Longer (base64-encoded policy embedded in URL) |
| **Reusability** | One URL per file | One policy for many files via wildcards |

### Key Groups vs CloudFront Key Pairs (Legacy)

To create signed URLs/cookies, you need a **signer** -- the entity whose private key creates the signature. CloudFront offers two approaches:

- **Trusted key groups** (recommended): You create an RSA 2048 or ECDSA 256 key pair (note: field-level encryption requires RSA only), upload the public key to CloudFront, create a key group, and reference it in the cache behavior. You manage this through normal [IAM permissions](2026-03-05-iam-advanced-patterns.md) -- any IAM user/role with the right permissions can manage key groups. You can have multiple public keys per key group for rotation.
- **CloudFront key pairs** (legacy): Created only by the AWS root account user. No IAM delegation possible. Limited to one active and one inactive key pair. Deprecated -- do not use for new implementations.

### Geo-Restriction

CloudFront can block or allow access based on the **viewer's country** using either an allow-list (only these countries can access) or a deny-list (all countries except these can access). This operates at the edge before the request reaches your origin -- CloudFront returns a 403.

This is distinct from Route53 geolocation routing (which you covered on [Mar 11](2026-03-11-route53-routing-policies.md)): Route53 geolocation routes users to different endpoints based on location, while CloudFront geo-restriction blocks users from accessing content at all. They solve different problems and can be used together.

**Layered content protection**: Signed URLs/cookies and OAC solve different problems at different layers. OAC prevents direct S3 access bypassing CloudFront. Signed URLs/cookies prevent unauthorized users from accessing content *through* CloudFront. Both are needed for a complete protection strategy -- OAC without signed content means anyone with the CloudFront URL can access everything, and signed content without OAC means tech-savvy users can bypass CloudFront and hit S3 directly.

**For finer-grained control** (city-level, organization-level), use a third-party geolocation service combined with Lambda@Edge or CloudFront Functions reading the `CloudFront-Viewer-Country`, `CloudFront-Viewer-City`, or other geo headers.

### Field-Level Encryption

Field-level encryption adds an extra layer of protection for sensitive form data (credit card numbers, personal IDs). CloudFront encrypts specific POST fields at the edge using an RSA public key you provide, and only your origin application -- holding the corresponding private key -- can decrypt them. This means the data is encrypted throughout the entire request path, even within your own infrastructure. This pairs with the [envelope encryption model](2026-03-09-aws-kms-encryption-deep-dive.md) you studied -- but field-level encryption uses RSA asymmetric keys directly, not KMS.

---

## Part 7: Cache Invalidation vs Versioned File Names

### The Analogy

Cache invalidation is a **recall notice** sent to every newsstand in the network: "Pull yesterday's front page off the rack immediately." It works, but it is expensive (the courier has to visit 750+ newsstands), slow (takes seconds to minutes to propagate to all locations), and incomplete (you cannot recall papers that readers already picked up -- i.e., browser caches).

Versioned file names take a different approach entirely. Instead of recalling the old paper, you **print a new edition with a different name**. Yesterday's paper was `front-page-v1.pdf`. Today's is `front-page-v2.pdf`. The HTML entry point (the "table of contents") references the new file name, so readers naturally pick up the new edition. The old edition expires from newsstand racks naturally when its TTL expires. No recall notice needed.

### Practical Decision

| Approach | When to Use | Cost | Speed |
|----------|------------|------|-------|
| **Versioned file names** | Static assets (CSS, JS, images) -- build tools generate content hashes automatically (`style.abc123.css`) | Free | Instant (new URL = new cache entry) |
| **Invalidation** | HTML entry points, configuration files, emergency content removal, or any file whose URL cannot change | First 1,000 paths/month free; $0.005/path after that; `/*` counts as one path | Seconds to minutes to propagate globally |

### The Two-Tier TTL Strategy

The best practice combines both approaches:

```
TWO-TIER TTL STRATEGY
════════════════════════════════════════════════

  index.html              (short TTL: 60 seconds)
  ├── /static/style.a1b2c3.css    (long TTL: 1 year, versioned)
  ├── /static/app.d4e5f6.js       (long TTL: 1 year, versioned)
  └── /static/logo.g7h8i9.png     (long TTL: 1 year, versioned)

  Deploy workflow:
  1. Build generates new hashed file names
  2. Upload new static assets to S3 (new names = new cache keys)
  3. Upload new index.html referencing new file names
  4. (Optional) Invalidate /index.html to refresh within seconds
  5. Old assets expire naturally; new assets are fetched on demand

  Result:
  - Static assets: 99%+ cache hit ratio, instant updates, no invalidation
  - HTML: always fresh within 60 seconds (or immediately if invalidated)
  - Zero cache pollution -- old and new versions coexist safely
```

### Invalidation Mechanics

- You can have up to **3,000 invalidation paths** in progress simultaneously (with a separate limit of **15 concurrent wildcard paths**)
- An invalidation request can include up to **3,000 paths** per request (or use `/*` for everything)
- `/*` counts as **one path** toward your free 1,000/month quota -- making it the cheapest way to flush everything
- Invalidation does **not** clear browser caches or intermediate proxy caches -- only CloudFront edge caches
- Invalidation respects the same path pattern as the viewer would use, including query strings if they are part of the cache key

---

## Part 8: Logging and Monitoring

### Standard Access Logs (v2)

Standard logs deliver **detailed per-request records** to one of three destinations: S3, CloudWatch Logs, or Data Firehose. The logs include: timestamp, edge location, client IP, HTTP method, URI path, status code, bytes transferred, cache hit/miss status, time-to-first-byte, user agent, and more.

- **Delivery delay**: Minutes (not real-time)
- **Completeness**: Best-effort delivery; logs may be delayed or incomplete during high-traffic spikes
- **Cost**: Log delivery is free; you pay only for the storage destination (S3 storage, CloudWatch Logs ingestion)

### Real-Time Logs

For near-instant visibility, real-time logs deliver records to **Kinesis Data Streams** within seconds.

- **Configurable sampling rate**: Send 100% of requests for full visibility or sample (e.g., 10%) to control cost
- **Field selection**: Choose which fields to include in the log record to reduce volume
- **Use cases**: Real-time cache hit ratio dashboards, immediate 5xx spike detection, live traffic analysis

### CloudWatch Metrics

CloudFront publishes these metrics to CloudWatch at **no additional cost**:

| Metric | What It Tells You |
|--------|-------------------|
| `Requests` | Total number of viewer requests |
| `BytesDownloaded` | Total bytes served to viewers |
| `BytesUploaded` | Total bytes received from viewers (uploads through CloudFront) |
| `4xxErrorRate` | Percentage of requests resulting in 4xx status codes |
| `5xxErrorRate` | Percentage of requests resulting in 5xx status codes |
| `TotalErrorRate` | Combined 4xx + 5xx error rate |

**Additional metrics** (per-distribution fee to enable):

| Metric | What It Tells You |
|--------|-------------------|
| `CacheHitRate` | Percentage of requests served from cache (the most important operational metric) |
| `OriginLatency` | Time between CloudFront forwarding a request to the origin and receiving the first response byte |
| `401ErrorRate`, `403ErrorRate`, `404ErrorRate`, `502ErrorRate`, `503ErrorRate`, `504ErrorRate` | Per-status-code error rates for targeted alerting |

### Connecting Security Monitoring

CloudFront integrates with the security services you have already studied:

- **WAF Web ACLs** ([Mar 10](2026-03-10-aws-network-application-protection.md)): Attach a Web ACL to your CloudFront distribution to inspect and filter requests at the edge. WAF logs (when enabled) capture details about blocked/allowed requests separately from CloudFront access logs.
- **Shield Standard**: Automatically protects every CloudFront distribution against L3/L4 DDoS at no cost.
- **Shield Advanced**: Adds L7 DDoS detection, the DDoS Response Team (DRT), automatic L7 mitigation via WAF rules, and cost protection credits for scaling charges during attacks.

---

## Complete Architecture: Putting It All Together

```
PRODUCTION CLOUDFRONT ARCHITECTURE
═══════════════════════════════════════════════════════════════════════════

                         VIEWERS (global)
                              │
                    DNS: app.example.com
                    Route53 alias ──▶ d111.cloudfront.net
                              │
                              ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  SHIELD STANDARD (automatic L3/L4 DDoS protection)               │
  └──────────────────────────────┬─────────────────────────────────────┘
                                 ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  WAF WEB ACL (us-east-1)                                         │
  │  ├── AWS Managed Rules (SQLi, XSS, Bad Inputs)                   │
  │  ├── Rate limiting (2000 req/5min per IP)                        │
  │  ├── Geo-blocking (deny-list: sanctioned countries)              │
  │  └── Bot Control (optional premium)                              │
  └──────────────────────────────┬─────────────────────────────────────┘
                                 ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  CLOUDFRONT DISTRIBUTION                                         │
  │                                                                   │
  │  ┌─ CloudFront Function (viewer request) ────────────────────┐   │
  │  │  • URL rewrite: /about → /about/index.html                │   │
  │  │  • Add security headers in viewer response                 │   │
  │  └────────────────────────────────────────────────────────────┘   │
  │                                                                   │
  │  Cache Behaviors:                                                 │
  │  ┌────────────────────────────────────────────────────────────┐   │
  │  │  /static/*  ──▶ S3 Origin (OAC + SSE-KMS)                 │   │
  │  │               Cache: CachingOptimized (1yr TTL)            │   │
  │  │               Compress: gzip + Brotli                      │   │
  │  ├────────────────────────────────────────────────────────────┤   │
  │  │  /api/*     ──▶ ALB Origin Group (VPC Origin, private)     │   │
  │  │               Cache: CachingDisabled                       │   │
  │  │               ORP: AllViewerExceptHostHeader               │   │
  │  │               Methods: ALL                                 │   │
  │  │               Primary: ALB us-east-1                       │   │
  │  │               Secondary: ALB us-west-2 (failover)          │   │
  │  ├────────────────────────────────────────────────────────────┤   │
  │  │  /media/*   ──▶ S3 Origin (OAC)                            │   │
  │  │               Cache: CachingOptimized                      │   │
  │  │               Signed cookies required (trusted key group)  │   │
  │  ├────────────────────────────────────────────────────────────┤   │
  │  │  * (default) ──▶ S3 Origin (OAC)                           │   │
  │  │               Cache: CachingOptimized (60s default TTL)    │   │
  │  │               Serves SPA index.html                        │   │
  │  └────────────────────────────────────────────────────────────┘   │
  │                                                                   │
  │  Origin Shield: us-east-1 (collapses cache misses)               │
  │                                                                   │
  │  Logging:                                                        │
  │  ├── Standard logs ──▶ S3 bucket (long-term analysis)            │
  │  ├── Real-time logs ──▶ Kinesis (10% sampling, dashboards)       │
  │  └── CloudWatch metrics (CacheHitRate, OriginLatency enabled)    │
  └────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **CloudFront is a four-tier cache hierarchy**: POPs (750+) serve viewers, RECs (15) absorb POP misses, Origin Shield (optional, 1 region) collapses REC misses, and the origin handles only truly unique requests. Every cache hit at any tier prevents traffic from going further.

- **Proxy methods bypass regional edge caches**: PUT, POST, PATCH, and DELETE requests go straight from the POP to the origin. Only GET and HEAD responses are cached in the REC tier. This is why CloudFront handles both static CDN and dynamic reverse proxy use cases.

- **Cache behaviors are the routing engine**: Path patterns evaluated in order (first match wins) let you serve static assets from S3 and API requests from an ALB under a single domain. Each behavior has its own origin, cache policy, and edge compute settings.

- **Separate caching from forwarding**: Cache policies control cache key composition (fewer values = higher hit ratio). Origin request policies control additional data forwarded to the origin. Everything in the cache policy is automatically forwarded -- never duplicate between the two.

- **Use OAC, not OAI**: OAC supports SSE-KMS, all regions, dynamic requests, and uses short-term SigV4 credentials. OAI is legacy and deprecated. If your S3 objects use KMS encryption, OAC is the only option that works.

- **VPC Origins eliminates the custom header hack**: Private ALBs and NLBs can be CloudFront origins without any public internet exposure. CloudFront connects over the AWS backbone. This is the recommended pattern for securing ALB origins.

- **CloudFront Functions for simple, high-volume tasks; Lambda@Edge for complex logic**: CloudFront Functions run at 750+ POPs in sub-millisecond time but cannot make network calls. Lambda@Edge runs at 15 RECs with full Node.js/Python, network access, and all four trigger points.

- **Prefer versioned file names over invalidation**: Versioned files (style.abc123.css) provide instant updates, zero cost, and enable rollback. Use invalidation only for files whose URLs cannot change (HTML entry points) or emergency content removal. The `/*` wildcard counts as one path.

- **The two-tier TTL strategy is best practice**: Long TTLs (1 year) on versioned static assets, short TTLs (60 seconds) on HTML entry points. New deploys generate new file names; HTML references the new names; no invalidation needed for assets.

- **WAF Web ACLs for CloudFront must be in us-east-1**: CloudFront is a global service with its control plane in us-east-1. ACM certificates and WAF Web ACLs must also be in us-east-1.

- **Monitor CacheHitRate as your primary operational metric**: Enable the additional CloudWatch metrics (paid per-distribution) to track cache hit ratio and origin latency. A dropping cache hit rate means your cache key is too fragmented or your TTLs are too short.

---

## Further Reading

- [How CloudFront Delivers Content](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html) -- The official conceptual overview of the POP/REC hierarchy
- [Cache Behavior Settings Reference](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html) -- Exhaustive reference for every behavior setting
- [Understanding Cache Policies and Origin Request Policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cache-key-understand-cache-policy.html) -- The cache key separation model
- [Restricting Access to S3 Origins (OAC)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) -- OAC setup with bucket and KMS key policies
- [VPC Origins](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html) -- Private ALB/NLB as CloudFront origins
- [Choosing Between CloudFront Functions and Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html) -- Official comparison and decision framework
- [Serving Private Content with Signed URLs and Cookies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html) -- Signed content access patterns
- [Cache Invalidation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html) -- Mechanics, costs, and best practices
- Related docs in this repo: [WAF & Shield](2026-03-10-aws-network-application-protection.md), [Route53 Routing](2026-03-11-route53-routing-policies.md), [KMS & Encryption](2026-03-09-aws-kms-encryption-deep-dive.md), [IAM Advanced Patterns](2026-03-05-iam-advanced-patterns.md)
