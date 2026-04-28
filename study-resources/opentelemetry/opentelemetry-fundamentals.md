# OpenTelemetry Fundamentals — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-27
**Difficulty:** Intermediate (assumes Prometheus / Grafana / Kubernetes fluency)

## Overview

This plan introduces OpenTelemetry (OTel) as the vendor-neutral standard for collecting traces, metrics, and logs across cloud-native systems. After two focused hours you will understand the API/SDK/Collector separation, OTLP as the wire protocol, the receivers->processors->exporters pipeline model, agent-vs-gateway deployment patterns on Kubernetes, the Operator's auto-instrumentation injection mechanic, semantic conventions for `service.*` and `k8s.*` attributes, and how Grafana Alloy fits into the OTel Collector ecosystem you already touch via kube-prometheus-stack.

Resources are sequenced so you build the conceptual model first (signals -> components -> protocol), then move into the Collector pipeline, then into Kubernetes-specific deployment patterns, and finally into the strategic "why OTel over a vendor agent" framing.

## Resources

### 1. OpenTelemetry — Observability Primer + Signals Overview ⏱️ 15 min
- **URL:** https://opentelemetry.io/docs/concepts/observability-primer/ and https://opentelemetry.io/docs/concepts/signals/
- **Type:** Official Docs
- **Summary:** Establishes the canonical "three signals" mental model — traces, metrics, logs — and explains how OTel unifies them under a shared context-propagation subsystem. Read both pages back-to-back: the primer defines vocabulary (telemetry, instrumentation, observability) and the signals page maps each signal to its purpose. Focus on the framing that traces give you request flow, metrics give you aggregate health, and logs give you point-in-time records — and that OTel's value is making them correlatable via TraceID/SpanID propagation. You already know metrics from Prometheus week, so skim metrics quickly and spend most of the time on traces and the signals overview.

### 2. OpenTelemetry — Components (API, SDK, Collector, Instrumentation) ⏱️ 15 min
- **URL:** https://opentelemetry.io/docs/concepts/components/
- **Type:** Official Docs
- **Summary:** This is THE page that disambiguates the API/SDK/Collector layering — the single most common point of confusion for OTel learners. Focus on why the API and SDK are deliberately separated (libraries depend on the API only; applications wire in the SDK), what the Collector adds beyond an in-process SDK exporter, and how auto-instrumentation libraries plug into this stack. Read alongside the linked `opentelemetry.io/docs/concepts/instrumentation/` page to see the auto-vs-manual instrumentation tradeoff explicitly — auto for breadth (HTTP frameworks, DB clients), manual for business-meaningful spans/attributes.

### 3. OpenTelemetry Collector — Architecture + Configuration ⏱️ 25 min
- **URL:** https://opentelemetry.io/docs/collector/architecture/ and https://opentelemetry.io/docs/collector/configuration/
- **Type:** Official Docs
- **Summary:** The architecture page diagrams the receiver -> processor -> exporter pipeline; the configuration page shows the actual YAML you write. Focus on: (a) the four component types (receivers, processors, exporters, connectors) and the role of `service.pipelines.{traces,metrics,logs}` in wiring them, (b) the must-have processors `memory_limiter` (back-pressure, always first) and `batch` (throughput, always last before exporter), and (c) `resource` / `k8sattributes` processors for enriching telemetry with environment context. Pay attention to the gotcha that a component must appear in `service.pipelines` to actually be used — declaring it under top-level `processors:` is necessary but not sufficient. This mirrors the relabel-config two-phase pattern you saw in Prometheus Fundamentals.

### 4. SigNoz — OpenTelemetry Collector from A to Z: A Production-Ready Guide ⏱️ 20 min
- **URL:** https://signoz.io/blog/opentelemetry-collector-complete-guide/
- **Type:** Blog (vendor-neutral, OSS-friendly)
- **Summary:** Bridges the official docs into production-shaped configurations. Walks through a real Collector YAML with `otlp` receivers (gRPC 4317 + HTTP 4318), `memory_limiter` + `batch` + `resource` processors, and multi-backend exporters (e.g., `prometheusremotewrite`, `otlp/tempo`, `loki`). Focus on the canonical pipeline ordering (`memory_limiter` first, `batch` last) and on the Contrib distribution vs Core distribution distinction — Core ships only OTLP-native components, Contrib adds vendor and protocol-specific receivers/exporters that you'll actually need (Prometheus scrape receiver, Loki exporter, k8sattributes processor). Also covers the Collector's internal telemetry endpoint at `:8888` for self-monitoring.

### 5. OpenTelemetry — Agent-to-Gateway Deployment Pattern ⏱️ 15 min
- **URL:** https://opentelemetry.io/docs/collector/deployment/gateway/ and https://opentelemetry.io/docs/collector/deployment/agent/
- **Type:** Official Docs
- **Summary:** The two canonical Collector topologies on Kubernetes. The agent (DaemonSet) collects node-local data — kubelet/cAdvisor metrics, container logs from the node filesystem, and OTLP from local pods over `localhost`. The gateway (Deployment) provides a cluster-wide OTLP endpoint that does heavy processing (tail sampling, fan-out exports, auth to backends). Focus on why you almost always want both in production: agents stay thin and survive node-local pressure, gateways centralize policy (sampling, redaction, multi-tenant routing). Compare mentally to Prometheus federation — agent-to-gateway is the OTel equivalent of regional Prometheus -> global Prometheus, but with richer processing in the middle tier.

