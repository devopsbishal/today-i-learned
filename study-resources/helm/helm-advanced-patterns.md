# Helm Advanced Patterns — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-15
**Difficulty:** Intermediate-to-Advanced (assumes Helm Fundamentals from Apr 13)

## Overview

This plan takes you past single-chart authoring into the patterns real teams use to run Helm at scale: composing umbrella charts from subcharts, propagating configuration via `global`, running lifecycle logic with hooks, extracting shared template logic into library charts, distributing charts through OCI registries like ECR/GHCR, orchestrating many releases across environments with Helmfile, and testing charts with `helm test`, `ct`, and `helm-unittest`. After completing it you'll be able to reason about a multi-chart deployment (e.g. a platform chart bundling Postgres + Redis + your app), write a DB-migration pre-upgrade hook safely, publish a signed chart to ECR, and design a Helmfile-driven dev/staging/prod pipeline.

## Resources

### P0 — Must-Read (Core 90 minutes)

### 1. Subcharts and Global Values (Official Docs) ⏱️ 15 min
- **URL:** https://helm.sh/docs/chart_template_guide/subcharts_and_globals/
- **Type:** Official Docs
- **Summary:** The canonical walkthrough of how parent charts see their subcharts' values but not vice-versa, how `global:` is the only section that propagates downward, and how to override a subchart's values from the parent's `values.yaml`. Read this first — every other advanced pattern (umbrella charts, Helmfile composition, shared config across bundled Postgres/Redis) builds on this mental model.

### 2. Chart Dependencies — Chart.yaml, conditions, tags, alias, import-values (Official Docs) ⏱️ 15 min
- **URL:** https://helm.sh/docs/topics/charts/#chart-dependencies
- **Type:** Official Docs
- **Summary:** The `dependencies:` block reference — how `condition:` toggles subcharts on/off via a values flag (`postgres.enabled=true`), how `tags:` group multiple subcharts under a single switch, how `alias:` lets you include the same chart twice (two Redis instances named `cache` and `queue`), and how `import-values` exports a subchart's values up to the parent namespace. Essential for understanding umbrella charts like `prometheus-community/kube-prometheus-stack`.

### 3. Chart Hooks — Lifecycle, Weights, Delete Policies (Official Docs) ⏱️ 15 min
- **URL:** https://helm.sh/docs/topics/charts_hooks/
- **Type:** Official Docs
- **Summary:** The full hook taxonomy — `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`, `test` — plus `helm.sh/hook-weight` for ordering (ascending, negative-to-positive, ties break alphabetically) and `helm.sh/hook-delete-policy` (`before-hook-creation` default, `hook-succeeded`, `hook-failed`). Internalize the critical gotcha: hook resources are NOT tracked as part of the release, so a failed hook leaves orphaned Jobs/ConfigMaps unless you set delete policies explicitly.

### 4. Library Charts (Official Docs) ⏱️ 10 min
- **URL:** https://helm.sh/docs/topics/library_charts/
- **Type:** Official Docs
- **Summary:** How `type: library` in `Chart.yaml` turns a chart into a pure template-definition package that can't be installed on its own but is depended on by application charts to share `define` blocks across a fleet. The core use case: one `_deployment.tpl` shared by fifty microservice charts so that a single label or securityContext change propagates everywhere. Read alongside the `bitnami/common` chart as a reference implementation.

### 5. Use OCI-Based Registries (Official Docs) ⏱️ 10 min
- **URL:** https://helm.sh/docs/topics/registries/
- **Type:** Official Docs
- **Summary:** The modern chart distribution model — `helm push chart.tgz oci://<registry>/<repo>`, `helm pull oci://...`, `helm install oci://...` — covering auth flow (`helm registry login`), the mandatory `--version` flag on OCI pulls (no "latest"), and how charts are stored as OCI artifacts alongside container images. Since Azure, AWS ECR, and GHCR all support this and classic HTTP `index.yaml` repos are being deprecated, this is the distribution approach to default to going forward.

### 6. Pushing Helm Charts to Amazon ECR (AWS Docs) ⏱️ 10 min
- **URL:** https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html
- **Type:** Official Docs (AWS)
- **Summary:** The concrete ECR workflow you'll actually use at work: `aws ecr get-login-password | helm registry login`, `helm package`, `helm push oci://<account>.dkr.ecr.<region>.amazonaws.com`, and IAM permissions required (`ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`). Pairs directly with resource 5 — read the generic OCI docs first, then this AWS-specific page.

### 7. Helmfile — Getting Started + Best Practices (Official Docs) ⏱️ 15 min
- **URL:** https://helmfile.readthedocs.io/ (and https://helmfile.readthedocs.io/en/stable/writing-helmfile/)
- **Type:** Official Docs
- **Summary:** Helmfile's declarative spec for orchestrating many Helm releases — the `helmfile.yaml` structure (`repositories:`, `releases:`, `environments:`), how `environments:` drives dev/staging/prod value selection, templating with `{{ .Environment.Name }}`, and `helmfile apply`/`diff`/`sync` semantics. This is the tool that fills Helm's single-release-at-a-time gap. Read the getting-started page and the best-practices guide — skip the CLI reference for now.

### P1 — Should-Read (Testing, 20 minutes)

