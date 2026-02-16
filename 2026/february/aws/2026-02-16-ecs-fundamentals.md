# AWS ECS Fundamentals: Task Definitions, Services, and Capacity Providers - The Film Studio Analogy

> Understanding the core building blocks of Amazon Elastic Container Service through a real-world analogy of a film studio that produces shows on demand -- from screenplays (task definitions) to production teams (services) to sound stages and equipment (capacity providers), with everything orchestrated to keep the studio running smoothly and profitably.

---

## TL;DR

| AWS ECS Concept | Real-World Analogy | One-Liner |
|-----------------|-------------------|-----------|
| ECS Cluster | Film studio lot | The physical campus where all productions happen; a logical grouping of resources |
| Task Definition | Screenplay / production blueprint | An immutable document specifying every detail: actors needed, costumes, props, stage requirements |
| Task Definition Revision | New edition of the screenplay | Each edit creates a new numbered version; old versions remain available for rollback |
| Container Definition | Role description within the screenplay | Each actor's part: their costume (image), lines (commands), and stage directions (resource limits) |
| Task | A single live performance | One running instance of the screenplay -- actors on stage, cameras rolling |
| Service | The production manager | Ensures the right number of performances are always running, replaces any that fail, and manages show transitions |
| Desired Count | Target number of simultaneous performances | "Keep 4 shows running at all times across all stages" |
| Task Role | The actor's security clearance badge | What the actor can access backstage (S3, DynamoDB, SQS) while performing |
| Task Execution Role | The stage manager's credentials | What the stage manager needs to set up the performance: pull costumes (images), set up lighting (logs) |
| Fargate Launch Type | Renting a fully managed sound stage | Show up, perform, leave -- the studio handles the building, electricity, and cleanup |
| EC2 Launch Type | Owning your own sound stages | You manage the buildings and equipment, but you control every detail |
| Capacity Provider | The studio's resource procurement department | Decides where to find sound stages -- rent them (Fargate), use owned ones (EC2 ASG), or mix both |
| Capacity Provider Strategy | The procurement policy | "Use 70% owned stages, 30% rented stages" or "rent first, overflow to owned" |
| awsvpc Network Mode | Each performance gets its own phone line | Every task gets a dedicated ENI with its own IP address and security groups |
| Rolling Update | Gradually replacing actors between shows | New actors start performing while old ones finish their current show; audience never notices the swap |
| Blue/Green Deployment | Opening the new show on a second stage | Run both old and new shows simultaneously, shift the audience to the new stage, then close the old one |
| Service Auto Scaling | Audience demand tracker | Automatically adds or removes simultaneous performances based on ticket demand |
| Deployment Circuit Breaker | The studio's quality control inspector | If the new show keeps failing during rehearsal, automatically revert to the previous production |

---

## The Big Picture

Imagine you run a **film studio** that produces live performances on demand. Audiences (users) arrive at your box office (load balancer), and your studio must have enough shows running to serve them all. You need three fundamental things:

1. **Screenplays** (task definitions) -- Detailed blueprints that describe every aspect of a production: which actors are needed, what costumes they wear, how much stage space they require, and what props they need access to.
2. **A production manager** (service) -- Someone who ensures the right number of shows are always running, replaces any that fail mid-performance, and smoothly transitions from old productions to new ones.
3. **Sound stages and equipment** (capacity providers) -- The physical infrastructure where performances happen. You can rent fully managed stages (Fargate) or own and operate your own (EC2).

All of this happens within a **studio lot** (cluster) -- the logical boundary that contains your productions, stages, and staff.

```
ECS RESOURCE HIERARCHY
=============================================================================

  ┌─────────────────────────────────────────────────────────────────────┐
  │                         CLUSTER (Studio Lot)                        │
  │                                                                     │
  │  CAPACITY PROVIDERS (Where shows happen)                           │
  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────────┐  │
  │  │ Fargate         │  │ Fargate Spot    │  │ EC2 ASG Provider  │  │
  │  │ (rented stages) │  │ (discount rent) │  │ (owned stages)    │  │
  │  └─────────────────┘  └─────────────────┘  └───────────────────┘  │
  │                                                                     │
  │  SERVICES (Production managers)                                    │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │ Service: "web-frontend"                                      │  │
  │  │ ├── Task Definition: web-app:14 (screenplay v14)            │  │
  │  │ ├── Desired Count: 4                                        │  │
  │  │ ├── Capacity Provider Strategy: Fargate=1, Fargate_Spot=2   │  │
  │  │ ├── Load Balancer: ALB target group                         │  │
  │  │ └── Auto Scaling: Target 1000 req/target                    │  │
  │  │                                                              │  │
  │  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                       │  │
  │  │  │Task 1│ │Task 2│ │Task 3│ │Task 4│  Running tasks         │  │
  │  │  └──────┘ └──────┘ └──────┘ └──────┘                       │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │ Service: "api-backend"                                       │  │
  │  │ ├── Task Definition: api-server:7                            │  │
  │  │ ├── Desired Count: 6                                        │  │
  │  │ └── Capacity Provider Strategy: EC2 ASG Provider             │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────────────┘

  KEY INSIGHT: The hierarchy flows top-down:
  Cluster → contains Services → reference Task Definitions + Capacity Providers
  Task Definitions → describe Containers
  Services → launch and maintain Tasks (running instances of Task Definitions)
  Capacity Providers → supply the compute for those Tasks
```

