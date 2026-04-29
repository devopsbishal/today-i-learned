# Today I Learned

A personal learning journal where I document what I learn each day. Writing things down helps me understand concepts better and gives me a reference to look back on when I need a refresher.

---

## 📚 What's This?

This is my collection of bite-sized learnings - concepts, insights, and "aha moments" I pick up along the way. Each entry is written in my own words, often with real-world analogies that help me remember.

## 📁 Structure

```
today-i-learned/
├── 2025/
│   ├── november/
│   │   └── 2025-11-30-aws-vpc.md
│   └── december/
├── 2026/
│   └── ...
└── Readme.md
```

Organized by: `year/month/YYYY-MM-DD-topic.md`

## 📝 Recent Learnings

> Showing the latest 10 entries. Older entries live in the [Archive](#-archive) below and in the year folders.

| Date | Topic | Description |
|------|-------|-------------|
| 2026-04-29 | [Log Aggregation with Loki](./2026/april/loki/2026-04-29-log-aggregation-loki.md) | Library-card-catalog vs Google-for-books analogy; labels-define-streams cardinality model with 100k cap, structured metadata (3.0+) + bloom filters (3.3+) for trace_id queries. Three deployment modes, full LogQL pipeline order, Alloy DaemonSet replacing EOL'd Promtail. Trace-to-logs pivot, 21 gotchas, 10 Commandments. |
| 2026-04-28 | [Distributed Tracing Deep Dive](./2026/april/opentelemetry/2026-04-28-distributed-tracing-deep-dive.md) | Span anatomy + W3C Trace Context propagation; head vs tail sampling with `loadbalancing` exporter and full policy stacks. TraceQL structural operators + TraceQL Metrics, three-signal correlation via spanmetrics + exemplars + service graph. Two-tier production topology, 18 gotchas, 10 Commandments. |
| 2026-04-27 | [OpenTelemetry Fundamentals](./2026/april/opentelemetry/2026-04-27-opentelemetry-fundamentals.md) | Three signals + API/SDK/Collector split, OTLP wire protocol, processor pipeline canonical order. K8s DaemonSet+Gateway pattern, OTel Operator with Instrumentation CRD, Grafana Alloy convergence. 15 production gotchas, five decision frameworks, 10 Commandments. |
| 2026-04-24 | [Grafana Fundamentals](./2026/april/grafana/2026-04-24-grafana-fundamentals.md) | Mission-control room analogy; data sources, dashboards, variables with `$__rate_interval` trap, transformations. File vs Terraform provisioning, Grafana-managed vs data-source-managed alerting decision framework, dynamic dashboards + Git Sync. 15 gotchas, four decision frameworks, 10 Commandments. |
| 2026-04-23 | [Alertmanager & Alerting Strategy](./2026/april/prometheus/2026-04-23-alertmanager-alerting-strategy.md) | Routing tree with `matchers`/`group_by`/timers, inhibition vs silence, receivers (Slack/PD/OpsGenie). SLO-based multi-window multi-burn-rate alerting (14.4/6/3/1) + PrometheusRule/AlertmanagerConfig CRDs, HA topology with gossip. 12 gotchas, four decision frameworks, 10 Commandments. |
| 2026-04-21 | [PromQL Deep Dive](./2026/april/prometheus/2026-04-21-promql-deep-dive.md) | Instant vs range vector mental model, vector matching as SQL JOIN, the rate family with extrapolation math + rate-then-sum rule. Classic vs native histogram queries, recording rules, subqueries vs `@` modifiers, RED + USE methods with full queries. 15 gotchas, decision frameworks, cheat sheet. |
| 2026-04-20 | [Prometheus Fundamentals](./2026/april/prometheus/2026-04-20-prometheus-fundamentals.md) | Pull-based scrape architecture, four metric types, cardinality as #1 risk, `kubernetes_sd_configs` with relabeling actions. Local TSDB (WAL + 2h blocks), remote_write to Thanos/Mimir/AMP, hierarchical federation. Production gotchas (counter resets, Pushgateway misuse, retention math). |
| 2026-04-19 | [ArgoCD + Helm in Production Patterns](./2026/april/kubernetes/2026-04-19-argocd-helm-production-patterns.md) | Three Helm-chart-source patterns + values precedence chain, repo structure (app vs config), per-env hierarchies. ArgoCD Image Updater + dev->stg->prod promotion, Argo Rollouts canary/blue-green with AnalysisTemplate. Six production gotchas (hook reinterpretation, release-ownership conflict, secrets discipline). |
| 2026-04-17 | [ArgoCD Advanced -- Multi-App & Multi-Cluster Patterns](./2026/april/kubernetes/2026-04-17-argocd-advanced-patterns.md) | App of Apps + nested patterns, all seven ApplicationSet generators with goTemplate, multi-cluster registration, Application Controller sharding strategies. AppProject Casbin RBAC, three secrets strategies (Sealed/ESO/SOPS), Dex vs direct OIDC. Notifications, advanced hooks, production Terraform. |
| 2026-04-16 | [ArgoCD Fundamentals -- GitOps Model](./2026/april/kubernetes/2026-04-16-argocd-gitops-fundamentals.md) | Four CNCF GitOps principles, push vs pull CD, ArgoCD's seven components, Application CRD with sync/health status. Auto-sync policies + per-resource sync options, the critical Helm bridge (helm template not install -> no release Secrets, lookup empty), sync waves/hooks. ArgoCD vs Helm hook semantics, custom Lua health checks. |

## 📂 Archive

| Year | Entries | Topics |
|------|---------|--------|
| [2026](./2026/) | 51 | Docker (Networking, Storage, Multi-Stage Builds, Runtimes), Terraform (State, Modules, Workspaces, Functions, Data Sources, Providers, Variables, Outputs, Security, Testing, Cloud/Enterprise, Project Structure), AWS (EC2, EC2 Purchasing Options, EC2 Networking, Auto Scaling, ECS, ECS Advanced, VPC Advanced, VPC Peering vs Transit Gateway, Transit Gateway Deep Dive, Direct Connect, Site-to-Site VPN, Organizations & SCPs, Control Tower & Landing Zones, IAM Advanced Patterns, Security Services -- Threat Detection, KMS & Encryption, Network & Application Protection, Route53 Routing Policies & Health Checks, Route53 Advanced -- Hybrid DNS, CloudFront Deep Dive, ALB vs NLB vs GLB, Global Accelerator & Edge Services, S3 Advanced Deep Dive, Block & File Storage -- EBS, EFS, FSx, RDS & Aurora Deep Dive, DynamoDB Deep Dive, ElastiCache & Caching Strategies, Disaster Recovery Strategies, Multi-Region Architecture Patterns, AWS Backup & Centralized Protection, Lambda Deep Dive, API Gateway Deep Dive, Step Functions Workflow Orchestration, EventBridge & Event-Driven Architecture, SQS SNS & Messaging Patterns), Kubernetes (Helm Fundamentals, Helm Advanced Patterns, ArgoCD Fundamentals) |
| [2025](./2025/) | 10 | AWS (VPC, EKS, VPC CNI, Networking, IAM, Storage, Autoscaling, Security, Observability, Upgrades) |

---

## 🧠 Why I Do This

1. **Learn by teaching** - Writing forces me to truly understand
2. **Future reference** - Quick refresher when concepts get fuzzy
3. **Track progress** - See how far I've come over time

---

### 🔗 Connect

[![GitHub](https://img.shields.io/badge/GitHub-@devopsbishal-181717?style=flat&logo=github)](https://github.com/devopsbishal)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Bishal%20Rayamajhi-0A66C2?style=flat&logo=linkedin)](https://www.linkedin.com/in/bishal-rayamajhi-02523a243/)