### 8. Chart Tests — helm test (Official Docs) ⏱️ 10 min
- **URL:** https://helm.sh/docs/topics/chart_tests/
- **Type:** Official Docs
- **Summary:** How to annotate a Job with `helm.sh/hook: test` so `helm test <release>` runs it against a deployed release — the classic "is my Postgres reachable from inside the cluster?" smoke-test pattern. Short, but the conceptual distinction from unit testing matters: `helm test` runs *post-deploy* against a real cluster; unit tests run *pre-deploy* against rendered templates.

### 9. helm-unittest Plugin (GitHub README) ⏱️ 10 min
- **URL:** https://github.com/helm-unittest/helm-unittest
- **Type:** Official GitHub (plugin)
- **Summary:** BDD-style YAML unit tests for chart templates — `suite:`, `templates:`, `tests:`, assertions like `isKind`, `equal`, `matchSnapshot`. Scan the README's Write Test Suite File section and the Asserts list. This is how you validate that changing a value actually changes the right field in the rendered Deployment without spinning up a cluster, and it's what CI pipelines typically run alongside `ct lint`.

### P2 — Optional (Bonus, pick one if time allows, ~10 min)

### 10. Advanced Helm Techniques — Post-Renderers & lookup (Official Docs) ⏱️ 10 min
- **URL:** https://helm.sh/docs/topics/advanced/
- **Type:** Official Docs
- **Summary:** Two advanced escape hatches you'll eventually need: (1) post-renderers, which pipe Helm's rendered output through an external binary (typically `kustomize build`) so you can patch a third-party chart without forking it — indispensable for injecting sidecars, securityContexts, or labels upstream charts don't expose; (2) the `lookup` function for reading existing cluster state at render time (e.g. preserving an auto-generated Secret across upgrades). Skim if you have time; revisit when you hit the first upstream chart that lacks a value you need.

## Study Tips

- **Build one umbrella chart as you read.** Before starting, scaffold `platform-chart/` with two stub subcharts (`api/` and `worker/`) and a shared `common/` library chart. Then as you move through resources 1, 2, and 4, actually wire up `dependencies:`, a `global.imageRegistry`, a `condition: worker.enabled`, and one shared `define "common.labels"`. Hands-on beats reading here — composition patterns only click when you've seen `helm dependency update` pull a tarball into `charts/` and watched `helm template` resolve the merged values.
- **Draw the hook timeline.** Sketch the install order on paper: CRDs → `pre-install` hooks (sorted by weight) → main resources → `post-install` hooks → `helm test` on demand. Then repeat for upgrade and rollback. The single most common production bug is a `pre-upgrade` DB-migration job that runs before the new ConfigMap exists because the author forgot the hook sees a mid-flight state, not the final state.
- **Internalize: hooks are not release resources.** A Job you annotate as a hook is tracked separately from the release. That means `helm uninstall` won't clean it up unless you set a `hook-delete-policy`, and `helm rollback` will not re-run it unless it's also a `pre-rollback` hook. This trips up almost everyone once.
- **Treat OCI as the default, HTTP repos as legacy.** When you build your chart-publishing CI this week, default to `helm push oci://` to ECR/GHCR. Only reach for classic `index.yaml` HTTP repos when interoperating with older tooling that doesn't yet support OCI (some ArgoCD versions, some corporate proxies).
- **Helmfile vs ArgoCD is not either/or.** Helmfile is imperative orchestration for local dev and CI pipelines; ArgoCD is declarative GitOps for the cluster. A common production pattern: developers use Helmfile locally for fast iteration, and ArgoCD reconciles a git-committed `Application` manifest in production. You'll see the ArgoCD side of this later in the week — don't try to pick a winner today.

## Next Steps

- **Day 3 (Kustomize basics):** The other major templating approach — overlays, patches, strategic merge — and when to choose it over Helm (or combine them via post-renderers).
- **Later this week (ArgoCD + GitOps):** How ArgoCD's Application CRD consumes Helm charts declaratively (`spec.source.helm.valueFiles`, `valuesObject`, `parameters`), and why GitOps replaces imperative `helm install` in production clusters.
- **Practice project:** Convert your Helm Fundamentals practice chart into an umbrella chart with (a) a `postgresql` subchart gated by a `condition`, (b) a library chart that defines shared labels and a securityContext, (c) a `pre-upgrade` hook Job that runs `alembic upgrade head` with `hook-delete-policy: before-hook-creation,hook-succeeded`, (d) a `helm test` Job that curls `/healthz`, (e) a `helm-unittest` suite asserting the Deployment's image tag changes when you set `image.tag`, and (f) a Helmfile with `dev` and `prod` environments pushing the chart to ECR.

## Sources

- [Subcharts and Global Values | Helm](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/)
- [Charts — Dependencies | Helm](https://helm.sh/docs/topics/charts/#chart-dependencies)
- [Chart Hooks | Helm](https://helm.sh/docs/topics/charts_hooks/)
- [Library Charts | Helm](https://helm.sh/docs/topics/library_charts/)
- [Use OCI-based Registries | Helm](https://helm.sh/docs/topics/registries/)
- [Pushing a Helm chart to Amazon ECR | AWS Docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html)
- [Helmfile Documentation](https://helmfile.readthedocs.io/)
- [Helmfile Best Practices Guide](https://helmfile.readthedocs.io/en/stable/writing-helmfile/)
- [Chart Tests | Helm](https://helm.sh/docs/topics/chart_tests/)
- [helm-unittest Plugin | GitHub](https://github.com/helm-unittest/helm-unittest)
- [Advanced Helm Techniques (post-renderers, lookup) | Helm](https://helm.sh/docs/topics/advanced/)