---

## Part 1: Task Definitions - The Screenplay

### The Analogy

A **screenplay** is the complete production blueprint. It specifies every detail about a performance: the number of actors (containers), each actor's costume and props (container image, environment variables), how much stage space each actor needs (CPU and memory), which doors they can use to enter and exit (port mappings), and what backstage areas they can access (IAM roles). The screenplay is **immutable** -- once published, it cannot be changed. If you need to make edits, you publish a new edition (revision). Old editions stay on file so you can always go back to a previous version.

### The Technical Reality

A task definition is a JSON document that serves as the blueprint for running containers in ECS. It defines one or more containers, their resource requirements, networking configuration, storage, logging, and IAM permissions. Task definitions are versioned -- each update creates a new revision (e.g., `web-app:1`, `web-app:2`, `web-app:3`).

```
TASK DEFINITION ANATOMY
=============================================================================

  TASK-LEVEL SETTINGS (apply to the entire production):
  ────────────────────────────────────────────────────
  ├── Family name: "web-app" (the screenplay title)
  ├── Revision: 14 (edition number, auto-incremented)
  ├── Task Role: what containers can access (S3, DynamoDB, etc.)
  ├── Execution Role: what ECS needs (pull images, write logs)
  ├── Network Mode: awsvpc, bridge, host, or none
  ├── CPU: total vCPU allocated to the task (required for Fargate)
  ├── Memory: total memory allocated to the task (required for Fargate)
  ├── Requires Compatibilities: FARGATE, EC2, or both
  └── Volumes: shared storage accessible to all containers in the task

  CONTAINER-LEVEL SETTINGS (per actor in the production):
  ────────────────────────────────────────────────────
  ├── Name: "nginx" (the actor's stage name)
  ├── Image: "nginx:1.25" (the costume)
  ├── CPU: container-level CPU (soft limit on EC2, ignored on Fargate)
  ├── Memory Hard Limit: maximum memory before OOM kill
  ├── Memory Soft Limit: memory reservation for placement decisions
  ├── Port Mappings: container port → host port
  ├── Environment Variables: configuration passed to the container
  ├── Secrets: values pulled from Secrets Manager / SSM at launch
  ├── Log Configuration: where to send stdout/stderr
  ├── Health Check: command to verify the container is healthy
  ├── Essential: if true, task stops when this container stops
  ├── Depends On: startup ordering between containers
  └── Entry Point / Command: override the container's default command


  TWO IAM ROLES — A COMMON CONFUSION:
  ────────────────────────────────────────────────────

  TASK ROLE (the actor's security badge):
  ├── Assumed BY the containers at runtime
  ├── Grants permissions for application code to call AWS APIs
  ├── Example: read from S3, write to DynamoDB, send to SQS
  └── This is what YOUR CODE uses

  TASK EXECUTION ROLE (the stage manager's credentials):
  ├── Assumed BY the ECS agent / Fargate platform
  ├── Grants permissions for ECS infrastructure operations
  ├── Example: pull images from ECR, write logs to CloudWatch,
  │   retrieve secrets from Secrets Manager / SSM Parameter Store
  └── This is what ECS ITSELF uses to set up the task

  If your container cannot pull its image → check execution role
  If your application cannot read from S3 → check task role
```

### Task Definition JSON Example

```json
{
  "family": "web-app",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/webAppTaskRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/web-app:v2.3.1",
      "essential": true,
      "cpu": 768,
      "memoryReservation": 1536,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "APP_ENV", "value": "production" },
        { "name": "LOG_LEVEL", "value": "info" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-password"
        },
        {
          "name": "API_KEY",
          "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/prod/api-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/web-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "web"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    },
    {
      "name": "log-router",
      "image": "amazon/aws-for-fluent-bit:latest",
      "essential": false,
      "cpu": 256,
      "memoryReservation": 512,
      "firelensConfiguration": {
        "type": "fluentbit"
      }
    }
  ]
}
```

### Networking Modes

