# AWS API Gateway Deep Dive -- The Grand Hotel Concierge Analogy

> You built the on-demand kitchen yesterday (Lambda) -- a serverless compute engine that spins up cooking stations per order, charges by the millisecond, and scales to thousands of concurrent dishes. But a kitchen without a front-of-house is just a room full of stoves. No one takes reservations. No one checks if the customer has a valid membership card. No one limits how many dishes a single party can order. No one caches the popular dishes that get ordered every thirty seconds. And no one decides whether tonight's new pasta recipe should go to just 10% of tables first. That front-of-house operation -- the entire experience between the customer walking through the door and the dish arriving at their table -- is **API Gateway**. Think of API Gateway as the **Grand Hotel Concierge Desk**. The hotel has restaurants (Lambda), room service (other AWS services), and external catering partners (HTTP backends). The concierge does not cook. The concierge **receives guests, verifies their identity, checks their membership tier, enforces rate limits on how many requests they can make, caches popular answers so the kitchen is not bothered repeatedly, transforms requests between languages, routes guests to the right service, splits traffic between the old and new restaurant during a renovation, and keeps detailed logs of every interaction**. The hotel operates three concierge desks: the **full-service desk** (REST API) with every bell and whistle, the **express desk** (HTTP API) that is 71% cheaper but handles only the basics, and the **phone desk** (WebSocket API) that maintains open lines for real-time two-way conversations. Choosing which desk to use -- and understanding the 30+ features each one does or does not support -- is the architectural decision that determines your API's cost, performance, security, and operational complexity.

---

**Date**: 2026-04-07
**Topic Area**: aws
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| REST API | The full-service concierge desk -- handles ID checks, membership tiers, rate limiting, caching, luggage transformation, and controlled renovation reveals | $3.50/million requests; full feature set including caching, WAF, usage plans, VTL transformations, canary deployments, and private endpoints |
| HTTP API | The express concierge desk -- faster, 71% cheaper, handles JWT badges natively, but no luggage transformation, no membership tiers, no caching | $1.00/million requests; lower latency (~10ms p99), native JWT authorizer, simpler routing, but missing caching/WAF/usage plans/VTL |
| WebSocket API | The hotel telephone switchboard -- maintains open lines between guests and staff for real-time two-way conversation | $1.00/million messages + $0.25/million connection minutes; $connect/$disconnect/$default routes with @connections callback API for push messaging |
| Stages | Named floors in the hotel (lobby=dev, penthouse=prod) -- each floor has its own guest policies, speed limits, and cached answers | Logical deployment environments (dev/staging/prod) each with independent settings for throttling, caching, logging, and stage variables |
| Stage variables | Sticky notes on each floor's bulletin board -- "Tonight's chef is Lambda-v3" -- readable by the concierge when routing requests | Key-value pairs per stage that act as environment variables; used in integration URIs to route to different Lambda aliases or HTTP endpoints per stage |
| Throttling (token bucket) | A ticket dispenser at the concierge desk -- it holds a bucket of tokens, refills at a steady rate, and each request takes one token; when the bucket is empty, guests wait | Token bucket algorithm with four priority tiers: usage plan per-client per-method > method-level > stage-level > account-level (10K RPS / 5K burst default) |
| Caching | A filing cabinet behind the concierge desk -- when guests ask the same question repeatedly, the concierge checks the cabinet before calling the kitchen | Per-stage response caching (0.5-237 GB), 300s default TTL (max 3600s), cache keys from parameters, invalidation via Cache-Control: max-age=0 requiring IAM permission |
| Lambda authorizer (TOKEN) | A bouncer who examines a single badge (bearer token) and decides if the guest enters -- with a memory for recent guests so he does not re-check every visit | Receives a bearer token from a specified header, returns an IAM policy, cached by token value for up to 1 hour |
| Lambda authorizer (REQUEST) | A bouncer who examines the guest's entire dossier -- badge, ID card, referral letter, the floor they are visiting -- then makes a decision | Receives full request context (headers, query strings, path params, stage variables), returns an IAM policy, cached by configured identity sources |
| Cognito authorizer | An automated badge scanner at the door -- scans the Cognito JWT, checks the photo matches, verifies it has not expired, and optionally checks VIP scope | Validates Cognito JWT tokens directly at API Gateway (no Lambda invocation); authentication only, not custom authorization logic |
| Usage plans + API keys | A tiered membership system -- Silver members get 1,000 requests/day at 10 RPS, Gold members get 10,000/day at 100 RPS; the membership card (API key) identifies the tier | Per-client throttling and quotas; API keys identify callers and map to plans; API keys are NOT for authentication (they are identification tokens) |
| Integration types | How the concierge communicates with the kitchen: AWS_PROXY (pass the full order slip directly), AWS (rewrite the order into kitchen format), HTTP_PROXY (forward to external caterer as-is), HTTP (translate then forward), MOCK (answer from the script without calling anyone) | Five types controlling transformation depth: proxy types pass through, non-proxy types use VTL mapping templates, mock returns hardcoded responses |
| Mapping templates (VTL) | A translator at the concierge desk who rewrites requests from French to English before sending to the kitchen, and translates responses back | Apache Velocity Template Language transforms request/response payloads between client format and backend format; REST API non-proxy integrations only |
| Canary deployments | During a restaurant renovation, sending 10% of diners to the new dining room while 90% still eat in the old one -- with separate review cards for each | Stage-level traffic splitting between current deployment (production) and new deployment (canary) with configurable percentage and separate CloudWatch logs |
| Private APIs | A concierge desk inside the building's private lobby -- only tenants (VPC resources) can reach it through the internal elevator (VPC endpoint) | REST APIs accessible only via Interface VPC Endpoints; combined with resource policies to restrict access to specific VPCs/VPC endpoints |
| Mutual TLS (mTLS) | Both the guest and the hotel verify each other's identity cards before any conversation begins -- the hotel checks the guest's certificate against a trusted list | Client presents X.509 certificate validated against a truststore in S3; server also presents its certificate; works with all API types |
| Resource policies | The hotel's posted rules about who can enter: "Only guests from Building A (VPC) and Building B (IP range) allowed; all others denied" | JSON IAM-like policies attached to the API itself controlling who can invoke it; combines with IAM auth via AND logic (both must Allow) |
| WAF integration | A security checkpoint before the concierge desk that blocks known troublemakers, rate-limits suspicious visitors, and filters out guests carrying contraband payloads | AWS WAF web ACL attached to REST API stage; provides SQL injection/XSS protection, IP blocking, rate-based rules, geo-blocking; REST API only |
| Endpoint types | Regional (local hotel with same-city guests), Edge-Optimized (global hotel chain routing guests through nearest CloudFront airport), Private (internal-only staff entrance) | Regional: same-region clients, you manage CloudFront; Edge-Optimized: CloudFront distribution auto-created; Private: VPC-only access via VPC endpoint |
| CORS | The hotel's guest policy: "We accept visitors from partner hotels (origins), allow them to carry these items (headers), and let them use these entrances (methods)" | Cross-Origin Resource Sharing headers returned to browsers; HTTP API has one-click CORS; REST API requires manual configuration on each method + OPTIONS mock integration |
| Request validation | The concierge checking that the guest's form is filled out correctly before disturbing the kitchen -- "You forgot to write your room number" | API Gateway validates request body against JSON Schema models and/or required parameters before invoking the backend; REST API only |
| Gateway responses | Custom rejection letters -- instead of a generic "403 Forbidden," the concierge hands the guest a branded card explaining the hotel's membership policy | Customize error responses (4xx/5xx) returned by API Gateway itself (not your backend) with custom headers, status codes, and body templates |
| Access logging vs execution logging | Guest register (access log) records every visitor's name, time, and room number; security camera footage (execution log) records every step the concierge took for each request | Access logs: structured per-request data (CLF/JSON/CSV) to CloudWatch/Firehose; Execution logs: detailed request pipeline debugging including authorizer, integration, mapping template output |
| Binary media types | The concierge passing sealed packages (images, PDFs) without opening and reading them -- just checking the label and forwarding directly | Configure binary media types so API Gateway base64-encodes/decodes binary payloads (images, files) instead of treating everything as UTF-8 text |
| SDK generation | The concierge handing guests a pre-printed instruction booklet for how to interact with the hotel -- in their preferred language | Auto-generate client SDKs (Java, JavaScript, iOS, Android, Ruby) from your REST API definition; includes signature handling and retry logic |

---

## The Grand Hotel Concierge Framework

Every API Gateway concept maps to a role in a grand hotel's front-of-house operation. You do not manage the building (infrastructure). You design the concierge protocols (API configuration), and the **hotel management company** (the API Gateway service) handles scaling the desk staff, keeping the lights on, and charging you per guest interaction.

The hotel has five operational dimensions:

1. **The Concierge Desks (API Types)** -- Three desks serving different guest needs: full-service (REST), express (HTTP), and telephone switchboard (WebSocket).

2. **The Guest Processing Pipeline (Request Lifecycle)** -- The end-to-end journey from a guest arriving to receiving a response: identity verification, rate limiting, request translation, backend dispatch, response translation, caching.

3. **The Security Apparatus (Authorization & Protection)** -- Bouncers (Lambda authorizers), badge scanners (Cognito), posted rules (resource policies), security checkpoints (WAF), and mutual ID exchange (mTLS).

4. **The Traffic Management System (Throttling, Caching, Deployments)** -- Token dispensers (throttling), filing cabinets (caching), and renovation reveals (canary deployments).

5. **The Membership Program (Usage Plans & API Keys)** -- Tiered access with per-member rate limits and quotas.

```
THE GRAND HOTEL CONCIERGE ARCHITECTURE -- REQUEST LIFECYCLE (REST API)
══════════════════════════════════════════════════════════════════════════════

  GUEST                  FRONT OF HOUSE                        BACK OF HOUSE
  ─────                  ──────────────                        ─────────────

  ┌────────┐    ┌─────────────────────────────────────────┐    ┌──────────┐
  │ Client │    │              API GATEWAY                 │    │ Backend  │
  │ (HTTP  │───▶│                                         │───▶│ (Lambda, │
  │ request│    │  ┌─────┐  ┌───────┐  ┌───────┐         │    │  HTTP,   │
  │   )    │    │  │ WAF │─▶│Resource│─▶│Endpoint│         │    │  AWS     │
  └────────┘    │  │check│  │Policy │  │ Type  │         │    │ service) │
                │  └─────┘  └───────┘  └───┬───┘         │    └──────────┘
                │                          │              │         │
                │                          ▼              │         │
                │  ┌────────────────────────────────────┐ │         │
                │  │      METHOD REQUEST                │ │         │
                │  │  1. Request Validation (schema)    │ │         │
                │  │  2. Authorizer (Lambda/Cognito/IAM)│ │         │
                │  │  3. API Key check (if required)    │ │         │
                │  │  4. Throttle check (4-tier)        │ │         │
                │  └──────────────┬─────────────────────┘ │         │
                │                 │                        │         │
                │                 ▼                        │         │
                │  ┌────────────────────────────────────┐ │         │
                │  │    INTEGRATION REQUEST             │ │         │
                │  │  5. Cache lookup (if GET + enabled)│ │         │
                │  │  6. Mapping template (VTL)  ────────────────▶  │
                │  │     OR proxy passthrough           │ │         │
                │  └────────────────────────────────────┘ │         │
                │                                         │         │
                │  ┌────────────────────────────────────┐ │         │
                │  │    INTEGRATION RESPONSE            │ │    ◀────┘
                │  │  7. Mapping template (VTL)         │ │
                │  │     OR proxy passthrough           │ │
                │  └──────────────┬─────────────────────┘ │
                │                 │                        │
                │                 ▼                        │
                │  ┌────────────────────────────────────┐ │
                │  │      METHOD RESPONSE               │ │
                │  │  8. Response model validation      │ │
                │  │  9. Cache storage (if enabled)     │ │
                │  │ 10. CORS headers                   │ │
  ┌────────┐    │  └────────────────────────────────────┘ │
  │ Client │◀───│                                         │
  │ (HTTP  │    └─────────────────────────────────────────┘
  │response│
  └────────┘
```

