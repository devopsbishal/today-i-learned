# Kubernetes Metrics Stack -- The Operator, the CRDs, and the Four Pipelines That Watch Your Cluster

> Last week we learned the parts of the orchestra: Prometheus scrapes (Apr 20), PromQL queries (Apr 21), Alertmanager triages (Apr 23), Grafana visualizes (Apr 24). Today we wire them all together the way nobody actually wires them by hand in production -- through a **single Helm chart** (`kube-prometheus-stack`) that drops a fully-formed observability rig into your cluster, plus the **Prometheus Operator** that turns hand-written `prometheus.yml` into Kubernetes-native CRDs.
>
> The core analogy for the day: **the Prometheus Operator is the building's facilities manager.** In the old world (Apr 20) you were the building owner with a wrench in your hand: you wrote `prometheus.yml` directly, listed every scrape job, every relabel rule, every alerting target. It worked, but it didn't scale -- adding a new tenant meant editing the central config and bouncing the building. In the Kubernetes-native world, you don't write `prometheus.yml` anymore. You file a **work order** (`ServiceMonitor`, `PodMonitor`, `PrometheusRule`) describing *what* you want monitored ("scrape the API service's `/metrics` endpoint every 30 seconds, with this auth"), and **facilities figures out the wiring** -- the Operator watches CRDs, regenerates `prometheus.yml` from them, mounts it into the Prometheus pod, and triggers a hot-reload via `/-/reload`. The pod's actual config is now an *output*, not an *input*. Read that twice. It changes how you debug everything.
>
> Around the facilities manager sits a small ecosystem of specialist trades: **kube-state-metrics** is the building registry (who is the lease holder for unit 4B, are they in good standing, how many bedrooms does it have), **node-exporter** is the per-floor electrician (voltage, breaker status, HVAC load on each floor), **cAdvisor** is the security camera in every apartment showing live activity (this resident is using 80% of their power allotment *right now*), and **metrics-server** is the building manager's desk thermometer -- a separate, lightweight thing they glance at to decide whether to send a maintenance crew. They do not replace each other. You install all of them, and each answers a question the others cannot.
>
> Hold those four characters in your head for the next 2,000 lines: the **facilities manager** (Operator), the **registry** (KSM), the **electrician** (node-exporter), the **camera** (cAdvisor). Plus the chart that hires them all in one go.

---

**Date**: 2026-04-26
**Topic Area**: prometheus
**Difficulty**: Intermediate / Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Prometheus Operator | Building's facilities manager | A controller that watches CRDs and generates `prometheus.yml` from them; you describe *what* to monitor, it figures out *how* |
| `kube-prometheus-stack` chart | A general-contractor package -- "install the whole observability building in one PO" | Helm chart bundling Operator + Prometheus + Alertmanager + Grafana + node-exporter + KSM + default rules + dashboards |
| `Prometheus` CRD | The work-order form for "we need a Prometheus instance, sized this way, watching these CRs" | The Operator's input describing a Prometheus deployment (replicas, retention, storage, selectors) |
| `Alertmanager` CRD | Same but for the triage nurse | The Operator's input describing an Alertmanager deployment |
| `ServiceMonitor` CRD | Delivery instructions to the building's main reception | "When you find a Service with these labels, scrape its endpoints on port X every Y seconds" |
| `PodMonitor` CRD | Delivery instructions directly to the apartment door, bypassing reception | Same shape but matches Pods directly, used when no Service exists (sidecars, headless workloads) |
| `Probe` CRD | A milkman doing his rounds checking each address from the outside | Tells blackbox-exporter to probe a list of external URLs (HTTP / TCP / ICMP) |
| `PrometheusRule` CRD | The recipe book for vital-sign thresholds and shortcut formulas | Recording + alerting rules as a Kubernetes object; Operator regenerates rule ConfigMap on change |
| `AlertmanagerConfig` CRD | Each ward's local triage sub-flowchart | Namespace-scoped Alertmanager routing config (covered Apr 23) |
| `ScrapeConfig` CRD | Escape hatch -- raw `scrape_config` as a CR | For exotic targets that don't fit ServiceMonitor/PodMonitor; uses native Prometheus syntax |
| The selector chain | Four-link chain from Prometheus down to a pod | Prometheus -> ServiceMonitor (label) -> Service (label) -> Pod -- one wrong link, no scrape |
| `serviceMonitorSelector` | The facilities manager's filing cabinet -- which work orders does she even read? | Label selector on the `Prometheus` CRD; matches `ServiceMonitor` CRs by label |
| `*NamespaceSelector` | Which floors of the building does facilities have keys to | Pairs with `*Selector`; controls which namespaces are searched |
| `release: <name>` label gotcha | The work-order form must have the right department stamp or it goes in the trash | Default chart selectors match `release: kube-prometheus-stack`; foreign SMs need this label or won't be picked up |
| `*SelectorNilUsesHelmValues` | "If the field is empty, do you mean 'all' or do you mean 'use my preset'?" | Chart-specific switch; set `false` to make empty selectors mean "match all" |
| `{}` vs `null` selector | "I want all the things" vs "the field doesn't exist, default behavior please" | `{}` matches everything, `null`/missing means "use default" -- different in CRDs! |
| cAdvisor (in kubelet) | The apartment's security camera -- live footage of resource use | Container CPU/memory/network usage, scraped from each kubelet's `/metrics/cadvisor` |
| kube-state-metrics | The building registry -- desired state, leases, paperwork | Reflects API server objects as metrics: `kube_pod_status_phase`, `kube_deployment_spec_replicas` |
| node-exporter | The per-floor electrician's panel | Host OS metrics on every node via DaemonSet: CPU, memory, disk, network, filesystem |
| metrics-server | A separate pocket thermometer on the building manager's desk | Lightweight in-memory aggregator powering HPA / `kubectl top`; NOT Prometheus |
| The four-layer hierarchy | Building -> floor -> apartment -> outlet | Cluster (KSM) -> node (node-exporter) -> pod/container (cAdvisor) -> app (your `/metrics`) |
| CRD upgrade gotcha | The contractor doesn't replace the foundation when remodeling | `helm upgrade` does NOT touch CRDs; must `kubectl apply` the new ones first or upgrades silently break |
| ArgoCD bootstrap waves | The contractor schedules concrete before framing before drywall | `sync-wave: -2` (CRDs) -> `-1` (Operator) -> `0` (Prometheus, SMs, rules) for clean bootstrap |
| EKS control-plane gotcha | Some floors of the building are owned by AWS -- you don't have keys | `kubeControllerManager`, `kubeScheduler`, `kubeProxy` ServiceMonitors fail on EKS; must disable |
| ServiceMonitor reload | The facilities manager re-types the master config and pushes "reload" | Operator regenerates `prometheus-prometheus.yaml` ConfigMap; Prometheus reloads via `/-/reload` (~30-60s) |

---

## The Big Picture in One Diagram

Before we zoom in, here is the whole stack laid out:

```
                       kube-prometheus-stack (Helm chart)
   +--------------------------------------------------------------------------+
   |                                                                          |
   |   +-----------------+         +-------------------+                      |
   |   | Prometheus      |  reads  | ServiceMonitor    |  matches  +-------+  |
   |   | Operator        | -------->                   | --------> |Service|  |
   |   | (controller)    |  CRDs   | PodMonitor        |  by label +-------+  |
   |   |                 |         | PrometheusRule    |               |      |
   |   |                 |         | Probe             |               v      |
   |   |  watches API    |         | ScrapeConfig      |           +------+   |
   |   |  generates      |         +-------------------+           | Pod  |   |
   |   |  prometheus.yml |                                          | /metrics |
   |   |  via Secret     |                                          +------+   |
   |   +--------+--------+                                                     |
   |            | mounts                                                       |
   |            v                                                              |
   |   +-----------------+      remote_write    +------------------+           |
   |   | Prometheus      |--------------------->| Thanos / Mimir   |           |
   |   | StatefulSet     |     (optional)       | (long-term)      |           |
   |   | (scrapes)       |                      +------------------+           |
   |   +--------+--------+                                                     |
   |            | alerts                                                       |
   |            v                                                              |
   |   +-----------------+                                                     |
   |   | Alertmanager    |---> Slack / PagerDuty / webhooks                    |
   |   | StatefulSet     |                                                     |
   |   | (HA, gossip)    |                                                     |
   |   +-----------------+                                                     |
   |                                                                          |
   |   +-----------------+   +-------------------+   +-------------------+    |
   |   | node-exporter   |   | kube-state-       |   | Grafana           |    |
   |   | DaemonSet       |   | metrics Deploy.   |   | Deploy. + sidecar |    |
   |   | (per-node host) |   | (cluster-wide)    |   | (dashboards)      |    |
   |   +-----------------+   +-------------------+   +-------------------+    |
   |                                                                          |
   |   default ServiceMonitors target ALL of the above + kubelet/cAdvisor     |
   |   default PrometheusRules cover golden signals for K8s + nodes           |
   |   default Grafana dashboards visualize the metrics                       |
   +--------------------------------------------------------------------------+

   Outside the chart but adjacent:
   +-----------------+
   | metrics-server  |   <-- separate Deploy, used by HPA / `kubectl top`,
   | (in-memory)     |       NOT Prometheus; install alongside, not instead of
   +-----------------+
```

**Five things to remember about this diagram:**

1. **The Operator is the only thing that talks to the Kubernetes API on your behalf.** Prometheus itself doesn't watch the API for ServiceMonitors -- it only knows how to scrape. The Operator translates CRD changes into `prometheus.yml` and pushes the new config in.
2. **Every component has a default ServiceMonitor that ships with the chart.** node-exporter, KSM, kubelet, the Operator itself, Grafana, Alertmanager, even Prometheus scraping itself. You inherit ~20 ServiceMonitors on day one.
3. **Default selectors filter on `release: <chart-release-name>`.** This is the single biggest gotcha; we will hammer it relentlessly in Part 4.
4. **metrics-server is NOT in this chart.** It's a separate install (often via `kubectl apply` from its own repo or AWS's add-on for EKS). It powers HPA and `kubectl top`. You will run both.
5. **cAdvisor is not a separate Deployment.** It is *embedded in kubelet*. The chart installs a ServiceMonitor that scrapes `kubelet:10250/metrics/cadvisor` -- no extra pod required.

---

## Part 1: The Operator Pattern -- Why You Don't Write `prometheus.yml` Anymore

### The Old World

In Apr 20 you wrote scrape configs like this directly:

```yaml
# prometheus.yml -- by hand
scrape_configs:
  - job_name: 'api'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_label_app]
        regex: api
        action: keep
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        regex: metrics
        action: keep
    scrape_interval: 30s
```

It worked. But adding a new service meant editing the master config, pushing it through your config-management tool, and reloading Prometheus. The config grew, drifted, became no one's responsibility. Multi-team clusters were a nightmare -- the platform team owned a single YAML file with hundreds of jobs from a dozen application teams, and any team could break the parser for everyone else.

### The New World

The Prometheus Operator inverts the relationship. **Application teams describe what they want monitored as Kubernetes objects in their own namespace.** The Operator watches those CRDs and assembles the central `prometheus.yml` automatically.

```yaml
# A team's ServiceMonitor -- lives in their namespace
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server
  namespace: api
  labels:
    release: kube-prometheus-stack    # the magic label, see Part 4
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

That's it. No editing the central config. The team owns their monitoring contract; the platform team owns the Prometheus runtime. Separation of concerns, GitOps-friendly, namespace-scoped RBAC. It is, after the analogies, the actual reason this stack exists.

### The Reconciliation Loop in One Picture

```
+-----------------+      +------------------+      +--------------------+
| You apply       |      | Operator watches |      | Operator generates |
| ServiceMonitor  |----->| api-server       |----->| prometheus.yml     |
| via kubectl/    |      | watches CRDs     |      | (concatenates ALL  |
| Helm/ArgoCD     |      |                  |      | matching CRDs)     |
+-----------------+      +------------------+      +---------+----------+
                                                              |
                                                              v
