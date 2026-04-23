# Alertmanager & Alerting Strategy — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-23
**Difficulty:** Intermediate / Advanced

## Overview

This plan picks up directly from the PromQL Deep Dive (`2026/april/prometheus/2026-04-21-promql-deep-dive.md`) and moves from "how do I query metrics" to "how do those queries become pages that wake the right person at 3am — and not the wrong person, not five people, and not seventeen times in a row." It focuses on the Alertmanager pipeline (inhibit -> silence -> route -> dedupe -> notify), the routing-tree semantics (`group_by` / `group_wait` / `group_interval` / `repeat_interval`), inhibition vs silences, HA gossip and the notification log, SLO-based multi-window multi-burn-rate alerts, and the kube-prometheus-stack / PrometheusRule / AlertmanagerConfig CRD path for deploying all of this on EKS.

Beginner overviews ("what is an alert") are deliberately skipped. The PromQL side of the `expr` field is already in your head from Day 2 — today is about the decision graph *after* the expression fires.

## TL;DR — What You'll Be Able to Do After This

- **Write production-grade alerting rules** with correct `for:` durations, severity labels, `runbook_url` annotations, and templated `$labels`/`$value` descriptions — and know when `keep_firing_for` saves you from flap storms.
- **Read a routing tree** and predict exactly which receiver a given alert hits, how long it waits for siblings, and when the next notification fires. Explain `continue: true` and when nested routes beat a flat list.
- **Design inhibition rules** that suppress downstream noise (node down -> hide its pod alerts) and articulate why that is fundamentally different from a silence.
- **Justify grouping choices** — why `group_by: [alertname, cluster, service]` is usually right, why `group_by: [...]` (all labels) is a notification storm, and why `group_by: [alertname]` alone loses critical context.
- **Implement multi-window multi-burn-rate SLO alerts** from the Google SRE Workbook and explain fast-burn vs slow-burn, why two windows beat one, and why you still need a ticket-severity path for slow burns.
- **Run HA Alertmanager** (3 replicas, gossip on :9094) and explain why you don't get duplicate pages even though every replica independently evaluates the pipeline — the nflog is the trick.
- **Operate the system** with `amtool` (check-config, alert query, silence add), `promtool test rules`, and know the top production gotchas: resolve timeout, external labels for multi-cluster dedup, silence expiry, and the "for: 0s on deploy" rollout-flap trap.

## Resources

### 1. Prometheus Alerting Rules — Official Reference ⏱️ 10 min
- **URL:** https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- **Type:** Official Docs
- **Summary:** The canonical page for rule file syntax: `expr` + `for` + `keep_firing_for` + `labels` + `annotations`, the pending -> firing state machine, and Go templating with `{{ $labels.instance }}` / `{{ $value }}`. Short and load-bearing — every concept on this page ends up in the `PrometheusRule` CRD you'll write tomorrow.

### 2. Alertmanager Overview — What It Does and Doesn't ⏱️ 10 min
- **URL:** https://prometheus.io/docs/alerting/latest/alertmanager/
- **Type:** Official Docs
- **Summary:** The pipeline in one page: grouping, inhibition, silences, high availability, and the core promise that Alertmanager de-duplicates across a Prometheus HA pair *and* across its own HA replicas. Read this before the config page so the knobs on that page have semantic homes.

### 3. Alertmanager Configuration — Full Reference ⏱️ 25 min
- **URL:** https://prometheus.io/docs/alerting/latest/configuration/
- **Type:** Official Docs
- **Summary:** The single most important page for today. Covers the routing tree (`group_by`, `group_wait`, `group_interval`, `repeat_interval`, `continue`), `matchers` (v0.22+) vs the deprecated `match` / `match_re`, every built-in receiver (Slack, PagerDuty, OpsGenie, webhook, email, SNS, MSTeams, Discord, Pushover, VictorOps, WeChat, Telegram), and inhibition rules with `source_matchers` / `target_matchers` / `equal`. Don't try to memorize — read linearly once, then plan to live in this page.

### 4. Alertmanager Routing Tree Visual Editor (hands-on) ⏱️ 10 min
- **URL:** https://prometheus.io/webtools/alerting/routing-tree-editor/
- **Type:** Interactive Tool
- **Summary:** Paste your routing config and see a tree diagram of which alert labels route where, plus the effective `group_wait` / `group_interval` / `repeat_interval` inherited at each node. The fastest way to internalize `continue: true` and route inheritance — spend ten minutes feeding it non-trivial configs and watching the tree change.

### 5. Google SRE Workbook, Chapter 5 — Alerting on SLOs ⏱️ 30 min
- **URL:** https://sre.google/workbook/alerting-on-slos/
- **Type:** Book chapter (free, canonical)
- **Summary:** The reference for modern alerting philosophy. Walks through six successive strategies (target error rate -> increased alert window -> burn rate -> multi-burn-rate -> multi-window multi-burn-rate) and shows mathematically why each one fixes the previous one's failure mode. The table with the 14.4 / 6 / 3 / 1 burn-rate multipliers and their detection-time / alert-budget consumption tradeoffs is the one you will cite in design reviews forever. Non-optional.