---

## Part 1: The Three Concierge Desks -- REST API vs HTTP API vs WebSocket API

### REST API -- The Full-Service Desk

**The analogy**: The full-service concierge desk has been operating since the hotel opened in 2015. It offers every service imaginable: luggage transformation (VTL mapping templates), a filing cabinet for repeat questions (caching), a tiered membership program (usage plans + API keys), a security checkpoint at the entrance (WAF), controlled renovation reveals (canary deployments), automated request form validation (request validation), and a private staff-only entrance (private endpoints via VPC endpoints). It costs more because it does more.

**The technical reality**: REST API (formerly "API Gateway v1") is the original and most feature-complete API type. It supports:
- API keys and usage plans (per-client throttling and quotas)
- Request/response validation with JSON Schema models
- Response caching (0.5-237 GB per stage)
- VTL mapping templates for request/response transformation
- Canary release deployments
- AWS WAF integration
- Private API endpoints (VPC-only access)
- Edge-optimized endpoints (auto-provisioned CloudFront)
- Mock integrations
- SDK generation (Java, JavaScript, iOS, Android, Ruby)
- Gateway response customization
- Custom domain names with TLS 1.2
- Response streaming (November 2025) -- extends integration timeout up to 15 minutes and supports payloads larger than 10 MB for streaming use cases (LLM applications, SSE, large file downloads)

**Pricing**: $3.50 per million requests (first 333 million), $2.80 thereafter. Cache charges are separate and hourly.

### HTTP API -- The Express Desk

**The analogy**: The express concierge desk opened in 2019 as a budget-friendly, fast-lane alternative. It handles 90% of what most guests need -- identity verification with modern JWT badges (native JWT authorizer), basic routing, and forwarding to the kitchen -- at 71% less cost and with noticeably faster processing. But it has no filing cabinet (no caching), no membership program (no usage plans), no security checkpoint (no WAF), no luggage transformation service (no VTL), no controlled renovation reveals (no canary), and no staff-only entrance (no private endpoints).

**The technical reality**: HTTP API (API Gateway v2) is optimized for Lambda and HTTP backends:
- **Native JWT authorization** with any OIDC provider (Cognito, Auth0, Okta, Firebase Auth) -- no Lambda authorizer needed for JWT validation
- Lower latency: ~10ms overhead at p99 vs tens of milliseconds for REST API
- Automatic deployments (no manual deployment step required)
- Simplified CORS configuration (one-click)
- Lambda payload format version 2.0 (simplified event structure -- `routeKey` instead of `resource`, `cookies` as a separate field, no `multiValueHeaders`, `rawPath` and `rawQueryString` fields; important when migrating from REST to HTTP API as the Lambda handler must handle the different event shape)
- VPC Link to ALB, NLB, or AWS Cloud Map (REST API VPC Link only supports NLB)
- $1.00 per million requests (up to 71% cheaper than REST API)

**What HTTP API does NOT support**:
- Caching, WAF, usage plans/API keys, request validation
- VTL mapping templates, mock integrations
- Canary deployments, edge-optimized endpoints, private endpoints
- Lambda authorizer TOKEN type (only REQUEST type)
- SDK generation, gateway response customization

### WebSocket API -- The Telephone Switchboard

**The analogy**: The telephone switchboard desk maintains open phone lines between guests and hotel staff. Unlike the other desks where guests walk up, ask a question, get an answer, and leave (request-response), the switchboard keeps the line open for continuous two-way conversation. When a guest picks up the phone ($connect), the operator registers their line number (connection ID) in a directory (DynamoDB). When the guest speaks ($default or custom routes), the operator routes the message to the right department. When the kitchen has an update, it can call the guest back on their registered line (@connections callback API). When the guest hangs up ($disconnect), the operator removes them from the directory.

**The technical reality**: WebSocket APIs enable persistent, bidirectional communication:

**Three special routes**:
- `$connect` -- Triggered when a client initiates a WebSocket connection. Use for authentication and storing the connection ID in DynamoDB. If the integration returns an error, the connection is rejected.
- `$disconnect` -- Triggered on disconnect. Use for cleanup (delete connection ID from DynamoDB). Best-effort delivery -- not guaranteed if client drops abruptly.
- `$default` -- Fallback route when no custom route matches. Required if you want a catch-all handler.

**Custom routes** are selected using a `routeSelectionExpression` evaluated against the message body. For example, `$request.body.action` routes `{"action": "sendMessage", "text": "hello"}` to the `sendMessage` route integration.

**The @connections callback API** lets backends push messages to specific connected clients:
```
POST https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/@connections/{connectionId}
```
This is how server-initiated push works -- Lambda receives a message via a custom route, processes it, then uses the @connections API to send responses to one or many connected clients by their stored connection IDs.

**Connection limits**: 10-minute idle timeout, 2-hour maximum connection duration (clients must reconnect), 128 KB maximum frame size, 32 KB maximum payload with route selection.

**Pricing**: $1.00 per million messages + $0.25 per million connection minutes.

### The Decision Matrix

| Feature | REST API | HTTP API | WebSocket API | Lambda Function URLs |
|---------|----------|----------|---------------|---------------------|
| **Cost per million requests** | $3.50 | $1.00 | $1.00 msgs + $0.25/M conn-min | Free |
| **Latency overhead** | Tens of ms | ~10ms p99 | Persistent connection | ~0ms |
| **Caching** | Yes (0.5-237 GB) | No | No | No |
| **WAF** | Yes | No | No | No |
| **Usage plans / API keys** | Yes | No | No | No |
| **Request validation** | Yes (JSON Schema) | No | No | No |
| **VTL transformations** | Yes | No | No | No |
| **Lambda authorizer** | TOKEN + REQUEST | REQUEST only | REQUEST only | No (IAM only) |
| **Cognito authorizer** | Yes | Native JWT (any OIDC) | No | No |
| **Canary deployments** | Yes | No | No | No |
| **Edge-optimized endpoint** | Yes | No | No | No |
| **Private endpoint (VPC)** | Yes | No | No | No |
| **Custom domains** | Yes | Yes | Yes | No |
| **Mutual TLS (mTLS)** | Yes | Yes | No | No |
| **Throttling** | Per-method, per-client | Route-level only | Route-level | No (use reserved concurrency) |
| **Max payload** | 10 MB | 10 MB | 128 KB frame | 6 MB (streaming: 20 MB) |
| **Max timeout** | 29s default (adjustable; 15 min with streaming) | 30 seconds | 2 hours (conn lifetime) | 15 minutes (Lambda limit) |

**Decision framework**: Start with **HTTP API** as the default for new serverless APIs (cheaper, faster, JWT auth covers most needs). Upgrade to **REST API** when you need caching, WAF, usage plans, request validation, VTL transformations, private endpoints, or canary deployments. Use **Lambda Function URLs** for free internal webhooks and simple endpoints where you do not need gateway features. Use **WebSocket API** for real-time bidirectional communication (chat, dashboards, notifications).

---

## Part 2: Stages and Stage Variables -- The Hotel Floors

### Stages -- Deployment Environments

**The analogy**: Each floor of the hotel is a **stage** -- the lobby (dev), the mezzanine (staging), and the penthouse (prod). Each floor has its own guest policies, speed limits, filing cabinets, and logging cameras. When you design a new concierge protocol (API update), you create a **deployment** -- a snapshot-in-time of the protocol -- and assign it to a specific floor. The lobby might run tonight's experimental protocol while the penthouse runs last week's proven one.

**The technical reality**: A **deployment** is an immutable snapshot of your API configuration. A **stage** is a named reference (dev, staging, prod) that points to a specific deployment. You can have multiple stages pointing to different deployments, each with independent configuration:

```
┌─────────────────────────────────────────────────────────┐
│                    REST API (api-id)                      │
│                                                          │
│  Deployment A (2026-04-01)    Deployment B (2026-04-07)  │
│       │                              │                   │
│       ▼                              ▼                   │
│  ┌─────────┐                   ┌──────────┐              │
│  │ dev     │                   │ prod     │              │
│  │ stage   │                   │ stage    │              │
│  │─────────│                   │──────────│              │
│  │ Cache: OFF                  │ Cache: 0.5GB            │
│  │ Throttle: 100 RPS           │ Throttle: 5000 RPS      │
│  │ Logging: ALL                │ Logging: ERROR           │
│  │ Variables:                  │ Variables:               │
│  │   lambdaAlias=dev           │   lambdaAlias=prod       │
│  │   dbEndpoint=dev-db         │   dbEndpoint=prod-db     │
│  └─────────┘                   └──────────┘              │
│                                                          │
│  Invoke URL:                                             │
│  https://{api-id}.execute-api.{region}.amazonaws.com/dev │
│  https://{api-id}.execute-api.{region}.amazonaws.com/prod│
└──────────────────────────────────────────────────────────┘
```

### Stage Variables -- Configuration per Floor

Stage variables are key-value pairs available within the API configuration. The most powerful pattern is using them in integration URIs to route to different Lambda aliases per stage:

```
Integration URI with stage variable:
arn:aws:lambda:{region}:{account}:function:order-processor:${stageVariables.lambdaAlias}

dev stage:  lambdaAlias = dev   → invokes order-processor:dev alias
prod stage: lambdaAlias = prod  → invokes order-processor:prod alias
```

This pattern connects yesterday's Lambda alias concept to today's API Gateway stages: one API definition, multiple stages, each routing to different Lambda versions via aliases. The API configuration is identical -- only the stage variable changes the target.

---

## Part 3: Throttling -- The Token Bucket Rate Limiter

### The Token Bucket Algorithm

**The analogy**: Picture a bucket that holds tokens. The bucket refills at a steady rate (the **rate limit** -- tokens per second). Each incoming request takes one token. If the bucket is full, excess tokens overflow (the bucket has a maximum capacity called the **burst limit**). If the bucket is empty, the guest is turned away with a 429 "Too Many Requests" response. The key insight: burst capacity allows brief traffic spikes above the sustained rate, but only until the accumulated tokens are depleted.

**The technical reality**: API Gateway applies throttling at four tiers, evaluated in this priority order:

```
THROTTLING HIERARCHY (highest priority first)
══════════════════════════════════════════════════════════════════════

  TIER 1: Per-client, per-method throttle (Usage Plan)
  ┌──────────────────────────────────────────────────────┐
  │ API key "partner-abc" → GET /orders: 50 RPS, 25 burst│
  │ Most granular. Applied FIRST.                        │
  └──────────────────┬───────────────────────────────────┘
                     │ if no usage plan match, check ▼
  TIER 2: Per-method throttle (Stage Method Settings)
  ┌──────────────────────────────────────────────────────┐
  │ GET /orders: 500 RPS, 250 burst                      │
  │ Protects specific expensive endpoints                │
  └──────────────────┬───────────────────────────────────┘
                     │ if no method override, check ▼
  TIER 3: Stage-level default throttle
  ┌──────────────────────────────────────────────────────┐
  │ Stage "prod": 1000 RPS default, 500 burst default    │
  │ Blanket limit for all methods on this stage          │
  └──────────────────┬───────────────────────────────────┘
                     │ cannot exceed ▼
  TIER 4: Account-level limit (per Region)
  ┌──────────────────────────────────────────────────────┐
  │ 10,000 RPS steady-state (soft -- requestable increase│
  │   via Service Quotas)                                │
  │ 5,000 burst (HARD -- NOT adjustable)                 │
  │ Shared across ALL APIs in the account + region       │
  └──────────────────────────────────────────────────────┘

  KEY RULE: A lower-tier limit CANNOT exceed a higher-tier limit.
  If account limit is 10,000 RPS, setting a stage to 20,000 has no effect.
```

