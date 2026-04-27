# Kubernetes Metrics Stack (kube-prometheus-stack + Operator) — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-26
**Difficulty:** Intermediate

## Overview

This plan covers the canonical Kubernetes monitoring stack: the Prometheus Operator, the `kube-prometheus-stack` Helm chart, and the supporting metric pipelines (kube-state-metrics, node-exporter, cAdvisor, metrics-server). You already know Prometheus, PromQL, Alertmanager, and Grafana from earlier this week — today's focus is on the Kubernetes-native glue that wires them together via CRDs and how to operate the stack in production (especially on EKS where control-plane components are partially hidden).

Prerequisite knowledge assumed (already covered):
- Prometheus architecture, scraping, service discovery, federation (Apr 20)
- PromQL selectors, aggregations, recording rules, RED/USE (Apr 21)
- Alertmanager routing, AlertmanagerConfig CRD, burn-rate alerts (Apr 23)
- Grafana dashboards & provisioning (Apr 24)
- EKS, Helm, ArgoCD fundamentals

## Resources

### 1. Prometheus Operator — Design (Official Docs) — 20 min
- **URL:** https://prometheus-operator.dev/docs/getting-started/design/
- **Type:** Official Docs
- **Summary:** The single most important page to internalize today. Walks through every CRD (Prometheus, Alertmanager, ThanosRuler, ServiceMonitor, PodMonitor, Probe, ScrapeConfig, PrometheusRule, AlertmanagerConfig) and — critically — how selector fields on the Prometheus instance (`serviceMonitorSelector`, `podMonitorSelector`, `ruleSelector`, plus their namespaceSelector counterparts) bind config resources to instances. Pay close attention to the "empty selector matches everything / null selector matches nothing" semantics — this is the root cause of most "why isn't my ServiceMonitor being picked up" debugging sessions.