+----------------------+      +-----------------------+   +-------------+
| Prometheus pod sees  |<-----| Operator updates      |<--| Operator    |
| new config, reloads  |      | Secret/ConfigMap that |   | computes    |
| via /-/reload (~30s) |      | Prometheus mounts     |   | full config |
+----------------------+      +-----------------------+   +-------------+
```

When the loop is healthy, `kubectl apply -f my-servicemonitor.yaml` results in a new scrape target appearing in `Prometheus UI -> Status -> Service Discovery` within ~60 seconds. When it isn't, you debug the selector chain (Part 4).

### CRDs the Operator Owns

| CRD | Purpose | Cluster-scoped? |
|-----|---------|-----------------|
| `Prometheus` | Describes a Prometheus *instance* (StatefulSet) -- replicas, retention, resources, storage, selectors | Namespace-scoped |
| `Alertmanager` | Same for Alertmanager | Namespace-scoped |
| `ThanosRuler` | Same for Thanos Ruler (out of scope today) | Namespace-scoped |
| `ServiceMonitor` | "Scrape these Service endpoints" | Namespace-scoped |
| `PodMonitor` | "Scrape these Pods directly" | Namespace-scoped |
| `Probe` | "Tell blackbox-exporter to probe these URLs" | Namespace-scoped |
| `PrometheusRule` | Recording + alerting rules (Apr 23) | Namespace-scoped |
| `AlertmanagerConfig` | Per-namespace Alertmanager routing (Apr 23) | Namespace-scoped |
| `ScrapeConfig` | Raw scrape_config in CR form -- escape hatch for exotic targets | Namespace-scoped |
| `PodMonitor`/`Probe` etc. | (same family) | Namespace-scoped |

All CRDs are namespace-scoped except the controller's own permissions. The Operator runs cluster-wide and watches every namespace it has RBAC for.

---

## Part 2: The `kube-prometheus-stack` Chart -- One Helm Install, Whole Building

### What's in the Box

| Component | Type | Purpose |
|-----------|------|---------|
| Prometheus Operator | Deployment | The controller above |
| Prometheus | StatefulSet (1+ replicas) | The actual time-series DB |
| Alertmanager | StatefulSet (default 1, recommend 3) | Triage (Apr 23) |
| Grafana | Deployment | Dashboards (Apr 24) |
| node-exporter | DaemonSet | Per-node host metrics |
| kube-state-metrics | Deployment | Object-state metrics |
| Default ServiceMonitors | ~15-20 SMs | Scrape targets for all of the above + control plane |
| Default PrometheusRules | ~30+ rules | "KubePodCrashLooping", "KubeNodeNotReady", etc. |
| Default Grafana dashboards | ~30 dashboards | Pre-built K8s dashboards |

You install one Helm chart, and 90% of "what should we be monitoring on a vanilla K8s cluster" is answered out of the box. The remaining 10% is your applications and your operational tweaks.

### Two Charts Easily Confused

There are **two** Prometheus-Operator-flavored bundles in the wild and people mix them up constantly:

| Name | Source | What it is |
|------|--------|------------|
| `prometheus-community/kube-prometheus-stack` | https://github.com/prometheus-community/helm-charts | **The Helm chart you actually install.** Production-grade, frequently updated, supports CRD lifecycle. |
| `prometheus-operator/kube-prometheus` | https://github.com/prometheus-operator/kube-prometheus | A **jsonnet** library that generates raw manifests. Reference for what the Operator team considers a canonical install. Use as a reading source, not as something you `kubectl apply`. |

If someone says "kube-prometheus-stack," they mean the Helm chart. If they say just "kube-prometheus," they probably mean the jsonnet library. Add to vocabulary, move on.

### The Top-Level `values.yaml` Map

Skim the chart's values.yaml the first time you install it. The shape:

```yaml
crds:
  enabled: true                    # whether to ship CRDs at all (helm v3)

global:
  imageRegistry: ""                # override registry for air-gapped envs

defaultRules:
  create: true                     # ship the default 30+ alerting rules
  disabled:                        # turn specific ones off (Apr 23 covered this)
    - KubeMemoryOvercommit

additionalPrometheusRulesMap:      # inject custom PrometheusRules from values
  my-team-rules:
    groups: [...]

alertmanager:
  enabled: true
  alertmanagerSpec:                # passed straight to Alertmanager CRD
    replicas: 3
    storage:
      volumeClaimTemplate: {...}

prometheus:
  enabled: true
  prometheusSpec:                  # passed straight to Prometheus CRD
    replicas: 2
    retention: 30d
    serviceMonitorSelector: {}     # see Part 4
    serviceMonitorSelectorNilUsesHelmValues: false
    storageSpec:
      volumeClaimTemplate: {...}

prometheusOperator:
  enabled: true
  admissionWebhooks:
    enabled: true                  # validates ServiceMonitors at admission time
  resources: {...}

grafana:
  enabled: true
  adminPassword: ""                # use existingSecret in production
  sidecar:
    dashboards:
      enabled: true                # auto-discover ConfigMaps with `grafana_dashboard: "1"` label
  defaultDashboardsEnabled: true

nodeExporter:
  enabled: true

kube-state-metrics:                # subchart with hyphens because of upstream naming
  enabled: true

# Control plane components -- the EKS gotcha lives here
kubeApiServer:
  enabled: true
kubeControllerManager:
  enabled: true                    # FAILS on EKS -- see Part 9
kubeScheduler:
  enabled: true                    # FAILS on EKS -- see Part 9
kubeProxy:
  enabled: true                    # FAILS on some EKS configs -- see Part 9
kubelet:
  enabled: true
coreDns:
  enabled: true
kubeEtcd:
  enabled: true                    # only useful on self-hosted control planes
```

About 80% of production configuration touches `prometheus.prometheusSpec`, `alertmanager.alertmanagerSpec`, `grafana.*`, and the control-plane toggles for your cloud provider. The defaults are sensible; tweak the things that actually matter.

### The CRD Upgrade Gotcha (Read This Twice)

**Helm does not upgrade CRDs.** This is by design -- Helm v3's CRD lifecycle is "install once, never touch again," because deleting a CRD wipes all its CRs (catastrophic). On a `helm upgrade kube-prometheus-stack ...`, the chart's templates render fine, but the new CRD schemas are silently ignored. Your new `ServiceMonitor` field with the new validation passes Helm rendering, fails admission webhook because the cluster's CRD is the old version.

**The fix every time you upgrade:**

```bash
# 1. Apply the new CRDs from the chart's crds/ directory
CHART_VERSION=58.0.0
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-community/helm-charts/kube-prometheus-stack-${CHART_VERSION}/charts/kube-prometheus-stack/charts/crds/crds/

# 2. THEN upgrade the chart
helm upgrade kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --version ${CHART_VERSION} \
  -n monitoring \
  -f values.yaml
```

`--server-side` matters here -- the CRDs are large enough that the standard client-side apply hits the **256 KB annotation limit** that we saw in Apr 16's ArgoCD doc. Use server-side apply for any CRD bootstrap from now on.

If you skip this step, the symptom is "my new ServiceMonitor field isn't taking effect" and you'll spend an hour debugging selectors before realizing the CRD itself is two versions stale.

---

## Part 3: The Selector Chain -- Four Links That Must All Hold

This is the single most important section of today's doc. Get this right and 90% of "why isn't this scraping" tickets disappear. Get it wrong and you'll be debugging it for the rest of your career.

### The Chain

```
+--------------+   serviceMonitorSelector   +----------------+
| Prometheus   |--------------------------->| ServiceMonitor |
| CRD          |   matches SMs by label     | CRD            |
+------+-------+                            +--------+-------+
       |                                             |
       | also: serviceMonitorNamespaceSelector       | spec.selector
       | (which namespaces to look in)               | matches Services by label
       |                                             |
       v                                             v
   (filters)                                  +------------+
                                              | Service    |
                                              | object     |
                                              +-----+------+
                                                    |
                                                    | spec.selector
                                                    | matches Pods by label
                                                    v
                                              +------------+
                                              | Pod        |
                                              | (with /metrics)
                                              +------------+
```

**Four links, all must hold:**

1. The **Prometheus CRD's `serviceMonitorSelector`** must match the **ServiceMonitor's labels**.
2. The **Prometheus CRD's `serviceMonitorNamespaceSelector`** must include the **ServiceMonitor's namespace**.
3. The **ServiceMonitor's `selector.matchLabels`** must match the **Service's labels**.
4. The **Service's `spec.selector`** must match **Pod labels** (this is plain Kubernetes, not Prometheus-specific).

If any link breaks, the target does not appear in `Prometheus UI -> Status -> Service Discovery`. The trace-through-top-down debugging method:

```bash
# Step 1: is the Prometheus CRD configured to look at my SM?
kubectl get prometheus -n monitoring -o yaml | yq '.items[0].spec | {serviceMonitorSelector, serviceMonitorNamespaceSelector}'

# Step 2: does my SM have the right labels?
kubectl get servicemonitor -n my-app api-server -o yaml | yq '.metadata.labels'

# Step 3: does my SM's selector find the Service?
kubectl get svc -n my-app -l app=api -o name

# Step 4: does the Service find Pods?
kubectl get endpoints -n my-app api -o yaml
# If the .subsets[].addresses[] is empty, the Service's selector doesn't match Pods.
```

### The `release` Label Trap

Out of the box, `kube-prometheus-stack` configures the Prometheus CRD with:

```yaml
serviceMonitorSelector:
  matchLabels:
    release: kube-prometheus-stack    # whatever your Helm release name is
ruleSelector:
  matchLabels:
    release: kube-prometheus-stack
podMonitorSelector:
  matchLabels:
    release: kube-prometheus-stack
probeSelector:
  matchLabels:
    release: kube-prometheus-stack
```

This is sensible defaults logic: "only pick up SMs/rules/PMs that were installed by *this* Helm release." It's a multi-tenancy guardrail. But it has an aggressively bad consequence: **any ServiceMonitor created by another chart (ingress-nginx, cert-manager, your own apps) will be ignored** unless it carries that label.

You hit this within a week of every fresh install. Symptoms:

- You install `ingress-nginx` with `controller.metrics.serviceMonitor.enabled: true`. ingress-nginx happily creates a ServiceMonitor.
- You wait. No metrics in Prometheus.
- You check the SM with `kubectl get servicemonitor -A`. It's there. It looks correct.
- You stare at it for an hour.
- You finally check the Prometheus CRD's selector. It's filtering on `release: kube-prometheus-stack`. The ingress-nginx SM has `release: ingress-nginx` (or no release label at all). Mismatch.

We covered this same pattern for `ruleSelector` in Apr 23. **Today's lesson: it applies to ALL the `*Selector` fields on the Prometheus CRD.** ServiceMonitors, PodMonitors, Probes, PrometheusRules -- same selector logic, same trap.

### Three Fixes (Pick One Strategy and Stick With It)

**Strategy A: Make selectors match-all.** Recommended for single-tenant clusters or any team with control over what gets installed:

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelector: {}              # match all
    serviceMonitorNamespaceSelector: {}     # in any namespace
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
    ruleSelector: {}
    ruleNamespaceSelector: {}
    probeSelector: {}
    probeNamespaceSelector: {}
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
```

