# Helm Fundamentals — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-13
**Difficulty:** Intermediate (assumes EKS/Kubernetes fluency)

## Overview

This plan builds a working mental model of Helm as Kubernetes' package manager: how charts are structured, how the Go template engine renders manifests, how values flow from defaults to `--set` overrides, and how the install/upgrade/rollback lifecycle works. After completing it, you'll be able to read any production Helm chart, write your own from scratch with `_helpers.tpl` partials, debug template rendering locally with `helm template` and `--dry-run`, and make informed decisions about value layering — the foundation for the ArgoCD + GitOps deployment work later this week.

## Resources

### 1. Helm Quickstart + Using Helm (Official Intro) ⏱️ 15 min
- **URL:** https://helm.sh/docs/intro/quickstart/ and https://helm.sh/docs/intro/using_helm/
- **Type:** Official Docs
- **Summary:** Baseline mental model — what a chart, release, and repository are, plus the `helm install / upgrade / rollback / uninstall / history` command surface. Skim fast; you likely know most of this from EKS work, but the release-vs-chart distinction is the one concept to nail down before anything else.

### 2. Helm Charts — Structure & Chart.yaml (Official Docs) ⏱️ 20 min
- **URL:** https://helm.sh/docs/topics/charts/
- **Type:** Official Docs
- **Summary:** The canonical reference for chart anatomy: `Chart.yaml` fields (apiVersion v2, appVersion vs version, dependencies), the `templates/`, `charts/`, and `crds/` directories, and how subcharts and dependencies resolve. This is the page you'll return to most often — read it end-to-end and bookmark it.

### 3. Chart Template Guide — Getting Started through Named Templates (Official Docs) ⏱️ 35 min
- **URL:** https://helm.sh/docs/chart_template_guide/
- **Type:** Official Docs
- **Summary:** The core of today's study. Work through these sections in order: Getting Started, Built-In Objects (`.Release`, `.Values`, `.Chart`, `.Files`, `.Capabilities`), Values Files, Template Functions and Pipelines, Flow Control (`if/with/range`), Named Templates (`define`/`include`/`template` and the `_helpers.tpl` convention). Focus on *why* `include` is preferred over `template` (pipelining into `indent`/`nindent`) and why named templates need `.` passed explicitly for scope.

### 4. Debugging Templates (Official Docs) ⏱️ 10 min
- **URL:** https://helm.sh/docs/chart_template_guide/debugging/
- **Type:** Official Docs
- **Summary:** The three debugging tools you'll use daily: `helm lint` for structural/best-practice checks, `helm template` for fully local rendering without a cluster, and `helm install --dry-run --debug` for server-aware simulation that catches conflicting resources. Short but high-ROI — internalize the decision of which tool to reach for when.

### 5. Chart Development Tips and Tricks (Official Docs) ⏱️ 15 min
- **URL:** https://helm.sh/docs/howto/charts_tips_and_tricks/
- **Type:** Official Docs
- **Summary:** The practitioner-level page covering `tpl` (evaluating strings as templates — critical for passing templated values through configmaps), `toYaml`/`toJson`, `default`, `required` for enforcing mandatory values with helpful error messages, and the `lookup` function for reading existing cluster state. These are the functions that separate trivial charts from production-grade ones.

### 6. Helm Template Guide — Advanced Functions & Examples (Palark Blog) ⏱️ 20 min
- **URL:** https://blog.palark.com/advanced-helm-templating/
- **Type:** Blog (practitioner deep dive)
- **Summary:** Widely cited real-world article showing how to build reusable wrapper functions around `tpl`, handle the Bitnami `common.tplvalues.render` pattern for values that may themselves contain templates, and structure `_helpers.tpl` for charts that outgrow trivial examples. Reinforces the official docs with patterns you will actually see in production charts.

