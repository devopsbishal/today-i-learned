# ArgoCD Fundamentals -- GitOps Model -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-16
**Difficulty:** Intermediate (assumes strong Kubernetes/EKS and Helm knowledge)

## Overview

This plan builds a working mental model of ArgoCD as a GitOps continuous delivery engine for Kubernetes: the four OpenGitOps principles that define what GitOps actually means, ArgoCD's pull-based architecture (API Server, Repo Server, Application Controller, Redis, Dex), the Application CRD as the central abstraction, sync policies (auto-sync, self-heal, prune), health assessment (built-in and custom Lua checks), sync waves and hooks for deployment ordering, and how ArgoCD consumes Helm charts using `helm template` rather than `helm install`. After completing it, you will understand why ArgoCD replaces imperative `helm install` in production, how to structure an Application CRD that deploys a Helm chart from Git with environment-specific values, and how sync waves, hooks, and health checks coordinate multi-resource deployments.

## Resources

### 1. OpenGitOps Principles + The New Stack Explainer -- 12 min

- **URL:** <https://opengitops.dev/> and <https://thenewstack.io/4-core-principles-of-gitops/>
- **Type:** CNCF Standard + Article
- **Summary:** Start here to internalize the four CNCF-standardized GitOps principles -- Declarative, Versioned and Immutable, Pulled Automatically, Continuously Reconciled -- before touching any ArgoCD-specific material. The OpenGitOps page gives you the canonical one-liner definitions (2 min), then The New Stack article provides the "why" behind each principle with quotes from practitioners at Red Hat, Weaveworks, and the New York Times (10 min). This framing prevents the common mistake of treating ArgoCD as "just another CD tool" rather than an implementation of a principled operational model.

### 2. ArgoCD Official Docs -- Core Concepts + Understand the Basics -- 15 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/core_concepts/> and <https://argo-cd.readthedocs.io/en/stable/understand_the_basics/>
- **Type:** Official Docs
- **Summary:** The canonical definitions of every term you will encounter: Application, Project, sync status (Synced/OutOfSync), health status (Healthy/Progressing/Degraded/Suspended/Missing/Unknown), target state vs live state, and refresh vs sync. These two pages are short but precise -- they establish the vocabulary that every subsequent resource assumes you know. Pay particular attention to how "sync" differs from what you know as `helm upgrade`: ArgoCD compares rendered manifests against live cluster state, not Helm release revisions.

### 3. ArgoCD Architecture -- DevOpsCube Deep Dive -- 20 min

- **URL:** <https://devopscube.com/argo-cd-architecture/>
- **Type:** Blog (practitioner deep dive)
- **Summary:** The best single-article explanation of ArgoCD's seven components and how they interact. Covers the API Server (gRPC/REST gateway for UI and CLI), Repo Server (clones Git repos, runs `helm template` or `kustomize build` to generate manifests), Application Controller (the reconciliation loop that compares desired vs live state and triggers syncs), Redis (caching layer for manifest and cluster state), Dex (OIDC/SAML SSO), ApplicationSet Controller (multi-app generation), and Notification Controller. Includes architecture diagrams showing data flow. Read this to understand what happens when you create an Application CRD -- which component does what, and why the repo server never talks to the cluster directly.

### 4. ArgoCD Official Docs -- Helm Integration -- 15 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/user-guide/helm/>
- **Type:** Official Docs
- **Summary:** Critical for you given your Helm background. This page explains the fundamental shift: ArgoCD runs `helm template` to inflate charts into raw manifests, then applies them with `kubectl apply` -- it does NOT run `helm install` or `helm upgrade`. This means no Helm release Secrets in the cluster, no `helm rollback`, and no Tiller-era state. Covers the values precedence chain (parameters > valuesObject > values > valueFiles > chart defaults), `helm.valuesObject` for inline values in the Application CRD, `helm.fileParameters` for binary data, `helm.passCredentials`, and `helm.releaseName`. If you only read one official docs page today, make it this one -- it bridges your existing Helm mental model to the ArgoCD world.

### 5. Automated Sync Policy + Sync Options (Official Docs) -- 15 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/> and <https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/>
- **Type:** Official Docs
- **Summary:** Two pages that define the three auto-sync toggles -- `automated` (enable the reconciliation loop), `prune` (delete resources removed from Git), and `selfHeal` (revert manual cluster changes back to Git state) -- plus the safety mechanism `allowEmpty` (protect against accidentally wiping all resources). The sync options page covers per-resource annotations like `argocd.argoproj.io/sync-options: Replace=true` (force replace instead of apply), `ServerSideApply=true`, `CreateNamespace=true`, and `PruneLast=true`. Understanding which options are set at the Application level vs per-resource annotation is essential for production use.

### 6. Codefresh -- ArgoCD Sync Policies Practical Guide -- 15 min

