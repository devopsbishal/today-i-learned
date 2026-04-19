# ArgoCD Advanced -- Multi-App & Multi-Cluster Patterns -- 2-Hour Study Plan

**Total Time:** ~125 minutes
**Created:** 2026-04-17
**Difficulty:** Advanced (builds on `argocd-gitops-fundamentals.md` -- assumes you already know Application CRD, sync policies, health checks, sync waves/hooks, and the repo server / application controller architecture)

## Overview

This plan moves past "one Application CRD deploys one Helm chart" into the patterns required to run ArgoCD as the control plane for an organization: generating hundreds of Applications from templates (App-of-Apps + ApplicationSets), managing dozens of clusters from a single ArgoCD instance (hub-and-spoke + controller sharding), enforcing multi-tenant boundaries with AppProjects and SSO-backed RBAC, keeping secrets out of Git (Sealed Secrets vs External Secrets Operator vs SOPS), and wiring deployment events to Slack/PagerDuty via argocd-notifications. After completing it, you will understand the full decision tree for structuring ArgoCD repos at scale, know which generator to reach for in any given scenario, and have a defensible opinion on secret management strategy.

## Resources

### 1. Cluster Bootstrapping -- App of Apps Pattern (Official Docs) -- 12 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/>
- **Type:** Official Docs
- **Summary:** The canonical reference for the App-of-Apps pattern. A single root Application points at a Git directory full of child Application manifests -- when you apply the root, ArgoCD discovers and syncs every child. Read this for the mechanics: `directory.recurse: true`, the sync-wave trick to order bootstrap components (cert-manager before ingress before apps), the `resources-finalizer.argocd.argoproj.io` finalizer that cascades deletion to children, and the critical detail that the root Application should be excluded from its own directory scan to avoid self-reconciliation loops. This is the foundation every advanced pattern builds on.

### 2. Codefresh -- Structuring Argo CD Repositories with ApplicationSets -- 15 min

- **URL:** <https://codefresh.io/blog/how-to-structure-your-argo-cd-repositories-using-application-sets/>
- **Type:** Blog (practitioner deep dive)
- **Summary:** The best single article on the "App-of-Apps vs ApplicationSets vs nested" decision. Walks through four repo layouts -- monolithic, per-environment, per-team, and hybrid -- with concrete YAML. Explains why nested App-of-Apps (a root App that points at child Apps that themselves point at more Apps) is the right answer when you have organizational separation (platform team owns cluster addons, product teams own their apps), and when to collapse that into a single ApplicationSet. Read this before the ApplicationSet generator docs so you have a concrete use case in mind for each generator.

### 3. ApplicationSet Generators -- Official Docs (Git, Cluster, List, Matrix, Merge, SCM, PR) -- 25 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/> (index page; follow links to each generator)
- **Type:** Official Docs
- **Summary:** The seven generators are the entire reason ApplicationSet exists. Budget ~3-4 min per generator. **Git generator** (files + directories) for "one app per directory in a monorepo" or "one app per YAML config file with parameters". **Cluster generator** for "deploy this app to every registered cluster matching a label selector". **List generator** for a hand-maintained enumeration. **Matrix generator** combines two generators as a cross-product (e.g., every app in every cluster) -- the workhorse for multi-tenant platforms. **Merge generator** joins generators on a key (override cluster-specific values from one generator with overrides from another). **SCM Provider generator** auto-discovers every repo in a GitHub org matching filters -- true "deploy anything that exists". **Pull Request generator** spins up a preview Application for every open PR and tears it down on merge. Focus on recognizing which shape fits which problem.

### 4. ApplicationSet Templates, Go Template & Template Overrides -- 15 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/GoTemplate/> and <https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Template/>
- **Type:** Official Docs
- **Summary:** Two pages that cover the template rendering layer that sits on top of any generator. `goTemplate: true` switches from the legacy fasttemplate engine (simple `{{ .param }}` substitution) to full Go templates with sprig functions, conditionals, and range loops -- always set this for new ApplicationSets. `goTemplateOptions: ["missingkey=error"]` turns silent typos into loud failures. Template overrides let a generator's element provide a partial `template` block that patches the base template for just that element (e.g., one service in the matrix points at a different repo). `preservedFields` tells the ApplicationSet controller to NOT overwrite specific fields on generated Applications if a human edits them in the cluster -- essential when operators need to set `spec.source.targetRevision` for a hotfix without the controller clobbering the change on next reconcile.

