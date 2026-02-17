# AWS ECS Advanced: Service Discovery, Service Connect, App Mesh, ECS Anywhere, and Capacity Provider Strategies - The Corporate Communications Analogy

> Understanding advanced ECS patterns through a real-world analogy of a large corporation managing internal communications. From the department phone directory (Cloud Map) to the smart switchboard (Service Connect) to the legacy PBX being retired (App Mesh), then into satellite offices with limited connectivity (ECS Anywhere) and the facilities team optimizing office space (capacity provider strategies).

---

## TL;DR

| AWS ECS Concept | Real-World Analogy | One-Liner |
|-----------------|-------------------|-----------|
| AWS Cloud Map | Corporate department directory | A centralized registry that maps department names to phone numbers (IPs) and extensions (ports) |
| Service Discovery Namespace | A domain in the directory (e.g., production.internal) | Logical grouping that gives all registered services a shared suffix like "engineering.corp" |
| DNS-Based Discovery | Looking up a number in the phone book | Query "accounting.corp" and get back a list of direct-dial numbers for the team |
| API-Based Discovery | Calling the reception desk to ask for someone | Richer metadata returned (department, floor, availability) but requires a phone call vs just reading the book |
| SRV Records | Directory entries that include extension numbers | Returns both the phone line (IP) AND the extension (port) -- needed when multiple people share one line (bridge mode) |
| ECS Service Connect | Smart corporate switchboard | Operators who know every department, automatically route calls, retry busy lines, and avoid known-broken extensions |
| Service Connect Namespace | The switchboard's coverage area | All departments served by this switchboard can call each other by first name ("just ask for Accounting") |
| Service Connect Client Alias | Short name on the switchboard | Instead of dialing 555-0142-ext-8080, just say "mysql" and the switchboard connects you |
| Service Connect Proxy (Envoy) | A personal assistant at each department's desk | Every department gets a dedicated assistant who handles all outgoing and incoming calls on their behalf |
| Client Service | A department that only makes calls | Marketing calls Analytics for data but never receives internal calls from other departments |
| Client-Server Service | A department that makes AND receives calls | The Database team both serves queries from other teams and makes calls to the Storage team |
| Outlier Detection | Operator notes that extension 4 is always busy/failing | Stops routing calls to that extension and distributes to healthy ones instead |
| AWS App Mesh (EOL) | Legacy PBX phone system being decommissioned | A powerful but complex phone system where you had to install, wire, and maintain a sub-switch at every desk |
| App Mesh Virtual Node | A PBX station configuration | Manual configuration describing one desk's phone setup, trunk lines, and routing rules |
| App Mesh Virtual Router | PBX call routing table | Rules like "send 80% of calls to extension group A, 20% to extension group B" |
| ECS Anywhere | Satellite office with limited headquarters connectivity | Remote offices that run the same work using local facilities, but cannot use the main building's intercom, switchboard, or mailroom |
| EXTERNAL Launch Type | The satellite office badge type | Identifies an employee as working from the remote office -- different access rules than headquarters staff |
| SSM Agent on External Instance | The satellite office's secure phone line to HQ | An encrypted dedicated line for receiving instructions, checked every 30 minutes for credential rotation |
| Capacity Provider Strategy | Facilities team's space allocation policy | "Fill the main building first (binpack), then overflow to the annex, keeping floors balanced (spread)" |
| targetCapacity | How full to pack each floor before opening a new one | 80% means keep 20% of desks empty on each floor for walk-ins; 100% means pack every desk |
| CapacityProviderReservation | Current occupancy rate metric | "We need 50 desks but only have 40 available" = reservation of 125%, triggering a new floor to open |
| Managed Termination Protection | "Do not close a floor while people are still working" | Prevents the facilities team from shutting down a floor that has active employees |
| Binpack Placement | Fill one floor completely before opening another | Minimizes the number of floors you pay rent on -- cost-efficient but less resilient |
| Spread Placement | Distribute employees evenly across all floors | If one floor floods, you only lose a fraction of your workforce -- resilient but potentially wasteful |
| Zonal Capacity Providers | One landlord per building wing | A separate ASG per availability zone, ensuring perfectly even distribution across wings |

---

## The Big Picture

In the [ECS Fundamentals doc](./2026-02-16-ecs-fundamentals.md), we covered how the film studio operates: screenplays (task definitions), production managers (services), and sound stages (capacity providers). Now we zoom out to a **large corporation** with many departments that need to communicate. The fundamentals taught you how to run individual shows. This document teaches you how those shows **find each other, talk to each other, run in remote offices, and optimize the real estate they occupy**.

Five questions drive this document:

1. **How do services find each other?** -- Service Discovery with Cloud Map (the department directory)
2. **How do services talk reliably?** -- Service Connect (the smart switchboard)
3. **What was the old approach?** -- App Mesh (the legacy PBX, now EOL)
4. **Can services run outside AWS?** -- ECS Anywhere (the satellite office)
5. **How do you optimize where services run?** -- Advanced Capacity Provider Strategies (the facilities team)