**Critical behavior**: Throttled requests receive **429 Too Many Requests**. For Lambda proxy integrations, this is returned BEFORE Lambda is invoked -- throttling protects your backend from overload.

**Throttling and Lambda concurrency interaction** (connecting yesterday's doc): API Gateway throttling and Lambda reserved concurrency are independent controls. If your API Gateway stage allows 10,000 RPS but your Lambda has reserved concurrency of 50, the effective limit is 50 concurrent executions -- excess requests receive 502/503 from Lambda, not 429 from API Gateway. For proper capacity planning, set API Gateway throttle limits to match or stay below your Lambda concurrency capacity to ensure callers receive clean 429 responses rather than 502 errors.

**The burst vs rate distinction**:
- **Rate** (tokens per second): Sustained throughput. The bucket refills at this rate.
- **Burst** (bucket size): Maximum instantaneous capacity. Handles traffic spikes.
- Example: Rate=100, Burst=200. A sudden spike of 200 requests is served immediately (draining the bucket). Then the rate is limited to 100/second until the bucket refills.

---

## Part 4: Caching -- The Filing Cabinet

**The analogy**: Behind the concierge desk sits a filing cabinet. When a guest asks "What is the weather forecast?" the concierge checks the cabinet first. If a recent answer exists (cache hit), the concierge hands it over without calling the weather service. If not (cache miss), the concierge calls the service, gives the answer to the guest, and files a copy for the next person who asks. The cabinet has limited space (0.5-237 GB), answers expire after a set time (TTL), and VIP guests with the right credentials can demand a fresh answer by saying "I want a new forecast, not the cached one" (cache invalidation).

**The technical reality**: Caching is available on **REST API only** and is configured per stage.

**Cache behavior**:
- Only **GET methods** are cached by default (you can enable caching on other methods but it is unusual)
- Cache key is formed from: URL path + configured query string parameters + configured headers
- Default TTL: 300 seconds (5 minutes), configurable 0-3600 seconds (1 hour max)
- Cache sizes: 0.5 GB, 1.6 GB, 6.1 GB, 13.5 GB, 28.4 GB, 58.2 GB, 118 GB, 237 GB
- Creating or deleting a cache takes approximately 4 minutes
- **Changing cache capacity deletes all cached data** -- plan capacity changes during maintenance windows

**Cache invalidation**:
- Clients send the header `Cache-Control: max-age=0`
- This requires the caller to have `execute-api:InvalidateCache` IAM permission
- If you do not require authorization for invalidation, **any client can flush your cache** -- this is a security and cost risk (every invalidation causes a backend invocation)
- You can require authorization by setting `Require authorization` on the stage cache settings

**Cache key parameters** -- controlling what makes a cache entry unique:
```
GET /products?category=electronics&sort=price
                ────────┬────────  ────┬────
                        │              │
    If both are cache key params: separate cache entries for
    ?category=electronics&sort=price  vs  ?category=books&sort=name

    If neither is a cache key param: ALL queries to /products
    return the same cached response (regardless of query params)
```

**Monitoring**: `CacheHitCount` and `CacheMissCount` CloudWatch metrics tell you if your cache is effective. A low hit ratio suggests your cache keys are too granular or your TTL is too short.

**Pricing**: Cache is charged hourly by size, regardless of utilization. A 0.5 GB cache in us-east-1 costs ~$0.020/hour (~$14.60/month). This is NOT included in the free tier.

---

## Part 5: Authorization -- The Security Apparatus

### Lambda Authorizers -- The Custom Bouncers

**The analogy**: You hire a bouncer (Lambda function) who stands at the door. When a guest arrives, the bouncer either examines just their badge (**TOKEN type**) or their entire dossier -- badge, ID card, the floor they are visiting, and who referred them (**REQUEST type**). The bouncer then writes a verdict on a slip of paper (IAM policy document) saying which rooms the guest can enter. To avoid checking every returning guest, the bouncer keeps a memory book (authorization cache) -- if the same badge was approved 2 minutes ago, the guest walks right in.

**TOKEN type** (REST API only):
- Receives a single bearer token from a specified header (typically `Authorization`)
- Caches the policy by the token value
- Use case: Simple JWT or custom token validation

**REQUEST type** (REST API and HTTP API):
- Receives the full request context: headers, query strings, path parameters, stage variables, request context
- Caches by a combination of configured identity sources (e.g., `method.request.header.Authorization, method.request.querystring.petId`)
- Use case: Authorization decisions based on multiple request attributes

**The Lambda authorizer returns three things**:
1. **principalId** -- Identifies the caller (e.g., user ID)
2. **policyDocument** -- IAM policy with Allow/Deny on specific API resources
3. **context** (optional) -- Key-value pairs injected into the integration request, accessible by the backend Lambda without re-parsing the token

```python
# Lambda authorizer example (REQUEST type)
def handler(event, context):
    token = event['headers'].get('Authorization', '')
    
    # Your custom auth logic here (validate JWT, check database, etc.)
    user = validate_token(token)
    
    if user:
        return {
            'principalId': user['id'],
            'policyDocument': {
                'Version': '2012-10-17',
                'Statement': [{
                    'Action': 'execute-api:Invoke',
                    'Effect': 'Allow',
                    'Resource': event['methodArn']  # Or use wildcard for broader access
                }]
            },
            'context': {
                'userId': user['id'],        # Passed to backend Lambda
                'userRole': user['role'],     # via event.requestContext.authorizer
                'tenantId': user['tenantId']  # No need to re-parse token downstream
            }
        }
    else:
        raise Exception('Unauthorized')  # Returns 401
```

**Authorization caching**:
- Default TTL: 300 seconds (5 minutes)
- Maximum TTL: 3600 seconds (1 hour)
- Setting TTL to 0 disables caching (every request invokes the authorizer Lambda)
- **Gotcha**: TOKEN authorizer caches by token value. If two users have the same token (impossible with proper JWT, but possible with weak token schemes), one user's policy is returned for the other.
- **Gotcha**: REQUEST authorizer caches by the configured identity sources. If you cache by `header.Authorization` alone but your authorization logic also depends on the path, you will get stale/wrong policies for different paths with the same token.

### Cognito User Pool Authorizers -- The Badge Scanner

**The analogy**: Instead of hiring a bouncer, you install an automated badge scanner (Cognito authorizer) at the door. Guests swipe their Cognito JWT badge, the scanner verifies the photo matches (signature validation), checks it has not expired (expiration check), and optionally verifies VIP status (OAuth scope validation). No Lambda function is invoked -- the scanning is built into the door.

**The technical reality**: Cognito authorizers validate JWT tokens issued by Amazon Cognito User Pools directly at the API Gateway layer.

- **No Lambda invocation** -- lower latency, no Lambda cost
- Validates: token signature, expiration, issuer, and optionally required OAuth scopes
- Does NOT support custom authorization logic (e.g., "is this user allowed to access THIS specific resource?")
- REST API: Cognito User Pool authorizer type
- HTTP API: Native JWT authorizer (works with ANY OIDC provider, not just Cognito)

**Authorization decision framework**:

| Scenario | Recommended Authorizer |
|----------|----------------------|
| Cognito users, need only token validation | Cognito/JWT authorizer (no Lambda cost) |
| Any OIDC provider (Auth0, Okta), HTTP API | HTTP API JWT authorizer (native, no Lambda) |
| Custom auth logic (check DB, ABAC, multi-tenant) | Lambda authorizer (REQUEST type) |
| AWS service-to-service (IAM credentials) | IAM authorization |
| Public API with API key identification | API key (not auth -- combine with another method) |

### IAM Authorization

Callers sign requests with AWS Signature Version 4. API Gateway validates the signature and evaluates the caller's IAM policy against the API resource ARN. Best for service-to-service calls where the caller has IAM credentials (e.g., another Lambda function, an EC2 instance, cross-account access).

---

## Part 6: Usage Plans and API Keys -- The Membership Program

**The analogy**: The hotel runs a tiered membership program. **Silver** members (free tier partners) get 1,000 requests per day at 10 requests per second. **Gold** members (paying customers) get 50,000 requests per day at 100 requests per second. Each member carries a **membership card** (API key) that the concierge scans to identify their tier. The card does NOT verify their identity (that is the bouncer's job) -- it only determines their rate limit and quota.

**The technical reality**:

```
USAGE PLAN ARCHITECTURE
══════════════════════════════════════════════════════════════════════

  ┌─── Usage Plan: "Silver" ──────────────────────────────────┐
  │  Throttle: 10 RPS rate, 20 burst                          │
  │  Quota: 1,000 requests/day                                │
  │  API Stages: [api-123/prod, api-456/prod]                 │
  │  Method Throttle: POST /orders → 5 RPS (override)         │
  │                                                            │
  │  ┌── API Key: partner-abc ──┐  ┌── API Key: partner-def ─┐│
  │  │  x-api-key: abc123...    │  │  x-api-key: def456...   ││
  │  └──────────────────────────┘  └──────────────────────────┘│
  └────────────────────────────────────────────────────────────┘

  ┌─── Usage Plan: "Gold" ────────────────────────────────────┐
  │  Throttle: 100 RPS rate, 200 burst                        │
  │  Quota: 50,000 requests/day                               │
  │  API Stages: [api-123/prod]                               │
  │                                                            │
  │  ┌── API Key: enterprise-x ─┐                             │
  │  │  x-api-key: ent789...    │                             │
  │  └──────────────────────────┘                             │
  └────────────────────────────────────────────────────────────┘
```

**Critical rule: API keys are NOT for authentication.** They are identification tokens that map callers to usage plans. API keys are sent in cleartext in the `x-api-key` header. Anyone who obtains the key can use it. Always combine API keys with a proper authentication mechanism (Lambda authorizer, Cognito, IAM).

**Quota behavior**: When a client exhausts their quota, they receive `429 Too Many Requests` with a `Retry-After` header indicating when the quota resets. Quotas reset at midnight UTC for daily quotas, or at the start of the period for weekly/monthly.

---

## Part 7: Integration Types -- How the Concierge Talks to the Kitchen

**The analogy**: The concierge has five ways to communicate with backends:

1. **AWS_PROXY (Lambda Proxy)** -- Hand the guest's entire request slip directly to the chef. The chef reads everything (headers, body, path, query params) and writes the complete response (status code, headers, body). The concierge does zero translation. This is the recommended default for Lambda.

2. **AWS (Lambda Non-Proxy / AWS Service)** -- The concierge translates the guest's French request into English (VTL mapping template), hands it to the chef (or directly to the AWS service -- e.g., DynamoDB PutItem, SQS SendMessage, Step Functions StartExecution -- skipping the chef entirely), then translates the English response back to French for the guest.

3. **HTTP_PROXY** -- Forward the guest's request as-is to an external caterer. No translation.

4. **HTTP** -- Translate the request, forward to the external caterer, translate the response back.

5. **MOCK** -- The concierge answers from a prepared script without calling anyone. No backend is invoked.

### AWS_PROXY (Lambda Proxy Integration) -- The 90% Case

This is the integration you will use for the vast majority of Lambda-backed APIs. API Gateway passes the entire HTTP request as a structured JSON event to Lambda, and Lambda must return a response object with a specific structure.

**What Lambda receives** (REST API v1.0 payload format):
```json
{
  "resource": "/orders/{orderId}",
  "path": "/orders/12345",
  "httpMethod": "GET",
  "headers": { "Authorization": "Bearer eyJ...", "Content-Type": "application/json" },
  "queryStringParameters": { "include": "items" },
  "pathParameters": { "orderId": "12345" },
  "stageVariables": { "lambdaAlias": "prod" },
  "requestContext": {
    "authorizer": {
      "userId": "user-789",
      "userRole": "admin"
    },
    "identity": { "sourceIp": "203.0.113.1" },
    "stage": "prod"
  },
  "body": null,
  "isBase64Encoded": false
}
```

**What Lambda must return**:
```json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json",
    "Access-Control-Allow-Origin": "*"
  },
  "body": "{\"orderId\": \"12345\", \"status\": \"shipped\"}",
  "isBase64Encoded": false
}
```

**Key constraint**: The `body` must be a **string** (not a JSON object). For JSON responses, you must `JSON.stringify()` or `json.dumps()` the body. Returning a raw object causes a 502 Bad Gateway -- one of the most common API Gateway debugging issues.

### AWS (Non-Proxy / Direct AWS Service Integration)

This is the cost-optimization pattern: **skip Lambda entirely** and have API Gateway call AWS services directly via VTL mapping templates.

```
DIRECT AWS SERVICE INTEGRATION (no Lambda)
══════════════════════════════════════════════════════════════════════

  ┌────────┐     ┌────────────────┐     ┌──────────┐
  │ Client │────▶│  API Gateway   │────▶│ DynamoDB │  No Lambda
  │ POST   │     │  VTL template  │     │ PutItem  │  invocation!
  │/orders │     │  transforms    │     │          │
  │        │◀────│  request →     │◀────│          │
  └────────┘     │  DynamoDB fmt  │     └──────────┘
                 └────────────────┘

  Cost savings: Eliminates Lambda invocation charges + cold start latency
  Trade-off: VTL is harder to debug than Lambda code
  Best for: Simple CRUD, proxying to SQS/SNS/Step Functions
```

Example VTL mapping template for DynamoDB PutItem:
```velocity
#set($inputRoot = $input.path('$'))
{
  "TableName": "orders",
  "Item": {
    "PK": {"S": "$inputRoot.orderId"},
    "SK": {"S": "ORDER#$inputRoot.orderId"},
    "customerName": {"S": "$inputRoot.customerName"},
    "amount": {"N": "$inputRoot.amount"},
    "createdAt": {"S": "$context.requestTimeEpoch"}
  }
}
```

### MOCK Integration

Returns hardcoded responses without invoking any backend. Two primary use cases:
1. **CORS preflight responses**: The OPTIONS method returns CORS headers without hitting Lambda
2. **API prototyping**: Design API responses before the backend exists

---

## Part 8: Canary Deployments -- The Controlled Renovation Reveal

**The analogy**: The hotel is renovating one restaurant. Instead of closing the old restaurant and opening the new one overnight (risky), you send 10% of tonight's diners to the new restaurant (canary) while 90% still eat in the old one (production). Both restaurants keep separate review cards (CloudWatch log groups). If reviews are bad, you cancel the renovation and send everyone back to the old restaurant. If reviews are good, you open the new restaurant to everyone (promote the canary).

**The technical reality**: Canary deployments are a **stage-level** feature on REST APIs.

```
CANARY DEPLOYMENT ON A STAGE
══════════════════════════════════════════════════════════════════════

  "prod" Stage
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  ┌─── Production (90%) ───┐   ┌─── Canary (10%) ─────────┐ │
  │  │ Deployment A            │   │ Deployment B              │ │
  │  │ (current proven config) │   │ (new API changes)         │ │
  │  │                         │   │                           │ │
  │  │ Logs: API-Gateway-      │   │ Logs: API-Gateway-        │ │
  │  │   Execution-Logs/       │   │   Execution-Logs/         │ │
  │  │   {id}/prod             │   │   {id}/prod/Canary        │ │
  │  │                         │   │                           │ │
  │  │ Stage vars: inherited   │   │ Stage vars: can override  │ │
  │  └─────────────────────────┘   └───────────────────────────┘ │
  └──────────────────────────────────────────────────────────────┘
```

**Canary vs Lambda weighted alias routing** (connecting yesterday's Lambda doc):
- **API Gateway canary**: Tests the entire API configuration (new routes, changed authorizers, modified integrations, different stage variable values)
- **Lambda weighted alias**: Tests new function code behind the same API configuration
- In production, you might use both: Lambda alias routing for code changes, API Gateway canary for API structure changes

**Canary stage variable overrides**: The canary deployment can use different stage variable values than production. This lets you route canary traffic to a different Lambda alias, a different database endpoint, or enable feature flags -- all without changing the API definition.

---

## Part 9: Endpoint Types -- Where the Concierge Desk Sits

**The analogy**:
- **Regional**: The concierge desk is in the local hotel. Guests from the same city (same AWS region) walk in directly. If you want global guests, you build your own airport shuttle (CloudFront distribution) so you can customize the route.
- **Edge-Optimized**: The hotel chain automatically routes guests through the nearest airport (CloudFront edge location) before sending them to the main hotel. Convenient but you cannot customize the shuttle route -- AWS manages the CloudFront distribution for you.
- **Private**: The concierge desk is inside the building's private lobby. Only tenants (VPC resources) can reach it through the internal elevator (Interface VPC Endpoint). No external guests allowed.

| Endpoint Type | How Clients Reach It | CloudFront | Use Case |
|--------------|---------------------|------------|----------|
| **Regional** | Direct to API Gateway in one region | You manage your own (recommended) | Same-region clients, or when you want full CloudFront control |
| **Edge-Optimized** | Via AWS-managed CloudFront distribution | AWS-managed (limited control) | Default for REST API; legacy pattern; geographically distributed clients |
| **Private** | Via Interface VPC Endpoint only | N/A | Internal microservices, compliance-restricted APIs, backend-for-backend |

**Best practice**: Use **Regional** endpoints and front them with your own CloudFront distribution. This gives you full control over cache behaviors, custom headers, Lambda@Edge/CloudFront Functions, WAF at the CloudFront layer, and custom error pages. Edge-optimized endpoints use an AWS-managed CloudFront distribution that you cannot configure.

---

## Part 10: Private APIs and Resource Policies -- The Staff-Only Entrance

### Private APIs

**The analogy**: The concierge desk is inside a restricted building. Only tenants with access to the internal elevator (Interface VPC Endpoint for execute-api) can reach it. Even if someone knows the desk's address, they cannot walk in from the street.

**The technical reality**: Private REST APIs are accessible only through Interface VPC Endpoints:

1. Create an Interface VPC Endpoint for `execute-api` in your VPC
2. Create a REST API with endpoint type `PRIVATE`
3. Attach a **resource policy** that allows access from the VPC endpoint

**Invoke URL**: `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}` -- same format as public APIs, but DNS resolution only works from within the VPC (or via the VPC endpoint's DNS).

### Resource Policies

**The analogy**: The posted rules at the hotel entrance: "Entry permitted for residents of Building A and visitors from the downtown district. All others must present special authorization."

Resource policies are JSON documents (IAM policy syntax) attached to the API that control who can invoke it. They work independently of method-level authorization.

**Common patterns**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:{region}:{account}:{api-id}/*",
      "Condition": {
        "StringEquals": {
          "aws:sourceVpce": "vpce-0123456789abcdef0"
        }
      }
    }
  ]
}
```

**Resource policy + IAM authorization interaction**: The evaluation model differs based on whether the caller is in the **same AWS account** or a **different account**. An explicit Deny in either policy always wins regardless of account context.

**Same-account callers (union / OR logic)**: An explicit Allow from EITHER the resource policy OR the IAM policy is sufficient. The resource policy being silent (implicit deny) does NOT block the request if the IAM policy allows it. This follows standard IAM same-account evaluation -- each policy is independently sufficient.

| Resource Policy Result | IAM Auth Result | Final Decision (Same-Account) |
|-----------------------|-----------------|-------------------------------|
| Allow | Allow | **Allow** |
| Allow | Neither Allow nor Deny | **Allow** |
| Neither Allow nor Deny | Allow | **Allow** (IAM Allow is sufficient on its own) |
| Allow | Deny | **Deny** (explicit Deny wins) |
| Deny | Allow | **Deny** (explicit Deny wins) |
| Neither Allow nor Deny | Neither Allow nor Deny | **Deny** (nothing allows) |

**Cross-account callers (intersection / AND logic)**: BOTH the resource policy AND the caller's IAM policy must explicitly Allow. This follows the same cross-account pattern as S3 bucket policies -- the resource owner must grant access in the resource policy, AND the caller's account must grant access in the IAM policy.

| Resource Policy Result | IAM Auth Result | Final Decision (Cross-Account) |
|-----------------------|-----------------|--------------------------------|
| Allow | Allow | **Allow** |
| Allow | Neither Allow nor Deny | **Deny** (both must explicitly Allow) |
| Neither Allow nor Deny | Allow | **Deny** (both must explicitly Allow) |
| Deny | Allow | **Deny** (explicit Deny wins) |

**Common interview trap**: Candidates often state "both must Allow" without specifying the account context. For same-account, the IAM Allow alone is sufficient -- a resource policy that does not match the request does NOT result in a deny. For cross-account, both must explicitly Allow. Getting this distinction wrong leads to over-restrictive or under-restrictive API security designs.

**VPC endpoint policy -- the hidden fourth layer**: When calling a Private API or any API through an Interface VPC Endpoint, the VPC endpoint itself has a resource-based policy that is evaluated BEFORE the request reaches API Gateway. If this policy does not permit the API, the request is rejected with 403 at the endpoint layer -- the API's resource policy and IAM authorization are never consulted. This is the most common cause of "everything looks right but I still get 403" when accessing APIs through VPC endpoints. Diagnose by checking for the absence of `x-amzn-RequestId` in the 403 response (the request never reached API Gateway) and no entries in CloudWatch access/execution logs.

---

## Part 11: Security Features -- WAF, mTLS, and CORS

### WAF Integration

**The analogy**: A security checkpoint installed BEFORE the concierge desk. Guards check guests against a blocklist (IP rules), scan for contraband (SQL injection, XSS payloads), enforce visit frequency limits (rate-based rules), and block guests from restricted countries (geo-blocking). If a guest passes the checkpoint, they proceed to the concierge. If not, they are turned away before consuming any concierge resources.

**The technical reality**: AWS WAF web ACLs attach to **REST API stages only** (not HTTP API, not WebSocket). WAF rules evaluate before any API Gateway processing, so blocked requests do not count against your throttle limits or invoke authorizers/backends.

Common WAF rules for API Gateway:
- **AWS Managed Rules**: AWSManagedRulesCommonRuleSet (OWASP Top 10), AWSManagedRulesSQLiRuleSet, AWSManagedRulesKnownBadInputsRuleSet
- **Rate-based rules**: Block IPs exceeding N requests per 5-minute window
- **IP allowlist/blocklist**: Restrict access to known IP ranges
- **Geo-blocking**: Block or allow by country

### Mutual TLS (mTLS)

**The analogy**: Normally, only the guest verifies the hotel's identity (standard TLS -- the hotel's certificate). With mutual TLS, both sides verify each other: the hotel checks the guest's identity card (client certificate) against a trusted list (truststore), and the guest verifies the hotel's certificate. This is the highest-assurance identity verification -- used for B2B integrations and regulated industries.

**The technical reality**:
- Upload a truststore (PEM bundle of trusted CA certificates) to S3
- Configure the custom domain name with the truststore S3 URI
- Clients must present a certificate signed by one of the trusted CAs
- Works with REST API and HTTP API
- Requires a custom domain name (cannot use the default execute-api domain)
- Client certificate details (subject, issuer, serial number, validity) are available in `$context.identity.clientCert` for downstream processing

### CORS Configuration

**REST API**: Manual. You must configure CORS headers on each method AND create an OPTIONS method with a MOCK integration that returns the appropriate headers. This is tedious and error-prone.

**HTTP API**: One-click. Configure allowed origins, methods, and headers at the API level. API Gateway automatically handles OPTIONS preflight requests.

```
CORS FLOW
══════════════════════════════════════════════════════════════════════

  Browser                    API Gateway               Backend
    │                            │                         │
    │── OPTIONS /orders ────────▶│                         │
    │   (preflight check)        │  Check CORS config      │
    │                            │  (MOCK integration for  │
    │◀── 200 + CORS headers ─────│   REST API -- backend   │
    │   Access-Control-Allow-    │   is NOT invoked)       │
    │   Origin: https://app.com  │                         │
    │   Access-Control-Allow-    │                         │
    │   Methods: GET, POST       │                         │
    │                            │                         │
    │── GET /orders ────────────▶│────────────────────────▶│
    │   Origin: https://app.com  │                         │
    │                            │◀────────────────────────│
    │◀── 200 + CORS headers ─────│                         │
    │   + response body          │                         │
```

---

## Part 12: Request Validation and Gateway Responses

### Request Validation (REST API Only)

**The analogy**: The concierge checks that the guest's request form is filled out correctly BEFORE disturbing the kitchen. "You forgot to write your room number." "The 'amount' field must be a number, not a word." This prevents wasted kitchen time (Lambda invocations) on malformed requests.

**The technical reality**: API Gateway can validate:
- **Required parameters**: Query strings, headers, path parameters marked as required
- **Request body**: Validated against a JSON Schema model

```json
{
  "type": "object",
  "required": ["orderId", "items"],
  "properties": {
    "orderId": { "type": "string", "pattern": "^ORD-[0-9]{6}$" },
    "items": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["productId", "quantity"],
        "properties": {
          "productId": { "type": "string" },
          "quantity": { "type": "integer", "minimum": 1 }
        }
      }
    }
  }
}
```

Validation failures return **400 Bad Request** with a message like `"Invalid request body"` before the backend is invoked. This saves Lambda invocation costs and reduces backend error noise.

### Gateway Responses

**The analogy**: Custom rejection letters. Instead of the concierge handing a generic "Access Denied" card, they hand a branded card: "Dear guest, your membership has expired. Please visit the front desk to renew. Error reference: GW-4032."

You can customize the error responses that API Gateway itself generates (not your backend's errors):
- `DEFAULT_4XX` / `DEFAULT_5XX` -- catch-all for client/server errors
- `UNAUTHORIZED` (401) -- authorization failure
- `ACCESS_DENIED` (403) -- resource policy or WAF denial
- `THROTTLED` (429) -- rate limit exceeded
- `QUOTA_EXCEEDED` (429) -- usage plan quota exhausted
- `MISSING_AUTHENTICATION_TOKEN` (403) -- no auth credentials provided
- `INTEGRATION_FAILURE` (504) -- backend did not respond in time
- `BAD_REQUEST_BODY` (400) -- request validation failure

Each gateway response can have custom headers, a custom status code, and a custom body template (VTL) with access to `$context` variables like `$context.error.message` and `$context.requestId`.

---

## Part 13: Observability -- Access Logging vs Execution Logging

**The analogy**: The guest register (access log) at the front desk records every visitor: name, arrival time, which room they visited, and whether they were admitted. It is structured and compact. The security camera footage (execution log) records the concierge's every step: checking the ID, looking up the membership, calling the kitchen, waiting for the response, transforming the answer. It is detailed, noisy, and expensive to store.

### Access Logging

- **What**: One log entry per request with structured data
- **Format**: Customizable using `$context` variables (CLF, JSON, CSV, XML -- you design the format)
- **Destination**: CloudWatch Logs (or Kinesis Data Firehose for REST API)
- **Cost**: Low (one line per request)
- **Use case**: Monitoring, dashboards, alerting, analytics

Recommended JSON access log format:
```json
{
  "requestId": "$context.requestId",
  "ip": "$context.identity.sourceIp",
  "requestTime": "$context.requestTime",
  "httpMethod": "$context.httpMethod",
  "resourcePath": "$context.resourcePath",
  "status": "$context.status",
  "responseLength": "$context.responseLength",
  "integrationLatency": "$context.integrationLatency",
  "responseLatency": "$context.responseLatency",
  "errorMessage": "$context.error.message",
  "authorizerLatency": "$context.authorize.latency",
  "apiKeyId": "$context.identity.apiKeyId"
}
```

### Execution Logging

- **What**: Detailed debug trace of every step in the request pipeline
- **Detail**: Shows authorizer input/output, mapping template evaluation, integration request/response, cache lookup results
- **Cost**: High (multiple log entries per request, verbose output)
- **Use case**: Debugging specific request failures -- NOT production always-on
- **Levels**: ERROR (errors only) or INFO (full pipeline trace)

**Prerequisite**: Execution logging requires a CloudWatch Logs IAM role configured at the account level via `aws_api_gateway_account`. This is a region-wide singleton -- without it, enabling logging on a stage silently fails. This is the #1 reason people cannot get execution logs working. See the Terraform example for the `aws_api_gateway_account` resource.

**Best practice**: Enable access logging in JSON format on all stages. Enable execution logging at ERROR level in production. Switch to INFO level temporarily when debugging specific issues. Never leave INFO-level execution logging on permanently -- it generates enormous log volume and cost.

### Key CloudWatch Metrics

| Metric | What It Tells You |
|--------|------------------|
| `Count` | Total API requests |
| `4XXError` | Client errors (auth failures, validation errors, throttled) |
| `5XXError` | Server errors (backend failures, timeouts) |
| `Latency` | End-to-end time from request receipt to response sent |
| `IntegrationLatency` | Time spent waiting for backend (Lambda execution time) |
| `CacheHitCount` | Requests served from cache (REST API with caching) |
| `CacheMissCount` | Requests that missed cache and hit backend |

**The gap**: `Latency - IntegrationLatency = API Gateway overhead` (authorization, throttle checks, mapping templates, cache lookup). If this gap is large, investigate authorizer caching and mapping template complexity.

---

## Part 14: Binary Media Types, SDK Generation, and VPC Links

### Binary Media Types

By default, API Gateway treats all payloads as UTF-8 text. For binary content (images, PDFs, Protobuf, compressed data), you must register the content types as binary media types:

- Register types like `image/png`, `application/pdf`, `application/octet-stream`, or `*/*` (all types)
- API Gateway base64-encodes the binary payload in the `body` field of the Lambda event (`isBase64Encoded: true`)
- Lambda must base64-encode binary response bodies and set `isBase64Encoded: true` in the response
- The client must send the `Accept` header matching a registered binary media type

### SDK Generation (REST API Only)

API Gateway can auto-generate client SDKs from your REST API definition in Java, JavaScript, iOS (Swift/Objective-C), Android, and Ruby. The generated SDK includes method signatures for each API method, request signing (for IAM auth), and retry logic. Useful for distributing API access to partners who want a native SDK rather than raw HTTP calls.

### VPC Links -- Connecting to Private Backends

**The analogy**: A private underground tunnel from the concierge desk to a restaurant in a restricted building. Guests interact with the concierge (public API), but the kitchen is in a locked building (private subnet) accessible only through the tunnel (VPC Link).

- **REST API VPC Link**: Connects to an **NLB** (Network Load Balancer) in your VPC
- **HTTP API VPC Link**: Connects to an **ALB**, **NLB**, or **AWS Cloud Map** service in your VPC (more flexible)

The VPC Link enables API Gateway to integrate with ECS services, EC2 instances, or on-premises resources (via Direct Connect) in private subnets without exposing them to the internet.

---

## Part 15: API Gateway Limits Reference

| Limit | Value | Type |
|-------|-------|------|
| Account-level steady-state throttle (per region) | 10,000 RPS | Soft (requestable increase) |
| Account-level burst throttle (per region) | 5,000 requests | Hard (NOT adjustable) |
| Maximum integration timeout | REST: 29 seconds (default, adjustable via Service Quotas), HTTP: 30 seconds | Soft (REST Regional/Private), Hard (HTTP) |
| WebSocket connection duration | 2 hours max, 10 min idle timeout | Hard |
| WebSocket frame size | 128 KB | Hard |
| Maximum payload size | 10 MB (REST & HTTP) | Hard |
| Resource policy size | 8,192 bytes | Hard |
| Custom domain names per account per region | 120 | Soft |
| API keys per account per region | 10,000 | Soft |
| Usage plans per account per region | 300 | Soft |
| Usage plans per API key | 10 | Soft |
| APIs per account per region | 600 (REST + HTTP combined) | Soft |
| Resources per REST API | 300 | Hard |
| Stages per REST API | 10 | Soft |
| Cache size options | 0.5, 1.6, 6.1, 13.5, 28.4, 58.2, 118, 237 GB | Fixed |
| Cache TTL | 0-3600 seconds (default 300) | Configurable |
| Authorizer result TTL | 0-3600 seconds (default 300) | Configurable |
| Lambda authorizer timeout | 10 seconds | Hard |
| Mapping template size | 300 KB | Hard |
| Header values total size | 10,240 bytes | Hard |
| Canary traffic percentage | 0.0-100.0% | Configurable |
| WebSocket routes per API | 300 | Hard |
| mTLS truststore size | 1 MB PEM file, up to 1,000 CA certificates | Hard |

---

## Part 16: Pricing Model -- Pay Per Guest Interaction

**The analogy**: The full-service desk (REST API) charges $3.50 for every million guest interactions. The express desk (HTTP API) charges $1.00 for the same million -- 71% cheaper because it offers fewer services. The telephone switchboard (WebSocket) charges per message AND per connection minute. The filing cabinet (cache) costs extra hourly rent regardless of how many guests use it.

| Component | Price (us-east-1) | Free Tier |
|-----------|-------------------|-----------|
| **REST API requests** | $3.50 per million (first 333M), $2.80 (next 667M), $2.38 (next 19B) | 1 million requests/month for 12 months |
| **HTTP API requests** | $1.00 per million (first 300M), $0.90 thereafter | 1 million requests/month for 12 months |
| **WebSocket messages** | $1.00 per million messages | 1 million messages/month for 12 months |
| **WebSocket connection minutes** | $0.25 per million connection minutes | 2 million connection minutes/month for 12 months |
| **REST API cache** | $0.020/hour (0.5 GB) to $3.800/hour (237 GB) | None |
| **Data transfer** | Standard AWS data transfer rates apply | Standard free tier |

**Cost comparison example** (1 billion requests/month):
- REST API: ~$3,300/month (requests only, no cache)
- HTTP API: ~$1,000/month
- Savings with HTTP API: ~$2,300/month (70% less)

**Cost optimization strategies**:
1. **Use HTTP API when possible** -- 71% cheaper for simple proxy scenarios
2. **Enable caching on REST API** -- Cache reduces backend invocations (saves Lambda cost) but adds cache cluster cost; calculate the break-even
3. **Use direct AWS service integrations** -- Skip Lambda entirely for simple CRUD (API Gateway -> DynamoDB); eliminate Lambda invocation charges
4. **Request validation at the gateway** -- Reject malformed requests before they invoke Lambda
5. **Authorization caching** -- Reduce Lambda authorizer invocations with appropriate TTL

---

## Production Terraform Example

This Terraform configuration deploys a production-ready REST API with Lambda proxy integration, Lambda authorizer, usage plans, caching, canary deployment, WAF, and access logging, plus an HTTP API with JWT authorizer for comparison.

```hcl
# ═══════════════════════════════════════════════════════════════════════
# API GATEWAY DEEP DIVE -- PRODUCTION ARCHITECTURE
# ═══════════════════════════════════════════════════════════════════════
#
# Deploys:
# - REST API with Lambda proxy integration (prod stage)
# - Lambda authorizer with 300s cache TTL
# - Stage-level caching (0.5 GB, 300s TTL)
# - Method-level throttling (GET /orders: 500 RPS)
# - Usage plan with API key (Silver tier: 1000/day, 10 RPS)
# - Canary deployment (10% traffic)
# - WAF web ACL with rate limiting and managed rules
# - CloudWatch access logging (JSON format)
# - HTTP API with Cognito JWT authorizer for comparison
# ═══════════════════════════════════════════════════════════════════════

terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_region" "current" {}
data "aws_caller_identity" "current" {}

# ═══════════════════════════════════════════════════════════════════════
# REST API DEFINITION
# ═══════════════════════════════════════════════════════════════════════

resource "aws_api_gateway_rest_api" "orders" {
  name        = "orders-api"
  description = "Production orders API with Lambda proxy integration"

  endpoint_configuration {
    types = ["REGIONAL"]  # Use Regional + your own CloudFront for full control
  }

  # Binary media types -- enable for image/PDF uploads
  binary_media_types = ["application/octet-stream", "image/*"]
}

# /orders resource
resource "aws_api_gateway_resource" "orders" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  parent_id   = aws_api_gateway_rest_api.orders.root_resource_id
  path_part   = "orders"
}

# /orders/{orderId} resource
resource "aws_api_gateway_resource" "order_by_id" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  parent_id   = aws_api_gateway_resource.orders.id
  path_part   = "{orderId}"
}

# ═══════════════════════════════════════════════════════════════════════
# REQUEST VALIDATION MODEL
# ═══════════════════════════════════════════════════════════════════════

resource "aws_api_gateway_model" "create_order" {
  rest_api_id  = aws_api_gateway_rest_api.orders.id
  name         = "CreateOrderRequest"
  description  = "Validates POST /orders request body"
  content_type = "application/json"

  schema = jsonencode({
    type     = "object"
    required = ["items", "customerId"]
    properties = {
      customerId = { type = "string", pattern = "^CUS-[0-9]{6}$" }
      items = {
        type     = "array"
        minItems = 1
        items = {
          type     = "object"
          required = ["productId", "quantity"]
          properties = {
            productId = { type = "string" }
            quantity  = { type = "integer", minimum = 1, maximum = 100 }
          }
        }
      }
    }
  })
}

resource "aws_api_gateway_request_validator" "body_and_params" {
  rest_api_id                 = aws_api_gateway_rest_api.orders.id
  name                        = "validate-body-and-params"
  validate_request_body       = true
  validate_request_parameters = true
}

# ═══════════════════════════════════════════════════════════════════════
# LAMBDA AUTHORIZER
# ═══════════════════════════════════════════════════════════════════════

# Authorizer Lambda function (assume it exists -- focus on API Gateway config)
resource "aws_api_gateway_authorizer" "token_auth" {
  rest_api_id                      = aws_api_gateway_rest_api.orders.id
  name                             = "token-authorizer"
  type                             = "TOKEN"                                  # TOKEN or REQUEST
  authorizer_uri                   = "arn:aws:apigateway:${data.aws_region.current.name}:lambda:path/2015-03-31/functions/${var.authorizer_lambda_arn}/invocations"
  authorizer_credentials           = aws_iam_role.api_gateway_invocation.arn  # Role allowing API GW to invoke Lambda
  identity_source                  = "method.request.header.Authorization"    # Header containing the token
  authorizer_result_ttl_in_seconds = 300                                      # Cache auth result for 5 minutes
  # WARNING: TTL > 0 caches by token value. Ensure tokens are unique per user.
  # Set to 0 for development/debugging to disable caching.
}

# IAM role for API Gateway to invoke the authorizer Lambda
resource "aws_iam_role" "api_gateway_invocation" {
  name = "api-gateway-lambda-invocation"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "apigateway.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "invoke_authorizer" {
  name = "invoke-authorizer-lambda"
  role = aws_iam_role.api_gateway_invocation.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "lambda:InvokeFunction"
      Resource = var.authorizer_lambda_arn
    }]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# METHODS -- GET /orders and POST /orders with Lambda proxy
# ═══════════════════════════════════════════════════════════════════════

# GET /orders -- cached, uses Lambda authorizer
resource "aws_api_gateway_method" "get_orders" {
  rest_api_id   = aws_api_gateway_rest_api.orders.id
  resource_id   = aws_api_gateway_resource.orders.id
  http_method   = "GET"
  authorization = "CUSTOM"
  authorizer_id = aws_api_gateway_authorizer.token_auth.id

  # API key required (usage plan throttling applies)
  api_key_required = true

  # Request parameters (for cache key and validation)
  request_parameters = {
    "method.request.querystring.status"  = false  # Optional query param
    "method.request.querystring.limit"   = false
  }
}

resource "aws_api_gateway_integration" "get_orders" {
  rest_api_id             = aws_api_gateway_rest_api.orders.id
  resource_id             = aws_api_gateway_resource.orders.id
  http_method             = aws_api_gateway_method.get_orders.http_method
  type                    = "AWS_PROXY"
  integration_http_method = "POST"  # Lambda is always invoked via POST
  uri                     = "arn:aws:apigateway:${data.aws_region.current.name}:lambda:path/2015-03-31/functions/${var.backend_lambda_arn}:$${stageVariables.lambdaAlias}/invocations"
  # $$ escapes the dollar sign in Terraform; evaluates to ${stageVariables.lambdaAlias} at runtime
}

# POST /orders -- validated, uses Lambda authorizer
resource "aws_api_gateway_method" "post_orders" {
  rest_api_id          = aws_api_gateway_rest_api.orders.id
  resource_id          = aws_api_gateway_resource.orders.id
  http_method          = "POST"
  authorization        = "CUSTOM"
  authorizer_id        = aws_api_gateway_authorizer.token_auth.id
  api_key_required     = true
  request_validator_id = aws_api_gateway_request_validator.body_and_params.id

  request_models = {
    "application/json" = aws_api_gateway_model.create_order.name
  }
}

resource "aws_api_gateway_integration" "post_orders" {
  rest_api_id             = aws_api_gateway_rest_api.orders.id
  resource_id             = aws_api_gateway_resource.orders.id
  http_method             = aws_api_gateway_method.post_orders.http_method
  type                    = "AWS_PROXY"
  integration_http_method = "POST"
  uri                     = "arn:aws:apigateway:${data.aws_region.current.name}:lambda:path/2015-03-31/functions/${var.backend_lambda_arn}:$${stageVariables.lambdaAlias}/invocations"
}

# OPTIONS /orders -- CORS preflight (MOCK integration)
resource "aws_api_gateway_method" "options_orders" {
  rest_api_id   = aws_api_gateway_rest_api.orders.id
  resource_id   = aws_api_gateway_resource.orders.id
  http_method   = "OPTIONS"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "options_orders" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  resource_id = aws_api_gateway_resource.orders.id
  http_method = aws_api_gateway_method.options_orders.http_method
  type        = "MOCK"

  request_templates = {
    "application/json" = "{\"statusCode\": 200}"
  }
}

resource "aws_api_gateway_method_response" "options_200" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  resource_id = aws_api_gateway_resource.orders.id
  http_method = aws_api_gateway_method.options_orders.http_method
  status_code = "200"

  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin"  = true
    "method.response.header.Access-Control-Allow-Methods" = true
    "method.response.header.Access-Control-Allow-Headers" = true
  }
}

