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
| 2026-05-12 | [AWS CloudWatch Comprehensive](./2026/may/aws/2026-05-12-aws-cloudwatch-comprehensive.md) | Factory-instrumentation-room analogy; Phase 2.8 opener. Metrics vocabulary (namespace+name+dimension as unique billed unit, the cardinality trap), EMF as the modern emission path with Lambda Powertools (Python+TypeScript) and FireEye 65% case study, log classes (Standard vs IA with the irreversibility gotcha + vended-log IA wins), Logs Insights pipeline with `filter`-first cost rule and 5 production queries. Three alarm types (metric / composite for noise suppression / anomaly detection 3x cost), Synthetics + RUM pairing, Insights suite map distinguishing Container vs Lambda vs Application Insights vs Application Signals APM vs ServiceLens. CloudWatch-vs-Prometheus-vs-OTel decision framework with InfraCloud benchmark + the awsemf-OTel-fan-out pattern, 8-cause cost triage, 20 gotchas, 6 frameworks, 10 Commandments. |
| 2026-05-11 | [GitHub Actions Security, Secrets & Production Hardening](./2026/may/cicd/2026-05-11-github-actions-security-production-hardening.md) | High-security bank vault analogy; Phase 2.7 capstone tying May 5/6/7/10 into eight independent defense layers. SHA-pinning regime with the `tj-actions` CVE-2025-30066 case study as the spine, Dependabot + cooldown + `pinact --check` + CODEOWNERS-on-workflows. Three-tier secret precedence (env > repo > org), `secrets: inherit` cross-org silent-empty trap, GitHub App tokens vs PATs vs `GITHUB_TOKEN`. Two-signature pattern (Cosign + `attest-build-provenance@v2`) with `gh attestation verify` and Kyverno admission gate, Rulesets vs classic branch protection with the "skipped is not success" status-aggregator fix, ARC ephemeral runners + JIT tokens (NEVER on public repos), Jan 2026 pricing with arm64 37% savings, audit-log + CloudTrail OIDC `sub` correlation, 20 gotchas, 6 frameworks, 10 Commandments. |
| 2026-05-10 | [GitHub Actions GitOps CD -- ArgoCD Integration](./2026/may/cicd/2026-05-10-github-actions-gitops-cd-integration.md) | Shipping-department / customs / dispatcher analogy; CI/CD handoff principle (CI commits to Git, never `kubectl apply`) with five-axis defense, three image-update strategies (CI-PR vs Image Updater `git` write-back vs Kargo) with decision matrix, two-repo vs single-repo + folder-per-env (Kapelonis), GitHub Environments deep dive (reviewers/wait/branch+tag policies/custom rules/prevent-self-review/plan caveats) with OIDC `sub:environment:X` for prod, three promotion patterns (auto/PR/tag) + `workflow_dispatch`, ArgoCD Notifications closing the loop (commit-status/Deployments-API/PR-comment), 16 gotchas (GITHUB_TOKEN-PR-no-trigger, recursion-fuse, `argocd`-write-back-drift, `git-branch` mismatch, helm-image-name silent no-op, env-secret-without-`environment:`, prod-promotion-sub-mismatch), 5 decision frameworks, 10 Commandments. |
| 2026-05-07 | [GitHub Actions Container CI: Build, Scan, Sign & Push](./2026/may/cicd/2026-05-07-github-actions-container-ci-pipeline.md) | Bakery-pickup-window-with-tamper-evident-seal analogy; the four-action stack (setup-qemu/buildx/login/build-push) with current 2026 majors, BuildKit cache backends + scope= + immutability rule, AWS OIDC federation as the security spine with the four `sub` formats + 2026 immutable-subject-claims rollout + the `pull_request` foot-gun. Trivy two-tier gating with upload-sarif@v4, BuildKit SBOM/provenance attestations + builder-driver-first debug, Cosign keyless via Fulcio/Rekor with sign-by-digest discipline. The pwn-request attack chain connecting `pull_request_target` + fork-checkout + workflow-level `id-token: write` + `sub`-only trust policy as four independent layers, 15 gotchas, 5 frameworks, 10 Commandments. |
| 2026-05-06 | [GitHub Actions Custom Actions & Reusable Patterns](./2026/may/cicd/2026-05-06-github-actions-custom-actions-reusable-patterns.md) | Recipe-library-vs-catering-service analogy; three custom action types (JS with ncc bundling + pre/post hooks, Docker Linux-only, composite with shell-mandatory + secrets-as-inputs foot-gun) and reusable workflows with two-layer outputs + secrets:inherit cross-org silent-empty. actions/cache key composition + immutability rule + restore-keys hierarchy vs setup-* built-in cache, upload-artifact v4 immutability + fan-out matrix break, dorny/paths-filter with fetch-depth:0 + status-aggregator job for required checks, 15 gotchas, 5 frameworks, 10 Commandments. |
| 2026-05-05 | [GitHub Actions Core Review & Advanced Workflows](./2026/may/cicd/2026-05-05-github-actions-advanced-workflows.md) | Automated-factory-floor analogy; workflow/job/step/runner model with sterile-workbench job isolation, the 12-context availability matrix as the silent-empty bug source, four event triggers including the `pull_request_target` foot-gun. Concurrency cancel-in-progress=true vs false for PR-CI vs deploys, reusable workflows vs composite actions decision framework, matrix with include/exclude/fail-fast/dynamic-fromJson, 18 gotchas, 10 Commandments. |
| 2026-05-01 | [SLOs, Error Budgets & Production Observability](./2026/may/observability/2026-05-01-slos-error-budgets-production-observability.md) | Hospital-ER-triage analogy; four SLI categories (request/pipeline/storage/journey), good-events/valid-events with denominator clamping, cost-curve and dependency-math for target setting. Error budget POLICY as the org artifact (signed, escalation ladder, dispute resolution); MWMBR going deeper than Apr 23 with per-tier burn rates and severity matrix. Sloth vs Pyrra vs Grafana SLO vs OpenSLO vs Nobl9 decision matrix, runbook-as-code, maturity model, 16 gotchas, 10 Commandments. |
| 2026-04-30 | [Unified Observability -- Correlating Signals](./2026/april/observability/2026-04-30-unified-observability-correlation.md) | Three-witnesses analogy (camera/GPS/diary); the four pivots -- exemplars (metrics->traces), tracesToLogsV2 (traces->logs), derivedFields (logs->traces), LogQL ruler (logs->metrics). Resource attributes as durable join key, semconv 1.27 `deployment.environment.name` rename, Grafana Correlations vs per-DS configs, Alloy as unified emitter, 18 gotchas, 10 Commandments. |
| 2026-04-29 | [Log Aggregation with Loki](./2026/april/loki/2026-04-29-log-aggregation-loki.md) | Library-card-catalog vs Google-for-books analogy; labels-define-streams cardinality model with 100k cap, structured metadata (3.0+) + bloom filters (3.3+) for trace_id queries. Three deployment modes, full LogQL pipeline order, Alloy DaemonSet replacing EOL'd Promtail. Trace-to-logs pivot, 21 gotchas, 10 Commandments. |
| 2026-04-28 | [Distributed Tracing Deep Dive](./2026/april/opentelemetry/2026-04-28-distributed-tracing-deep-dive.md) | Span anatomy + W3C Trace Context propagation; head vs tail sampling with `loadbalancing` exporter and full policy stacks. TraceQL structural operators + TraceQL Metrics, three-signal correlation via spanmetrics + exemplars + service graph. Two-tier production topology, 18 gotchas, 10 Commandments. |

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