```
ECS ADVANCED — THE FIVE TOPICS
=============================================================================

  ┌──────────────────────────────────────────────────────────────────────┐
  │                        ECS CLUSTER                                   │
  │                                                                      │
  │  HOW DO SERVICES FIND EACH OTHER?                                   │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │  Cloud Map Service Discovery (Part 1)                          │  │
  │  │  "The department phone directory"                               │  │
  │  │  DNS-based (A/SRV records) or API-based (rich metadata)        │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                      │
  │  HOW DO SERVICES TALK RELIABLY?                                     │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │  ECS Service Connect (Part 2)                                  │  │
  │  │  "The smart switchboard"                                       │  │
  │  │  Cloud Map + Envoy proxy = discovery + load balancing + retry  │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                      │
  │  WHAT WAS THE OLD APPROACH?                                         │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │  AWS App Mesh (Part 3) — EOL September 30, 2026                │  │
  │  │  "The legacy PBX phone system"                                 │  │
  │  │  Powerful but complex — manual Envoy sidecars, virtual nodes   │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                      │
  │  CAN SERVICES RUN OUTSIDE AWS?                                      │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │  ECS Anywhere (Part 4) — EXTERNAL launch type                  │  │
  │  │  "The satellite office"                                        │  │
  │  │  On-premises servers registered to ECS with limited features   │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                      │
  │  HOW DO YOU OPTIMIZE WHERE SERVICES RUN?                            │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │  Advanced Capacity Provider Strategies (Part 5)                │  │
  │  │  "The facilities team"                                         │  │
  │  │  Managed scaling, targetCapacity, binpack vs spread, zonal    │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Service Discovery with AWS Cloud Map - The Department Directory

### The Analogy

Your corporation has dozens of departments -- Accounting, Engineering, Marketing, Analytics. When the Marketing team needs to send data to Analytics, they need to look up the phone number. The **department directory** (Cloud Map) is a centralized registry where every department registers its contact information. When someone needs to reach a department, they look it up in the directory and get back a list of direct-dial numbers. The directory is **automatically updated** -- when a new employee joins, their number is added; when they leave, it is removed. No manual edits.

The directory supports two lookup methods:
- **Phone book lookup (DNS-based)**: Fast, simple, returns a list of numbers. But the phone book has a printing delay (DNS TTL caching), so you might call a number that was recently disconnected.
- **Reception desk call (API-based)**: You call the front desk and ask "Who is available in Accounting?" The receptionist gives you richer information -- not just phone numbers but also which floor they are on, their specialty, and whether they are currently available. Slower but more accurate.

### The Technical Reality

AWS Cloud Map is a service registry that ECS integrates with natively. When you enable service discovery on an ECS service, each task automatically registers its IP address and port with Cloud Map. Other services can then discover these tasks through DNS queries or the Cloud Map API.

```
CLOUD MAP ARCHITECTURE
=============================================================================

  ┌───────────────────────────────────────────────────────────────────┐
  │                    AWS CLOUD MAP                                   │
  │                                                                   │
  │  NAMESPACE: production.local                                      │
  │  (Type: Private DNS — backed by Route 53 hosted zone)            │
  │                                                                   │
  │  ┌─────────────────────┐    ┌─────────────────────┐              │
  │  │ Service: web-api    │    │ Service: payments    │              │
  │  │                     │    │                      │              │
  │  │ Instance: task-a    │    │ Instance: task-x     │              │
  │  │  IP: 10.0.1.23     │    │  IP: 10.0.2.45      │              │
  │  │  Port: 8080        │    │  Port: 3000          │              │
  │  │  AZ: us-east-1a    │    │  AZ: us-east-1b     │              │
  │  │                     │    │                      │              │
  │  │ Instance: task-b    │    │ Instance: task-y     │              │
  │  │  IP: 10.0.1.87     │    │  IP: 10.0.2.91      │              │
  │  │  Port: 8080        │    │  Port: 3000          │              │
  │  │  AZ: us-east-1b    │    │  AZ: us-east-1a     │              │
  │  └─────────────────────┘    └─────────────────────┘              │
  │                                                                   │
  │  DNS QUERY: web-api.production.local                              │
  │  RESPONSE:  A record → [10.0.1.23, 10.0.1.87]                   │
  │                                                                   │
  │  DNS QUERY: payments.production.local (SRV)                       │
  │  RESPONSE:  SRV record → [10.0.2.45:3000, 10.0.2.91:3000]      │
  └───────────────────────────────────────────────────────────────────┘


  THE FOUR-LAYER HIERARCHY:
  ────────────────────────────────────────────────────

  NAMESPACE (production.local)
    └── SERVICE (web-api)
          └── INSTANCE (task-a: 10.0.1.23:8080, AZ=us-east-1a)
          └── INSTANCE (task-b: 10.0.1.87:8080, AZ=us-east-1b)
    └── SERVICE (payments)
          └── INSTANCE (task-x: 10.0.2.45:3000)
          └── INSTANCE (task-y: 10.0.2.91:3000)


  DNS RECORD TYPES BY NETWORK MODE:
  ────────────────────────────────────────────────────

  awsvpc mode:
  ├── A records (IPv4) — each task has its own ENI and IP
  ├── AAAA records (IPv6) — for dual-stack or IPv6-only
  └── SRV records — IP + port (useful but not required since port is fixed)

  bridge / host mode:
  └── SRV records ONLY — because multiple tasks share an instance IP
      and use dynamic port mapping, you NEED both IP and port


  HEALTH CHECK INTEGRATION:
  ────────────────────────────────────────────────────
  ├── ECS manages Cloud Map health automatically via HealthCheckCustomConfig
  │   (ECS — not Route 53 — controls instance health status)
  ├── New tasks register as UNHEALTHY until container health check passes
  ├── Passing checks → status updated to HEALTHY → appears in DNS
  ├── Failing checks → instance deregistered from DNS routing
  └── No extra cost for container-level health checks


  REGISTERED INSTANCE ATTRIBUTES:
  ────────────────────────────────────────────────────
  ├── AWS_INSTANCE_IPV4: private IP (ALWAYS private, even with public namespace)
  ├── AWS_INSTANCE_IPV6: private IPv6 address
  ├── AWS_INSTANCE_PORT: container port
  ├── AVAILABILITY_ZONE: AZ where task launched
  ├── REGION: AWS Region
  ├── ECS_SERVICE_NAME: the ECS service name
  ├── ECS_CLUSTER_NAME: the cluster name
  └── EC2_INSTANCE_ID: container instance (EC2 only, not Fargate)
```

### Key Limitations of Raw Service Discovery

```
WHAT RAW CLOUD MAP DOES NOT GIVE YOU:
=============================================================================

  1. NO LOAD BALANCING
     ├── DNS returns a list of IPs, your application picks one
     ├── Most DNS clients cache the first result and reuse it
     └── You need client-side load balancing logic or short TTLs

  2. NO RETRIES
     ├── If a task is unhealthy between health checks, DNS still returns it
     ├── Your application must implement retry logic
     └── DNS TTL caching means stale entries persist

  3. NO TRAFFIC MANAGEMENT
     ├── No circuit breaking, outlier detection, or rate limiting
     ├── No traffic shaping (canary, weighted routing)
     └── All of this must be handled in application code

  4. HARD LIMITS
     ├── 1,000 instances (tasks) per Cloud Map service (Route 53 quota)
     ├── Private IPs only — public IPs are never registered
     └── Shared Cloud Map namespaces are NOT supported for raw service discovery
         registration (Service Connect does support shared namespaces via AWS RAM)

  BOTTOM LINE: Raw service discovery gives you a phone book.
  You still need to dial the number, handle busy signals, and retry.
  Service Connect (Part 2) adds the smart switchboard on top.
```

### Terraform: Cloud Map Service Discovery

```hcl
# Private DNS namespace — the "domain" in your directory
resource "aws_service_discovery_private_dns_namespace" "main" {
  name        = "production.local"
  description = "Service discovery namespace for production ECS services"
  vpc         = var.vpc_id
}

# Cloud Map service — a directory entry for the web-api service
resource "aws_service_discovery_service" "web_api" {
  name = "web-api"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10    # Low TTL for faster failover (seconds)
      type = "A"   # A records for awsvpc mode
    }

    routing_policy = "MULTIVALUE"  # Return all healthy IPs
  }

  # HealthCheckCustomConfig — lets ECS (not Route 53) manage health status
  health_check_custom_config {
    failure_threshold = 1  # Deregister after 1 failed check
  }
}

# ECS Service with service discovery enabled
resource "aws_ecs_service" "web_api" {
  name            = "web-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.web_api.arn
  desired_count   = 3

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 1
  }

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.web_api.id]
    assign_public_ip = false
  }

  # Register tasks with Cloud Map automatically
  service_registries {
    registry_arn = aws_service_discovery_service.web_api.arn
  }
}