### 6. New Relic — OpenTelemetry Collector Deployment Modes in Kubernetes ⏱️ 15 min
- **URL:** https://newrelic.com/blog/infrastructure-monitoring/opentelemetry-collector-deployment-modes-in-kubernetes
- **Type:** Blog
- **Summary:** Concrete K8s deployment-mode walkthrough: DaemonSet vs Deployment vs StatefulSet vs Sidecar, with the tradeoffs of each. Focus on the DaemonSet's role for node telemetry (logs, kubelet metrics) and the Deployment's role as the cluster gateway. Pay attention to the StatefulSet variant for collectors that hold state (e.g., tail sampling that needs to keep a span buffer keyed by traceID — load-balancer routing must stick all spans of one trace to the same Collector replica via the `loadbalancing` exporter). The sidecar pattern is increasingly rare — note when it still makes sense (per-pod attribute injection, isolation in shared-tenant clusters).

### 7. OpenTelemetry — Operator + Auto-Instrumentation Injection ⏱️ 15 min
- **URL:** https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/
- **Type:** Official Docs
- **Summary:** Explains how the OTel Operator's mutating webhook + the `instrumentation.opentelemetry.io/inject-{python,java,nodejs,dotnet,go}: "true"` pod annotation transforms a regular Deployment into an auto-instrumented one with zero code changes. Focus on the mechanic: the webhook adds an init container that copies a language-specific agent into a shared volume, then mounts it into the app container and sets env vars (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`, `JAVA_TOOL_OPTIONS`, etc.) so the runtime auto-loads the agent. Also covers the `Instrumentation` CRD — a per-namespace resource that defines defaults (endpoint, sampler, propagators) that injected pods inherit. This is the K8s-native answer to "how do I instrument 200 microservices without touching their Dockerfiles."

### 8. OpenTelemetry — Resource Semantic Conventions (skim) + K8s Attributes ⏱️ 10 min
- **URL:** https://opentelemetry.io/docs/concepts/semantic-conventions/ and https://opentelemetry.io/docs/specs/semconv/non-normative/k8s-attributes/
- **Type:** Official Docs
- **Summary:** Semantic conventions are OTel's "agreed vocabulary" — the reason a Tempo trace from a Java service can be joined to a Loki log from a Python service: both emit `service.name`, `service.namespace`, `deployment.environment.name`, and the `k8s.*` family (`k8s.pod.name`, `k8s.namespace.name`, `k8s.deployment.name`, `k8s.cluster.name`). Skim the resource conventions page for the must-set attributes, then read the K8s attributes page in full — it documents the `resource.opentelemetry.io/<attribute>` annotation pattern that the `k8sattributes` Collector processor uses to auto-discover and stamp pod metadata onto telemetry. Recently (2026) `k8s.*` attributes were promoted to release-candidate stability — meaning dashboards and queries built on them are now safe to commit to.

### 9. Grafana Labs — Introducing Grafana Alloy: An OTel Collector Distribution ⏱️ 10 min
- **URL:** https://grafana.com/blog/grafana-alloy-opentelemetry-collector-with-prometheus-pipelines/
- **Type:** Blog
- **Summary:** Closes the loop with the metrics/Grafana sub-block you just finished. Alloy is Grafana's OSS distribution of the OTel Collector, fully OTLP-compatible but adding native Prometheus pipelines (scrape configs, remote_write) on top of OTel components. Focus on three things: (1) why a "distribution" exists at all — the upstream OTel Collector ships in Core/Contrib/k8s flavors, and vendors like Grafana, AWS (ADOT), and Splunk publish their own builds with selected component subsets, (2) Alloy's River-syntax DSL as an alternative to OTel YAML for advanced pipelines, and (3) the EOL of Grafana Agent on Nov 1 2025 — Alloy is the migration target. This is the natural Collector choice when your backend is Mimir/Tempo/Loki.

## Study Tips

- **Use the Prometheus mental model as scaffolding, not as substitute.** Metric model maps cleanly (counters/gauges/histograms exist in both), but traces and logs have no Prometheus analogue — context propagation via TraceID is the genuinely new concept. When reading the traces page, force yourself to articulate "what does W3C `traceparent` header carry, and how is it different from a Prometheus label?"
- **Read every Collector YAML out loud, naming each component's type.** "`otlp` is a receiver, `memory_limiter` is a processor, `batch` is a processor, `prometheusremotewrite` is an exporter, and `service.pipelines.metrics` is the wiring." This habit catches the most common bug: declaring a component but forgetting to reference it in `service.pipelines`.
- **Map every concept to a deployment topology before moving on.** API + SDK = inside your app pod. Auto-instrumentation init container = injected by Operator. DaemonSet agent = per-node Collector. Deployment gateway = cluster-wide Collector. Mimir/Tempo/Loki = the backends the gateway exports to. If you can draw this end-to-end on paper, you've internalized the architecture.
- **Defer language-specific SDK details.** You don't need to know how the Python `opentelemetry-instrumentation-flask` package works internally today — that comes when you instrument a real service. Focus on the conceptual shape (API contract -> SDK implementation -> exporter -> OTLP -> Collector).

## Next Steps

After this foundational session, the natural follow-ups in the Logs/Traces/OpenTelemetry sub-block are:
- **Distributed tracing fundamentals** — span/trace anatomy, parent/child relationships, W3C Trace Context propagation, sampling strategies (head vs tail), and the role of Tempo as a backend.
- **Logs in OpenTelemetry + Loki** — the OTel logs data model, log-to-trace correlation via TraceID injection, and the `filelog` receiver vs application-side OTLP log emission, then routing to Loki.
- **Hands-on Collector deployment via the OpenTelemetry Operator on EKS** — install the operator, deploy an `OpenTelemetryCollector` CRD in agent + gateway mode, annotate a sample app for auto-instrumentation, and watch traces flow into Tempo (likely via your kube-prometheus-stack-adjacent setup).
- **Tail-based sampling** — when head sampling isn't enough, how the `loadbalancing` exporter + `tailsamplingprocessor` work together across a Collector pool.
