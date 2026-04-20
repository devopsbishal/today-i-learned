# ArgoCD + Helm in Production Patterns -- 2-Hour Study Plan

**Total Time:** ~125 minutes
**Created:** 2026-04-19
**Difficulty:** Advanced (capstone -- assumes Helm Fundamentals (Apr 13), Helm Advanced Patterns (Apr 15), ArgoCD Fundamentals (Apr 16), and ArgoCD Advanced Patterns (Apr 17/18). You should already understand chart authoring, subcharts/hooks, the Application CRD, sync waves, ApplicationSets, App-of-Apps, and AppProjects.)

## Overview

This is the capstone day on GitOps with the Argo stack. The previous four sessions taught Helm and ArgoCD as separate tools; today's plan stitches them into the workflow real platform teams run: Helm charts as ArgoCD Application sources (from Git or OCI), per-environment value file hierarchies that avoid duplication, multi-source Applications that decouple chart from values, automated image-tag promotion via Argo CD Image Updater (write-back to Git), progressive delivery with Argo Rollouts (canary, blue/green, analysis templates), repo structure decisions (mono-repo vs separate config repo), PR-based dev/staging/prod promotion, and the production gotchas that bite everyone the first time -- Helm hooks running on every sync, `helm template` losing release ownership semantics, and tracking-label drift conflicts. After completing this plan you can defend a full production GitOps design end-to-end and recognize the failure modes before they happen in your own cluster.

## Resources

### P0 -- Must-Read (Core ~85 minutes)

### 1. Helm in Argo CD -- valueFiles, parameters, valuesObject, multiple sources (Official Docs) ⏱️ 15 min
- **URL:** <https://argo-cd.readthedocs.io/en/stable/user-guide/helm/>
- **Type:** Official Docs
- **Why this matters:** The single source of truth for how an Application's `spec.source.helm` block actually behaves -- the precedence chain (`parameters` > `valuesObject` > `values` > `valueFiles` > chart's own `values.yaml`), the `--ignore-missing-value-files` flag that makes optional per-env override files possible, glob support in `valueFiles`, and the critical fact that ArgoCD runs `helm template` (not `helm install`) so there is no Helm release object in `kube-system` and no client-side `--set` semantics. Internalize this precedence order -- almost every "why isn't my override winning" debugging session traces back to it.

### 2. Multiple Sources for an Application -- chart from one repo, values from another (Official Docs) ⏱️ 10 min
- **URL:** <https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/>
- **Type:** Official Docs
- **Why this matters:** The 2.6+ pattern that lets you point at a Helm chart in an OCI registry (or upstream Git) AND pull `valueFiles` from your own private config repo in the same Application. This is the production answer to "I want to deploy `kube-prometheus-stack` from the upstream chart but keep my customer values out of a fork." Pay attention to the `$values` reference syntax (`$values/envs/prod/values.yaml`) and the `ref:` field that names the values-source so `valueFiles` can reference it. This pattern is what makes the entire "app config repo separate from chart repo" architecture work.

### 3. Codefresh -- Using Helm Hierarchies in Multi-Source ArgoCD Applications for Promoting to GitOps Environments ⏱️ 15 min
- **URL:** <https://codefresh.io/blog/helm-values-argocd/>
- **Type:** Blog (Kostis Kapelonis -- the canonical practitioner write-up)
- **Why this matters:** The single best end-to-end example of building a value hierarchy (`common.yaml` -> `region.yaml` -> `env.yaml` -> `cluster.yaml`, last one wins) and exposing it through a multi-source Application driven by an ApplicationSet Git generator. Includes a public GitHub repo with working YAML you can clone. Every "promote dev to staging to prod" pipeline ends up looking like this -- read it before you design your own repo structure.

### 4. Red Hat Developer -- 3 Patterns for Deploying Helm Charts with Argo CD ⏱️ 10 min
- **URL:** <https://developers.redhat.com/articles/2023/05/25/3-patterns-deploying-helm-charts-argocd>
- **Why this matters:** Crisp comparison of the three practical chart-source patterns: (1) chart-and-values-in-the-same-Git-repo (simplest, fine for your own apps), (2) external chart from a Helm/OCI registry with values inline in the Application (good for upstream charts where you change few values), (3) external chart with values in a separate Git repo via multi-source (the production pattern from Resource 3). Read this to anchor your decision tree -- you will pick a different pattern for your own app vs `ingress-nginx` vs `kube-prometheus-stack`.