resource "aws_api_gateway_integration_response" "options_200" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  resource_id = aws_api_gateway_resource.orders.id
  http_method = aws_api_gateway_method.options_orders.http_method
  status_code = aws_api_gateway_method_response.options_200.status_code

  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin"  = "'https://app.example.com'"
    "method.response.header.Access-Control-Allow-Methods" = "'GET,POST,OPTIONS'"
    "method.response.header.Access-Control-Allow-Headers" = "'Content-Type,Authorization,x-api-key'"
  }

  depends_on = [aws_api_gateway_integration.options_orders]
}

# ═══════════════════════════════════════════════════════════════════════
# LAMBDA PERMISSIONS -- Allow API Gateway to invoke backend Lambda
# ═══════════════════════════════════════════════════════════════════════

resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = var.backend_lambda_arn
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.orders.execution_arn}/*"
  qualifier     = "prod"  # Allow invocation of the prod alias
}

# ═══════════════════════════════════════════════════════════════════════
# DEPLOYMENT AND STAGE -- With Caching and Canary
# ═══════════════════════════════════════════════════════════════════════

resource "aws_api_gateway_deployment" "orders" {
  rest_api_id = aws_api_gateway_rest_api.orders.id

  # Force redeployment when API resources change
  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.orders.id,
      aws_api_gateway_resource.order_by_id.id,
      aws_api_gateway_method.get_orders.id,
      aws_api_gateway_method.post_orders.id,
      aws_api_gateway_integration.get_orders.id,
      aws_api_gateway_integration.post_orders.id,
      aws_api_gateway_authorizer.token_auth.id,
    ]))
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_api_gateway_stage" "prod" {
  rest_api_id   = aws_api_gateway_rest_api.orders.id
  deployment_id = aws_api_gateway_deployment.orders.id
  stage_name    = "prod"
  description   = "Production stage with caching, throttling, and WAF"

  # Stage variables -- route to Lambda prod alias
  variables = {
    lambdaAlias = "prod"
    environment = "production"
  }

  # Enable caching on the stage
  cache_cluster_enabled = true
  cache_cluster_size    = "0.5"  # GB -- smallest cluster for cost efficiency

  # Canary deployment settings (10% traffic to new deployment)
  canary_settings {
    percent_traffic = 10.0

    # Override stage variables for canary traffic
    stage_variable_overrides = {
      lambdaAlias = "canary"  # Canary traffic goes to Lambda canary alias
    }
  }

  # Access logging
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_access_logs.arn
    format = jsonencode({
      requestId        = "$context.requestId"
      ip               = "$context.identity.sourceIp"
      requestTime      = "$context.requestTime"
      httpMethod       = "$context.httpMethod"
      resourcePath     = "$context.resourcePath"
      status           = "$context.status"
      responseLength   = "$context.responseLength"
      integrationLatency = "$context.integrationLatency"
      responseLatency  = "$context.responseLatency"
      errorMessage     = "$context.error.message"
      apiKeyId         = "$context.identity.apiKeyId"
    })
  }

  # X-Ray tracing
  xray_tracing_enabled = true

  depends_on = [aws_cloudwatch_log_group.api_access_logs]
}

# Per-method settings: caching and throttling overrides
resource "aws_api_gateway_method_settings" "get_orders" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  stage_name  = aws_api_gateway_stage.prod.stage_name
  method_path = "orders/GET"

  settings {
    # Caching: enable for GET /orders with specific cache key params
    caching_enabled      = true
    cache_ttl_in_seconds = 300  # 5 minutes
    cache_data_encrypted = true
    # Require IAM permission to invalidate cache
    require_authorization_for_cache_control = true

    # Throttling: method-level override
    throttling_rate_limit  = 500  # 500 RPS for GET /orders
    throttling_burst_limit = 250

    # Logging
    logging_level   = "ERROR"  # ERROR in prod, INFO for debugging
    data_trace_enabled = false # Do NOT log full request/response bodies in prod
    metrics_enabled = true
  }
}

# Stage-level default method settings
resource "aws_api_gateway_method_settings" "all_methods" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  stage_name  = aws_api_gateway_stage.prod.stage_name
  method_path = "*/*"

  settings {
    throttling_rate_limit  = 1000  # Default: 1000 RPS for all methods
    throttling_burst_limit = 500
    logging_level          = "ERROR"
    metrics_enabled        = true
  }
}