# Other services can now reach web-api at:
# web-api.production.local → resolves to task IPs
```

---

## Part 2: ECS Service Connect - The Smart Switchboard

### The Analogy

Raw service discovery gave you a phone directory -- you look up a number and dial it yourself. **Service Connect** is the **smart switchboard** that sits between every department. Instead of dialing a number directly, you tell the switchboard "connect me to Accounting" and it handles everything: finding an available person, automatically retrying if the line is busy, noting which extensions are consistently failing and stopping routing to them, and load-balancing calls across the whole team.

The key innovation: **every department gets its own personal assistant** (Envoy proxy) sitting right at their desk. When Marketing wants to call Analytics, Marketing's assistant intercepts the call, looks up all available Analytics extensions, picks the best one (round-robin with outlier detection), and connects the call. If the call fails, the assistant retries on a different extension -- all transparently to Marketing. The assistants are coordinated by the switchboard system but act locally, so each department's calls are handled without a single centralized bottleneck.

The switchboard's **coverage area** is the namespace. All departments within the same namespace can reach each other by **short names** ("just say Accounting") instead of full phone numbers. Departments outside the namespace cannot use the switchboard -- they need to dial the full number through other means (load balancer, VPC Lattice, etc.).

### The Technical Reality

ECS Service Connect combines Cloud Map service discovery with a managed Envoy proxy sidecar, injected automatically into every task in the service. It provides:
- **Automatic service registration** (no manual Cloud Map configuration)
- **Short-name DNS resolution** (client aliases like `mysql` or `api`)
- **Round-robin load balancing** across healthy endpoints
- **Outlier detection** (routes around failing tasks)
- **Automatic retries** on transient failures
- **Standardized metrics** in CloudWatch (connection counts, latency, error rates)
- **Optional TLS encryption** via AWS Private CA (short-lived mode: certificates rotated every 5 days with max 7-day validity; general-purpose mode uses longer-lived certificates at higher cost)

All of this without any application code changes.

```
SERVICE CONNECT ARCHITECTURE
=============================================================================

  NAMESPACE: production (Cloud Map namespace)

  ┌──────────────────────────────────────────────────────────────────────┐
  │                         ECS CLUSTER                                   │
  │                                                                       │
  │  SERVICE: frontend (client service — only makes calls)               │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │ TASK                                                           │  │
  │  │ ┌──────────────┐    ┌─────────────────────────────────────┐   │  │
  │  │ │  Frontend    │───▶│  Service Connect Proxy (Envoy)      │   │  │
  │  │ │  Container   │    │                                     │   │  │
  │  │ │              │    │  Intercepts: http://api:8080        │   │  │
  │  │ │  Calls:      │    │  Resolves to: [10.0.1.23, 10.0.2.7]│   │  │
  │  │ │  api:8080    │    │  Load balances: round-robin         │   │  │
  │  │ │  cache:6379  │    │  Outlier detection: active          │   │  │
  │  │ └──────────────┘    └──────────┬──────────────────────────┘   │  │
  │  └────────────────────────────────┼──────────────────────────────┘  │
  │                                   │                                  │
  │                                   │ (proxy-to-proxy communication)   │
  │                                   ▼                                  │
  │  SERVICE: api (client-server service — makes AND receives calls)    │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │ TASK                                                           │  │
  │  │ ┌─────────────────────────────────────┐    ┌──────────────┐   │  │
  │  │ │  Service Connect Proxy (Envoy)      │───▶│  API         │   │  │
  │  │ │                                     │    │  Container   │   │  │
  │  │ │  Listens on: api:8080 (client alias)│    │  :8080       │   │  │
  │  │ │  Registered in Cloud Map as "api"   │    │              │   │  │
  │  │ │  Intercepts outgoing calls too      │    │  Calls:      │   │  │
  │  │ │                                     │◀───│  db:5432     │   │  │
  │  │ └─────────────────────────────────────┘    └──────────────┘   │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                       │
  │  SERVICE: database (client-server service)                           │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │ TASK                                                           │  │
  │  │ ┌─────────────────────────────────────┐    ┌──────────────┐   │  │
  │  │ │  Service Connect Proxy (Envoy)      │───▶│  Database    │   │  │
  │  │ │                                     │    │  Container   │   │  │
  │  │ │  Listens on: db:5432               │    │  :5432       │   │  │
  │  │ └─────────────────────────────────────┘    └──────────────┘   │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────┘


  KEY TERMINOLOGY:
  ────────────────────────────────────────────────────

  PORT NAME:
  ├── Defined in the task definition (e.g., name: "api-http")
  ├── Assigns a logical name to a port mapping
  └── Service Connect uses this to identify discoverable endpoints

  DISCOVERY NAME:
  ├── Optional name registered in Cloud Map
  ├── Defaults to the port name if not specified
  └── Must be unique within the namespace

  CLIENT ALIAS:
  ├── The short DNS name other services use to connect
  ├── Example: "api" with port 8080 → http://api:8080
  ├── Can override the discovery name
  └── Multiple aliases per service are supported

  CLIENT SERVICE:
  ├── Only makes outgoing calls (consumes other services)
  ├── Has the proxy for outgoing traffic interception
  └── Does not register endpoints in Cloud Map

  CLIENT-SERVER SERVICE:
  ├── Makes AND receives calls
  ├── Registers endpoints in Cloud Map
  ├── Proxy listens on the client alias port for incoming traffic
  └── Most microservices are client-server


  TRAFFIC FLOW (what actually happens):
  ────────────────────────────────────────────────────

  1. Frontend container resolves "api" → 127.0.0.1 (loopback, intercepted by proxy)
     HOW? The Envoy proxy modifies the task's local DNS so that client alias
     names (like "api") point to the loopback address where the proxy listens.
     This is why zero application code changes are needed — the app thinks
     it is calling "api:8080" normally, but DNS silently redirects to the proxy.
  2. Frontend's proxy looks up "api" in Cloud Map → [10.0.1.23:8080, 10.0.2.7:8080]
  3. Proxy picks an endpoint via round-robin, skipping any marked as outliers
  4. Proxy sends the request to the API task's proxy
  5. API task's proxy forwards to the API container on localhost:8080
  6. If the request fails, frontend's proxy retries on a different endpoint
  7. Metrics (latency, success rate, connection count) published to CloudWatch
```

### Service Connect vs Raw Service Discovery

```
DECISION: WHEN TO USE WHICH
=============================================================================

                            Raw Cloud Map          Service Connect
                            ──────────────         ────────────────
  Discovery                 DNS (A/SRV) or API     Proxy-based (automatic)
  Load balancing            None (app handles)     Round-robin + outlier detection
  Retries                   None (app handles)     Automatic
  Health routing            DNS TTL lag             Real-time proxy routing
  Metrics                   None built-in          CloudWatch (latency, errors, conns)
  TLS                       Manual                 Optional managed (Private CA)
  Short names               FQDN only              Client aliases (e.g., "api")
                            (api.prod.local)
  Network modes             awsvpc, bridge, host   awsvpc, bridge
  Cross-namespace           Possible               Not supported
  Cross-cluster             Yes (Cloud Map has       Supported (same namespace)
                            no cluster concept)
  Application changes       DNS client needed       Zero code changes
  Extra containers          None                   Envoy proxy (auto-injected)
  Cost                      Route 53 + Cloud Map   No extra charge (proxy shares
                            query costs             task CPU/memory)

  RECOMMENDATION:
  ├── Use SERVICE CONNECT for all new ECS service-to-service communication
  ├── Use RAW CLOUD MAP only when you need cross-namespace discovery
  │   or when Service Connect is not available (ECS Anywhere, etc.)
  └── For external traffic (internet-facing), use ALB/NLB — neither
      Service Connect nor raw discovery replaces a load balancer
```

### Terraform: ECS Service Connect

```hcl
# Cloud Map namespace for Service Connect
resource "aws_service_discovery_http_namespace" "production" {
  name        = "production"
  description = "Service Connect namespace for production services"
}

# Task definition with port name (required for Service Connect)
resource "aws_ecs_task_definition" "api" {
  family                   = "api"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.api_task.arn

  container_definitions = jsonencode([
    {
      name      = "api"
      image     = "${aws_ecr_repository.api.repository_url}:latest"
      essential = true

      portMappings = [
        {
          containerPort = 8080
          protocol      = "tcp"
          name          = "api-http"       # Port name — Service Connect uses this
          appProtocol   = "http"           # Supports "http", "http2", or "grpc"
                                            # Protocol choice affects proxy behavior
                                            # (e.g., gRPC health checking, HTTP/2 multiplexing)
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/api"
          "awslogs-region"        = var.region
          "awslogs-stream-prefix" = "api"
        }
      }
    }
  ])
}