### 2. kube-prometheus-stack Helm Chart README (Official) — 25 min
- **URL:** https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md
- **Type:** Official Docs
- **Summary:** The chart bundles Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics + a curated set of default ServiceMonitors, PrometheusRules, and Grafana dashboards covering the K8s golden signals. Read the "Components included", "Configuration", and especially the upgrade-notes sections (CRDs are not upgraded by Helm — that's the #1 operational gotcha). Skim the values.yaml structure: the `prometheus.prometheusSpec.serviceMonitorSelector{,NilUsesHelmValues}` block is the one you'll touch most often.

### 3. ServiceMonitor and PodMonitor — Service Discovery Guide — 20 min
- **URL:** https://medium.com/@helia.barroso/a-guide-to-service-discovery-with-prometheus-operator-how-to-use-pod-monitor-service-monitor-6a7e4e27b303
- **Type:** Article (community deep-dive, free / no paywall)
- **Summary:** Side-by-side worked examples of ServiceMonitor vs PodMonitor vs ScrapeConfig with full YAML. Clarifies when each fits: ServiceMonitor when a `Service` exists and you want to scrape its endpoints, PodMonitor when scraping pods directly (sidecar metrics, headless workloads, no Service), ScrapeConfig for arbitrary external targets. Covers `endpoints[].port` (must match the Service's named port, not container port), `namespaceSelector.matchNames` vs `any: true`, `interval`, `scrapeTimeout`, and `metricRelabelings` for dropping high-cardinality labels at scrape time.

### 4. Making Sense of Kubernetes Metrics — Kevin Sookocheff — 20 min
- **URL:** https://sookocheff.com/post/kubernetes/making-sense-of-kubernetes-metrics/
- **Type:** Article (engineering blog)
- **Summary:** The clearest explanation of the four overlapping metrics pipelines and what each one is actually for: **cAdvisor** (embedded in kubelet, container resource usage like `container_cpu_usage_seconds_total` and `container_memory_working_set_bytes`, scraped at `/metrics/cadvisor`), **kube-state-metrics** (object-state metrics like `kube_pod_status_phase`, `kube_deployment_spec_replicas`, `kube_node_status_condition` — read from the API server, NOT resource usage), **node-exporter** (host-level CPU/memory/disk/network/filesystem on each node via DaemonSet), and **metrics-server** (lightweight in-memory aggregator powering HPA/VPA/`kubectl top` — not Prometheus-compatible storage, no history). Confusing these is the #1 conceptual mistake; this article fixes that permanently.

### 5. Amazon EKS Control Plane Metrics with Prometheus — 20 min
- **URL:** https://aws.amazon.com/blogs/containers/amazon-eks-enhances-kubernetes-control-plane-observability/
- **Type:** Official AWS Blog
- **Summary:** On managed Kubernetes you don't get direct access to kube-controller-manager and kube-scheduler — the default kube-prometheus-stack ServiceMonitors for those components fail silently on EKS. This post explains the EKS-specific solution: the new `metrics.eks.amazonaws.com` API group exposes scheduler metrics at `/apis/metrics.eks.amazonaws.com/v1/ksh/container/metrics` and controller-manager metrics at `/apis/metrics.eks.amazonaws.com/v1/kcm/container/metrics`, scraped via bearer-token auth. Essential reading if you're targeting EKS — also references AMP and the agentless managed scraper as alternatives.

### 6. kube-state-metrics — Official Docs — 10 min
- **URL:** https://github.com/kubernetes/kube-state-metrics/tree/main/docs
- **Type:** Official Docs
- **Summary:** Skim the metric reference (especially `pod-metrics`, `deployment-metrics`, `node-metrics`) so you know what's available without guessing. Then read the "Scaling kube-state-metrics" / cardinality docs: labels like `pod` rotate every deploy and can blow up Prometheus storage; use `--metric-labels-allowlist` and `--metric-annotations-allowlist` carefully. Also note kube-state-metrics is *stateless* and reflects the API server — it does NOT measure resource usage (that's cAdvisor's job).

### 7. kube-prometheus-stack Troubleshooting — Operator Docs — 15 min
- **URL:** https://prometheus-operator.dev/docs/platform/troubleshooting/
- **Type:** Official Docs
- **Summary:** The production gotchas page. Covers the canonical failure modes you'll hit: (1) ServiceMonitor created but not in Prometheus `/service-discovery` — almost always a label or namespace selector mismatch, with the chart's `release: <release-name>` label as the usual culprit; (2) cross-namespace ServiceMonitors not picked up because `serviceMonitorNamespaceSelector` is too strict; (3) RBAC failures when Prometheus tries to list endpoints in other namespaces; (4) duplicated/missing rules from `ruleSelector` mismatch. Bookmark this — you'll come back to it every time something doesn't scrape.

## Study Tips

- **Build a mental model of the selector chain before touching YAML.** The Operator pattern is: `Prometheus` CR has `serviceMonitorSelector` → matches `ServiceMonitor` CRs by label → each `ServiceMonitor` has its own `selector` matching `Service` objects → those Services point at Pods. Four levels. When something doesn't scrape, trace the chain top-down: is the ServiceMonitor in Prometheus's `/service-discovery`? If yes, are targets being discovered? If no, is the Service correct? Most "broken" stacks have one selector wrong somewhere in this chain.
- **Remember: kube-state-metrics ≠ metrics-server ≠ cAdvisor.** Different sources, different purposes, different consumers. KSM = object state (for Prometheus). cAdvisor = container resource usage (for Prometheus). metrics-server = current pod/node CPU+memory only (for HPA/`kubectl top`, not Prometheus). Node-exporter = host OS metrics. Repeat this distinction until it's reflexive.
- **Read the chart's values.yaml alongside the README.** Many "how do I configure X" questions are answered by searching the values.yaml file directly. The defaults are sensible; production tweaks usually live in `prometheus.prometheusSpec.{retention,resources,storageSpec,additionalScrapeConfigs}`, `alertmanager.config`, and `grafana.{adminPassword,sidecar.dashboards}`.

## Next Steps

After completing this plan, you'll be ready for:
- **Thanos / Cortex / Mimir** — long-term storage and global query view across multiple Prometheus instances (the natural next step once a single Prometheus runs out of room)
- **OpenTelemetry Collector for Kubernetes** — the emerging alternative/complement to Prometheus exporters, especially for traces + metrics convergence
- **Prometheus Adapter** — exposing custom Prometheus metrics to the HPA via the `custom.metrics.k8s.io` API for scaling on application-level signals (request rate, queue depth) instead of just CPU
- **GitOps the whole stack with ArgoCD** — handle the CRD bootstrapping wave (sync-wave annotations) so kube-prometheus-stack installs cleanly on a fresh cluster