# ═══════════════════════════════════════════════════════════════════════
# USAGE PLAN AND API KEY -- Silver Tier
# ═══════════════════════════════════════════════════════════════════════

resource "aws_api_gateway_usage_plan" "silver" {
  name        = "silver-tier"
  description = "Silver tier: 1,000 requests/day, 10 RPS rate limit"

  api_stages {
    api_id = aws_api_gateway_rest_api.orders.id
    stage  = aws_api_gateway_stage.prod.stage_name

    # Per-method throttle override within this usage plan
    throttle {
      path        = "orders/POST"
      rate_limit  = 5     # POST /orders limited to 5 RPS for Silver tier
      burst_limit = 10
    }
  }

  # Global throttle for this plan
  throttle_settings {
    rate_limit  = 10   # 10 requests per second
    burst_limit = 20
  }

  # Quota
  quota_settings {
    limit  = 1000       # 1,000 requests per day
    period = "DAY"
  }
}

resource "aws_api_gateway_api_key" "partner_abc" {
  name    = "partner-abc"
  enabled = true
}

resource "aws_api_gateway_usage_plan_key" "partner_abc_silver" {
  key_id        = aws_api_gateway_api_key.partner_abc.id
  key_type      = "API_KEY"
  usage_plan_id = aws_api_gateway_usage_plan.silver.id
}