```
ECS NETWORKING MODES
=============================================================================

  awsvpc (RECOMMENDED — required for Fargate)
  ────────────────────────────────────────────
  ├── Each TASK gets its own ENI with a private IP from your VPC subnet
  ├── Tasks are first-class VPC citizens with their own security groups
  ├── Container port = host port (no port mapping conflicts)
  ├── The ONLY option for Fargate
  ├── Limitation: ENI density limits tasks per EC2 instance
  │   (mitigated by ENI trunking on supported instance types)
  └── Think: "every performance gets its own phone line"

  bridge (EC2 only — Docker default)
  ────────────────────────────────────────────
  ├── Containers share the EC2 instance's network via Docker bridge
  ├── Uses dynamic port mapping (container port 8080 → host port 49153)
  ├── Multiple tasks can run on one instance (different host ports)
  ├── ALB handles dynamic port discovery automatically
  └── Think: "all actors share one phone line with extension numbers"

  host (EC2 only)
  ────────────────────────────────────────────
  ├── Container ports map directly to the EC2 host ports
  ├── Maximum network performance (no Docker networking overhead)
  ├── Limitation: only one task per port per instance
  └── Think: "the actor IS the phone line — no sharing"

  none
  ────────────────────────────────────────────
  ├── No external network connectivity
  └── Rare: only for tasks that need no network access


  DECISION:
  ────────────────────────────────────────────
  Using Fargate?          → awsvpc (only option)
  Using EC2, need SGs per task? → awsvpc
  Using EC2, need density?      → bridge (more tasks per instance)
  Using EC2, need raw perf?     → host (but one task per port per host)
```

### Terraform: Task Definition

```hcl
# ECS Task Definition
resource "aws_ecs_task_definition" "web_app" {
  family                   = "web-app"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 1024    # 1 vCPU
  memory                   = 2048    # 2 GB

  # The stage manager's credentials — ECS uses this to pull images and write logs
  execution_role_arn = aws_iam_role.ecs_execution.arn

  # The actor's security badge — application code uses this for AWS API calls
  task_role_arn = aws_iam_role.web_app_task.arn

  container_definitions = jsonencode([
    {
      name      = "web"
      image     = "${aws_ecr_repository.web_app.repository_url}:latest"
      essential = true
      cpu       = 768
      memoryReservation = 1536

      portMappings = [
        {
          containerPort = 8080
          protocol      = "tcp"
        }
      ]

      environment = [
        { name = "APP_ENV", value = "production" }
      ]

      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = aws_secretsmanager_secret.db_password.arn
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.web_app.name
          "awslogs-region"        = var.region
          "awslogs-stream-prefix" = "web"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])

  tags = {
    Environment = "production"
    Service     = "web-app"
  }
}

# Task Execution Role — what ECS needs to set up the task
resource "aws_iam_role" "ecs_execution" {
  name = "ecs-task-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Additional execution role policy for Secrets Manager access
resource "aws_iam_role_policy" "execution_secrets" {
  name = "secrets-access"
  role = aws_iam_role.ecs_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue"
      ]
      Resource = [aws_secretsmanager_secret.db_password.arn]
    }]
  })
}

# Task Role — what the application code can access at runtime
resource "aws_iam_role" "web_app_task" {
  name = "web-app-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "web_app_task" {
  name = "web-app-permissions"
  role = aws_iam_role.web_app_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::my-app-bucket/*"
      },
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:Query"
        ]
        Resource = "arn:aws:dynamodb:us-east-1:123456789012:table/my-table"
      }
    ]
  })
}
```

---

## Part 2: Services - The Production Manager

### The Analogy

A **production manager** takes a screenplay and keeps the show running. Their responsibilities include maintaining the right number of simultaneous performances (desired count), replacing any performance that fails mid-show (task replacement), smoothly transitioning from the old screenplay to a new edition without the audience noticing (deployments), and working with the box office (load balancer) to direct audience members to available shows.

The production manager has two scheduling strategies: **replica** mode ("keep exactly 4 shows running, spread across our stages") and **daemon** mode ("run exactly one show on every stage in the lot, no matter how many stages we have").

When a new edition of the screenplay arrives, the production manager must transition from the old version to the new one. They can do this as a **rolling update** (gradually replace actors one at a time -- start a new actor, verify they are performing well, then let an old actor leave) or as a **blue/green deployment** (open the new show on a separate stage, shift the audience over, then close the old stage).

### The Technical Reality

An ECS service is a long-running configuration that maintains a specified number of task instances from a given task definition. It integrates with Elastic Load Balancing for traffic distribution, manages deployments when the task definition is updated, and supports auto scaling to adjust task count based on demand.