### 5. InfraCloud -- Sharding Clusters Across Application Controller Replicas -- 12 min

- **URL:** <https://www.infracloud.io/blogs/sharding-clusters-across-argo-cd-application-controller-replicas/>
- **Type:** Blog (practitioner deep dive)
- **Summary:** The application-controller is the memory hog in any ArgoCD install -- it holds the live state cache for every managed cluster. Once you pass ~20-30 clusters from a single hub, the controller OOMs. This article explains the sharding model: set `controller.replicas`, choose a sharding algorithm (`legacy` hashes cluster UIDs, `round-robin` distributes evenly, `consistent-hashing` minimizes reshuffles when clusters come and go), and optionally pin specific clusters to specific shards via the `shard` field in the cluster Secret. Covers the operational gotchas (shard assignments persist in Secret annotations, a dying shard's clusters stay "stuck" until rescheduled, and the round-robin + dynamic-scaling combo needs Argo CD 2.8+).

### 6. AWS EKS Blueprints -- Multi-Cluster Hub & Spoke with ArgoCD -- 15 min

- **URL:** <https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/gitops/gitops-multi-cluster-hub-spoke-argocd/>
- **Type:** Official AWS Reference Architecture
- **Summary:** Concrete, AWS-specific walkthrough of the hub-and-spoke model: one management EKS cluster runs ArgoCD, additional workload EKS clusters are registered as "spokes", and the hub deploys platform addons (cert-manager, external-dns, Karpenter, aws-load-balancer-controller) plus workloads to every spoke via ApplicationSets. Shows the cluster-registration mechanics (IAM role for cross-account access, the cluster Secret with the kubeconfig, label-based cluster selectors driving ApplicationSet Cluster generators), and contrasts hub-and-spoke with "standalone ArgoCD per cluster" (simpler blast radius, worse operational overhead, no cross-cluster visibility). Relevant to your EKS path specifically -- this is the pattern most AWS shops end up running.

### 7. ArgoCD RBAC + SSO -- Official Docs -- 15 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/> and <https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/>
- **Type:** Official Docs
- **Summary:** RBAC in ArgoCD is a Casbin policy (`policy.csv`) with two built-in roles (`admin`, `readonly`) plus custom roles scoped globally or per AppProject. The grammar is `p, <subject>, <resource>, <action>, <object>, <effect>` -- e.g., `p, role:dev, applications, sync, dev-project/*, allow`. The **AppProject is the multi-tenant boundary**: it restricts which Git repos, destination clusters/namespaces, and resource kinds its Applications can use, and it hosts project-scoped roles and JWT tokens for automation (CI pipelines sync Apps without a human login). For SSO, read the user-management index page to understand when to use Dex (bundled, supports GitHub/SAML/LDAP connectors) vs direct OIDC (simpler, better for Okta/Auth0/Keycloak/Google). The magic of SSO + RBAC is group-to-role mapping: `g, my-org:platform-team, role:admin` maps an OIDC group claim directly to an ArgoCD role.

### 8. Codefresh -- ArgoCD Secrets: Two Technical Approaches -- 12 min

- **URL:** <https://codefresh.io/learn/argo-cd/argocd-secrets/>
- **Type:** Blog (practitioner comparison)
- **Summary:** Clear head-to-head of the three secret strategies you must choose between. **Sealed Secrets** (kubeseal CLI encrypts with the controller's public key; only the in-cluster controller can decrypt; secrets live in Git as `SealedSecret` CRs): simple, zero external dependencies, but tied to the cluster (the controller's private key must be backed up, rotation is painful, and re-sealing for N clusters is tedious). **External Secrets Operator** (ESO pulls from AWS Secrets Manager / Vault / GCP Secret Manager at runtime via `ExternalSecret` CRs referencing a `SecretStore` or `ClusterSecretStore`): secrets never touch Git, auto-rotates, the winner for AWS shops with existing Secrets Manager investment. **SOPS** (Mozilla SOPS encrypts YAML files with KMS/age keys; decryption happens in ArgoCD via helm-secrets plugin or KSOPS): not Kubernetes-specific, multi-recipient key support for team access, but requires a custom ArgoCD plugin image. Read this with the question "which would I pick for an AWS EKS platform with 10 clusters?" in mind (spoiler: ESO).

### 9. argocd-notifications -- Triggers, Templates, Services (Official Docs) -- 10 min

- **URL:** <https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/> (follow links to Triggers, Templates, and Services/Slack)
- **Type:** Official Docs
- **Summary:** Four moving parts: **services** (Slack, webhook, email, PagerDuty, Teams, GitHub -- configured in `argocd-notifications-cm`), **triggers** (named conditions with a CEL-style expression against the Application object, e.g., `app.status.sync.status == 'OutOfSync'`), **templates** (the message body referencing the Application context), and **subscriptions** (annotations on the Application or AppProject like `notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts` that bind a trigger to a service to a destination). Ships with useful default triggers (`on-deployed`, `on-sync-failed`, `on-health-degraded`, `on-sync-running`). Skim the Slack services page for the OAuth token setup and the `groupingKey` trick that threads related messages. This is the last-mile "how do humans find out something broke" piece every production install needs.

## Study Tips

- **Build the generator decision tree first.** Before diving into YAML, map the seven ApplicationSet generators onto concrete shapes: "one app per Git dir" = Git directories, "one app per cluster" = Cluster, "one app per PR" = Pull Request, "one app per repo in my GitHub org" = SCM, "cross-product of apps and clusters" = Matrix, "layer config from multiple sources" = Merge. Most ApplicationSet confusion comes from picking the wrong generator for the problem -- picking correctly makes the templating trivial.
- **Treat AppProjects as the tenant boundary, not a UI organizer.** The tempting mistake is to create one AppProject per environment (dev/staging/prod) and dump everything into it. The correct model is one AppProject per team or per trust boundary: each Project restricts source repos, destination clusters/namespaces, and allowed resource kinds, and hosts its own RBAC roles. This is what makes multi-tenant ArgoCD safe -- a compromised or misconfigured Application in one Project literally cannot deploy to another Project's namespaces even if the YAML says to.
- **For secrets, start from the cluster count and secret source of truth.** One cluster + no existing secret manager = Sealed Secrets is fine. Many clusters + AWS/Vault already in play = External Secrets Operator, full stop. Cross-team repos where different people need to decrypt different values locally = SOPS with per-file key policies. Do not pick based on popularity -- pick based on where your secrets already live and how many clusters need them.
- **Sketch the hub-and-spoke data flow.** Hub cluster runs: argocd-server, repo-server, application-controller (sharded), Redis, Dex, applicationset-controller, notifications-controller. Spoke clusters run: nothing ArgoCD-specific -- just a ServiceAccount + token (or IRSA for EKS) that the hub uses to authenticate. The hub's application-controller holds a kubeconfig per spoke in a cluster Secret and maintains a watch stream per spoke. Memory and API-server QPS on the hub scales with (clusters x apps x resources per app) -- this is why sharding exists.

## Next Steps

- **ApplicationSet Progressive Syncs:** Rolling out a change across N clusters in controlled waves (one region first, then canary, then the rest) using `strategy.type: RollingSync` on the ApplicationSet. Still beta as of ArgoCD 2.x -- worth tracking.
- **Argo Rollouts integration:** Combine ArgoCD (what to deploy) with Argo Rollouts (how to deploy -- canary, blue/green, analysis templates from Prometheus). This is the full "GitOps progressive delivery" stack that Intuit and others run in production.
- **Disaster recovery drills:** Document and practice "the hub cluster is gone" scenarios. What does it take to rebuild the ArgoCD hub from Git alone? (Answer: install ArgoCD, apply the root App-of-Apps manifest, wait -- everything else reconciles back. This is THE proof that your GitOps setup actually works.)
- **Observability:** Wire argocd-metrics into Prometheus, import the official Grafana dashboards, and alert on `argocd_app_info{sync_status!="Synced"}` holding true for > 10 minutes. The notifications controller tells humans about deploy events; metrics tell SRE about systemic drift.