# ═══════════════════════════════════════════════════════════════════════
# WAF WEB ACL -- Attached to REST API Stage
# ═══════════════════════════════════════════════════════════════════════

resource "aws_wafv2_web_acl" "api_gateway" {
  name  = "orders-api-waf"
  scope = "REGIONAL"  # API Gateway uses REGIONAL (not CLOUDFRONT)

  default_action {
    allow {}
  }

  # Rule 1: AWS Managed Common Rule Set (OWASP Top 10)
  rule {
    name     = "aws-managed-common"
    priority = 1

    override_action {
      none {}  # Use rule group's actions as-is
    }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
    }
  }

  # Rule 2: Rate limiting (1000 requests per 5-minute window per IP)
  rule {
    name     = "rate-limit"
    priority = 2

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 1000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
    }
  }

  # Rule 3: SQL injection protection
  rule {
    name     = "aws-managed-sqli"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesSQLiRuleSet"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLiRuleSet"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "OrdersApiWAF"
  }
}

# Associate WAF with API Gateway stage
resource "aws_wafv2_web_acl_association" "api_gateway" {
  resource_arn = aws_api_gateway_stage.prod.arn
  web_acl_arn  = aws_wafv2_web_acl.api_gateway.arn
}

# ═══════════════════════════════════════════════════════════════════════
# RESOURCE POLICY -- Restrict Access (example: IP allowlist)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_api_gateway_rest_api_policy" "orders" {
  rest_api_id = aws_api_gateway_rest_api.orders.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "execute-api:Invoke"
        Resource  = "${aws_api_gateway_rest_api.orders.execution_arn}/*"
      },
      {
        Effect    = "Deny"
        Principal = "*"
        Action    = "execute-api:Invoke"
        Resource  = "${aws_api_gateway_rest_api.orders.execution_arn}/*"
        Condition = {
          NotIpAddress = {
            "aws:SourceIp" = var.allowed_ip_ranges
          }
        }
      }
    ]
  })
}