# ECS Service with Service Connect enabled (client-server)
resource "aws_ecs_service" "api" {
  name            = "api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 3

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
  }

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.api.id]
    assign_public_ip = false
  }

  # Service Connect configuration — this is the switchboard setup
  service_connect_configuration {
    enabled   = true
    namespace = aws_service_discovery_http_namespace.production.arn

    # This service is a CLIENT-SERVER: it receives calls from others
    service {
      port_name = "api-http"   # Must match port name in task definition

      client_alias {
        port     = 8080        # The port other services use
        dns_name = "api"       # The short name: http://api:8080
      }
    }

    # Log configuration for the auto-injected Envoy proxy container
    log_configuration {
      log_driver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/service-connect-proxy"
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "api"
      }
    }
  }
}

# Frontend service — CLIENT ONLY (makes calls but does not receive)
resource "aws_ecs_service" "frontend" {
  name            = "frontend"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.frontend.arn
  desired_count   = 2

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
  }

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.frontend.id]
    assign_public_ip = false
  }

  # Client-only Service Connect — no "service" block, just the namespace
  service_connect_configuration {
    enabled   = true
    namespace = aws_service_discovery_http_namespace.production.arn
    # No "service" block = client only
    # This service can call "api:8080" but does not register its own endpoints
  }

  # Frontend still uses an ALB for external traffic
  load_balancer {
    target_group_arn = aws_lb_target_group.frontend.arn
    container_name   = "frontend"
    container_port   = 3000
  }
}
```

---

## Part 3: AWS App Mesh - The Legacy PBX (EOL September 30, 2026)

### The Analogy

Before the smart switchboard (Service Connect) existed, the corporation used a **legacy PBX phone system** (App Mesh). App Mesh was like **running your own phone company** -- you had to set up not just each phone (virtual node), but also write the routing rules between area codes (virtual router), define the directory listings (virtual services), and make sure every phone's configuration was updated whenever any other phone changed. You also had to install one to three pieces of hardware at each desk (Envoy sidecar, X-Ray daemon, proxy init container), manually wired and configured.

The smart switchboard (Service Connect) is like switching to a **managed phone service** where you just plug in the phone and the carrier handles everything. It does 80% of what the PBX did with 20% of the complexity, because ECS manages the wiring automatically. The PBX is being **decommissioned on September 30, 2026**, and all departments should migrate to the switchboard before that date.

### The Technical Reality

AWS App Mesh was a service mesh that provided application-level networking across ECS, EKS, and EC2. It used Envoy proxies as sidecars, but unlike Service Connect, you had to **manually configure and deploy** these sidecars.

```
APP MESH ARCHITECTURE (LEGACY — EOL SEPT 30, 2026)
=============================================================================

  ┌──────────────────────────────────────────────────────────────────────┐
  │                        APP MESH                                      │
  │                                                                      │
  │  MESH: production-mesh                                               │
  │                                                                      │
  │  ┌─────────────────────┐         ┌──────────────────────┐           │
  │  │ VIRTUAL SERVICE     │         │ VIRTUAL SERVICE      │           │
  │  │ "api.prod.local"    │         │ "db.prod.local"      │           │
  │  │                     │         │                      │           │
  │  │ ┌─────────────────┐ │         │ ┌──────────────────┐ │           │
  │  │ │ VIRTUAL ROUTER  │ │         │ │ VIRTUAL NODE     │ │           │
  │  │ │                 │ │         │ │ "db-v1"          │ │           │
  │  │ │ Route:          │ │         │ │                  │ │           │
  │  │ │  80% → api-v1   │ │         │ │ Listener: :5432  │ │           │
  │  │ │  20% → api-v2   │ │         │ │ Service: db-svc  │ │           │
  │  │ └─────────────────┘ │         │ └──────────────────┘ │           │
  │  │                     │         └──────────────────────┘           │
  │  │ ┌─────────┐ ┌─────────┐                                         │
  │  │ │ VIRTUAL │ │ VIRTUAL │                                         │
  │  │ │ NODE    │ │ NODE    │                                         │
  │  │ │ api-v1  │ │ api-v2  │                                         │
  │  │ └─────────┘ └─────────┘                                         │
  │  └─────────────────────┘                                            │
  └──────────────────────────────────────────────────────────────────────┘


  APP MESH KEY CONSTRUCTS:
  ────────────────────────────────────────────────────

  MESH:
  └── Top-level container for all mesh resources (like the PBX system itself)

  VIRTUAL NODE:
  ├── Represents a specific task group / service version
  ├── Defines listeners (ports the service accepts traffic on)
  ├── Defines backends (other virtual services this node can call)
  └── Maps to an ECS service or Cloud Map service discovery entry

  VIRTUAL SERVICE:
  ├── An abstraction that other services target
  ├── Can be backed by a virtual node OR a virtual router
  └── Example: "api.prod.local" — callers use this name

  VIRTUAL ROUTER:
  ├── Routes traffic to one or more virtual nodes
  ├── Supports weighted routing (canary, blue/green)
  ├── Supports path-based and header-based routing
  └── Example: send 80% to api-v1, 20% to api-v2

  ROUTE:
  ├── A rule within a virtual router
  ├── Matches on path prefix, headers, method
  ├── Specifies retry policy, timeout
  └── Routes to weighted targets (virtual nodes)


  WHY APP MESH WAS COMPLEX:
  ────────────────────────────────────────────────────
  ├── You deployed 1-3 sidecar containers per task:
  │   ├── Envoy proxy container
  │   ├── X-Ray daemon container (for tracing)
  │   └── Proxy init container (to set up iptables rules)
  ├── You configured iptables rules to intercept traffic through Envoy
  │   (contrast: Service Connect uses DNS manipulation — no iptables needed)
  ├── You managed virtual node, service, router, and route definitions
  ├── You kept all mesh configurations in sync with ECS service changes
  └── A misconfigured proxy could silently drop traffic


  APP MESH → SERVICE CONNECT MIGRATION MAP:
  ────────────────────────────────────────────────────

  App Mesh Construct         →  Service Connect Equivalent
  ─────────────────────         ──────────────────────────
  Mesh                       →  Cloud Map namespace
  Virtual Node               →  Service Connect endpoint (client-server)
  Virtual Service            →  Client alias (short DNS name)
  Virtual Router (weighted)  →  Not directly supported — use CodeDeploy
                                for canary/weighted traffic shifting
  Manual Envoy sidecars      →  Auto-injected proxy container
  iptables init container    →  Not needed (ECS manages routing)
  X-Ray sidecar              →  X-Ray integration via task definition
                                (tracing still works — just not a separate
                                sidecar; note: X-Ray traces and CloudWatch
                                metrics are different — traces are per-request
                                journey maps, metrics are aggregated numbers)


  WHAT SERVICE CONNECT DOES NOT REPLACE:
  ────────────────────────────────────────────────────
  ├── Weighted routing between versions (App Mesh virtual router)
  │   → Use CodeDeploy blue/green with traffic shifting instead
  ├── Cross-service-type mesh (ECS + EKS + EC2 in one mesh)
  │   → Use VPC Lattice for cross-compute-type networking
  └── Header-based / path-based routing rules
      → Use ALB listener rules or VPC Lattice