```
ECS SERVICE — WHAT IT MANAGES
=============================================================================

  ┌─────────────────────────────────────────────────────────────────────┐
  │                         ECS SERVICE                                 │
  │                                                                     │
  │  Core Responsibility: "Keep N healthy tasks running at all times"   │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ TASK MAINTENANCE                                              │  │
  │  │ ├── Launches tasks to reach desired count                    │  │
  │  │ ├── Replaces tasks that fail health checks                   │  │
  │  │ ├── Replaces tasks that exit unexpectedly                    │  │
  │  │ └── Spreads tasks across AZs (AZ-balanced spread)            │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ LOAD BALANCER INTEGRATION                                     │  │
  │  │ ├── Registers healthy tasks as targets in ALB/NLB            │  │
  │  │ ├── Deregisters unhealthy or draining tasks                  │  │
  │  │ └── Supports multiple target groups (one task, multiple LBs) │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ DEPLOYMENTS                                                   │  │
  │  │ ├── Rolling update (ECS native)                              │  │
  │  │ ├── Blue/Green (via CodeDeploy)                              │  │
  │  │ └── Deployment circuit breaker (auto-rollback on failure)    │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ AUTO SCALING                                                  │  │
  │  │ ├── Target tracking (keep metric at target value)            │  │
  │  │ ├── Step scaling (graduated response to alarm thresholds)    │  │
  │  │ └── Scheduled scaling (time-based desired count changes)     │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────────────┘


  SCHEDULING STRATEGIES:
  ────────────────────────────────────────────────────

  REPLICA (default):
  ├── Maintains a specified number of task instances
  ├── Spreads tasks across AZs for high availability
  ├── Supports auto scaling to adjust desired count
  └── Use for: web servers, APIs, workers — most workloads

  DAEMON:
  ├── Runs exactly ONE task on each container instance
  ├── New instances automatically get a task
  ├── Cannot use Fargate (no "instances" to target)
  ├── Does not support desired count or auto scaling
  └── Use for: monitoring agents, log collectors, host-level utilities
```

### Deployment Strategies Deep Dive

```
ROLLING UPDATE — THE DEFAULT DEPLOYMENT
=============================================================================

  How it works:
  ├── ECS launches new tasks (new task definition revision)
  ├── Waits for new tasks to pass health checks
  ├── Registers new tasks with load balancer
  ├── Deregisters old tasks from load balancer
  ├── Stops old tasks after deregistration delay
  └── Repeats until all tasks are on the new revision

  Controlled by two parameters:
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  minimumHealthyPercent: Floor of healthy tasks during       │
  │  deployment (as % of desired count)                         │
  │                                                             │
  │  maximumPercent: Ceiling of total tasks during              │
  │  deployment (as % of desired count)                         │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  EXAMPLE: desired = 4 tasks

  Config: min 100%, max 200% (ZERO-DOWNTIME — start new before stopping old)
  ────────────────────────────────────────────────────
  Step 1: [OLD][OLD][OLD][OLD]                    4 tasks (100%)
  Step 2: [OLD][OLD][OLD][OLD][NEW][NEW]          6 tasks (150%)
  Step 3: [OLD][OLD][NEW][NEW][NEW][NEW]          6 tasks (150%)
  Step 4: [NEW][NEW][NEW][NEW]                    4 tasks (100%)
  Requires extra capacity — new tasks start before old ones stop.

  Config: min 50%, max 100% (SAVE CAPACITY — stop old before starting new)
  ────────────────────────────────────────────────────
  Step 1: [OLD][OLD][OLD][OLD]                    4 tasks (100%)
  Step 2: [OLD][OLD]                              2 tasks (50%)
  Step 3: [OLD][OLD][NEW][NEW]                    4 tasks (100%)
  Step 4: [NEW][NEW]                              2 tasks (50%)
  Step 5: [NEW][NEW][NEW][NEW]                    4 tasks (100%)
  Temporarily reduces capacity — old tasks stop before new ones start.

  Config: min 100%, max 150% (BALANCED — moderate overhead)
  ────────────────────────────────────────────────────
  Step 1: [OLD][OLD][OLD][OLD]                    4 tasks (100%)
  Step 2: [OLD][OLD][OLD][OLD][NEW][NEW]          6 tasks (150%) ← max ceiling hit
  Step 3: [OLD][OLD][NEW][NEW][NEW][NEW]          6 tasks (150%) ← stop 2 old, start 2 new
  Step 4: [NEW][NEW][NEW][NEW]                    4 tasks (100%)
  Can only launch 2 new tasks at a time (ceiling is 6), so deployment takes
  more batches than max 200% (which could launch all 4 new tasks at once).
  Always above desired count, but slower than 200% with less overhead.


  DEPLOYMENT CIRCUIT BREAKER:
  ────────────────────────────────────────────────────
  ├── Monitors new tasks during deployment
  ├── If new tasks repeatedly fail to reach RUNNING + healthy:
  │   ├── Automatically STOPS the deployment
  │   └── Rolls back to the last stable task definition revision
  ├── Failure threshold: based on minimum number of launch attempts
  ├── Enabled with: deployment_circuit_breaker { enable = true, rollback = true }
  └── Prevents a bad image or misconfiguration from killing all tasks


  BLUE/GREEN (via CodeDeploy):
  ────────────────────────────────────────────────────
  ├── Creates a SECOND set of tasks (green) alongside the existing (blue)
  ├── Traffic shifting options:
  │   ├── AllAtOnce: shift 100% immediately
  │   ├── Linear: shift X% every Y minutes (e.g., 10% every 5 min)
  │   └── Canary: shift X% first, wait, then shift remaining
  ├── Automatic rollback on CloudWatch alarm triggers
  ├── Requires: ALB with two target groups + CodeDeploy config
  └── More complex but safer for critical services
```