# ═══════════════════════════════════════════════════════════════════════
# CUSTOM GATEWAY RESPONSES
# ═══════════════════════════════════════════════════════════════════════

resource "aws_api_gateway_gateway_response" "throttled" {
  rest_api_id   = aws_api_gateway_rest_api.orders.id
  response_type = "THROTTLED"
  status_code   = "429"

  response_parameters = {
    "gatewayresponse.header.Retry-After"                 = "'60'"
    "gatewayresponse.header.Access-Control-Allow-Origin"  = "'https://app.example.com'"
  }

  response_templates = {
    "application/json" = jsonencode({
      message   = "Rate limit exceeded. Please retry after 60 seconds."
      requestId = "$context.requestId"
    })
  }
}

resource "aws_api_gateway_gateway_response" "unauthorized" {
  rest_api_id   = aws_api_gateway_rest_api.orders.id
  response_type = "UNAUTHORIZED"
  status_code   = "401"

  response_templates = {
    "application/json" = jsonencode({
      message   = "Invalid or missing authentication token."
      requestId = "$context.requestId"
    })
  }
}

# ═══════════════════════════════════════════════════════════════════════
# CLOUDWATCH LOG GROUPS
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_log_group" "api_access_logs" {
  name              = "/aws/apigateway/orders-api/access-logs"
  retention_in_days = 30
}

# API Gateway needs permission to write to CloudWatch Logs (account-level setting)
resource "aws_api_gateway_account" "main" {
  cloudwatch_role_arn = aws_iam_role.api_gateway_cloudwatch.arn
}

resource "aws_iam_role" "api_gateway_cloudwatch" {
  name = "api-gateway-cloudwatch-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "apigateway.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "api_gateway_cloudwatch" {
  role       = aws_iam_role.api_gateway_cloudwatch.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs"
}

# ═══════════════════════════════════════════════════════════════════════
# CLOUDWATCH ALARMS
# ═══════════════════════════════════════════════════════════════════════

resource "aws_cloudwatch_metric_alarm" "api_5xx_errors" {
  alarm_name          = "orders-api-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  threshold           = 10
  alarm_description   = "REST API 5XX error count exceeds 10 in 2 consecutive periods"

  metric_name = "5XXError"
  namespace   = "AWS/ApiGateway"
  period      = 300
  statistic   = "Sum"

  dimensions = {
    ApiName = aws_api_gateway_rest_api.orders.name
    Stage   = aws_api_gateway_stage.prod.stage_name
  }

  alarm_actions = [var.sns_alarm_topic_arn]
}

resource "aws_cloudwatch_metric_alarm" "api_latency" {
  alarm_name          = "orders-api-high-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  threshold           = 5000  # 5 seconds
  alarm_description   = "API p99 latency exceeds 5 seconds"

  metric_name = "Latency"
  namespace   = "AWS/ApiGateway"
  period      = 300
  extended_statistic = "p99"  # Percentile stats require extended_statistic, not statistic

  dimensions = {
    ApiName = aws_api_gateway_rest_api.orders.name
    Stage   = aws_api_gateway_stage.prod.stage_name
  }

  alarm_actions = [var.sns_alarm_topic_arn]
}

# ═══════════════════════════════════════════════════════════════════════
# HTTP API WITH JWT AUTHORIZER (for comparison)
# ═══════════════════════════════════════════════════════════════════════

resource "aws_apigatewayv2_api" "orders_http" {
  name          = "orders-http-api"
  protocol_type = "HTTP"
  description   = "HTTP API with native JWT authorizer -- lower cost, lower latency"

  cors_configuration {
    allow_origins     = ["https://app.example.com"]
    allow_methods     = ["GET", "POST", "OPTIONS"]
    allow_headers     = ["Content-Type", "Authorization"]
    expose_headers    = ["X-Request-Id"]
    max_age           = 3600
    allow_credentials = true
  }
}

# JWT authorizer using Cognito User Pool (works with any OIDC provider)
resource "aws_apigatewayv2_authorizer" "jwt" {
  api_id           = aws_apigatewayv2_api.orders_http.id
  authorizer_type  = "JWT"
  name             = "cognito-jwt"
  identity_sources = ["$request.header.Authorization"]

  jwt_configuration {
    audience = [var.cognito_app_client_id]
    issuer   = "https://cognito-idp.${data.aws_region.current.name}.amazonaws.com/${var.cognito_user_pool_id}"
  }
}

# Lambda integration for HTTP API
resource "aws_apigatewayv2_integration" "orders_lambda" {
  api_id                 = aws_apigatewayv2_api.orders_http.id
  integration_type       = "AWS_PROXY"
  integration_uri        = var.backend_lambda_arn
  integration_method     = "POST"
  payload_format_version = "2.0"  # HTTP API uses v2.0 format (simplified event)
}

# Routes
resource "aws_apigatewayv2_route" "get_orders" {
  api_id             = aws_apigatewayv2_api.orders_http.id
  route_key          = "GET /orders"
  target             = "integrations/${aws_apigatewayv2_integration.orders_lambda.id}"
  authorization_type = "JWT"
  authorizer_id      = aws_apigatewayv2_authorizer.jwt.id

  # Require specific OAuth scope
  authorization_scopes = ["orders:read"]
}

resource "aws_apigatewayv2_route" "post_orders" {
  api_id             = aws_apigatewayv2_api.orders_http.id
  route_key          = "POST /orders"
  target             = "integrations/${aws_apigatewayv2_integration.orders_lambda.id}"
  authorization_type = "JWT"
  authorizer_id      = aws_apigatewayv2_authorizer.jwt.id

  authorization_scopes = ["orders:write"]
}

# Auto-deploy stage
resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.orders_http.id
  name        = "prod"
  auto_deploy = true  # HTTP API: automatic deployment on changes

  default_route_settings {
    throttling_rate_limit  = 1000
    throttling_burst_limit = 500
  }

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.http_api_access_logs.arn
    format = jsonencode({
      requestId      = "$context.requestId"
      ip             = "$context.identity.sourceIp"
      requestTime    = "$context.requestTime"
      httpMethod     = "$context.httpMethod"
      routeKey       = "$context.routeKey"
      status         = "$context.status"
      responseLength = "$context.responseLength"
      integrationLatency = "$context.integrationLatency"
      errorMessage   = "$context.error.message"
    })
  }
}

resource "aws_cloudwatch_log_group" "http_api_access_logs" {
  name              = "/aws/apigateway/orders-http-api/access-logs"
  retention_in_days = 30
}

# ═══════════════════════════════════════════════════════════════════════
# VARIABLES
# ═══════════════════════════════════════════════════════════════════════

variable "backend_lambda_arn" {
  description = "ARN of the backend Lambda function (from yesterday's Lambda deployment)"
  type        = string
}

variable "authorizer_lambda_arn" {
  description = "ARN of the Lambda authorizer function"
  type        = string
}

variable "cognito_user_pool_id" {
  description = "Cognito User Pool ID for HTTP API JWT authorizer"
  type        = string
}

variable "cognito_app_client_id" {
  description = "Cognito App Client ID for JWT audience validation"
  type        = string
}

variable "allowed_ip_ranges" {
  description = "CIDR blocks allowed to invoke the API (resource policy)"
  type        = list(string)
  default     = ["0.0.0.0/0"]  # Allow all by default -- restrict in production
}

variable "sns_alarm_topic_arn" {
  description = "SNS topic ARN for CloudWatch alarm notifications"
  type        = string
}