```

---

## Part 4: ECS Anywhere - The Satellite Office

> We leave the communications department and move to real estate — the next two sections cover where your services physically run, not how they talk to each other.

### The Analogy

Your corporation opens a **satellite office** in a remote location. The employees there do the same work (run the same containers), follow the same management chain (ECS control plane), and report to the same headquarters. But the satellite office has **limited connectivity to headquarters**: no intercom system (no load balancer), no entry in the department directory (no service discovery), no connection to the smart switchboard (no Service Connect), and no corporate VPN phone lines to every desk (no awsvpc networking). Employees use shared phone lines (bridge/host networking) and are best suited for **outbound tasks** -- processing data, running batch jobs, and local edge computing -- rather than receiving inbound customer calls.

Each satellite office worker carries two badges: an **SSM badge** (SSM Agent) for secure communication with headquarters, which gets re-validated every 30 minutes using a hardware fingerprint, and an **ECS badge** (ECS Agent) for receiving task assignments and reporting status.

### The Technical Reality

ECS Anywhere allows you to register on-premises servers and VMs as **external instances** in an ECS cluster using the **EXTERNAL launch type**. These instances run the SSM Agent and ECS Agent alongside Docker, enabling ECS to schedule and manage container workloads on your own hardware.

The single root cause behind all limitations: **external instances sit outside the AWS VPC**. Nearly every advanced ECS feature — ALB, Cloud Map, Service Connect, awsvpc — is built on VPC-native networking primitives (ENIs, VPC IP addresses, VPC-internal routing). Without VPC membership, none of these features can function.

```
ECS ANYWHERE ARCHITECTURE
=============================================================================

  ┌───────────────────────────────────────────┐
  │              AWS CLOUD                     │
  │                                           │
  │  ┌─────────────────────────────────────┐  │
  │  │           ECS CONTROL PLANE         │  │
  │  │  ├── Task scheduling               │  │
  │  │  ├── Service management            │  │
  │  │  └── Health monitoring             │  │
  │  └───────────────┬─────────────────────┘  │
  │                  │                        │
  │  ┌───────────────┼─────────────────────┐  │
  │  │  ENDPOINTS    │                     │  │
  │  │  ├── ecs.region.amazonaws.com       │  │
  │  │  ├── ssm.region.amazonaws.com       │  │
  │  │  ├── ec2messages.region...          │  │
  │  │  └── ssmmessages.region...          │  │
  │  └───────────────┼─────────────────────┘  │
  └──────────────────┼────────────────────────┘
                     │ (outbound HTTPS from on-prem)
                     │
  ═══════════════════╪══════════════════════════  Network boundary
                     │
  ┌──────────────────┼────────────────────────┐
  │  ON-PREMISES     │                        │
  │                  │                        │
  │  ┌───────────────▼─────────────────────┐  │
  │  │  EXTERNAL INSTANCE                  │  │
  │  │                                     │  │
  │  │  ┌──────────┐  ┌──────────┐        │  │
  │  │  │ SSM      │  │ ECS      │        │  │
  │  │  │ Agent    │  │ Agent    │        │  │
  │  │  │          │  │          │        │  │
  │  │  │ Creds    │  │ Task     │        │  │
  │  │  │ rotated  │  │ lifecycle│        │  │
  │  │  │ every    │  │ mgmt     │        │  │
  │  │  │ 30 min   │  │          │        │  │
  │  │  └──────────┘  └──────────┘        │  │
  │  │                                     │  │
  │  │  ┌──────────────────────────────┐   │  │
  │  │  │         DOCKER               │   │  │
  │  │  │  ┌────────┐  ┌────────┐     │   │  │
  │  │  │  │ Task A │  │ Task B │     │   │  │
  │  │  │  │(bridge)│  │ (host) │     │   │  │
  │  │  │  └────────┘  └────────┘     │   │  │
  │  │  └──────────────────────────────┘   │  │
  │  └─────────────────────────────────────┘  │
  │                                            │
  │  ┌─────────────────────────────────────┐  │
  │  │  EXTERNAL INSTANCE 2                │  │
  │  │  (same agents, same setup)          │  │
  │  └─────────────────────────────────────┘  │
  └────────────────────────────────────────────┘


  SUPPORTED OPERATING SYSTEMS:
  ────────────────────────────────────────────────────
  ├── Amazon Linux 2023 (x86_64, ARM64)
  ├── Ubuntu 20.04, 22.04, 24.04 (x86_64, ARM64)
  └── RHEL 9 (x86_64, ARM64) — Docker must be pre-installed

  DEPRECATED (EOL August 7, 2026):
  ├── Amazon Linux 2, CentOS Stream 9, RHEL 7-8
  ├── Ubuntu 18, Debian 9-12
  ├── Fedora 32/33/40, openSUSE Tumbleweed, SUSE Enterprise Server 15
  └── Windows Server 2016-2022 (Windows fully deprecated)


  CRITICAL LIMITATIONS:
  ────────────────────────────────────────────────────

  ┌────────────────────────────────┬──────────────────────────────────┐
  │ Feature                        │ ECS Anywhere Support             │
  ├────────────────────────────────┼──────────────────────────────────┤
  │ Elastic Load Balancing         │ NOT supported                    │
  │ Service Discovery (Cloud Map)  │ NOT supported                    │
  │ Service Connect                │ NOT supported                    │
  │ awsvpc Network Mode            │ NOT supported                    │
  │ Capacity Providers             │ NOT supported                    │
  │ EFS Volumes                    │ NOT supported                    │
  │ App Mesh                       │ NOT supported                    │
  │ Fargate                        │ NOT supported (no Fargate on-prem│
  │ SELinux                        │ NOT supported                    │
  ├────────────────────────────────┼──────────────────────────────────┤
  │ bridge/host/none networking    │ Supported                        │
  │ ECS Exec                       │ Supported                        │
  │ Custom placement attributes    │ Supported                        │
  │ CloudWatch Logs                │ Supported (requires exec role)   │
  │ Secrets Manager / SSM Params   │ Supported                        │
  └────────────────────────────────┴──────────────────────────────────┘

  NETWORKING REQUIREMENT:
  ────────────────────────────────────────────────────
  External instances must have outbound HTTPS (port 443) access to:
  ├── ecs.region.amazonaws.com
  ├── ecs-a-*.region.amazonaws.com (task management)
  ├── ecs-t-*.region.amazonaws.com (metrics)
  ├── ssm.region.amazonaws.com
  ├── ec2messages.region.amazonaws.com
  └── ssmmessages.region.amazonaws.com


  WHEN TO USE ECS ANYWHERE (AND WHEN NOT TO):
  ────────────────────────────────────────────────────

  GOOD FIT:
  ├── Data processing at the edge (IoT, factory floors, retail stores)
  ├── Batch workloads that must run on-premises (compliance/data gravity)
  ├── Unified management plane — same ECS API for cloud and on-prem
  ├── Migration bridge — run ECS on-prem while migrating to cloud
  └── Outbound-heavy workloads that push results to S3/SQS/DynamoDB

  BAD FIT:
  ├── Web services or APIs that need load balancing (no ALB/NLB support)
  ├── Microservices that need service discovery or service mesh
  ├── Workloads requiring awsvpc networking or per-task security groups
  ├── Workloads already running on-prem Kubernetes (use EKS Anywhere instead)
  └── Anything that needs Fargate simplicity on-premises
```

### Terraform: ECS Anywhere

```hcl
# IAM role for external instances
resource "aws_iam_role" "ecs_anywhere" {
  name = "ecsAnywhereRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ssm.amazonaws.com"   # SSM assumes this role, not EC2
      }
    }]
  })
}

# Attach the ECS Anywhere policy
resource "aws_iam_role_policy_attachment" "ecs_anywhere" {
  role       = aws_iam_role.ecs_anywhere.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy_attachment" "ecs_anywhere_ecs" {
  role       = aws_iam_role.ecs_anywhere.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}

# SSM Activation — generates the activation ID and code needed to register
# external instances. Without this, you have no credentials for the install script.
resource "aws_ssm_activation" "ecs_anywhere" {
  name               = "ecs-anywhere-activation"
  iam_role           = aws_iam_role.ecs_anywhere.id
  registration_limit = 10
  depends_on         = [aws_iam_role_policy_attachment.ecs_anywhere]
}