- **URL:** <https://codefresh.io/learn/argo-cd/argocd-sync-policies-a-practical-guide/>
- **Type:** Article (practitioner guide)
- **Summary:** Reinforces the official docs with concrete YAML examples and CLI commands showing how to configure auto-sync, self-heal, and prune in both the Application manifest and via `argocd app set`. Covers the subtle behaviors the docs mention but do not illustrate well: what happens when self-heal fires (5-second timeout before correction), the difference between `Prune=true` and `PruneLast=true`, selective sync for large applications, and server-side apply for CRD-heavy deployments. Read this after the official docs to see the concepts applied in realistic manifests.

### 7. Sync Phases, Waves, and Hooks (Official Docs + Codefresh Examples) -- 18 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/> and <https://codefresh.io/learn/argo-cd/understanding-argo-cd-sync-waves-with-examples/>
- **Type:** Official Docs + Article
- **Summary:** The deployment ordering system. ArgoCD syncs resources in three phases (PreSync, Sync, PostSync) with a SyncFail phase for error handling. Within each phase, resources are ordered by wave number (`argocd.argoproj.io/sync-wave: "5"`), lowest first. A wave only proceeds after all resources in the previous wave are healthy. Hooks are resources annotated with `argocd.argoproj.io/hook: PreSync` (typically Jobs) that run at phase boundaries -- the ArgoCD equivalent of Helm hooks, but with a critical difference: ArgoCD hooks participate in health assessment and wave ordering, while Helm hooks are fire-and-forget. The Codefresh article adds a practical DB migration example (PreSync Job) and Slack notification (PostSync hook). Note the pruning reversal: during deletion, ArgoCD processes waves in reverse order (highest first).

### 8. Resource Health -- Built-in and Custom Lua Checks (Official Docs) -- 10 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/operator-manual/health/>
- **Type:** Official Docs
- **Summary:** ArgoCD ships with built-in health assessments for standard Kubernetes resources (Deployment checks `.status.updatedReplicas`, Service checks endpoint existence, Ingress checks load balancer assignment, etc.). For CRDs or custom logic, you write Lua scripts in the `argocd-cm` ConfigMap under `resource.customizations.health.<group>_<kind>`. The script receives the resource as `obj` and returns `{status, message}` where status is one of Healthy/Progressing/Degraded/Suspended/Missing/Unknown. This is the mechanism that makes sync waves actually work -- ArgoCD waits for health before advancing waves. Skim the built-in checks table to know what is covered by default, then read the custom Lua example to understand extension points.

## Study Tips

- **Map ArgoCD concepts to Helm concepts you already know.** ArgoCD Application CRD replaces `helm install/upgrade` commands. Sync replaces `helm upgrade --install`. Self-heal replaces the manual `helm rollback` workflow. Sync waves replace Helm hooks with `hook-weight`. The key difference: ArgoCD is declarative and pull-based (controller watches Git), while Helm is imperative and push-based (human runs CLI). Building this mapping explicitly prevents the "two separate mental models" problem.
- **Pay attention to what ArgoCD removes from Helm.** Because ArgoCD uses `helm template` and not `helm install`, there are no Helm release Secrets in the cluster, no `helm history`, no `helm rollback`, and `lookup` always returns empty (no cluster access during rendering). This is the biggest "gotcha" for someone coming from deep Helm knowledge -- features you relied on disappear, replaced by Git history and ArgoCD's own diff/rollback mechanisms.
- **Sketch the sync lifecycle on paper.** Draw the flow: Git commit -> Repo Server detects change -> `helm template` generates manifests -> Application Controller compares to live state -> PreSync hooks (wave -1, 0) -> Sync resources (wave 0, 1, 2...) -> PostSync hooks -> health check gates between waves. This single diagram ties together architecture, sync policies, waves, hooks, and health checks into one coherent picture.

## Next Steps

- **ArgoCD Advanced Patterns:** ApplicationSets (generating Applications from templates for multi-cluster/multi-env), App of Apps pattern, Projects for RBAC scoping, multi-source Applications (Helm chart + values from separate repos), and Notifications for Slack/webhook integration.
- **ArgoCD + Helmfile/Kustomize:** Explore how Kustomize post-rendering works with ArgoCD (`kustomize` as a source type), and the trade-offs vs pure Helm sources.
- **Hands-on lab:** Install ArgoCD on a local kind/minikube cluster, create an Application CRD pointing at a Helm chart in a Git repo with `valuesObject`, enable auto-sync with self-heal, make a Git commit, and watch the reconciliation loop in the ArgoCD UI. Then add sync waves to order a ConfigMap (wave 0) before a Deployment (wave 1) and observe the wave gating behavior.
- **Production hardening:** ArgoCD RBAC with Projects, SSO via Dex/OIDC, repo credential management, resource tracking methods (annotation vs label), and performance tuning for large clusters (sharding, Redis scaling).