### 7. Best Practices — Values & Templates (Official Docs) ⏱️ 10 min
- **URL:** https://helm.sh/docs/chart_best_practices/values/ and https://helm.sh/docs/chart_best_practices/templates/
- **Type:** Official Docs
- **Summary:** Short, opinionated guidance on naming conventions (camelCase values, snake_case is discouraged), flat vs nested value structures, template file naming (`deployment.yaml` not `Deployment.yaml`), and chart-prefixed named templates to avoid collisions when your chart is used as a subchart. Read this before writing your first chart so you don't have to refactor later.

### 8. Helm Repositories + OCI Registries (Official Docs — Skim) ⏱️ 10 min
- **URL:** https://helm.sh/docs/topics/chart_repository/ and https://helm.sh/docs/topics/registries/
- **Type:** Official Docs
- **Summary:** Just enough to understand the two distribution models — the traditional HTTP repo with `index.yaml` + tarballs (`helm repo add`, `helm search repo`) and the newer OCI-based approach (`helm push oci://...`, `helm pull oci://...` to ECR, GHCR, etc.). Skim only — a deeper dive into OCI + ECR lands on Day 2.

## Study Tips

- **Render as you read.** When working through the Chart Template Guide (Resource 3), run `helm create mychart` and actually run `helm template mychart` after each new function you learn. Template logic only sticks once you've seen its output — Go templates are surprisingly easy to misread on paper.
- **Trace values precedence concretely.** Make a scratch chart with a single value like `replicaCount`, then deliberately override it via `values.yaml`, `-f override.yaml`, and `--set replicaCount=3`. Confirm the final rendered output matches the rule: defaults < chart `values.yaml` < parent `values.yaml` < `-f` (rightmost wins) < `--set` (rightmost wins). This one exercise prevents hours of debugging later.
- **Read a real production chart.** After the fundamentals land, spend a few minutes reading the `bitnami/common` chart's `_helpers.tpl` on GitHub (https://github.com/bitnami/charts/tree/main/bitnami/common) to see named-template patterns used at scale — labels, image references, tplvalues.render. This bridges the gap between toy examples and what you'll encounter at work.
- **Keep `include` vs `template` straight.** `include` returns a string and can be piped (`| nindent 4`, `| toYaml`); `template` is an action statement and cannot. When in doubt, use `include`. This is the single most common stumbling block when `_helpers.tpl` output indents wrong.

## Next Steps

- **Day 2 (Helm OCI + chart publishing):** Pushing charts to ECR as OCI artifacts, versioning strategies, `helm package`, chart signing/provenance, and dependency management with `Chart.lock`.
- **Day 3+ (ArgoCD):** How ArgoCD consumes Helm charts declaratively — the Application CRD, `helm.valuesObject` vs `valueFiles`, and why the GitOps model replaces imperative `helm install` in production.
- **Practice project:** Take an existing Kubernetes manifest set (Deployment + Service + ConfigMap + Ingress) and convert it into a chart with `_helpers.tpl` for shared labels, a required `image.repository`, and environment-specific `values-dev.yaml` / `values-prod.yaml` files. This is the exercise that consolidates everything above.

## Sources

- [Helm Quickstart Guide](https://helm.sh/docs/intro/quickstart/)
- [Using Helm](https://helm.sh/docs/intro/using_helm/)
- [Charts | Helm](https://helm.sh/docs/topics/charts/)
- [Chart Template Guide | Helm](https://helm.sh/docs/chart_template_guide/)
- [Debugging Templates | Helm](https://helm.sh/docs/chart_template_guide/debugging/)
- [Chart Development Tips and Tricks | Helm](https://helm.sh/docs/howto/charts_tips_and_tricks/)
- [Helm Template Guide — Advanced Functions & Examples (Palark)](https://blog.palark.com/advanced-helm-templating/)
- [Best Practices: Values | Helm](https://helm.sh/docs/chart_best_practices/values/)
- [Best Practices: Templates | Helm](https://helm.sh/docs/chart_best_practices/templates/)
- [Chart Repository Guide | Helm](https://helm.sh/docs/topics/chart_repository/)
- [Registries (OCI) | Helm](https://helm.sh/docs/topics/registries/)