That last set of `*NilUsesHelmValues: false` is **chart-specific** (not a CRD field) and is required because the chart's templating layer interprets `selector: {}` as "use my preset" by default. The flag tells Helm to interpret nil/empty as "no selector, match all."

**Strategy B: Label every foreign ServiceMonitor with the release label.** Stricter but more control:

```yaml
# When installing ingress-nginx
controller:
  metrics:
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: kube-prometheus-stack
```

You do this for every chart you install. It works, but it's tedious and error-prone -- forget the label once and you spend an afternoon debugging.

**Strategy C: Multi-tenant pattern -- segment by label, not by chart.** When multiple teams install their own monitoring stacks (uncommon, but exists in very large orgs):

```yaml
# Platform team's Prometheus picks up only platform-labeled SMs
prometheus:
  prometheusSpec:
    serviceMonitorSelector:
      matchLabels:
        prometheus: platform
```

Teams then label their ServiceMonitors with `prometheus: platform` (or `prometheus: app-team` for their own instance) to opt in.

**Industry default**: Strategy A. It's what 80% of production clusters do. Strategy B is what enterprise orgs with tight chart governance do. Strategy C is rare.

### `{}` vs `null` -- The Subtle Difference

We noted this in Apr 23 for `ruleSelector` but it deserves a re-emphasis because it bites here too. In Kubernetes API semantics:

- `serviceMonitorSelector: {}` = "empty selector, matches everything"
- `serviceMonitorSelector: null` (or field omitted entirely) = "no selector configured, default behavior"

What "default behavior" is depends on the *Operator version* and the chart's `*NilUsesHelmValues` flag. Older Operators interpreted nil as "match nothing" -- the worst of all defaults, because your fresh install had a Prometheus that scraped exactly nothing. Newer Operators are kinder. Do not rely on the default; **always be explicit**: either `{}` or a real selector with `matchLabels`. Never leave the field empty.

### Namespace Selectors

`serviceMonitorNamespaceSelector` is the partner field. Even if your ServiceMonitor has the right labels, Prometheus won't scrape it if its *namespace* isn't in scope.

```yaml
# Two flavors:
serviceMonitorNamespaceSelector: {}                         # all namespaces
# vs
serviceMonitorNamespaceSelector:
  matchLabels:
    monitored: "true"                                       # only namespaces with this label
```

Default chart behavior in recent versions: `{}` (all namespaces). But on locked-down multi-tenant clusters this is sometimes set to a label-restricted subset, in which case **you must label your namespace** (`kubectl label namespace api monitored=true`) for any SMs in there to be picked up. Add this to your application namespace's bootstrap manifests.

### Selectors Are Operational, Not Security

A common interview-grade misconception worth nailing down: **selectors are not a security boundary**. They are an *operational filter* the Operator uses to decide what to read. Anyone with `kubectl edit prometheus` (or who can mutate the Prometheus CR via Helm/ArgoCD) can change them. They keep teams' configs out of each other's hair on a *cooperating* cluster; they do not stop a determined or careless actor from making the Operator pick up something it shouldn't.

The actual security boundary is **RBAC on `monitoring.coreos.com` resources**, enforced by `kube-apiserver`. If you want a team to write their own ServiceMonitors but not touch the platform's:

- Give them `create/update/patch/delete` on `monitoring.coreos.com/v1` resources scoped to their team namespace via a `Role` (not a `ClusterRole`).
- Keep platform CRs in the `monitoring` namespace, with no team principals authorized there.
- Set the Prometheus CR's `*NamespaceSelector: {}` so the Operator picks up team-owned monitors anywhere RBAC allows them to live.

**The framing**: RBAC is the security wall; selectors, `enforced*` limits, Kyverno admission policies, and per-namespace `AlertmanagerConfig` (Apr 23, with `alertmanagerConfigMatcherStrategy: OnNamespace`) are layered defense-in-depth on top of it. Selectors alone are not enough.

---

## Part 4: ServiceMonitor -- Full Anatomy

The ServiceMonitor is the workhorse CRD. 80% of what you write is ServiceMonitors. Master this YAML.

### Complete Example with Every Important Field

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server
  namespace: api
  labels:
    release: kube-prometheus-stack    # match Prometheus's serviceMonitorSelector
    team: api                          # for your own filtering
spec:
  # ----- WHICH SERVICES TO MATCH -----
  selector:
    matchLabels:
      app: api                          # matches `Service` labels (NOT pod labels)
  namespaceSelector:
    matchNames: [api, api-staging]      # OR: any: true (all namespaces)
  jobLabel: app                          # which Service label becomes Prometheus's `job` label
  targetLabels: [team, tier]             # additional Service labels to copy onto every metric
  podTargetLabels: [version]             # Pod labels to copy onto metrics

  # ----- HOW TO SCRAPE -----
  endpoints:
    - port: metrics                      # named port on the Service (NOT container port number!)
      interval: 30s                      # scrape interval (default: Prometheus global)
      scrapeTimeout: 10s                 # timeout per scrape (must be <= interval)
      path: /metrics                     # default
      scheme: http                       # http or https
      honorLabels: false                 # if true, target's labels override server-side labels
      honorTimestamps: true              # respect timestamps from /metrics output

      # ----- AUTH (modern Operator: prefer `authorization`, not bearerTokenFile) -----
      authorization:
        type: Bearer
        credentials:
          name: api-bearer-token            # Secret in the SAME namespace as the SM
          key: token
      # OR (legacy, still works -- kept for backward compat):
      # bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      # OR for mTLS:
      # tlsConfig:
      #   ca:   { secret: { name: api-tls, key: ca.crt } }
      #   cert: { secret: { name: api-tls, key: tls.crt } }
      #   keySecret: { name: api-tls, key: tls.key }
      #   insecureSkipVerify: false
      # OR for basic auth:
      # basicAuth:
      #   username: {name: api-creds, key: username}
      #   password: {name: api-creds, key: password}

      # ----- RELABELING -----
      relabelings:                       # rewrite labels on the SCRAPE CONFIG (before scrape)
        - sourceLabels: [__meta_kubernetes_pod_label_version]
          targetLabel: app_version
        - sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel: node

      metricRelabelings:                 # rewrite labels on individual METRICS (after scrape)
        - sourceLabels: [__name__]
          regex: 'go_.*'                 # drop noisy Go runtime metrics
          action: drop
        - sourceLabels: [le]             # drop high-cardinality histogram bucket label for one metric
          regex: '.*'
          action: replace
          targetLabel: le
          replacement: 'masked'

  # ----- SAFETY LIMITS (CARDINALITY GUARDRAILS) -----
  sampleLimit: 10000                     # drop the scrape if it returns > 10K samples
  labelLimit: 30                         # drop if any metric has > 30 labels
  labelValueLengthLimit: 200             # drop if a label value is longer than 200 chars
  targetLimit: 100                       # drop if SM matches > 100 targets
```

### Field-by-Field Notes

| Field | Critical detail |
|-------|-----------------|
| `selector.matchLabels` | Matches **Service** objects, not Pods. The Service then matches Pods through its own selector. |
| `namespaceSelector.matchNames` | A literal list. Use `any: true` for "all namespaces." |
| `endpoints[].port` | The **named port on the Service**, not the container port. If your Service exposes `port: 9090` named `metrics`, write `port: metrics`, not `port: 9090`. Anonymous ports work but are bad practice. |
| `jobLabel` | Sets Prometheus's `job` label to the value of the named Service label. Default `job` is `<namespace>/<service-name>` -- usually fine, but `jobLabel: app` makes it cleaner. |
| `targetLabels` / `podTargetLabels` | The "I want this Service/Pod label to show up on every metric scraped" knob. Use sparingly -- each one increases cardinality. |
| `relabelings` | Rewrites labels on the scrape **target metadata** (the `__meta_kubernetes_*` family from kubernetes_sd_configs). Done before scraping. |
| `metricRelabelings` | Rewrites labels on **scraped metrics**. Done after scraping. The cardinality kill-switch lives here. |
| `sampleLimit` | Hard cap on samples per scrape. The single most effective cardinality safety belt. Set it. |
| `authorization` / `tlsConfig` / `basicAuth` | Standard auth modes. **Use `authorization.credentials` referencing a Secret as the modern pattern**; `bearerTokenFile` still works but is the legacy path (Secrets give you rotation, RBAC, and avoid baking tokens into the Prometheus pod). |
| `honorLabels: true` | Means "if the target sets a label `instance` / `job`, don't override it." Useful for pushgateway / federation; almost always wrong for direct application scrapes. |

#### When `honorLabels: true` is actually right

The default (`false`) is correct for ~95% of scrapes -- you want Prometheus's automatic `instance` and `job` labels to win, so all of `api-pod-0`, `api-pod-1`, `api-pod-2` show up correctly with their pod-derived `instance` value.

Set `honorLabels: true` only when scraping **a source that legitimately owns its own `instance` / `job` labels**:

- **Pushgateway** -- the whole point is that batch jobs registered themselves under a specific `instance`/`job`; you want those preserved, not overwritten with "pushgateway-0".
- **Federation** (`/federate` from another Prometheus) -- the upstream Prometheus already labeled each series with the original target's `instance`/`job`; if Prometheus overrode them, you'd lose all original target identity and every series would say `instance=upstream-prometheus-0`.
- **Any exporter that proxies metrics on behalf of multiple real targets** (some SNMP exporters, blackbox-exporter probes use this pattern through different mechanisms).

The damage of getting it wrong: target-set labels overwrite Prometheus's automatic labels, breaking `up{instance=...}` semantics, breaking your alerting on which pod is actually down, and breaking every dashboard that groups by `instance`.

### When to Use ServiceMonitor

Default. If your app is fronted by a Service, write a ServiceMonitor. The discovery is more idiomatic to Kubernetes (Services are first-class), the dedup is automatic (multiple replicas behind one Service all show up correctly), and the YAML is shorter. **Use ServiceMonitor unless you have a specific reason to use PodMonitor.**

### Aside: `endpoints` vs `endpointslice` Discovery Role

Under the hood, ServiceMonitor's discovery uses Prometheus's `kubernetes_sd_configs` with either the `endpoints` or `endpointslice` role. Modern Prometheus (2.21+) and recent Operator versions default to **`endpointslice`** because it scales better on large clusters -- the `Endpoints` resource is one object per Service capped at 1000 endpoints, while `EndpointSlice` shards across multiple objects.

You usually don't touch this setting. But two cases where it matters:

- **Headless Services (`clusterIP: None`).** Discovery still works because EndpointSlices are populated even when there's no clusterIP. ServiceMonitor against a headless StatefulSet's per-pod DNS (`pod-0.svc.namespace`) is a fine pattern.
- **Very large Services (>1000 endpoints).** If your cluster is on the older `endpoints` role, you'll see truncated targets. Modern Operator versions handle this transparently.

The Operator exposes `serviceMonitorEndpointsRoles` / `endpointSliceSupported` knobs on some versions; just use the defaults unless you have a specific scaling problem.

### Platform-Level Capacity Caps: the `enforced*` Family

Per-ServiceMonitor `sampleLimit`/`labelLimit`/`targetLimit` are *suggestions* -- a team can write a ServiceMonitor with `sampleLimit: 10000000` and torch your Prometheus the next time a misbehaving exporter explodes. The platform-team answer is the **`enforced*` family on the `Prometheus` CR**:

```yaml
prometheus:
  prometheusSpec:
    enforcedSampleLimit: 50000           # ceiling on samples per scrape, cluster-wide
    enforcedTargetLimit: 1000            # ceiling on targets per ServiceMonitor
    enforcedLabelLimit: 30               # max labels per metric
    enforcedLabelValueLengthLimit: 200   # max length of a label value
    enforcedLabelNameLengthLimit: 200    # max length of a label name