# ECS Cluster — external instances register here
resource "aws_ecs_cluster" "hybrid" {
  name = "hybrid-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# Task definition for EXTERNAL launch type
resource "aws_ecs_task_definition" "edge_processor" {
  family                   = "edge-processor"
  requires_compatibilities = ["EXTERNAL"]   # EXTERNAL launch type
  network_mode             = "bridge"        # Only bridge, host, or none

  # Execution role needed for CloudWatch Logs on external instances
  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.edge_task.arn

  # Note: NO cpu/memory at task level for EXTERNAL (unlike Fargate)
  container_definitions = jsonencode([
    {
      name      = "processor"
      image     = "123456789012.dkr.ecr.us-east-1.amazonaws.com/edge-processor:latest"
      essential = true
      cpu       = 512
      memory    = 1024

      environment = [
        { name = "QUEUE_URL", value = "https://sqs.us-east-1.amazonaws.com/123456789012/edge-work" }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/edge-processor"
          "awslogs-region"        = "us-east-1"
          "awslogs-stream-prefix" = "edge"
        }
      }
    }
  ])
}

# ECS Service for external instances
resource "aws_ecs_service" "edge_processor" {
  name            = "edge-processor"
  cluster         = aws_ecs_cluster.hybrid.id
  task_definition = aws_ecs_task_definition.edge_processor.arn
  desired_count   = 2
  launch_type     = "EXTERNAL"    # Run on external instances

  # No load_balancer block — not supported
  # No network_configuration — not awsvpc
  # No service_registries — service discovery not supported
  # No capacity_provider_strategy — not supported for EXTERNAL

  deployment_maximum_percent         = 200
  deployment_minimum_healthy_percent = 100

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}

# Registration: on the external instance, run:
# curl --proto "https" -o "/tmp/ecs-anywhere-install.sh" \
#   "https://amazon-ecs-agent.s3.amazonaws.com/ecs-anywhere-install-latest.sh"
# sudo bash /tmp/ecs-anywhere-install.sh \
#   --cluster hybrid-cluster \
#   --activation-id "${aws_ssm_activation.ecs_anywhere.id}" \
#   --activation-code "${aws_ssm_activation.ecs_anywhere.activation_code}" \
#   --region us-east-1
```

---

## Part 5: Advanced Capacity Provider Strategies - The Facilities Team

### The Analogy

Your corporation's **facilities team** manages office space. They decide how to allocate desks across floors and buildings. Their decisions involve several trade-offs:

- **Binpack** ("fill one floor before opening another"): Pack employees tightly onto a few floors. This minimizes rent because you pay per floor, but if one floor loses power, many employees are affected.
- **Spread** ("distribute evenly across all floors"): Spread employees across every available floor. If one floor loses power, you only lose a fraction of the workforce. But you pay rent on all those floors even if some are half-empty.
- **targetCapacity** ("how full to pack each floor"): Do you fill each floor to 100% capacity, or keep 20% of desks empty for walk-ins? A target of 80% means the facilities team opens a new floor when any existing floor hits 80% occupancy, always keeping 20% headroom.
- **Zonal strategy** ("one landlord per building wing"): Instead of one large building with all floors managed together, you have separate wings (one ASG per AZ), each with its own landlord. This guarantees perfectly even distribution because each wing manages its own capacity independently.

The facilities team also has a **termination protection** policy: they will never close a floor while employees are still working, even if the building is being consolidated.

### The Technical Reality

Advanced capacity provider strategies optimize cost, availability, and task placement across EC2-backed ECS clusters. The key mechanisms are managed scaling (automated ASG adjustments), the CapacityProviderReservation metric, targetCapacity tuning, placement strategies, and zonal capacity providers.

```
MANAGED SCALING ALGORITHM — HOW IT WORKS
=============================================================================

  STEP 1: ECS calculates how many instances are needed
  ────────────────────────────────────────────────────
  ├── Groups pending tasks by identical resource requirements
  ├── Uses binpack calculation: vCPU, memory, ENIs, ports, GPUs
  ├── Determines MINIMUM number of instances to fit all tasks
  └── Example: 10 tasks × 1 vCPU each, instances have 4 vCPU
      → minimum 3 instances (packing ~3-4 tasks per instance)

  STEP 2: ECS publishes the CapacityProviderReservation metric
  ────────────────────────────────────────────────────

  CapacityProviderReservation = (instances needed / instances running) × 100

  Example scenarios:
  ├── Need 5, have 5 → 100% reservation → at target if targetCapacity=100
  ├── Need 5, have 4 → 125% reservation → SCALE OUT (over target)
  ├── Need 5, have 8 → 62.5% reservation → SCALE IN (under target)
  └── Need 0, have 0 → 100% reservation → no action
      (AWS defines 0/0 as 100% to prevent division by zero and avoid unnecessary scaling)

  STEP 3: CloudWatch alarms trigger scaling
  ────────────────────────────────────────────────────
  ├── Reservation > targetCapacity → scale OUT alarm fires
  │   → ASG desired count increases → new instances launch
  ├── Reservation < targetCapacity → scale IN alarm fires
  │   → ASG desired count decreases → instances terminate
  └── Reservation = targetCapacity → no action needed


  targetCapacity TUNING:
  ────────────────────────────────────────────────────

  targetCapacity = 100%:
  ├── Pack instances fully — no spare capacity
  ├── Maximum cost efficiency
  ├── New tasks must WAIT for a new instance to launch (~2-5 min)
  └── Good for: predictable workloads with no burst requirements

  targetCapacity = 80%:
  ├── Keep 20% headroom on each instance
  ├── New tasks can be placed immediately on spare capacity
  ├── Slightly higher cost (paying for unused capacity)
  └── Good for: most production workloads (RECOMMENDED DEFAULT)

  targetCapacity = 50%:
  ├── Keep 50% headroom — half the fleet is spare capacity
  ├── Very fast task placement but expensive
  └── Good for: latency-sensitive workloads where task startup matters


  SPECIAL BEHAVIORS:
  ────────────────────────────────────────────────────

  SCALING FROM ZERO:
  ├── When ECS needs to scale from 0 → N instances, it always
  │   launches 2 instances initially (not 1)
  ├── WHY? Target tracking scaling needs at least 2 data points to
  │   calculate CapacityProviderReservation accurately
  └── This also provides redundancy during the cold-start phase

  STABILIZATION WINDOW:
  ├── 15-minute waiting period between scale-out and scale-in
  └── Prevents rapid oscillation (thrashing) during deployment churn

  INSTANCE WARMUP PERIOD (instanceWarmupPeriod):
  ├── Default: 300 seconds (5 minutes)
  ├── New instances are NOT counted as available capacity during this window
  ├── Prevents the algorithm from thinking capacity exists before
  │   the instance is ready to accept tasks
  └── Set this to match your instance bootstrap + agent startup time

  STEP SIZE LIMITS:
  ├── minimumScalingStepSize: minimum number of instances to add/remove
  │   per scaling action (default: 1)
  └── maximumScalingStepSize: maximum instances to add/remove
      per scaling action (default: 10,000)


  CRITICAL EDGE CASE:
  ────────────────────────────────────────────────────
  If a task's resource requirements EXCEED the capacity of the smallest
  instance type in your ASG, the task remains stuck in PROVISIONING
  state forever. No scaling occurs — the system silently fails.

  Example: task needs 8 vCPU but ASG uses t3.medium (2 vCPU) instances
  → task will never be placed, no alarm fires, no instances launch
  → SOLUTION: ensure your ASG instance types can accommodate your
    largest task definition