### Terraform: ECS Service with Rolling Update

```hcl
# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "production"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# ECS Service
resource "aws_ecs_service" "web_app" {
  name            = "web-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.web_app.arn
  desired_count   = 4

  # Use capacity provider strategy instead of launch_type
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 2  # Always run at least 2 on Fargate
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 2  # 2:1 ratio — twice as many on Spot
  }

  # Networking (required for awsvpc mode)
  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  # Load balancer integration
  load_balancer {
    target_group_arn = aws_lb_target_group.web_app.arn
    container_name   = "web"
    container_port   = 8080
  }

  # Deployment configuration
  deployment_maximum_percent         = 200  # Allow up to 2x tasks during deploy
  deployment_minimum_healthy_percent = 100  # Never drop below desired count

  deployment_circuit_breaker {
    enable   = true
    rollback = true  # Auto-rollback on deployment failure
  }

  # Note: Fargate handles AZ spreading automatically.
  # ordered_placement_strategy is only for EC2 launch type.

  # Ignore changes to desired_count when auto scaling manages it
  lifecycle {
    ignore_changes = [desired_count]
  }

  depends_on = [aws_lb_listener.web_app]
}

# Service Auto Scaling
resource "aws_appautoscaling_target" "web_app" {
  max_capacity       = 20
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.web_app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Target tracking: keep ALB request count per target at 1000
resource "aws_appautoscaling_policy" "web_app_requests" {
  name               = "requests-per-target"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.web_app.resource_id
  scalable_dimension = aws_appautoscaling_target.web_app.scalable_dimension
  service_namespace  = aws_appautoscaling_target.web_app.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label = "${aws_lb.web_app.arn_suffix}/${aws_lb_target_group.web_app.arn_suffix}"
    }
    target_value       = 1000.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Target tracking: keep average CPU at 60%
resource "aws_appautoscaling_policy" "web_app_cpu" {
  name               = "cpu-utilization"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.web_app.resource_id
  scalable_dimension = aws_appautoscaling_target.web_app.scalable_dimension
  service_namespace  = aws_appautoscaling_target.web_app.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 60.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## Part 3: Capacity Providers - The Resource Procurement Department

### The Analogy

The studio's **resource procurement department** decides where performances actually happen. They have three options:

1. **Rent fully managed stages (Fargate)** -- Show up, perform, leave. The rental company handles the building, electricity, cleaning, and maintenance. You pay per minute of stage time. Simple but premium-priced.

2. **Rent discounted stages (Fargate Spot)** -- Same quality stages but at up to 70% off. The catch: the rental company can reclaim the stage with a 2-minute warning if someone else needs it. Great for workloads that can handle interruptions.

3. **Use owned stages (EC2 capacity provider)** -- You own the buildings and equipment. You control maintenance schedules, can install custom equipment, and pay for the building whether or not a show is running. The capacity provider connects your fleet of owned stages (Auto Scaling Group) to ECS, and **managed scaling** automatically builds or demolishes stages based on demand.

The **capacity provider strategy** is the procurement policy that determines the mix. You can say "always use 2 owned stages as a base, then fill additional demand with 70% rented and 30% discount." This gives you cost optimization with reliability guarantees.

### The Technical Reality

Capacity providers are the link between ECS services and the compute infrastructure that runs tasks. They abstract away the underlying infrastructure decisions, letting services declare what they need rather than where to run.

```
CAPACITY PROVIDERS — THE THREE OPTIONS
=============================================================================

  FARGATE CAPACITY PROVIDER:
  ────────────────────────────────────────────────────
  ├── Serverless — no instances to manage
  ├── Pay per vCPU-second and GB-second of task runtime
  ├── AWS handles provisioning, patching, scaling of hosts
  ├── Each task gets its own ENI (awsvpc required)
  ├── Task sizes: 0.25 to 16 vCPU, 0.5 to 120 GB memory
  └── Best for: most workloads, especially variable/unpredictable load

  FARGATE SPOT CAPACITY PROVIDER:
  ────────────────────────────────────────────────────
  ├── Same as Fargate but uses AWS spare capacity
  ├── Up to 70% discount vs standard Fargate
  ├── Tasks can be interrupted with 2-minute warning (SIGTERM)
  ├── ECS attempts to replace interrupted tasks automatically
  └── Best for: batch processing, dev/test, fault-tolerant workloads

  EC2 CAPACITY PROVIDER (backed by ASG):
  ────────────────────────────────────────────────────
  ├── You manage the EC2 instances via an Auto Scaling Group
  ├── Capacity provider links the ASG to ECS
  ├── MANAGED SCALING: ECS automatically adjusts ASG desired count
  │   based on pending tasks (no manual scaling policies needed)
  ├── Target capacity %: how full to pack instances (100% = fully packed)
  ├── MANAGED TERMINATION PROTECTION: prevents scale-in from killing
  │   instances that still have running tasks
  └── Best for: GPU workloads, custom AMIs, licensing, sustained load


  CAPACITY PROVIDER STRATEGY — THE PROCUREMENT POLICY:
  ────────────────────────────────────────────────────

  A strategy is a list of providers with weights and an optional base:

  ┌──────────────────┬────────┬──────┐
  │ Capacity Provider │ Weight │ Base │
  ├──────────────────┼────────┼──────┤
  │ FARGATE          │ 1      │ 2    │  ← First 2 tasks ALWAYS on Fargate
  │ FARGATE_SPOT     │ 2      │ 0    │  ← Remaining split 2:1 in favor of Spot
  └──────────────────┴────────┴──────┘

  With desired count = 8:
  ├── Base: 2 tasks on FARGATE (guaranteed, non-negotiable)
  ├── Remaining 6 tasks distributed by weight ratio 1:2
  │   ├── FARGATE:      6 × (1/3) = 2 tasks
  │   └── FARGATE_SPOT: 6 × (2/3) = 4 tasks
  ├── Total: 4 on FARGATE, 4 on FARGATE_SPOT
  └── Base guarantees a minimum on reliable compute even if Spot is unavailable


  EC2 MANAGED SCALING — HOW IT WORKS:
  ────────────────────────────────────────────────────

  1. ECS Service says: "I need 10 tasks, each requiring 1 vCPU + 2 GB"
  2. Capacity provider calculates: "I need 10 vCPU and 20 GB of capacity"
  3. Current ASG has 3 × m5.xlarge (4 vCPU, 16 GB each) = 12 vCPU, 48 GB
  4. But target capacity is set to 80%: only 80% of each instance is usable
     (20% headroom for burst)
  5. Effective capacity: 12 × 0.8 = 9.6 vCPU — not enough for 10 tasks
  6. ECS tells ASG: "Scale to 4 instances" → ASG launches 1 more instance
  7. Now: 4 × 4 × 0.8 = 12.8 effective vCPU — enough for 10 tasks

  Target capacity % meaning:
  ├── 100% → pack instances fully (maximum efficiency, no headroom)
  ├── 80%  → keep 20% headroom (good default for variable workloads)
  ├── 50%  → keep 50% headroom (expensive but ready for big spikes)
  └── Lower values = more spare capacity = faster task placement
      but higher cost from underutilized instances