```

These **override anything teams configure higher** at scrape time. A ServiceMonitor that requests `sampleLimit: 10000000` still gets capped at 50,000. The Operator translates them into the equivalent `prometheus.yml` `body_size_limit` / `sample_limit` / `label_limit` fields that Prometheus enforces during scrape.

**Why this matters as a platform engineer**: per-SM `sampleLimit` is a polite request that requires every team to remember to set it. `enforcedSampleLimit` is a non-negotiable ceiling enforced by the Operator regardless of what teams ask for. It's the actual cardinality-bomb prevention mechanism on a multi-tenant cluster -- you set it once on the Prometheus CR, application teams cannot exceed it, and you get a `prometheus_target_scrape_pool_exceeded_target_limit_total` counter you can alert on.

**Layered with**: per-SM `sampleLimit` for sensible per-target floors (so legitimate small targets aren't accidentally capped at the platform-wide ceiling), `metric_relabel_configs` with `action: drop` for surgical removal of known offenders, and Kyverno/OPA admission policies to reject ServiceMonitors that don't set their own `sampleLimit`. Each layer closes a different gap; together they keep one team's bad day from being everyone's bad day.

---

## Part 5: PodMonitor -- When There's No Service

Continuing the analogy: ServiceMonitor delivers via the building's main reception (the Service routes the package to the correct unit). PodMonitor knocks directly on the apartment door, bypassing reception. Useful when there's no doorman.

### When to Use PodMonitor

1. **No Service exists.** Headless workloads, sidecars, batch jobs, certain stateful apps that publish metrics on a port nobody else needs to talk to.
2. **You explicitly want Pod-level metadata.** PodMonitor's discovery surfaces pod-specific labels and annotations that ServiceMonitor smudges.
3. **The metrics port is on a pod sidecar that doesn't expose itself through the main Service.** E.g., Istio envoy-proxy, app-armor sidecar, custom metric exporter sidecars.

### Anatomy

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: api-sidecar
  namespace: api
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: api                          # matches Pod labels (NOT Service)
  namespaceSelector:
    matchNames: [api]
  podMetricsEndpoints:                   # NOT `endpoints` -- different field name!
    - port: sidecar-metrics              # named container port, NOT a Service port
      interval: 30s
      path: /metrics
      relabelings: [...]                 # same shape as ServiceMonitor
      metricRelabelings: [...]
  sampleLimit: 5000
  labelLimit: 30
```

The two differences from ServiceMonitor that trip people up:

