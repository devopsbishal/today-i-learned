# ECS Fundamentals: Task Definitions, Services, and Capacity Providers -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-02-16
**Difficulty:** Intermediate

## Overview

This study plan covers the core building blocks of Amazon Elastic Container Service: how task definitions serve as blueprints for your containers, how services maintain desired task counts with deployment strategies and load balancer integration, and how capacity providers manage and scale the underlying compute infrastructure across Fargate and EC2. After completing this plan, you will understand the ECS resource hierarchy from clusters down to containers, know how to configure task definitions with appropriate networking modes and resource allocations, understand the differences between rolling update and blue/green deployment strategies for services, and be able to design capacity provider strategies that balance cost, performance, and availability.

## Resources

### 1. What is Amazon Elastic Container Service? (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
- **Type:** Official Docs
- **Summary:** The essential starting point that establishes the complete ECS mental model: clusters as logical groupings of tasks and services, task definitions as immutable blueprints describing container configurations, tasks as running instances of those blueprints, and services as the control layer that maintains desired task counts. Covers the three compute options (Fargate serverless, EC2 self-managed, and ECS Managed Instances) and how ECS integrates with other AWS services like ECR, ELB, CloudWatch, and IAM. Read this first to build the conceptual foundation that everything else builds upon.

### 2. AWS ECS: A Beginner's Guide (AWS Fundamentals Blog) -- 20 min
- **URL:** https://awsfundamentals.com/blog/aws-ecs-beginner-guide
- **Type:** Blog
- **Summary:** A well-structured overview that reinforces the official docs with clearer explanations and practical context. Covers the ECS component hierarchy (clusters, container instances, task definitions, tasks, services), provides a detailed comparison of EC2 vs Fargate launch types with trade-offs for each, explains the role of the ECS Agent on EC2 instances, and includes real-world use cases from companies like Netflix and Samsung. Particularly valuable for understanding pricing differences between Fargate (pay per vCPU/memory per second) and EC2 (pay per instance regardless of utilization), which directly informs capacity provider strategy decisions later in this plan.

### 3. Amazon ECS Task Definitions (Official Docs) -- 25 min
- **URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html
- **Type:** Official Docs
- **Summary:** The definitive reference for task definitions, the JSON blueprints that specify everything about how your containers run. Covers the key parameters: container image, CPU and memory allocations (hard limits vs soft limits), port mappings, environment variables, logging configuration, IAM task roles (what the container can access) vs task execution roles (what ECS needs to pull images and write logs), volumes, and health checks. Pay close attention to the distinction between task-level resource allocation (required for Fargate) and container-level allocation (optional overrides within the task budget), and understand that task definitions are immutable -- updates create new revisions rather than modifying existing ones. Follow the links into the Fargate-specific and EC2-specific parameter pages for the networking mode details.

### 4. Amazon ECS Task Networking Options for EC2 (Official Docs) -- 15 min
- **URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html
- **Type:** Official Docs
- **Summary:** Covers the four ECS networking modes, which is one of the most consequential task definition decisions. The awsvpc mode gives each task its own ENI and private IP from your VPC subnet, making it the recommended default and the only option on Fargate. Bridge mode uses Docker's virtual network with port mapping, allowing multiple containers to share the host IP but requiring dynamic port allocation to avoid conflicts. Host mode maps container ports directly to the EC2 instance network interface for maximum performance but limits you to one task per port per host. None mode provides no external connectivity. Understanding these modes is critical because they determine how tasks communicate with each other, with load balancers, and with external services, and they constrain which security group and subnet configurations are possible.

### 5. Amazon ECS Services (Official Docs) -- 20 min
- **URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html
- **Type:** Official Docs
- **Summary:** The core reference for ECS services, the layer that turns individual task runs into reliable, scalable applications. Covers the service scheduler that maintains your desired task count and replaces failed tasks automatically, load balancer integration with ALB and NLB for traffic distribution, service auto scaling that adjusts desired count based on CloudWatch metrics (target tracking, step scaling, or scheduled), deployment configuration with minimum and maximum healthy percent parameters that control rollout speed vs availability, the deployment circuit breaker that automatically rolls back failed deployments, and the difference between replica scheduling (spread tasks across AZs) and daemon scheduling (one task per instance). Focus on how minimum/maximum healthy percent interact during deployments -- setting minimumHealthyPercent to 100 and maximumPercent to 200 means ECS starts new tasks before stopping old ones, ensuring zero-downtime rolling updates.