```

### Terraform: Capacity Providers and Strategy

```hcl
# ECS Cluster with capacity provider strategy
resource "aws_ecs_cluster" "main" {
  name = "production"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# Default capacity provider strategy for the cluster
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = [
    "FARGATE",
    "FARGATE_SPOT",
    aws_ecs_capacity_provider.ec2.name,
  ]

  # Default strategy: services that do not specify their own
  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 1
  }

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 2
  }
}

# EC2 Capacity Provider backed by an ASG
resource "aws_ecs_capacity_provider" "ec2" {
  name = "ec2-provider"

  auto_scaling_group_provider {
    auto_scaling_group_arn = aws_autoscaling_group.ecs.arn

    # ECS manages the ASG scaling — no need for separate scaling policies
    managed_scaling {
      status                    = "ENABLED"
      target_capacity           = 80         # Keep 20% headroom
      minimum_scaling_step_size = 1
      maximum_scaling_step_size = 10
      instance_warmup_period    = 300        # Seconds before counting new instance
    }

    # Prevent scale-in from killing instances with running tasks
    managed_termination_protection = "ENABLED"
  }
}

# ASG for EC2 capacity provider
resource "aws_autoscaling_group" "ecs" {
  name                = "ecs-ec2-asg"
  desired_capacity    = 2
  min_size            = 0
  max_size            = 20
  vpc_zone_identifier = var.private_subnet_ids

  # Must enable instance protection for managed termination protection
  protect_from_scale_in = true

  launch_template {
    id      = aws_launch_template.ecs.id
    version = "$Latest"
  }

  tag {
    key                 = "AmazonECSManaged"
    value               = true
    propagate_at_launch = true
  }
}

# Launch template for ECS EC2 instances
resource "aws_launch_template" "ecs" {
  name_prefix   = "ecs-"
  image_id      = data.aws_ami.ecs_optimized.id  # ECS-optimized AMI
  instance_type = "m6i.xlarge"

  iam_instance_profile {
    name = aws_iam_instance_profile.ecs.name
  }

  # Register instance with the ECS cluster
  user_data = base64encode(<<-EOF
    #!/bin/bash
    echo "ECS_CLUSTER=${aws_ecs_cluster.main.name}" >> /etc/ecs/ecs.config
    echo "ECS_ENABLE_TASK_ENI=true" >> /etc/ecs/ecs.config
    echo "ECS_ENABLE_CONTAINER_METADATA=true" >> /etc/ecs/ecs.config
  EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "ecs-instance"
    }
  }
}