1. **`podMetricsEndpoints`** vs `endpoints` -- different field name.
2. **`port` refers to a named container port** (the `name:` in the Pod spec's `containers[].ports[]`), not a Service port name. A PodMonitor talking about port `metrics` is asking each pod's container to expose a port named `metrics`.

### Side-by-Side: ServiceMonitor vs PodMonitor

| Dimension | ServiceMonitor | PodMonitor |
|-----------|---------------|------------|
| Selects | Services | Pods directly |
| Endpoint discovery via | Endpoints / EndpointSlice resource | Pod IPs from K8s API |
| Field for endpoints list | `endpoints[]` | `podMetricsEndpoints[]` |
| Port reference | Named **Service** port | Named **container** port |
| Idiomatic for | Most apps | Sidecars, headless workloads, no-Service jobs |
| Cardinality risk | Lower (one entry per replica) | Same (one per Pod), but easier to misconfigure |
| Default in chart | Yes (most defaults are SMs) | Used for control plane components |

**Decision rule**: ServiceMonitor by default. Use PodMonitor when you have evidence you need it.

---

## Part 6: Probe and ScrapeConfig -- The Other Two CRDs

### Probe -- For External Endpoints (Black-box Monitoring)

Probe is for synthetic monitoring of external HTTP/TCP/ICMP targets. It pairs with the `prometheus-blackbox-exporter` (a separate Helm chart, not in `kube-prometheus-stack`).

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: external-apis
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  jobName: external-http-probes
  interval: 60s
  module: http_2xx                       # blackbox-exporter module name
  prober:
    url: blackbox-exporter.monitoring.svc.cluster.local:9115
  targets:
    staticConfig:
      static:
        - https://api.partner-1.com/healthz
        - https://api.partner-2.com/healthz
      labels:
        partner: external
```

The milkman analogy: he doesn't go inside the house, he just rings the doorbell from the outside and notes whether someone answered (HTTP 200), how long it took (response time), and if the doorbell itself works (TCP probe). Prometheus then alerts when the doorbell stops answering.

### ScrapeConfig -- The Escape Hatch

When ServiceMonitor / PodMonitor / Probe don't fit (rare but it happens), `ScrapeConfig` lets you declare a raw `scrape_config` block as a CR. It graduated from `v1alpha1` to `monitoring.coreos.com/v1` in Operator v0.71+ (mid-2024), so on any reasonably current chart it's stable:

```yaml
apiVersion: monitoring.coreos.com/v1           # was v1alpha1 before Operator v0.71
kind: ScrapeConfig
metadata:
  name: external-rabbitmq
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  staticConfigs:
    - targets:
      - rabbitmq.example.com:15692
      labels:
        env: prod
  scrapeInterval: 30s
  metricsPath: /metrics
```

Use ScrapeConfig when:
- You're scraping a target outside the cluster that has no K8s representation.
- You need a `consul_sd_config` / `ec2_sd_config` / other non-Kubernetes service discovery.
- You're migrating from a hand-written `prometheus.yml` and want to lift-and-shift one job at a time.

In practice, most teams use **`additionalScrapeConfigs`** on the Prometheus CRD instead -- a Secret containing raw scrape_config YAML, mounted in. Same outcome, no new CRD:

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      name: my-additional-scrapes      # Secret name
      key: prometheus-additional.yaml  # key inside the Secret
```

`ScrapeConfig` CRD vs `additionalScrapeConfigs` Secret:

| Approach | Pros | Cons |
|----------|------|------|
| `ScrapeConfig` CRD | GitOps-friendly, validated by API server, namespace-scoped, stable in v0.71+ | Newer; older Operator deployments may still be on the v1alpha1 shape |
| `additionalScrapeConfigs` Secret | Battle-tested, used in every prod cluster for years | Secret content is opaque blob; harder to review in diffs |

Pick one and stick with it. Newer clusters can lean on `ScrapeConfig`; older ones often still use the Secret pattern.

---

## Part 7: PrometheusRule -- Quick Recap from Apr 23

We covered this in depth on Apr 23, so here's the K8s lifecycle angle only:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-rules
  namespace: api
  labels:
    release: kube-prometheus-stack    # ruleSelector match
spec:
  groups:
  - name: api.recording
    interval: 30s
    rules:
    - record: api:http_request_rate_5m
      expr: sum by (service) (rate(http_requests_total{namespace="api"}[5m]))
  - name: api.alerts
    rules:
    - alert: APIDown
      expr: up{job="api"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "API instance {{ $labels.instance }} is down"
        runbook_url: https://runbooks.example.com/api-down
```

**The K8s-specific lifecycle:**

1. You apply the `PrometheusRule` CR.
2. Operator notices, regenerates the `prometheus-prometheus-rulefiles-0` ConfigMap that aggregates all matching PrometheusRules.
3. The Prometheus pod sees the ConfigMap update (mounted as a volume), and the Operator triggers a hot-reload via `/-/reload`.
4. ~30-60 seconds later the new rule is active.

When rules don't fire after `kubectl apply`, the diagnosis order is:
1. Check `kubectl get prometheusrule -A` -- is the CR there?
2. Check the rule has the right `release:` label (or whatever your `ruleSelector` matches).
3. Check `kubectl get configmap -n monitoring -l prometheus-name=kube-prometheus-stack` -- did the rules ConfigMap update?
4. Check the Prometheus UI's `Status -> Rules` page -- is the rule loaded?

(See Apr 23 Part 7 for the deep dive on `ruleSelector` and the `defaultRules.disabled` / `additionalPrometheusRulesMap` chart values.)

### The Operator's Quiet Superpower: Bad Rules Are a Non-Event

This deserves its own callout because it is **the single biggest reason to run kube-prometheus-stack instead of a hand-rolled Prometheus**. When a junior engineer pushes a `PrometheusRule` with a malformed expression -- `expr: rate(http_requests_total[5m]` (missing closing paren), or a label-mismatch math error, or a typo'd metric name that returns no series -- here is the layered failure-handling pipeline:

1. **The Operator's admission webhook** (enabled by default in the chart via `prometheusOperator.admissionWebhooks.enabled: true`) parses every `PrometheusRule` server-side at admission time and rejects ones that don't compile. The engineer sees the rejection immediately, before the CR is ever stored in etcd.

2. **If the webhook is off** (or the rule slipped past it for some reason), the Operator's reconciliation loop validates each `PrometheusRule` individually and **filters bad ones out** of the regenerated rules ConfigMap. The Operator emits a Kubernetes Warning Event explaining why; the chart-shipped `PrometheusOperatorRejectedResources` alert fires in Alertmanager. **Existing valid rules are untouched**, Prometheus never sees the bad rule, no other rules in the same `PrometheusRule` group are affected.

3. **Worst-case user-facing symptom**: the rule the engineer wanted is silently inert. The team relying on it is unprotected; ArgoCD reports `Synced` because the YAML was applied successfully. *That's the worst case*. Nothing crashes.

Compare with the pre-Operator world: hand-rolled `prometheus.rules.yaml` mounted as a ConfigMap fails **atomically** -- one bad rule in a file makes Prometheus reject the entire file at reload, killing evaluation of *every* rule in it. A typo in the `WarehouseAPIDown` rule silently disables `PaymentsAPIDown` and `OrderProcessingDown` because they're in the same file. That failure mode is how teams used to lose alerting coverage and not find out until an incident.

**The Operator turns bad-rule from a runtime crash into a non-event.** It's quiet because it works, which is also why most engineers don't realize they're getting it. When you're justifying kube-prometheus-stack to leadership, this is the property to lead with -- not the dashboards, not the chart convenience.

**The pre-merge guardrails worth running together** to keep "silently inert" off the table:

- **CI-side `promtool check rules` and `promtool test rules`** -- catches malformed PromQL at PR review time, fastest feedback, free.
- **Operator admission webhook** -- server-side safety net for whatever slipped past CI.
- **Staged sync waves dev -> staging -> prod with soak periods** (via ArgoCD ApplicationSets, see Apr 17) -- catches *semantically valid but operationally bad* rules (the `vector(1) > 0` paging-storm class) that no static check could ever flag. Only soak time catches "fires too often by design."

---

## Part 8: The Component Zoo -- The Four Pipelines That Watch Your Cluster

This is where most engineers get permanently confused. There are **four different metric pipelines** in a kube-prometheus-stack install, and they answer **different questions**. Conflating them is the #1 conceptual mistake of K8s observability.

### The Four-Layer Hierarchy

```
+----------------------------------------------------------------+
| LAYER 1: CLUSTER-WIDE (the building registry)                  |
|   kube-state-metrics  -- desired state, K8s objects             |
|     "Should this Deployment have 3 replicas?"                   |
|     "Is this PVC bound?"                                        |
|     "How many pods are in CrashLoopBackOff?"                    |
+----------------------------------------------------------------+
| LAYER 2: NODE (per-floor electrician)                          |
|   node-exporter  -- host OS metrics                             |
|     "What's this node's CPU utilization?"                       |
|     "How much swap is being used?"                              |
|     "What's the load average?"                                  |
+----------------------------------------------------------------+
| LAYER 3: POD / CONTAINER (apartment camera)                    |
|   cAdvisor (in kubelet)  -- container resource usage            |
|     "Is this container using its CPU limit RIGHT NOW?"          |
|     "Memory working set vs limit?"                              |
|     "Network bytes received this second?"                       |
+----------------------------------------------------------------+
| LAYER 4: APPLICATION (the resident's own measurements)         |
|   ServiceMonitor on app's /metrics  -- whatever you instrument  |
|     "Request rate, error ratio, queue depth, latency p99..."    |
+----------------------------------------------------------------+
```

Different questions, different sources, **all four are needed**. Let's tour each.

### Layer 1: kube-state-metrics (KSM)

**What it is**: A single Deployment that connects to the Kubernetes API server, watches every object, and emits a metric for each interesting piece of object state.

**What it answers**: *"What does Kubernetes think the world should look like?"*

Key metrics:

| Metric | What it tells you |
|--------|-------------------|
| `kube_pod_status_phase{phase="Running"}` | 1 if pod is in Running phase, 0 otherwise |
| `kube_deployment_spec_replicas` | Desired replica count |
| `kube_deployment_status_replicas_available` | Currently available replicas |
| `kube_pod_container_status_restarts_total` | Container restart count (the CrashLoopBackOff signal) |
| `kube_node_status_condition` | Node Ready/MemoryPressure/DiskPressure/PIDPressure |
| `kube_persistentvolumeclaim_status_phase` | Pending/Bound/Lost |
| `kube_horizontalpodautoscaler_status_current_replicas` | What HPA decided |

**What it does NOT tell you**: Resource usage. KSM does not measure how much CPU a pod is using. It only tells you what the API server says about the pod's *configuration and lifecycle state*. Resource usage lives in cAdvisor.

**Cardinality concerns**: pods rotate. Every deploy creates new pod names (`api-deploy-7d9f8b6c5-xyz12`), and KSM emits metrics labeled with `pod=api-deploy-7d9f8b6c5-xyz12`. Old pod metrics linger until the API server expires them. Two mitigations:

```yaml
# In kube-state-metrics values
metricLabelsAllowlist:
  - "pods=[app,version]"               # only keep these pod labels on pod metrics
  - "deployments=[app,team]"

extraArgs:
  - "--metric-allowlist=kube_pod_status_phase,kube_pod_container_status_restarts_total,kube_deployment_spec_replicas,kube_deployment_status_replicas_available,kube_node_status_condition,kube_node_status_capacity,kube_node_status_allocatable"
  - "--metric-denylist=kube_pod_init_container_.*"   # drop init container noise
```

The `--metric-allowlist` is your nuclear option for cluster-cost. It cuts KSM's output to only what you actually query. We'll revisit cardinality in Part 11.

### Layer 2: node-exporter

**What it is**: A DaemonSet -- one Pod per node -- with the `host /` mounted in. Reads `/proc`, `/sys`, and `/dev` to expose host OS metrics.

**What it answers**: *"What is this node's underlying hardware doing?"*

Key metrics:

| Metric | What it tells you |
|--------|-------------------|
| `node_cpu_seconds_total{mode="idle"}` | Cumulative CPU seconds in `idle` (use `1 - rate(...)` for utilization) |
| `node_memory_MemAvailable_bytes` | Linux's "available memory" estimate (better than `MemFree`) |
| `node_filesystem_avail_bytes` | Disk space free per filesystem |
| `node_filesystem_files_free` | Inode count free (the silent killer of "df shows 80% but I can't write a file") |
| `node_load1` / `node_load5` / `node_load15` | Load averages |
| `node_network_receive_bytes_total` | NIC bytes received |
| `node_disk_io_time_seconds_total` | Disk IO time |

The metrics are **named by the underlying Linux source** -- if you've used `top`, `iostat`, `vmstat`, you'll recognize them. The `_total` suffix marks them as counters (Apr 20).

**Common gotcha**: `node_memory_MemAvailable_bytes` is what you actually want for "memory free." `node_memory_MemFree_bytes` is the kernel's literal "this RAM is sitting idle" count, which on Linux is misleading (the kernel uses spare RAM for page cache). `MemAvailable` accounts for reclaimable cache. Always use `MemAvailable`.

### Layer 3: cAdvisor (Container Advisor)

**What it is**: A library **embedded in kubelet**. There is no separate cAdvisor pod or DaemonSet -- it lives inside the kubelet binary on every node. The chart installs a ServiceMonitor that scrapes `kubelet:10250/metrics/cadvisor`.

**What it answers**: *"How much resource is this individual container using right now?"*

Key metrics:

| Metric | What it tells you |
|--------|-------------------|
| `container_cpu_usage_seconds_total{container="api",pod="..."}` | Cumulative CPU seconds used (counter) |
| `container_memory_working_set_bytes` | **The right metric to alert on.** Closest available proxy to the cgroup value the kernel checks; also what `kubectl top pod` shows. NOT RSS (undercounts), NOT raw cache. |
| `container_memory_rss` | Resident set size, less commonly used |
| `container_network_receive_bytes_total` | Pod-level network bytes received |
| `container_fs_usage_bytes` | Container ephemeral disk usage |
| `container_cpu_cfs_throttled_periods_total` | CPU throttle events (the "my CPU limit is too low" signal) |
| `container_cpu_cfs_throttled_seconds_total` | Total throttled time |

**The most important metric here is `container_memory_working_set_bytes`** -- and the framing matters. The kernel's OOM-killer technically fires when the cgroup's `memory.usage_in_bytes` exceeds `memory.limit_in_bytes`. cAdvisor doesn't expose `usage_in_bytes` directly; what it gives you is `working_set_bytes`, computed as `usage - inactive_file` (memory the kernel can't reclaim). Working set is the **closest available proxy** to the value Kubernetes uses for OOM decisions and is what `kubectl top pod` displays.

Why not RSS? `container_memory_rss` undercounts -- it excludes tmpfs, kernel page cache pinned by the workload, and other pieces the kernel still considers "yours." Why not raw `usage`? It overcounts by including reclaimable cache. Working set splits the difference and is what you should alert on. If your container has `resources.limits.memory: 1Gi`, alert when `container_memory_working_set_bytes` approaches `1073741824`. Don't write memory alerts against `_rss` -- they'll fire late or not at all.

**Why cAdvisor is in kubelet, not separate**: kubelet already has the cgroup info -- it's the process running the containers. Adding a cAdvisor sidecar would be redundant. The downside: scraping cAdvisor means scraping kubelet, and kubelet metrics access requires bearer-token auth against the kubelet API. The chart's default ServiceMonitor handles this for you with the right `tlsConfig` and `bearerTokenFile`.

### Layer 4: Application Metrics

**What it is**: Whatever your application code emits at `/metrics`. Standard libraries: `prom-client` (Node), `prometheus_client` (Python), `client_golang` (Go), `micrometer-registry-prometheus` (Java/Spring).

**What it answers**: *"How is my application behaving from its own perspective?"*

This is where you put RED metrics (Apr 21) -- request rate, error ratio, latency histogram. It's also where you put your business KPIs: `orders_processed_total`, `payments_failed_total{reason}`, `queue_depth`, etc.

You ship a ServiceMonitor with each application Helm chart pointing at the app's `/metrics` endpoint. The Operator picks it up. You query the metrics in Grafana.

### The Confusion Matrix: cAdvisor vs KSM (THE Interview Question)

```
+---------------------------------------+---------------------------------------+
| cAdvisor (kubelet /metrics/cadvisor)  | kube-state-metrics                    |
+---------------------------------------+---------------------------------------+
| "Live security camera"                | "Building registry"                   |
| Observable: actual resource usage     | Observable: object state              |
| Source: cgroups, kernel               | Source: K8s API server                |
| Updated: continuously by kernel       | Updated: when API server objects change|
| One per node (in kubelet)             | One Deployment, cluster-wide           |
+---------------------------------------+---------------------------------------+
| Question it answers:                  | Question it answers:                  |
|   "Is my container actually using      |   "Should this pod be running?"       |
|   80% of its CPU limit RIGHT NOW?"    |   "Did Deployment status reach 3      |
|   "How much memory is the working     |   ready replicas?"                    |
|   set right now?"                     |   "Is this PVC bound?"                |
+---------------------------------------+---------------------------------------+
| Examples:                             | Examples:                             |
|   container_cpu_usage_seconds_total   |   kube_pod_status_phase               |
|   container_memory_working_set_bytes  |   kube_deployment_status_replicas_available |
|   container_network_receive_bytes_total|   kube_pod_container_status_restarts_total |
+---------------------------------------+---------------------------------------+
```

Both have `pod` and `namespace` labels. They look similar in label space and are often joined. But they answer fundamentally different questions: **runtime vs desired state**.

A worked example: "is my Deployment OK?"

- **KSM angle**: `kube_deployment_status_replicas_available{deployment="api"} == kube_deployment_spec_replicas{deployment="api"}` -- does K8s think we have all desired replicas?
- **cAdvisor angle**: `sum(rate(container_cpu_usage_seconds_total{pod=~"api-.*"}[5m]))` -- are the containers actually doing work?

You need both to know if a Deployment is *truly* OK. KSM says "K8s thinks 3/3 replicas." cAdvisor says "but the pods are using 0% CPU and 0 network." That's a Deployment that's "Ready" by K8s lifecycle but actually broken. Welcome to observability.

### And metrics-server -- The Outlier

**metrics-server is NOT in kube-prometheus-stack.** It's a separate, lightweight component that powers:

- `kubectl top nodes` / `kubectl top pods`
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA, optional)
- Kubernetes "Metrics API" (`metrics.k8s.io`)

It is in-memory only. **No history**, no PromQL, no alerting. When you ask it "what's the CPU of pod X?" it responds with a snapshot. That's it.

**Decision: do you need both metrics-server AND Prometheus?**

| If you... | You need... |
|-----------|-------------|
| Want HPA based on CPU/memory | metrics-server |
| Want HPA based on custom metrics (request rate, queue depth) | metrics-server + Prometheus + prometheus-adapter |
| Want history, alerts, dashboards | Prometheus |
| Want `kubectl top` to work | metrics-server |
| Want everything | Both -- this is the production answer |

Yes, both. They overlap at the edges (both can tell you a pod's current CPU) but the use cases barely intersect. Install metrics-server too -- on EKS it's an EKS Add-on (`aws eks create-addon --addon-name metrics-server`).

---

## Part 9: EKS / Managed-K8s Control Plane Gotcha

On managed Kubernetes (EKS, GKE, AKS), the **control plane components are inside AWS/Google/Microsoft's VPC, not yours**. You cannot scrape `kube-controller-manager`, `kube-scheduler`, or sometimes `kube-proxy` because they're not exposed to your cluster.

The chart's defaults assume self-hosted K8s and try to scrape them anyway. Result: failing ServiceMonitors, failing alerts, noisy `KubeControllerManagerDown`/`KubeSchedulerDown` firings on day one.

### The Fix: Disable Them

```yaml
# values.yaml for EKS
kubeControllerManager:
  enabled: false                       # not exposed on EKS
kubeScheduler:
  enabled: false                       # not exposed on EKS
kubeProxy:
  enabled: false                       # depends on EKS network plugin -- check before disabling

# kube-apiserver IS reachable on EKS -- leave enabled
kubeApiServer:
  enabled: true

# kubelet IS reachable on every node -- leave enabled
kubelet:
  enabled: true
  serviceMonitor:
    cAdvisor: true
    probes: true
```

**Cluster-by-cluster guidance:**

| Provider | Disable | Keep |
|----------|---------|------|
| EKS | `kubeControllerManager`, `kubeScheduler`, often `kubeEtcd`, sometimes `kubeProxy` | `kubeApiServer`, `kubelet`, `coreDns` |
| GKE | Same as EKS | Same |
| AKS | Same as EKS | Same |
| kubeadm self-hosted | None -- everything works | All |
| KIND / k3d / minikube | `kubeEtcd` may need toggle | Most works |

### EKS's Newer Option: Control Plane Metrics via API Server Proxy

AWS has been iterating on exposing control-plane metrics through an API server proxy under the `metrics.eks.amazonaws.com` API group (kube-controller-manager, kube-scheduler, etc.). The setup involves an opt-in on the EKS cluster, a ClusterRole granting the Prometheus ServiceAccount `get` on the proxy paths, and a ServiceMonitor that targets the `kubernetes` Service in the `default` namespace with bearer-token auth.

**The exact paths and CRD shape have shifted between EKS versions** -- rather than ship a snippet that goes stale, refer to the AWS blog "Amazon EKS enhances Kubernetes Control Plane Observability" for the current setup. Verify the path, RBAC, and apiVersion against your specific EKS version before copy-pasting any example you find online.

In practice, most teams just disable the failing scrapes and live without controller-manager / scheduler metrics, because:
1. The control plane is AWS's responsibility -- they alert on it.
2. The metrics are only marginally useful for app teams.
3. The setup is non-trivial and the API has changed across EKS releases.

If you do want them, **Amazon Managed Prometheus (AMP) with the EKS managed scraper** is the cleanest answer -- AWS handles the auth, path, and version-skew problem for you.

---

## Part 10: Default ServiceMonitors and Default Dashboards

### Default ServiceMonitors That Ship With the Chart

| Target | What it scrapes |
|--------|-----------------|
| `node-exporter` | Per-node host metrics |
| `kube-state-metrics` | Object-state metrics |
| `kubelet` | Kubelet runtime metrics (volumes, pod lifecycle) |
| `kubelet/cadvisor` | Container resource usage |
| `kubelet/probes` | Probe metrics (livenessProbe, readinessProbe) |
| `kube-apiserver` | API server metrics |
| `kube-controller-manager` | (disabled on EKS) |
| `kube-scheduler` | (disabled on EKS) |
| `kube-proxy` | (sometimes disabled on EKS) |
| `coredns` | Cluster DNS metrics |
| `kube-etcd` | etcd metrics (only on self-hosted) |
| `prometheus` | Prometheus self-monitoring |
| `alertmanager` | Alertmanager self-monitoring |
| `grafana` | Grafana metrics |
| `prometheus-operator` | Operator metrics |

Combined, these are 15-20 ServiceMonitors on day one. You inherit a comprehensive baseline -- don't re-create what's already there.

### Default Grafana Dashboards (the gold)

The chart ships ~30 dashboards via `grafana.sidecar.dashboards.enabled: true` and ConfigMap discovery. The standard set includes:

- **Kubernetes / Compute Resources / Cluster** -- cluster-wide CPU/memory utilization
- **Kubernetes / Compute Resources / Namespace (Pods)** -- per-namespace breakdown
- **Kubernetes / Compute Resources / Workload** -- per-Deployment/StatefulSet view
- **Kubernetes / Compute Resources / Node (Pods)** -- per-node pod detail
- **Kubernetes / Networking / Cluster** -- network throughput
- **Node Exporter / Nodes** -- the canonical node-exporter dashboard (CPU/mem/disk/net)
- **Node Exporter / USE Method / Node** -- USE method (Apr 21) per node
- **Prometheus / Overview** -- Prometheus health
- **Alertmanager / Overview** -- Alertmanager health

**Pattern**: use these as your starting point. Don't rebuild them. When you need a custom view, **clone** an existing dashboard, modify it, and save it as a new ConfigMap with `grafana_dashboard: "1"` label so Grafana's sidecar picks it up:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"               # the magic label
data:
  my-app.json: |
    {
      "dashboard": { ... }
    }
```

The Grafana sidecar (`k8s-sidecar` in the values) watches every namespace for ConfigMaps with this label and provisions them into Grafana automatically. Same trick works for datasources with `grafana_datasource: "1"` and alerts.

---

## Part 11: Cardinality on Kubernetes -- Why Pods Are the Enemy

In Apr 21 we did the cardinality math: total time series = product of label-value cardinalities. On Kubernetes the worst offender is the `pod` label, because **pods rotate every deploy**. Every deploy creates new pod names, KSM emits new series for them, old ones linger until the API server forgets them.

A 50-deploy/day cluster with 100 pods can generate millions of pod-keyed series in a month. KSM is ground zero -- it has dozens of metrics keyed by pod.

### Mitigations (in order of effectiveness)

**1. Drop the worst KSM metrics with `--metric-allowlist` / `--metric-denylist`:**

```yaml
kube-state-metrics:
  extraArgs:
    - "--metric-denylist=kube_pod_init_container_.*,kube_pod_container_resource_.*"
```

Init container metrics in particular are useless and high-cardinality. Drop them.

**2. Drop high-cardinality labels with KSM's allowlist:**

```yaml
kube-state-metrics:
  metricLabelsAllowlist:
    - "pods=[app,version,team]"          # only keep these labels on pod metrics
    - "namespaces=[team,env]"
```

By default KSM exposes EVERY pod label as a metric label. You usually don't want that.

**3. metricRelabelings on the ServiceMonitor:**

```yaml
endpoints:
  - port: metrics
    metricRelabelings:
      # Drop completely
      - sourceLabels: [__name__]
        regex: 'kube_pod_init_container_.*'
        action: drop
      # Keep but strip a label
      - regex: 'pod_template_hash'
        action: labeldrop
      # Replace a high-cardinality value with a fixed one
      - sourceLabels: [endpoint]
        regex: '/api/v1/users/[0-9]+'
        action: replace
        targetLabel: endpoint
        replacement: '/api/v1/users/{id}'
```

`labeldrop` and value-replacement are your scalpels for cardinality surgery.

**4. `sampleLimit` on every ServiceMonitor**:

```yaml
spec:
  sampleLimit: 10000     # drop the entire scrape if it returns > 10K samples
```

Belt-and-suspenders. If an app accidentally explodes its metric output, the scrape gets dropped wholesale rather than blowing up Prometheus.

**5. Lower scrape frequency for chatty endpoints:**

```yaml
endpoints:
  - port: metrics
    interval: 60s        # default 30s -- halve the storage cost
```

Trade resolution for cost. Most metrics are fine at 60s. Burn-rate alerts (Apr 23) need 30s or finer.

### Cardinality Math, Recapped

A useful rule of thumb on K8s: every metric labeled by `pod` will have **~3-5x its "active" cardinality** at any moment because of:

- Currently running pods (~1x)
- Recently terminated pods still in the API (~1-2x)
- KSM's lingering series (~1x more)

Plan accordingly. A `kube_pod_status_phase` metric on a 100-pod cluster doesn't have 100 series; it has 300-500.

---

## Part 12: Cluster Sizing -- Rule-of-Thumb Resourcing

| Cluster size | Pods | Prometheus RAM | Prometheus disk | Retention | Pattern |
|--------------|------|----------------|-----------------|-----------|---------|
| Small | <500 | 4 GB | 50 GB | 15 days | Default chart values; single replica |
| Medium | <2000 | 8-16 GB | 200 GB | 15-30 days | HA pair (2 replicas); consider remote_write to Thanos/Mimir for >30d |
| Large | <10,000 | 32-64 GB | 500 GB-1 TB | 7 days local + remote_write | Sharded Prometheus by namespace OR `hashmod` sharding via additionalScrapeConfigs |
| Huge | >10,000 | Multiple Prometheus instances | -- | Local only | Federation hierarchy or full Mimir/Cortex/Thanos cluster |

The honest production answer is: **for any cluster you'd hesitate to lose, plan remote_write from day one**. Disk is cheap; rebuilding three weeks of metrics history is not.

### Storage Recommendations

```yaml
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 180GB                # leave 10-20% headroom under the volume size
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3         # AWS general-purpose SSD; faster than gp2
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 200Gi
    resources:
      requests:
        memory: 8Gi
        cpu: 1
      limits:
        memory: 12Gi                    # NO CPU LIMIT -- see gotchas
```

Note: **never set a CPU limit on Prometheus**. Memory limit yes (it's the OOM-kill ceiling). CPU limit no -- the kernel CFS scheduler throttles, scrape latency spikes, queries time out. Set a CPU **request** for scheduling, not a limit.

### Sharding -- `prometheusSpec.shards` for Very Large Clusters

When a single Prometheus pod can no longer scrape all your targets within the scrape interval, the Operator ships a built-in horizontal-sharding mode. Set:

```yaml
prometheus:
  prometheusSpec:
    shards: 4              # creates 4 separate StatefulSets
    replicas: 2            # each shard runs HA -- 8 pods total
```

Behind the scenes, the Operator creates N StatefulSets (`prometheus-<name>-0`, `prometheus-<name>-shard-1-0`, etc.) and injects a `hashmod` relabel into each shard's scrape configs so each target is owned by exactly one shard:

```yaml
relabel_configs:
  - source_labels: [__address__]
    modulus: 4
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: ${SHARD}        # 0, 1, 2, or 3 per shard
    action: keep
```

**Implications you must accept**:

1. **Queries go cross-shard.** Each shard only has 1/N of the data. Use Thanos Querier, Mimir, or the built-in Prometheus federation/Agent setup to fan out queries.
2. **Alerts evaluate per-shard.** If your alert needs data from multiple shards (e.g., `sum across services`), it has to run in a tier that sees all shards (Thanos Ruler, Mimir Ruler).
3. **Adding/removing shards rebalances every target.** Changing `shards: 4` to `shards: 5` rehashes; expect a brief storm of "new" series and "missing" series.

For most clusters this isn't needed -- vertical scaling and remote_write to Thanos/Mimir handle "Large" comfortably. Reach for `shards` only when scrape overlap warns are firing and you've already tuned scrape intervals + cardinality.

### Long-Term Storage: Thanos Sidecar vs Mimir/Cortex remote_write

Two architectures, two sets of trade-offs:

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Thanos sidecar** | A `thanos-sidecar` container in each Prometheus pod ships completed 2h TSDB blocks to S3 (or any object store). Thanos Querier federates across sidecar + S3 store-gateway. | No extra hot-path component; Prometheus stays canonical; cheap object storage | Querying recent (un-shipped) data still hits Prometheus directly; sidecar must be on same disk as Prometheus |
| **Mimir / Cortex remote_write** | Prometheus pushes every sample to Mimir via `remote_write`. Mimir owns durability, retention, and HA dedup. | Single global query plane; horizontal scaling on the storage side; built-in HA dedup; tenant isolation | Hot-path dependency (remote_write failures back-pressure Prometheus); higher infra complexity |

**Industry direction (2025-2026)**: most large orgs are picking **Mimir or Grafana Cloud Metrics** (which is hosted Mimir) over Thanos for new builds. Thanos is still excellent and many existing deployments stay on it; Mimir wins on operational simplicity and the global query plane. Both are valid; pick based on whether you want Prometheus to stay "the source of truth" (Thanos) or hand off durability to a dedicated tier (Mimir).

---

## Part 13: Production Gotchas (The Top 15)

The full operational pain list. Each entry is a real failure mode you will hit.

1. **The `release:` label mismatch.** ServiceMonitor created somewhere, never picked up. Default `serviceMonitorSelector` filters on `release: kube-prometheus-stack`. **Fix**: either label every foreign SM with the release label, or set selectors to `{}` plus `serviceMonitorSelectorNilUsesHelmValues: false`.

2. **CRDs not upgraded by `helm upgrade`.** Helm v3 explicitly does not touch CRDs. New chart fields silently fail. **Fix**: `kubectl apply --server-side -f charts/.../crds/` BEFORE `helm upgrade`.

3. **EKS control plane scrape failures.** Default `kubeControllerManager.enabled: true` and `kubeScheduler.enabled: true` produce `KubeControllerManagerDown` / `KubeSchedulerDown` alerts on EKS day one. **Fix**: disable both in values for any managed-K8s cluster.

4. **`serviceMonitorNamespaceSelector` too restrictive.** SM's labels match Prometheus's `serviceMonitorSelector`, but the SM's namespace doesn't match `serviceMonitorNamespaceSelector`. Silent skip. **Fix**: set `serviceMonitorNamespaceSelector: {}` for "all namespaces" or add the namespace label.

5. **`endpoints[].port` references the container port number, not the Service's named port.** ServiceMonitor with `port: 9090` finds nothing because the Service exposes port 9090 under the name `metrics`. **Fix**: always reference Service ports by name (`port: metrics`).

6. **PodMonitor `port` references a Service port name.** PodMonitor's `podMetricsEndpoints[].port` is a *container* port name, not Service. People copy-paste a ServiceMonitor and convert it to PodMonitor without updating. **Fix**: read Pod spec's `containers[].ports[].name`.

7. **kube-state-metrics unbounded label cardinality.** Default KSM exports every Pod label as a metric label. A team adding a high-cardinality label like `git-commit-sha` to their Pods explodes Prometheus storage. **Fix**: set `metricLabelsAllowlist` to a small fixed list of useful labels.

8. **Memory limit set on Prometheus, no swap, OOM-kill on query spikes.** A Grafana dashboard with a 30-day range query spikes RAM by 4-8 GB. **Fix**: size memory limit ~1.5x of expected steady state, never run with limit close to request.

9. **CPU limit set on Prometheus, scrape lag.** CFS throttling kicks in, scrape latencies climb, alerts based on `up` start firing. **Fix**: never set CPU limit on Prometheus; set request for scheduling.

10. **Default Grafana admin password unrotated, exposed via Ingress.** Chart default is `prom-operator` until you change it. **Fix**:
    ```yaml
    grafana:
      admin:
        existingSecret: grafana-admin-credentials
    ```

11. **Sidecar pattern: `grafana.sidecar.dashboards.enabled: true` not set.** You drop dashboard ConfigMaps with the magic label, nothing shows up in Grafana. **Fix**: enable the sidecar in chart values.

12. **`sampleLimit` not set, runaway scrape destroys Prometheus.** A misbehaving exporter emits 10M time series in one scrape. Prometheus chews through RAM and OOMs. **Fix**: set `sampleLimit: 10000` (or appropriate floor) on every SM.

13. **AZ-pinned PVCs after a zone failure.** A StatefulSet's `volumeClaimTemplates` give each replica its own PVC, and on AWS each EBS volume is pinned to one AZ. If `prometheus-0`'s AZ goes down, the pod can't reschedule into another AZ -- its PVC is stranded. **Fix**: spread replicas across AZs with `topologySpreadConstraints` (so the surviving replica keeps serving), accept that the down-AZ replica stays down until the AZ recovers, and lean on remote_write to Thanos/Mimir for any metric you can't afford to lose during a zonal outage.

14. **Prometheus Operator namespace permission scope.** By default, an Operator deployed via the chart has cluster-wide RBAC. Hand-rolled installs sometimes scope it to one namespace, then wonder why `serviceMonitorNamespaceSelector: {}` doesn't pick up SMs in other namespaces. **Fix**: check the Operator's ClusterRoleBinding -- it should have permission to list SMs/PMs/Probes/Rules cluster-wide.

15. **ArgoCD sync race on fresh install.** ArgoCD applies the Application manifest, the Operator deployment starts before its CRDs exist, fails admission. **Fix**: use `argocd.argoproj.io/sync-wave` annotations:
    - `-2` for CRDs
    - `-1` for the Operator Deployment + RBAC
    - `0` for everything else (Prometheus, Alertmanager, default rules, default SMs)

    Plus `ServerSideApply=true` in the Application's `syncPolicy.syncOptions` (see Apr 16) to handle the 256 KB CRD annotation limit.

16. **Default rules' `KubeMemoryOvercommit` / `KubeCPUOvercommit` always firing on dev clusters.** These rules check that `sum(requests) <= sum(node capacity)`. On a dev cluster running 50% over capacity (which is fine for dev), they fire constantly. **Fix**: `defaultRules.disabled: [KubeMemoryOvercommit, KubeCPUOvercommit]`.

17. **Helm release name embedded in everything.** If you `helm install kube-prometheus-stack` once, then redeploy as `helm install monitoring`, the new release's Prometheus has `serviceMonitorSelector: {release: monitoring}` and won't pick up the old SMs labeled `release: kube-prometheus-stack`. **Fix**: pick a release name on day one and never change it; if you must, do a full migration of all SM labels.

18. **`additionalScrapeConfigs` Secret typo silently broken.** The Secret key name must match the chart value's `key:` field. A typo means Prometheus's config-reload still succeeds (the Secret is just not mounted), and your scrape configs are silently absent. **Fix**: check `kubectl exec prometheus-pod -- cat /etc/prometheus/config_out/prometheus.env.yaml` to see what the pod actually got.

---

## Part 14: Decision Frameworks

### Framework 1: ServiceMonitor vs PodMonitor vs additionalScrapeConfigs

| Situation | Use |
|-----------|-----|
| App fronted by a Service | **ServiceMonitor** |
| Headless workload (no Service) | **PodMonitor** |
| Sidecar that exposes its own port | **PodMonitor** |
| External target (outside cluster) | **additionalScrapeConfigs** or `ScrapeConfig` CRD |
| Synthetic HTTP probe | **Probe** + blackbox-exporter |
| You're migrating from raw `prometheus.yml` | **additionalScrapeConfigs** initially, then refactor to SMs |

### Framework 2: kube-prometheus-stack vs Raw Operator vs Individual Components

| Use case | Recommended install |
|----------|---------------------|
| New cluster, want the full stack | **`kube-prometheus-stack`** Helm chart |
| Existing Prometheus, want to add Operator | **`prometheus-operator`** chart alone |
| Existing Operator, want to add HA Prometheus to existing one | Just create another `Prometheus` CR |
| Highly customized stack | Individual subcharts: `prometheus-operator`, `prometheus`, `alertmanager`, `grafana`, etc. |
| Reading reference for "what's canonical" | **`kube-prometheus`** (jsonnet, prometheus-operator org) |

### Framework 3: metrics-server vs Prometheus

| Need | Tool |
|------|------|
| `kubectl top` to work | metrics-server |
| HPA on CPU/memory | metrics-server |
| HPA on custom metrics (request rate, queue depth) | metrics-server + Prometheus + prometheus-adapter |
| Alerting on resource trends | Prometheus |
| Dashboards with history | Prometheus + Grafana |
| Resource snapshot in `kubectl describe pod` | metrics-server (via Metrics API) |

**Production answer**: install both. They overlap but neither replaces the other.

### Framework 4: cAdvisor vs KSM vs node-exporter vs Application Metrics

| Question | Look at |
|----------|---------|
| Is this container actually using CPU? | cAdvisor: `container_cpu_usage_seconds_total` |
| Is the container near its memory limit? | cAdvisor: `container_memory_working_set_bytes` |
| Did this Deployment achieve its desired replica count? | KSM: `kube_deployment_status_replicas_available` |
| Is this pod stuck in CrashLoopBackOff? | KSM: `kube_pod_container_status_restarts_total` |
| Is the node running out of disk? | node-exporter: `node_filesystem_avail_bytes` |
| What's the node's load average? | node-exporter: `node_load1` |
| What's my service's request error rate? | Application: your `http_requests_total` |
| What's my service's p99 latency? | Application: your `http_request_duration_seconds_bucket` |

### Framework 5: Helm-Deploy vs ArgoCD-Deploy

| Approach | When |
|----------|------|
| `helm upgrade --install` from a CI pipeline | Small teams, single environment, fast iteration |
| ArgoCD with sync waves | Multi-cluster, GitOps requirement, audit trail needed |

ArgoCD requires:
- **CRDs as sync-wave -2** with `Replace=true` and `ServerSideApply=true`
- **Operator Deployment as -1**
- **Default rules / SMs / PrometheusSpec as 0**
- **`SkipDryRunOnMissingResource=true`** to avoid validation failures during initial sync

(Cross-reference Apr 16's ArgoCD doc for the full syncPolicy.)

---

## Part 15: Production Helm Values (Full Example)

The complete `values.yaml` for a production EKS cluster, annotated:

```yaml
# kube-prometheus-stack values.yaml -- production EKS
crds:
  enabled: true                                # ship CRDs

defaultRules:
  create: true
  disabled:
    - KubeMemoryOvercommit                     # too noisy on bursty dev clusters
    - KubeCPUOvercommit
    - KubeSchedulerDown                        # disabled below
    - KubeControllerManagerDown                # disabled below

global:
  rbac:
    create: true

# ----- ALERTMANAGER -----
alertmanager:
  enabled: true
  alertmanagerSpec:
    replicas: 3                                # HA gossip cluster
    retention: 120h
    externalUrl: https://alertmanager.example.com
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 10Gi
    alertmanagerConfigSelector:
      matchLabels:
        alertmanagerConfig: team-managed
    alertmanagerConfigMatcherStrategy:
      type: OnNamespace                        # Apr 23 multi-tenancy
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi                          # Note: no CPU limit
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: default-null
      group_by: [alertname, cluster, service]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
    receivers:
      - name: default-null

# ----- PROMETHEUS -----
prometheus:
  enabled: true
  prometheusSpec:
    replicas: 2                                # HA pair
    externalLabels:
      cluster: prod-us-east-1
      region: us-east-1                        # Apr 23 multi-cluster dedup
    retention: 30d
    retentionSize: 180GB

    # SELECTOR CHAIN -- the most important block
    serviceMonitorSelector: {}                 # match all SMs
    serviceMonitorNamespaceSelector: {}        # in all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelector: {}
    ruleNamespaceSelector: {}
    ruleSelectorNilUsesHelmValues: false
    probeSelector: {}
    probeNamespaceSelector: {}
    probeSelectorNilUsesHelmValues: false

    # STORAGE
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 200Gi

    # RESOURCES (note: NO CPU LIMIT)
    resources:
      requests:
        memory: 8Gi
        cpu: 1
      limits:
        memory: 12Gi

    # REMOTE WRITE TO THANOS / MIMIR (commented; enable when scaling)
    # remoteWrite:
    #   - url: http://mimir-distributor.observability.svc/api/v1/push
    #     headers:
    #       X-Scope-OrgID: prod-us-east-1
    #     queueConfig:
    #       capacity: 10000
    #       maxSamplesPerSend: 2000

    # ESCAPE HATCH for non-K8s scrape targets
    additionalScrapeConfigs:
      name: prometheus-additional-scrape-configs
      key: scrape-configs.yaml

# ----- PROMETHEUS OPERATOR -----
prometheusOperator:
  enabled: true
  admissionWebhooks:
    enabled: true                              # validates SMs/PMs/Rules at admission
    failurePolicy: Fail
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi

# ----- GRAFANA -----
grafana:
  enabled: true
  defaultDashboardsEnabled: true               # ship K8s dashboards
  admin:
    existingSecret: grafana-admin-credentials  # don't use default password
    userKey: admin-user
    passwordKey: admin-password
  sidecar:
    dashboards:
      enabled: true                            # ConfigMap discovery
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: ALL                     # any namespace
      provider:
        allowUiUpdates: true                   # allow editing in UI; persisted via CM
    datasources:
      enabled: true
      label: grafana_datasource
      labelValue: "1"
  ingress:
    enabled: true
    ingressClassName: alb
    hosts: [grafana.example.com]
    tls:
      - hosts: [grafana.example.com]
        secretName: grafana-tls
  resources:
    requests:
      memory: 512Mi
      cpu: 200m
    limits:
      memory: 1Gi

# ----- NODE EXPORTER -----
nodeExporter:
  enabled: true

# ----- KUBE-STATE-METRICS -----
kube-state-metrics:
  enabled: true
  metricLabelsAllowlist:
    - "pods=[app,version,team]"
    - "deployments=[app,team]"
  extraArgs:
    - "--metric-denylist=kube_pod_init_container_.*"

# ----- CONTROL PLANE COMPONENTS (EKS-SPECIFIC) -----
kubeApiServer:
  enabled: true                                # exposed on EKS
kubeControllerManager:
  enabled: false                               # NOT exposed on EKS
kubeScheduler:
  enabled: false                               # NOT exposed on EKS
kubeProxy:
  enabled: false                               # depends on EKS network plugin
kubeEtcd:
  enabled: false                               # not exposed on EKS
kubelet:
  enabled: true
  serviceMonitor:
    cAdvisor: true                             # scrape /metrics/cadvisor
    probes: true                               # scrape /metrics/probes
coreDns:
  enabled: true
```

### A Sample Application ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-go-app
  namespace: my-app
  labels:
    release: kube-prometheus-stack             # match selector
    team: platform
spec:
  selector:
    matchLabels:
      app: my-go-app
  endpoints:
    - port: metrics                            # named Service port
      interval: 30s
      path: /metrics
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_version]
          targetLabel: app_version
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: 'go_(gc|memstats)_.*'         # drop Go runtime noise
          action: drop
  sampleLimit: 5000
  jobLabel: app
```

### A Sample Sidecar PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-envoy-sidecar
  namespace: my-app
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: my-go-app
  podMetricsEndpoints:
    - port: envoy-metrics                      # named container port
      interval: 30s
      path: /stats/prometheus
  sampleLimit: 5000
```

### A Sample PrometheusRule (Burn-Rate Alert from Apr 23)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-go-app-slo
  namespace: my-app
  labels:
    release: kube-prometheus-stack
    team: platform
spec:
  groups:
  - name: my-go-app.recording
    interval: 30s
    rules:
    - record: my_go_app:error_ratio_5m
      expr: |
        sum(rate(http_requests_total{app="my-go-app",status=~"5.."}[5m]))
        / sum(rate(http_requests_total{app="my-go-app"}[5m]))
    - record: my_go_app:error_ratio_1h
      expr: |
        sum(rate(http_requests_total{app="my-go-app",status=~"5.."}[1h]))
        / sum(rate(http_requests_total{app="my-go-app"}[1h]))
    - record: my_go_app:request_rate_5m
      expr: sum(rate(http_requests_total{app="my-go-app"}[5m]))

  - name: my-go-app.alerts
    rules:
    - alert: MyGoAppFastBurn
      expr: |
        (
          my_go_app:error_ratio_5m / 0.001 > 14.4
          and
          my_go_app:error_ratio_1h / 0.001 > 14.4
          and
          my_go_app:request_rate_5m > 0.5      # low-traffic gate (Apr 23)
        )
      for: 2m
      labels:
        severity: critical
        team: platform
        slo_type: fast_burn
      annotations:
        summary: "my-go-app burning SLO budget at >14.4x"
        runbook_url: https://runbooks.example.com/my-go-app-fast-burn
```

---

## Part 16: ArgoCD Application for the Whole Stack

The GitOps pattern for `kube-prometheus-stack`. The trick is sync waves.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: platform
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 58.0.0
    helm:
      releaseName: kube-prometheus-stack
      valueFiles:
        - $values/clusters/prod-us-east-1/kube-prometheus-stack-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true                   # Apr 16 -- 256 KB CRD annotation limit
      - SkipDryRunOnMissingResource=true       # CRDs may not exist on first sync
      - Replace=true                           # for CRDs that exceed annotation size
    retry:
      limit: 5
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 5m
```

### The CRD Bootstrap Problem

On a fresh cluster, `kube-prometheus-stack`'s ArgoCD Application will fail the first sync because:

1. The chart contains both CRDs AND CRs that use those CRDs.
2. ArgoCD's dry-run validates everything before applying.
3. `Prometheus` CR validation fails because the `Prometheus` CRD doesn't exist yet.

**The fix**: split into two Applications, with sync waves:

```yaml
# App 1: CRDs only
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack-crds
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  source:
    repoURL: https://github.com/prometheus-community/helm-charts.git
    targetRevision: kube-prometheus-stack-58.0.0
    path: charts/kube-prometheus-stack/charts/crds/crds
    directory:
      recurse: true
  syncPolicy:
    syncOptions:
      - ServerSideApply=true
      - Replace=true                           # CRDs >256 KB

---
# App 2: the rest of the chart (sync wave 0, after CRDs)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  source:
    chart: kube-prometheus-stack
    helm:
      skipCrds: true                           # CRDs handled by App 1
      releaseName: kube-prometheus-stack
```

The `helm.skipCrds: true` is critical -- it tells ArgoCD's Helm renderer not to include the chart's CRD templates, since App 1 handles them. Otherwise you get version conflicts.

---

## Part 17: The 10 Commandments of the K8s Metrics Stack

Distilled from years of operational pain. Match Apr 23's commandments format.

1. **Thou shalt write ServiceMonitors, not edit `prometheus.yml`.** The Operator pattern is the entire reason the stack exists. If you find yourself writing raw scrape configs by hand, stop and ask whether a CRD fits.

2. **Thou shalt label thy ServiceMonitors with `release: <chart-name>`** (or set selectors to `{}` plus `*NilUsesHelmValues: false` once and for all). 90% of "my SM isn't being scraped" tickets are this.

3. **Thou shalt apply CRDs separately before `helm upgrade`.** Helm v3 does not touch CRDs. `kubectl apply --server-side` the new CRDs first, every single upgrade.

4. **Thou shalt disable `kubeControllerManager`, `kubeScheduler` (and often `kubeProxy`/`kubeEtcd`) on managed K8s.** They're not exposed on EKS/GKE/AKS. Default-on means default-broken on those clusters.

5. **Thou shalt set `sampleLimit` on every ServiceMonitor.** The cardinality safety belt nobody regrets and many wish they had set the day before the outage.

6. **Thou shalt know the four pipelines: cAdvisor (live container resource), KSM (object state), node-exporter (host OS), application metrics (your code).** If you're querying the wrong one for the question you have, you'll get plausible-looking nonsense.

7. **Thou shalt not set a CPU limit on Prometheus.** Memory limit yes. CPU limit no -- CFS throttling murders scrape and query latency.

8. **Thou shalt run BOTH Prometheus and metrics-server.** They serve different consumers (Prometheus for history/alerts, metrics-server for HPA/`kubectl top`). Neither replaces the other.

9. **Thou shalt prefer ServiceMonitor over PodMonitor unless you specifically can't.** ServiceMonitor is the idiomatic K8s answer. Go to PodMonitor only for sidecars, headless workloads, or when no Service exists.

10. **Thou shalt alert on `container_memory_working_set_bytes`, not `_rss`.** Working set is the closest proxy to what Kubernetes actually checks for OOM-kill decisions and what `kubectl top` shows. RSS undercounts and your alerts will fire late.

---

## Key Takeaways

- **The Operator inverts the relationship**: you describe *what* to monitor as CRDs, the Operator generates the actual `prometheus.yml`. Stop thinking in scrape configs; start thinking in ServiceMonitors and PrometheusRules.

- **`kube-prometheus-stack` is the all-in-one Helm chart**: Operator + Prometheus + Alertmanager + Grafana + node-exporter + KSM + ~20 default ServiceMonitors + ~30 default rules + ~30 default dashboards. It is the right starting point for 95% of clusters.

- **The selector chain is four links**: Prometheus -> ServiceMonitor (label) -> Service (label) -> Pod. Default chart selectors filter on `release: kube-prometheus-stack`, which is the #1 source of "why isn't this scraping" tickets. Set selectors to `{}` plus `*NilUsesHelmValues: false` for sanity.

- **The four pipelines, memorized**: KSM (object state, "what does K8s think"), node-exporter (host OS), cAdvisor (container runtime, embedded in kubelet), application metrics (your code). Different questions, different sources.

- **cAdvisor != KSM, ever**. cAdvisor is the security camera (live runtime). KSM is the building registry (desired state). They have similar labels but answer fundamentally different questions.

- **`container_memory_working_set_bytes`** is the right metric for memory alerts -- the closest proxy to what Kubernetes checks for OOM and what `kubectl top` shows. Burn this into long-term memory. Most production memory alerts written against `container_memory_rss` are subtly wrong (they fire late or not at all).

- **CRDs are not upgraded by `helm upgrade`**. Always `kubectl apply --server-side` the new CRDs from the chart's `crds/` directory before the chart upgrade. ArgoCD installs split CRDs into a separate Application with `sync-wave: -2`.

- **EKS-specific defaults will fail** unless you disable `kubeControllerManager`, `kubeScheduler`, often `kubeProxy` and `kubeEtcd`. The control plane is AWS's responsibility, not yours.

- **metrics-server is separate from Prometheus** and serves different consumers (HPA, `kubectl top`). Run both.

- **Cardinality on K8s is mostly a `pod`-label problem**. KSM is the worst offender; `--metric-allowlist`, `metricLabelsAllowlist`, ServiceMonitor `metricRelabelings`, and `sampleLimit` are your scalpels.

- **Production ArgoCD bootstrap pattern**: CRDs at sync-wave -2 (with `ServerSideApply=true` for the 256 KB annotation limit), Operator at -1, everything else at 0. `helm.skipCrds: true` on the main chart Application.

---

## Further Reading

- Prometheus Operator design: https://prometheus-operator.dev/docs/getting-started/design/
- kube-prometheus-stack chart README: https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md
- ServiceMonitor / PodMonitor field reference: https://prometheus-operator.dev/docs/api-reference/api/
- Operator Troubleshooting: https://prometheus-operator.dev/docs/platform/troubleshooting/
- kube-state-metrics docs: https://github.com/kubernetes/kube-state-metrics
- Making Sense of Kubernetes Metrics (Sookocheff): https://sookocheff.com/post/kubernetes/making-sense-of-kubernetes-metrics/
- AWS EKS Control Plane Metrics: https://aws.amazon.com/blogs/containers/amazon-eks-enhances-kubernetes-control-plane-observability/
- Repo: `2026/april/prometheus/2026-04-20-prometheus-fundamentals.md` (scrape model, kubernetes_sd_configs, relabel_configs)
- Repo: `2026/april/prometheus/2026-04-21-promql-deep-dive.md` (PromQL for cardinality math, RED/USE)
- Repo: `2026/april/prometheus/2026-04-23-alertmanager-alerting-strategy.md` (PrometheusRule CRD, ruleSelector, defaultRules.disabled)
- Repo: `2026/april/grafana/2026-04-24-grafana-fundamentals.md` (dashboard sidecar, ConfigMap discovery)
- Repo: `2026/april/argocd/2026-04-16-argocd-fundamentals.md` (sync waves, ServerSideApply, 256 KB annotation limit)