# ═══════════════════════════════════════════════════════════════════════
# OUTPUTS
# ═══════════════════════════════════════════════════════════════════════

output "rest_api_invoke_url" {
  description = "REST API invoke URL"
  value       = "${aws_api_gateway_stage.prod.invoke_url}/orders"
}

output "rest_api_id" {
  value = aws_api_gateway_rest_api.orders.id
}

output "http_api_invoke_url" {
  description = "HTTP API invoke URL (71% cheaper, lower latency)"
  value       = aws_apigatewayv2_stage.prod.invoke_url
}

output "api_key_value" {
  description = "API key value for the Silver tier partner (sensitive)"
  value       = aws_api_gateway_api_key.partner_abc.value
  sensitive   = true
}
```

---

## Critical Gotchas and Interview Traps

**1. "API keys are NOT for authentication. They are identification tokens."**
API keys are sent in cleartext in the `x-api-key` header. They identify a caller and map them to a usage plan for throttling and quota enforcement. They do NOT verify identity. Always pair API keys with a real authentication mechanism (Lambda authorizer, Cognito, IAM). Using API keys as the sole security mechanism is a common anti-pattern that will be called out in architecture reviews.

**2. "REST API default integration timeout is 29 seconds -- but it is adjustable."**
The default is 29s for REST API and 30s for HTTP API. Since June 2024, Regional and Private REST APIs can request a timeout increase via Service Quotas (trade-off: increasing the timeout may require reducing your account-level throttle quota). Before this change, 29s was a hard limit. For operations that routinely exceed 29 seconds, consider: (a) requesting a quota increase, (b) enabling REST API response streaming (extends timeout to 15 minutes), or (c) using an async pattern where API Gateway returns 202 Accepted immediately, Lambda processes in the background, and the client polls a status endpoint.

**3. "Lambda proxy integration requires the response body to be a STRING."**
If your Lambda function returns `{"statusCode": 200, "body": {"key": "value"}}` (body is an object), API Gateway returns 502 Bad Gateway with an "Internal server error" message. The body must be a stringified JSON: `{"statusCode": 200, "body": "{\"key\": \"value\"}"}`. This is the single most common API Gateway debugging issue for new developers.

**4. "Changing cache cluster size DELETES all cached data."**
If you upgrade from 0.5 GB to 1.6 GB, all cached entries are gone. The cache cluster is recreated, which takes approximately 4 minutes. Plan cache size changes during low-traffic periods. Also, creating or deleting a cache cluster takes ~4 minutes during which caching is unavailable.

**5. "Cache invalidation without authorization lets anyone flush your cache."**
By default, any caller can send `Cache-Control: max-age=0` to bypass the cache. Without the `Require authorization for cache control` setting, this means an attacker can flush your cache at will, causing every request to hit your backend (cache stampede). Always require authorization for invalidation in production.

**6. "Lambda authorizer caching is keyed by identity source. Misconfiguration leaks policies across users."**
TOKEN authorizer: cached by token value. If your token scheme is weak (e.g., session IDs that could collide), one user's policy is returned for another. REQUEST authorizer: cached by the configured identity sources. If you cache by the Authorization header alone but your auth logic also checks the path or query parameters, you get stale policies for different requests with the same token. Always ensure your identity sources capture ALL variables that affect the authorization decision.

**7. "Resource policy + IAM authorization evaluation depends on same-account vs cross-account."**
Same-account callers use union/OR logic: an Allow from either the resource policy or IAM policy is sufficient. Cross-account callers use intersection/AND logic: both must explicitly Allow. Most engineers incorrectly state "both must Allow" without the account context, leading to over-restrictive designs for same-account or under-restrictive designs for cross-account. Additionally, when traffic flows through a VPC endpoint, the VPC endpoint's own resource policy is evaluated BEFORE the API -- a common hidden cause of 403s where the API-level policies are never even consulted.

**8. "WAF is REST API only. HTTP API has no WAF support."**
If you need WAF protection on an HTTP API, you must front the HTTP API with a CloudFront distribution and attach the WAF web ACL to CloudFront instead. This adds complexity and cost but is a valid pattern.

**9. "Edge-optimized REST API creates a hidden CloudFront distribution you cannot configure."**
The CloudFront distribution is AWS-managed. You cannot add custom cache behaviors, Lambda@Edge functions, custom error pages, or WAF at the CloudFront layer. Use Regional endpoints with your own CloudFront distribution for full control. If you later change from edge-optimized to regional, the invoke URL changes -- clients must update.

**10. "HTTP API does not support request validation, VTL mapping templates, or mock integrations."**
These are REST API-only features. If you need the gateway to validate request bodies against a JSON Schema before invoking Lambda, you must use REST API. With HTTP API, validation must happen in your Lambda code, which means you pay for the invocation even for invalid requests.

**11. "WebSocket $disconnect is best-effort, not guaranteed."**
If a client disconnects abruptly (network failure, browser close), the $disconnect route may not fire. Your backend must handle stale connection IDs gracefully -- catching GoneException (410) from the @connections API when trying to push to a disconnected client, then cleaning up the connection record in DynamoDB.

**12. "The aws_api_gateway_account resource for CloudWatch logging is a GLOBAL singleton per region."**
The `aws_api_gateway_account` resource sets the CloudWatch Logs role for ALL API Gateway APIs in the region. If two Terraform stacks both manage this resource, they will fight over it. Use a shared infrastructure stack for this resource, or import it carefully.

**13. "Canary deployments and Lambda weighted alias routing operate at DIFFERENT levels."**
API Gateway canary: tests the entire API configuration (new routes, changed authorizers, modified integrations, different stage variables). Lambda alias: tests new function code. They are complementary, not interchangeable. Use canary for API structure changes, alias routing for code changes.

**14. "REST API free tier is 12 months only, not perpetual."**
Unlike Lambda's perpetual free tier (1M requests/month forever), the API Gateway free tier (1M REST/HTTP requests per month) expires after 12 months. Plan for this in cost projections.

**15. "mTLS requires a custom domain name. It does not work with the default execute-api endpoint."**
You must set up a custom domain name (api.example.com) with an ACM certificate to use mutual TLS. The truststore (client CA certificates) is uploaded to S3 and referenced in the domain name configuration. The default `*.execute-api.amazonaws.com` endpoint does not support mTLS.

---

## Key Takeaways

- **Default to HTTP API for new serverless APIs.** HTTP API costs 71% less, has lower latency (~10ms p99), native JWT authorization, simpler CORS, and automatic deployments. Only upgrade to REST API when you need caching, WAF, usage plans, request validation, VTL transformations, private endpoints, or canary deployments.

- **API keys are identification, not authentication.** They map callers to usage plans for throttling and quotas. They are transmitted in cleartext. Always combine them with a real auth mechanism (Lambda authorizer, Cognito, IAM). Never use API keys as your sole security layer.

- **The throttling hierarchy has four tiers with strict priority.** Usage plan per-client per-method (highest) > method-level stage settings > stage-level defaults > account-level limit (10K RPS, soft limit). A lower-tier setting cannot exceed a higher-tier limit. Throttled requests return 429 before invoking the backend.

- **Lambda authorizer caching is powerful but dangerous.** Set the cache TTL appropriately (default 300s, max 3600s). Ensure identity sources capture ALL variables that affect the authorization decision. Setting TTL to 0 invokes the authorizer on every request (safe but expensive). Misconfigured caching can return another user's authorization policy.

- **Use direct AWS service integrations to eliminate Lambda invocations.** API Gateway -> DynamoDB PutItem, API Gateway -> SQS SendMessage, API Gateway -> Step Functions StartExecution -- all work without Lambda in the middle. Use VTL mapping templates to transform requests. The trade-off: VTL is harder to debug than Lambda code, but the cost savings and latency reduction are significant.

- **Cache invalidation must be secured.** Without `Require authorization for cache control`, any client can flush your cache by sending `Cache-Control: max-age=0`. This creates a cache stampede vulnerability where an attacker forces every request to hit your backend. Always require `execute-api:InvalidateCache` permission.

- **Use Regional endpoints with your own CloudFront distribution.** Edge-optimized endpoints create an AWS-managed CloudFront distribution you cannot configure. Regional endpoints with your own CloudFront give you full control over caching, WAF, Lambda@Edge, and custom error pages.

- **REST API default integration timeout is 29 seconds, but it is adjustable.** Since June 2024, Regional/Private REST APIs can request a timeout increase via Service Quotas. REST API response streaming (November 2025) extends the timeout to 15 minutes for streaming use cases. For standard request-response patterns exceeding the default, consider: requesting a quota increase, enabling response streaming, or implementing the async pattern (return 202 Accepted, process in background, poll status endpoint).

- **API Gateway canary and Lambda alias routing are complementary, not interchangeable.** API Gateway canary tests the entire API configuration (routes, authorizers, integrations). Lambda alias routing tests function code behind the same API configuration. Production deployments may use both depending on what changed.

- **WebSocket APIs require a connection management layer.** Store connection IDs in DynamoDB on $connect, clean up on $disconnect (best-effort), use the @connections API to push messages. Handle 410 GoneException for stale connections. Budget for DynamoDB costs in high-connection-count scenarios.

- **The access log vs execution log distinction is critical for operations.** Access logs (structured, one line per request) should be always-on in JSON format. Execution logs (verbose pipeline trace) should default to ERROR level and switch to INFO only when debugging. Never leave INFO-level execution logging on permanently -- the log volume and cost are enormous.

- **Request validation at the gateway saves money.** Every invalid request that reaches Lambda costs you a Lambda invocation. REST API's JSON Schema validation rejects malformed requests at the gateway for free. With HTTP API, you must validate in code, paying the invocation cost regardless.

---

## Study Resources

Curated reading list for this topic: [`study-resources/aws/api-gateway-deep-dive.md`](../../study-resources/aws/api-gateway-deep-dive.md)

**Key references**:
- [API Gateway Concepts](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-basic-concept.html) -- Foundational glossary: API types, endpoint types, stages, deployments, integration types
- [REST API vs HTTP API](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html) -- The definitive feature comparison table driving the architectural decision
- [Integration Types](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-integration-types.html) -- AWS_PROXY, AWS, HTTP_PROXY, HTTP, MOCK with transformation mechanics
- [Throttling](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html) -- Token bucket algorithm and four-tier hierarchy
- [Caching](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html) -- Per-stage caching with TTL, invalidation, and cache key parameters
- [Lambda Authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html) -- TOKEN vs REQUEST types with policy document format
- [Canary Releases](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html) -- Stage-level traffic splitting with monitoring

**Related docs in this repo**:
- [Lambda Deep Dive (Apr 6)](./2026-04-06-lambda-deep-dive.md) -- The kitchen behind the concierge desk: execution model, invocation types, Function URLs comparison table, versions/aliases
- [CloudFront Deep Dive (Mar 13)](../../march/aws/2026-03-13-cloudfront-deep-dive.md) -- Edge-optimized endpoints use CloudFront; front Regional APIs with your own distribution
- [WAF, Shield, Network Firewall (Mar 10)](../../march/aws/2026-03-10-waf-shield-network-firewall.md) -- WAF web ACLs that protect REST API stages
- [ALB vs NLB vs GLB (Mar 16)](../../march/aws/2026-03-16-alb-nlb-glb.md) -- ALB as API Gateway alternative, NLB for REST API VPC Links
- [IAM Advanced Patterns (Mar 5)](../../march/aws/2026-03-05-iam-advanced-patterns.md) -- IAM authorization, resource policy evaluation logic, Sig V4 signing