### 6. ECS Deployment Options: From Rolling Updates to Blue-Green and Canary (cloudonaut) -- 15 min
- **URL:** https://cloudonaut.io/ecs-deployment-options/
- **Type:** Blog
- **Summary:** A focused comparison of all three ECS deployment strategies that cuts through the complexity of the official docs. Covers native ECS rolling updates (simple but no rollback beyond circuit breaker), CodeDeploy-integrated blue/green deployments (supports canary and linear traffic shifting with automated rollback on CloudWatch alarms), and external/CloudFormation-based deployments using TaskSets for full custom control. The key insight is that each approach trades simplicity for control: rolling updates are the easiest to configure and sufficient for most services, blue/green through CodeDeploy adds traffic validation before full cutover at the cost of more infrastructure, and external deployments offer maximum flexibility but require significant custom orchestration. Read this to build a decision framework for which deployment strategy fits which use case.

### 7. Amazon ECS Capacity Providers for EC2 Workloads (Official Docs) -- 10 min
- **URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/asg-capacity-providers.html
- **Type:** Official Docs
- **Summary:** The essential reference for EC2-based capacity providers, which connect ECS services to Auto Scaling Groups and enable ECS-managed scaling of the underlying EC2 infrastructure. Covers how capacity providers work (ECS calculates the capacity needed based on pending tasks and adjusts the ASG target), the target capacity percentage parameter that controls how aggressively instances are utilized (100% means fully packed, lower values leave headroom for burst), managed scaling that automates ASG adjustments without manual scaling policies, and managed termination protection that prevents scale-in from killing instances running tasks. The critical concept is that capacity provider strategies let you define weighted distributions across multiple providers -- for example, 70% on On-Demand and 30% on Spot, or primary on Fargate with overflow to EC2 -- giving you fine-grained control over cost and availability trade-offs at the service level rather than the cluster level.

## Study Tips

- **Trace the resource hierarchy from top to bottom as you read:** Cluster contains services, services reference task definitions and capacity provider strategies, task definitions describe containers with their resource needs and networking mode, and capacity providers manage the compute that runs those containers. Every ECS decision flows through this hierarchy, so internalizing it prevents confusion when you encounter concepts like "a service uses a capacity provider strategy to place tasks defined by a task definition onto infrastructure managed by a capacity provider." Draw this hierarchy on paper and annotate it as you work through each resource.

- **Focus on the Fargate vs EC2 decision points rather than memorizing parameters:** The most important architectural question in ECS is which compute model to use, and this decision cascades into every configuration detail. Fargate forces awsvpc networking, requires task-level CPU/memory specification, and simplifies operations but costs more per unit of compute. EC2 gives you all four networking modes, allows bin-packing multiple tasks per instance, supports GPUs and custom AMIs, but requires you to manage capacity providers and instance lifecycle. Understanding these trade-offs is more valuable than memorizing individual task definition parameters.

- **Pay attention to how deployment configuration, capacity providers, and service auto scaling interact:** These three systems work together but are configured separately, which causes confusion. Service auto scaling adjusts the desired task count (horizontal scaling of your application). Capacity providers ensure enough compute infrastructure exists to place those tasks (infrastructure scaling). Deployment configuration controls how task replacements happen during updates (rollout behavior). A common production issue is configuring service auto scaling without appropriate capacity provider managed scaling -- the service wants more tasks but there is no infrastructure to place them on.

## Next Steps

After completing this study plan, consider exploring:

1. **ECS with Fargate hands-on** -- Deploy a containerized web application on Fargate with an ALB, configure service auto scaling based on request count, and observe how Fargate provisions and reclaims compute automatically. This makes the task definition, service, and networking concepts concrete.

2. **ECS task IAM roles and secrets management** -- Understand the difference between task roles and task execution roles in depth, learn how to pass secrets from Secrets Manager and SSM Parameter Store to containers without baking them into images, and configure fine-grained IAM policies for least-privilege container access.

3. **ECS observability with CloudWatch Container Insights** -- Learn how to configure centralized logging with awslogs and FireLens, set up Container Insights for cluster and service-level metrics, create CloudWatch alarms for task health and resource utilization, and use X-Ray for distributed tracing across ECS services.

4. **ECS with Terraform** -- Define ECS clusters, task definitions, services, and capacity providers as infrastructure as code using the AWS Terraform provider. This reinforces the resource relationships by forcing you to explicitly declare every dependency and configuration parameter.