### 5. Argo CD Image Updater -- Update Methods (git write-back) (Official Docs) ⏱️ 12 min
- **URL:** <https://argocd-image-updater.readthedocs.io/en/stable/basics/update-methods/>
- **Type:** Official Docs
- **Why this matters:** Image Updater is what closes the CI/CD loop for application code: CI builds + pushes a new image tag, Image Updater watches the registry, and on a new matching tag it commits an updated `.argocd-source-<appName>.yaml` (or a Helm values override) back to Git -- ArgoCD then syncs the new tag. Read the two methods (`argocd` annotation in the live Application vs `git` write-back to a tracked branch) and understand why `git` is the production answer (drift-proof, audit-trailed, survives Image Updater being down). Pay attention to `write-back-target` (where in the repo the override lands) and the four update strategies (`semver`, `latest`, `digest`, `name`).

### 6. CNCF -- Mastering Argo CD Image Updater with Helm: A Complete Configuration Guide ⏱️ 12 min
- **URL:** <https://www.cncf.io/blog/2024/11/05/mastering-argo-cd-image-updater-with-helm-a-complete-configuration-guide/>
- **Type:** Blog (CNCF -- vendor-neutral, recent)
- **Why this matters:** The Image Updater docs explain the mechanism; this article shows the end-to-end Helm wiring -- the `image-list` annotation that names a logical image alias, the `helm.image-tag` and `helm.image-name` annotations that map that alias to the values keys your chart actually reads, ECR/GHCR auth setup, and a worked `argocd-image-updater.argoproj.io/<alias>.update-strategy: semver` example with a `1.x.x` constraint. Concrete and current (Nov 2024). Combined with Resource 5 you have everything you need to wire this up to an EKS cluster pulling from ECR.

### 7. Medium / Houssein Dbouk -- ArgoCD & Helm Hooks: The Gotchas You Must Know ⏱️ 10 min
- **URL:** <https://medium.com/@h_dbouk/argocd-helm-hooks-the-gotchas-you-must-know-a56512fe96d1>
- **Type:** Blog (practitioner war story, freely accessible)
- **Why this matters:** The single most important production gotcha to internalize: ArgoCD runs `helm template` not `helm install`, so Helm hooks (`pre-install`, `post-upgrade`, etc.) don't fire the way you'd expect -- ArgoCD reinterprets them as ArgoCD sync hooks, which means a `post-upgrade` Job runs on every sync (not just on actual chart upgrades). Anything non-idempotent (DB migration, API call, license activation) will fire on drift reconciliation. The fix is `argocd.argoproj.io/hook: PostSync` semantics, sync-wave ordering, and `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded`. Read this before you ship any chart with hooks.

### P1 -- Strongly Recommended (~40 minutes)

### 8. Akuity -- How to Automate Blue-Green & Canary Deployments with Argo Rollouts ⏱️ 15 min
- **URL:** <https://akuity.io/blog/automating-blue-green-and-canary-deployments-with-argo-rollouts>
- **Type:** Blog (Akuity -- founded by the original ArgoCD creators)
- **Why this matters:** The clearest single intro to Argo Rollouts as the missing "how to deploy" layer alongside ArgoCD's "what to deploy". Walks the `Rollout` CRD as a drop-in replacement for `Deployment`, the canary strategy with weight steps + pauses, the blue/green strategy with `activeService`/`previewService` + `autoPromotionEnabled`, and AnalysisTemplates that query Prometheus / Datadog / a webhook to make automated promote-or-rollback decisions. This is what Intuit, Adobe, and Codefresh actually run in production -- ArgoCD syncs the Rollout manifest, Rollouts then drives the progressive traffic shift with metric-gated automation.

### 9. Argo Rollouts -- Analysis & Progressive Delivery (Official Docs) ⏱️ 15 min
- **URL:** <https://argo-rollouts.readthedocs.io/en/stable/features/analysis/>
- **Type:** Official Docs
- **Why this matters:** Goes deep on the four moving parts: `AnalysisTemplate` (the reusable metric-query definition), `AnalysisRun` (the instance kicked off by a Rollout step), the four provider types (`prometheus`, `datadog`, `web` for arbitrary HTTP, `job` for any custom logic), and the `successCondition`/`failureCondition` CEL expressions that decide promote vs abort. Pay attention to background analysis vs inline analysis (background runs alongside steps, inline blocks the next step), and `prePromotionAnalysis`/`postPromotionAnalysis` for blue-green. Internalize that an analysis failure causes Rollouts to abort and route 100% of traffic back to the stable ReplicaSet -- this is your safety net.