```

### Placement Strategies: Binpack vs Spread

```
PLACEMENT STRATEGIES — ORDERING MATTERS
=============================================================================

  ECS evaluates placement strategies IN ORDER. The first strategy is
  the primary grouping, the second is the tiebreaker within each group.

  STRATEGY 1: spread(attribute:ecs.availability-zone), binpack(cpu)
  ────────────────────────────────────────────────────
  First: spread tasks evenly across AZs
  Then: within each AZ, pack tightly by CPU

  AZ-a:  [████████│████    │         ]  ← packed toward fewer instances
  AZ-b:  [████████│████    │         ]  ← same distribution per AZ
  AZ-c:  [████████│████    │         ]

  Result: balanced across AZs (resilient), packed within each AZ (cost-efficient)
  This is the RECOMMENDED default for production.


  STRATEGY 2: binpack(cpu)
  ────────────────────────────────────────────────────
  Just pack as tightly as possible, ignoring AZ balance:

  AZ-a:  [████████│████████│████████ ]  ← all tasks here first
  AZ-b:  [████████│                  ]  ← overflow
  AZ-c:  [                           ]  ← empty

  Result: cheapest possible (fewest instances), but one AZ failure
  could take out 80%+ of capacity.


  STRATEGY 3: spread(attribute:ecs.availability-zone)
  ────────────────────────────────────────────────────
  Spread across AZs, but no packing within each AZ:

  AZ-a:  [██      │██      │██       ]  ← tasks spread across instances
  AZ-b:  [██      │██      │██       ]
  AZ-c:  [██      │██      │██       ]

  Result: maximum resilience, but many underutilized instances (expensive).


  CRITICAL RULE: BINPACK BEFORE SPREAD WHEN targetCapacity < 100%
  ────────────────────────────────────────────────────
  When targetCapacity is less than 100%, the placement strategy must
  have binpack listed BEFORE spread in the ordered_placement_strategy
  list (lower index = evaluated first). Otherwise:

  ├── Spread places one task per instance
  ├── Capacity provider sees each instance as "used" even if mostly empty
  ├── But reservation calculation says "I need more instances"
  ├── Premature scale-out occurs — you end up with way too many instances
  └── binpack consolidates tasks first, then spread distributes groups
```

### Zonal Capacity Providers: Guaranteeing Even AZ Distribution

```
ZONAL CAPACITY PROVIDERS — THE PRODUCTION PATTERN
=============================================================================

  PROBLEM: Default ECS placement with a single multi-AZ ASG is "best effort"
  ────────────────────────────────────────────────────
  A single ASG across 3 AZs might launch instances unevenly.

  WHY? The spread strategy operates PER PLACEMENT DECISION, not as a global
  optimizer across all services. When Service A deploys 4 tasks, 3 spread
  evenly and the "extra" goes to one AZ. Service B deploys — same pattern,
  same AZ gets the extra again. Over multiple deployments, one AZ accumulates
  more tasks and fills up first. If an instance type is temporarily unavailable
  in one AZ, the ASG launches in another, further skewing distribution.

  ┌────────────────────────────────────────────────────┐
  │  SINGLE ASG (multi-AZ)                              │
  │                                                    │
  │  AZ-a: [instance-1] [instance-2] [instance-3]     │  ← 6 tasks
  │  AZ-b: [instance-4]                               │  ← 2 tasks
  │  AZ-c: [instance-5] [instance-6]                  │  ← 4 tasks
  │                                                    │
  │  During rolling deployment or partial scale-in,    │
  │  imbalance gets WORSE over time.                   │
  └────────────────────────────────────────────────────┘

  SOLUTION: One ASG per AZ, each linked to its own capacity provider
  ────────────────────────────────────────────────────

  ┌────────────────────────────────────────────────────┐
  │  ZONAL CAPACITY PROVIDERS                           │
  │                                                    │
  │  CP: asg-az-a (weight=1)     ASG: ecs-az-a        │
  │  ├── AZ-a only                                     │
  │  └── [instance-1] [instance-2]  ← 4 tasks         │
  │                                                    │
  │  CP: asg-az-b (weight=1)     ASG: ecs-az-b        │
  │  ├── AZ-b only                                     │
  │  └── [instance-3] [instance-4]  ← 4 tasks         │
  │                                                    │
  │  CP: asg-az-c (weight=1)     ASG: ecs-az-c        │
  │  ├── AZ-c only                                     │
  │  └── [instance-5] [instance-6]  ← 4 tasks         │
  │                                                    │
  │  Equal weights → equal task distribution            │
  │  Each AZ scales independently                      │
  └────────────────────────────────────────────────────┘

  TRADE-OFF:
  ├── PRO: Guaranteed even AZ distribution (not "best effort")
  ├── PRO: Each AZ scales independently (no cross-AZ dependencies)
  ├── PRO: spread strategy becomes redundant — AZ distribution is enforced
  │         by capacity provider weights, so you only need binpack within each AZ
  ├── CON: May waste capacity if one AZ has a partially filled instance
  │         that cannot be shared with tasks in another AZ
  ├── CON: No cross-AZ overflow — if one AZ has a capacity constraint,
  │         tasks for that AZ's provider get stuck in PROVISIONING
  ├── CON: More ASGs to manage (3x the ASG resources)
  └── CON: Harder to use mixed instance types across AZs
```

### Terraform: Zonal Capacity Providers

```hcl
# Create one ASG per AZ, each linked to its own capacity provider
locals {
  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# One ASG per AZ
resource "aws_autoscaling_group" "ecs_zonal" {
  for_each = toset(local.azs)

  name                = "ecs-${each.value}"
  desired_capacity    = 2
  min_size            = 0
  max_size            = 10
  vpc_zone_identifier = [var.private_subnet_ids_by_az[each.value]]

  protect_from_scale_in = true  # Required for managed termination protection

  launch_template {
    id      = aws_launch_template.ecs.id
    version = "$Latest"
  }

  tag {
    key                 = "AmazonECSManaged"
    value               = true
    propagate_at_launch = true
  }

  tag {
    key                 = "Name"
    value               = "ecs-${each.value}"
    propagate_at_launch = true
  }
}

# One capacity provider per AZ
resource "aws_ecs_capacity_provider" "zonal" {
  for_each = toset(local.azs)

  name = "ec2-${each.value}"

  auto_scaling_group_provider {
    auto_scaling_group_arn = aws_autoscaling_group.ecs_zonal[each.value].arn

    managed_scaling {
      status                    = "ENABLED"
      target_capacity           = 80   # 20% headroom per AZ
      minimum_scaling_step_size = 1
      maximum_scaling_step_size = 5
      instance_warmup_period    = 300
    }

    managed_termination_protection = "ENABLED"
  }
}

# Cluster with equal-weight zonal strategy
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = [for az in local.azs : aws_ecs_capacity_provider.zonal[az].name]

  # Equal weights guarantee even distribution across AZs
  dynamic "default_capacity_provider_strategy" {
    for_each = toset(local.azs)
    content {
      capacity_provider = aws_ecs_capacity_provider.zonal[default_capacity_provider_strategy.value].name
      weight            = 1   # Equal weight = equal distribution
      base              = 0
    }
  }
}