# ECS-optimized AMI lookup
data "aws_ami" "ecs_optimized" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-ecs-hvm-*-x86_64-ebs"]
  }
}

# Instance profile for ECS EC2 instances
resource "aws_iam_instance_profile" "ecs" {
  name = "ecs-instance-profile"
  role = aws_iam_role.ecs_instance.name
}

resource "aws_iam_role" "ecs_instance" {
  name = "ecs-instance-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_instance" {
  role       = aws_iam_role.ecs_instance.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}
```

---

## Part 4: Fargate vs EC2 - Choosing Your Compute Model

```
FARGATE vs EC2 DECISION FRAMEWORK
=============================================================================

                        FARGATE                      EC2
                        ───────                      ───
  Operations            Fully managed                You manage instances,
                        (no patching, no AMIs)       AMIs, patching, agents

  Networking            awsvpc only                  awsvpc, bridge, host, none
                        (1 ENI per task)

  Pricing               Per vCPU-second +            Per instance (pay whether
                        per GB-second                or not tasks fill it)

  Cost at scale         Higher per unit              Lower per unit if well-
                        (premium for simplicity)     utilized (bin-packing)

  Startup time          ~30-60 seconds               Depends on instance launch
                                                     + ECS agent registration

  GPU support           No                           Yes

  Custom AMI            No                           Yes

  Task size limits      0.25-16 vCPU                 Limited by instance type
                        0.5-120 GB memory            (up to 448 vCPU, 24 TB)

  Persistent storage    EFS and EBS                  EBS, EFS, instance store

  Daemon tasks          Not supported                Supported

  Spot capacity         Fargate Spot (2 min warning)  EC2 Spot via ASG
                                                     (2 min warning)


  DECISION GUIDE:
  ────────────────────────────────────────────────────

  Choose FARGATE when:
  ├── You want minimal operational overhead
  ├── Workloads are variable or unpredictable
  ├── Team lacks dedicated infrastructure engineers
  ├── Running many small, independent services
  └── Cost predictability matters more than cost optimization

  Choose EC2 when:
  ├── You need GPUs (ML inference, video processing)
  ├── You need custom AMIs or kernel configurations
  ├── High, sustained utilization makes per-instance pricing cheaper
  ├── You need daemon services (monitoring agents on every host)
  ├── Tasks need more than 16 vCPU or 120 GB memory
  └── You need EBS volumes or instance store

  Choose BOTH (mixed strategy) when:
  ├── Base load on EC2 (cost-efficient for guaranteed minimum)
  ├── Burst/overflow on Fargate (elastic, no capacity planning)
  └── This is a common production pattern via capacity provider strategies
```

---

## Putting It All Together - A Production ECS Architecture

```
COMPLETE ECS ARCHITECTURE
=============================================================================

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                              VPC                                        │
  │                                                                         │
  │  ┌──────────────────────────────────────────────────────────────────┐   │
  │  │                     PUBLIC SUBNETS                                │   │
  │  │                                                                  │   │
  │  │  ┌──────────────────────────────────────────────────────────┐    │   │
  │  │  │               APPLICATION LOAD BALANCER                   │    │   │
  │  │  │   ┌──────────┐    ┌──────────────┐    ┌──────────────┐   │    │   │
  │  │  │   │ Listener │───▶│ Target Group │    │ Target Group │   │    │   │
  │  │  │   │ :443     │    │ (web-app)    │    │ (api)        │   │    │   │
  │  │  │   └──────────┘    └──────┬───────┘    └──────┬───────┘   │    │   │
  │  │  └──────────────────────────┼───────────────────┼───────────┘    │   │
  │  └─────────────────────────────┼───────────────────┼────────────────┘   │
  │                                │                   │                    │
  │  ┌─────────────────────────────┼───────────────────┼────────────────┐   │
  │  │                     PRIVATE SUBNETS             │                │   │
  │  │                                                                  │   │
  │  │  ┌──────────────────────────────────────────────────────────┐    │   │
  │  │  │                    ECS CLUSTER                            │    │   │
  │  │  │                                                          │    │   │
  │  │  │  Service: web-app          Service: api                  │    │   │
  │  │  │  Strategy: Fargate/Spot    Strategy: EC2 provider        │    │   │
  │  │  │  ┌──────┐ ┌──────┐       ┌──────┐ ┌──────┐ ┌──────┐   │    │   │
  │  │  │  │Task  │ │Task  │       │Task  │ │Task  │ │Task  │   │    │   │
  │  │  │  │(FG)  │ │(Spot)│       │(EC2) │ │(EC2) │ │(EC2) │   │    │   │
  │  │  │  └──────┘ └──────┘       └──────┘ └──────┘ └──────┘   │    │   │
  │  │  │                                                          │    │   │
  │  │  │  Service: worker (no LB)                                 │    │   │
  │  │  │  Strategy: Fargate Spot                                  │    │   │
  │  │  │  ┌──────┐ ┌──────┐ ┌──────┐                            │    │   │
  │  │  │  │Task  │ │Task  │ │Task  │  ← Polls SQS queue         │    │   │
  │  │  │  └──────┘ └──────┘ └──────┘                            │    │   │
  │  │  └──────────────────────────────────────────────────────────┘    │   │
  │  │                                                                  │   │
  │  └──────────────────────────────────────────────────────────────────┘   │
  │                                                                         │
  │  ┌──────────────────────────────────────────────────────────────────┐   │
  │  │  SUPPORTING SERVICES                                             │   │
  │  │  ├── ECR: Container image registry                              │   │
  │  │  ├── CloudWatch Logs: Centralized logging (/ecs/*)              │   │
  │  │  ├── Secrets Manager: DB passwords, API keys                    │   │
  │  │  ├── Service Discovery (Cloud Map): service-to-service comms    │   │
  │  │  └── Container Insights: Cluster and service metrics            │   │
  │  └──────────────────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────────────────┘


  REQUEST FLOW:
  ────────────────────────────────────────────────────

  User → Route 53 → ALB (:443) → Target Group → ECS Task (container:8080)
                                                       │
                                                       ├── Reads secrets from Secrets Manager
                                                       ├── Queries DynamoDB (via task role)
                                                       ├── Sends messages to SQS
                                                       └── Logs to CloudWatch via awslogs driver

  Worker tasks poll SQS independently (no load balancer needed).
```

---

## Key Takeaways

### Task Definitions
1. **Task definitions are immutable** -- Every change creates a new revision; you never modify an existing revision, which gives you a reliable audit trail and easy rollback by pointing your service at a previous revision
2. **Separate task role from execution role** -- The task role is what your application code uses to call AWS APIs (S3, DynamoDB); the execution role is what ECS uses to pull images and write logs; mixing them up is the most common IAM debugging headache in ECS
3. **Default to awsvpc networking** -- It gives each task its own ENI, private IP, and security group, making tasks first-class VPC citizens; it is the only option on Fargate and the recommended choice on EC2
4. **Use secrets, not environment variables, for sensitive data** -- The `secrets` field pulls values from Secrets Manager or SSM Parameter Store at task launch; environment variables are visible in the task definition and in `docker inspect`

### Services
5. **Set minimumHealthyPercent to 100 and maximumPercent to 200 for zero-downtime deployments** -- This ensures ECS starts new tasks before stopping old ones, so capacity never drops below desired count during a rolling update
6. **Always enable the deployment circuit breaker** -- Without it, a bad image or misconfigured container will keep trying to launch and fail indefinitely, potentially exhausting your capacity; with rollback enabled, ECS automatically reverts to the last working revision
7. **Use lifecycle ignore_changes on desired_count when auto scaling manages it** -- Otherwise Terraform will fight with auto scaling on every apply, resetting the count to whatever is in your HCL

### Capacity Providers
8. **Use base to guarantee minimum reliable capacity** -- In a capacity provider strategy, the `base` parameter places a fixed number of tasks on a specific provider before weights apply; use it to ensure critical tasks always run on Fargate (not Spot)
9. **EC2 managed scaling eliminates the need for separate ASG scaling policies** -- When you enable managed scaling on an EC2 capacity provider, ECS calculates the required capacity based on pending tasks and adjusts the ASG directly; do not add conflicting ASG scaling policies
10. **Target capacity below 100% provides headroom for burst** -- Setting target_capacity to 80% means instances are kept at 80% utilization, leaving 20% spare capacity for tasks that need to be placed immediately without waiting for a new instance to launch

### Fargate vs EC2
11. **Start with Fargate and move to EC2 only when you have a specific reason** -- GPU requirements, custom AMIs, sustained high utilization, or tasks needing more than 16 vCPU or 120 GB memory are the main reasons to bring in EC2
12. **Mixed strategies are the production sweet spot** -- Base load on EC2 (cost-efficient) with burst on Fargate (elastic, no capacity planning) gives you the best of both worlds via capacity provider strategies
13. **Fargate Spot is excellent for fault-tolerant workloads** -- Queue processors, batch jobs, and dev/test environments can save up to 70% with Fargate Spot; just ensure your application handles SIGTERM gracefully for the 2-minute interruption warning

### Operational Essentials
14. **Enable Container Insights from day one** -- It provides cluster-level and service-level metrics (CPU, memory, network, storage) without any agent installation; the cost is minimal compared to the debugging time it saves
15. **Three systems interact and must be configured together** -- Service auto scaling adjusts desired task count, capacity providers ensure enough compute exists to place those tasks, and deployment configuration controls how task replacements happen during updates; configuring service auto scaling without capacity provider managed scaling means the service wants more tasks but there is no infrastructure to place them on

---

*Written on February 16, 2026*