### 10. Akuity -- GitOps Best Practices Whitepaper (skim sections on repo structure & promotion) ⏱️ 10 min
- **URL:** <https://akuity.io/blog/gitops-best-practices-whitepaper>
- **Type:** Whitepaper (skim, don't read end-to-end)
- **Why this matters:** Skim two sections -- "Repository Structure" (mono-repo vs polyrepo trade-offs, the strong recommendation to separate app source code repo from GitOps config repo) and "Promotion Between Environments" (why environment branches are an anti-pattern, why directory-per-environment is the modern default, and how to gate prod sync on PR review + status checks). The rest is too high-level for where you are in your study path, but those two sections crystallize the architectural decisions you'll otherwise re-learn the hard way.

## Study Tips

- **Build the chart-source decision tree first.** For every Application you ever write, the first question is "where does the chart live and where do the values live?" Three answers: (a) chart + values in your own monorepo, single source -- fine for your own apps; (b) chart from upstream Helm/OCI registry + values inline in the Application -- fine for low-touch upstream charts; (c) chart from upstream + values from your private Git repo via multi-source -- the only sane answer for heavy customization of `kube-prometheus-stack`, `ingress-nginx`, etc. Get this decision right and the value hierarchy and ApplicationSet wiring fall out naturally.
- **Treat `helm template` as the mental model, not `helm install`.** The single most common bug class is assuming ArgoCD runs `helm install` / `helm upgrade`. It does not. There is no `Secret` of type `helm.sh/release.v1` in your namespace, `helm list` shows nothing, Helm hooks behave as ArgoCD sync hooks, and `Chart.yaml` `appVersion` does not magically reach your pod labels. Every gotcha in Resource 7 is downstream of this one fact.
- **Image Updater write-back must target a branch you actually track.** If your Application's `targetRevision` is a tag, a SHA, or `HEAD` of `main`, Image Updater write-back will either fail or write to a branch nobody syncs. Set `argocd-image-updater.argoproj.io/git-branch: main` (or a dedicated `image-updates` branch you merge via PR for prod), and have your Application track that branch. The "it commits but nothing deploys" failure mode is always a branch mismatch.
- **Sync waves are how you compose Helm hooks with ArgoCD ordering.** When a chart has both `helm.sh/hook: pre-install` resources AND ArgoCD-native resources, ArgoCD sequences them by sync-wave (`argocd.argoproj.io/sync-wave`, ascending integers, default 0). Use negative waves for setup (CRDs, namespaces, secrets), wave 0 for the main app, positive waves for post-deploy (smoke tests, cache warming, notifications). Hook resources without a wave inherit wave 0 and will collide with your main resources -- always set a wave explicitly on hooks.
- **Argo Rollouts replaces `Deployment`, it does not wrap it.** A common confusion is thinking Rollouts is a sidecar that observes a Deployment. It is not -- you delete the Deployment and replace it with a Rollout CRD that has nearly the same `spec.template`. Your existing HPA, PDB, NetworkPolicy, and Service configs work unchanged because the Rollout exposes the same selectors and pod template. Migration is a rename, not a rewrite.

## Next Steps

- **Kargo** (Akuity's promotion engine) -- once you've felt the pain of hand-rolled PR-based promotion, Kargo automates the "promote freight from stage to stage" workflow with a CRD-driven model. Worth a look after you've built the manual version once.
- **GitOps Promoter** (`gitops-promoter.readthedocs.io`) -- the open-source CNCF-aligned alternative to Kargo, built specifically for Argo CD environments. Models environments as a graph and auto-creates PRs between stages with status check gates.
- **ApplicationSet Progressive Syncs** (`strategy.type: RollingSync`) -- roll a single change across N clusters in waves (canary one region, observe, then the rest). Pairs naturally with Argo Rollouts: progressive delivery within a cluster (Rollouts) plus progressive delivery across clusters (ApplicationSet) is the full multi-region rollout story.
- **Disaster recovery drill: "rebuild the GitOps control plane from Git alone."** Tear down ArgoCD entirely in a lab cluster, then reinstall + apply only your root App-of-Apps. Time how long until everything reconciles back. This is THE proof your GitOps setup actually works -- if you can't do it in under 30 minutes, your bootstrap is too fragile.
- **Observability for Helm + ArgoCD + Rollouts:** wire `argocd-metrics`, `argo-rollouts-metrics`, and Helm release events into Prometheus, alert on `argocd_app_info{sync_status!="Synced"}` holding > 10 min, `rollout_phase{phase="Degraded"}`, and AnalysisRun failure rates. Notifications tell humans about deploy events; metrics tell SRE about systemic drift.