# Service using zonal capacity providers
resource "aws_ecs_service" "web_app" {
  name            = "web-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.web_app.arn
  desired_count   = 12   # 4 per AZ with equal weights

  # Inherits cluster default capacity provider strategy (equal-weight zonal)
  # OR explicitly specify:
  dynamic "capacity_provider_strategy" {
    for_each = toset(local.azs)
    content {
      capacity_provider = aws_ecs_capacity_provider.zonal[capacity_provider_strategy.value].name
      weight            = 1
    }
  }

  network_configuration {
    subnets          = values(var.private_subnet_ids_by_az)
    security_groups  = [aws_security_group.web_app.id]
    assign_public_ip = false
  }

  # Placement strategy: within each AZ (handled by zonal providers),
  # pack tightly by CPU
  ordered_placement_strategy {
    type  = "binpack"
    field = "cpu"
  }
}
```

### Managed Termination Protection

```
MANAGED TERMINATION PROTECTION
=============================================================================

  PROBLEM: During scale-in, the ASG might terminate an instance that
  still has running tasks, killing those tasks abruptly.

  SOLUTION: Managed termination protection prevents this.

  HOW IT WORKS:
  ────────────────────────────────────────────────────

  1. ECS sets instance scale-in protection on instances with running tasks
  2. When ASG wants to scale in, it skips protected instances
  3. ECS identifies the best candidate for termination:
     ├── Prefers instances with the fewest running tasks
     ├── Prefers instances with tasks that can be placed elsewhere
     └── Respects the ASG's termination policy for tiebreaking
  4. ECS drains the chosen instance:
     ├── Stops scheduling new tasks on it
     ├── Waits for running tasks to complete (or be replaced)
     └── Removes scale-in protection
  5. ASG terminates the now-empty instance

  REQUIREMENTS:
  ├── managed_termination_protection = "ENABLED" on the capacity provider
  ├── protect_from_scale_in = true on the ASG
  └── Both settings must be active — one without the other does not work

  WITHOUT TERMINATION PROTECTION:
  ├── ASG picks an instance based on its termination policy
  ├── Instance is terminated immediately
  ├── Running tasks are killed with SIGTERM → SIGKILL
  ├── ECS service detects missing tasks and launches replacements
  └── Brief capacity drop during the replacement window
```

---

## Service-to-Service Communication Decision Framework

```
CHOOSING THE RIGHT INTERCONNECTION METHOD
=============================================================================

  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  Are your services within the same ECS namespace?              │
  │  ├── YES → Use SERVICE CONNECT (recommended)                   │
  │  │         Managed proxy, load balancing, retries, metrics     │
  │  │                                                             │
  │  └── NO → Do they span accounts, VPCs, or compute types?      │
  │           ├── YES → Use VPC LATTICE                            │
  │           │         Cross-account, cross-VPC, ECS+EKS+Lambda  │
  │           │         (the recommended replacement for App Mesh  │
  │           │         cross-compute-type networking — provides   │
  │           │         service-to-service auth, traffic mgmt,     │
  │           │         and observability across compute types)    │
  │           │                                                    │
  │           └── NO → Do you need just DNS resolution?            │
  │                    ├── YES → Use RAW CLOUD MAP                 │
  │                    │         Simple, but no proxy features     │
  │                    │                                           │
  │                    └── NO → Is this external (internet)?       │
  │                             ├── YES → Use ALB / NLB            │
  │                             └── NO → Re-evaluate architecture  │
  └─────────────────────────────────────────────────────────────────┘


  COMPATIBILITY BY NETWORK MODE:
  ────────────────────────────────────────────────────

  ┌──────────────────┬──────────┬────────┬──────┬──────┐
  │ Method           │ awsvpc   │ bridge │ host │ none │
  ├──────────────────┼──────────┼────────┼──────┼──────┤
  │ Service Connect  │   Yes    │  Yes   │  No  │  No  │
  │ Cloud Map (DNS)  │ A/AAAA/  │  SRV   │ SRV  │  No  │
  │                  │ SRV      │  only  │ only │      │
  │ VPC Lattice      │   Yes    │  Yes   │ Yes  │  No  │
  │ ALB/NLB          │   Yes    │  Yes   │ Yes  │  No  │
  │ App Mesh (EOL)   │   Yes    │  Yes   │  No  │  No  │
  └──────────────────┴──────────┴────────┴──────┴──────┘
```

---

## Key Takeaways

### Service Discovery (Cloud Map)
1. **Cloud Map is the foundation, not the complete solution** -- It provides DNS-based or API-based service registration and lookup, but does not include load balancing, retries, or traffic management; your application code must handle all of that
2. **DNS record type depends on network mode** -- Use A records with awsvpc (each task has its own IP) and SRV records with bridge/host (you need both IP and port because tasks share the instance IP)
3. **Private IPs only, always** -- Even with a public namespace, Cloud Map registers the task's private IP; do not expect public IP resolution
4. **1,000 tasks per Cloud Map service** -- This is a Route 53 quota; if you have a service that might scale beyond this, plan for partitioning or use Service Connect

### Service Connect
5. **Service Connect is the recommended default for all new ECS service-to-service communication** -- It combines Cloud Map discovery with a managed Envoy proxy to provide load balancing, outlier detection, retries, and CloudWatch metrics without application code changes
6. **Client vs client-server is about whether you receive calls, not just make them** -- A client service gets a proxy for outgoing traffic only; a client-server service also registers endpoints and accepts incoming connections through the proxy
7. **Short names only work within the same namespace** -- Services in different namespaces cannot use Service Connect to reach each other; however, AWS supports shared Cloud Map namespaces across accounts via AWS RAM; for cross-namespace communication, use VPC Lattice or ALB
8. **The proxy shares your task's CPU and memory — budget at least 256 MB and 0.25 vCPU extra** -- There is no extra charge for Service Connect itself, but the Envoy proxy typically consumes ~256 MB of memory and 0.25 vCPU as a baseline; for a task sized at 512 MB / 0.25 vCPU, the proxy would consume half of the task's resources, so size your tasks accordingly

### App Mesh (Legacy)
9. **App Mesh reaches end of life on September 30, 2026** -- All App Mesh workloads should migrate to Service Connect (for ECS-only) or VPC Lattice (for cross-compute-type scenarios); the key mapping is virtual nodes become Service Connect endpoints, virtual services become client aliases
10. **Service Connect does not replace weighted routing between versions** -- If you relied on App Mesh virtual routers for canary deployments (80/20 traffic splits), use CodeDeploy blue/green with traffic shifting policies on the ECS service instead

### ECS Anywhere
11. **Focus on what ECS Anywhere cannot do, not what it can** -- No load balancers, no service discovery, no Service Connect, no awsvpc, no capacity providers, no EFS; this makes it suitable only for outbound workloads (batch processing, edge computing, data pipelines) rather than inbound request-serving services
12. **SSM Agent is the lifeline** -- Credentials are rotated every 30 minutes via hardware fingerprint; if the external instance loses connectivity to SSM endpoints, the ECS Agent cannot receive task assignments or report status
13. **One cluster per instance** -- Each external instance can only be registered to a single ECS cluster at a time; re-registration requires deregistering from the current cluster first

### Advanced Capacity Provider Strategies
14. **Set targetCapacity to 80% for most production workloads** -- This keeps 20% headroom on each instance for burst task placement without waiting for new instances to launch; 100% is cheaper but means every new task waits for scale-out
15. **Binpack must be listed BEFORE spread in ordered_placement_strategy when targetCapacity is below 100%** -- Otherwise spread places one task per instance, the instances look "used" but mostly empty, and the capacity provider prematurely scales out because reservation exceeds target
16. **Use zonal capacity providers for guaranteed even AZ distribution** -- One ASG per AZ with equal-weight capacity provider strategies enforces even distribution mathematically rather than relying on best-effort placement
17. **Always enable managed termination protection with protect_from_scale_in** -- Both settings must be active together; managed termination protection on the capacity provider tells ECS to manage draining, and protect_from_scale_in on the ASG tells the ASG to respect instance protection flags
18. **Scaling from zero always launches 2 instances** -- This is not configurable; ECS ensures redundancy during cold-start by launching a minimum of 2 instances even if the workload only needs one

---

*Written on February 17, 2026*