### 6. Grafana Labs — Multi-Window Multi-Burn-Rate Alerts, Concretely ⏱️ 15 min
- **URL:** https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/
- **Type:** Blog
- **Summary:** The SRE Workbook gives you the theory; this post gives you the actual PromQL and PrometheusRule YAML. Shows the two-window `and` expression for fast-burn, the longer-window ticket-severity alert for slow-burn, and how to source the SLO target as a recording rule. Use this as the template when you write your first burn-rate alert.

### 7. Prometheus Operator — PrometheusRule and AlertmanagerConfig CRDs ⏱️ 10 min
- **URL:** https://prometheus-operator.dev/docs/developer/alerting/
- **Type:** Official Docs (prometheus-operator)
- **Summary:** How the operator model changes everything: `PrometheusRule` objects replace rule files, `AlertmanagerConfig` objects let teams own namespace-scoped routing without editing the global config, and `alertmanagerConfigSelector` / `alertmanagerConfigNamespaceSelector` control which configs get merged in. This is what you'll actually ship via kube-prometheus-stack — read it before touching the chart values.

### 8. kube-prometheus-stack Chart — Default Alerting Rules and Customization ⏱️ 10 min
- **URL:** https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
- **Type:** Official Helm Chart README
- **Summary:** Scan the values schema for `alertmanager.config`, `alertmanager.alertmanagerSpec.replicas`, `defaultRules.create`, and `defaultRules.disabled`. You don't need to read the whole README — find the alerting sections, note the built-in rule groups (KubePodCrashLooping, KubeDeploymentReplicasMismatch, NodeFilesystemSpaceFillingUp, etc.), and learn the `additionalPrometheusRulesMap` pattern for layering your own rules on top without forking the chart.

### 9. amtool — Alertmanager CLI Reference ⏱️ 10 min
- **URL:** https://github.com/prometheus/alertmanager/tree/main/cli
- **Type:** Official CLI docs
- **Summary:** The operator toolkit: `amtool check-config alertmanager.yml` (catches routing-tree errors before rollout), `amtool alert` (list currently firing alerts), `amtool silence add alertname=Foo --duration=2h --comment="deploy window"` (the correct way to silence, with an audit trail). Scan the flag set so you know it exists; you'll come back when you need it in an incident.

## Study Tips

- **Follow one alert from YAML to Slack.** For each resource, mentally trace the path: `PrometheusRule.expr` fires -> Prometheus HTTP-POSTs to Alertmanager -> inhibit filter -> silence filter -> route match -> group_wait timer -> dedupe against nflog -> notify receiver. If any stage is fuzzy, re-read that section of the config reference.
- **Build the grouping intuition.** `group_by: [alertname, cluster, service]` is the default shape for a reason — alertname keeps "DiskFull on 20 nodes" as one page, cluster prevents cross-cluster contamination, service keeps team ownership clean. Whenever you see a `group_by`, ask "if 50 alerts matching this group fired at once, how many notifications would I get, and is that the right number?"
- **Inhibit vs silence is an interview question.** Inhibition is declarative and permanent in config — "if NodeDown is firing for instance=X, suppress PodCrashLooping for instance=X." Silence is imperative and temporary — "I'm deploying for 30 minutes, shut up about staging." Never conflate them.
- **Write the SRE Workbook table on paper.** The 2% / 5% / 10% budget-consumption thresholds with 1h / 6h / 3d windows and the matching 14.4 / 6 / 1 burn-rate multipliers — if you can redraw this table from memory, you understand multi-burn-rate alerting. If you can't, you don't yet.
- **Test rules before shipping.** `promtool test rules test.yml` lets you feed synthetic time series and assert which alerts fire when. Every non-trivial rule should have a test — this catches the "I meant `>` but wrote `>=` and nothing ever fires" class of bugs.

## Next Steps

- **Grafana fundamentals:** dashboards, variables, Grafana-native unified alerting (separate from Alertmanager — know the difference), and the Grafana contact-points / notification-policies model that mirrors Alertmanager's routing tree.
- **kube-prometheus-stack hands-on:** deploy the chart via ArgoCD on your EKS cluster, add a custom `PrometheusRule` for a burn-rate alert on a real service, wire an `AlertmanagerConfig` to a test Slack webhook, and watch a synthetic alert traverse the pipeline end to end.
- **SLO tooling:** explore Sloth (https://sloth.dev) for generating multi-burn-rate rules from a declarative SLO spec, and OpenSLO as the spec format. These automate what you'll learn to hand-write today.
- **Long-term storage + global alerting:** Thanos Ruler / Mimir alerting for cross-cluster rules, and the external-labels pattern that keeps multi-tenant Alertmanagers from deduplicating alerts that only *look* identical.
- **Alert-fatigue culture:** the "if you paged, write a postmortem or delete the alert" discipline, on-call rotation sanity, and how to use alert volume as a health metric for your monitoring itself.
