# Grafana Fundamentals — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-04-24
**Difficulty:** Intermediate

## Overview

A production-focused tour of Grafana as a platform (not a Prometheus+PromQL refresher). Covers data sources, dashboard design patterns, variables, transformations, file-based and Terraform-based provisioning, the modern Scenes / dashboard schema v2 architecture, and how Grafana-managed alerting compares to Prometheus/Mimir-managed alerting. By the end, you should be able to reason about how a real team runs Grafana at scale: dashboards-as-code, RBAC boundaries, and which alerting path belongs where.

## Resources

### 1. Dashboards Overview + Best Practices (Official Docs) ⏱️ 20 min
- **URL:** https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/
- **Type:** Official Docs
- **Summary:** Grafana Labs' canonical guidance on dashboard design — the "tell a story / answer a question" principle, the Z-pattern scanning layout, panel sizing for importance, and the separation between experimentation and production dashboards. This is the mental model every subsequent resource assumes you have.

### 2. Provision Grafana — File-based YAML (Official Docs) ⏱️ 20 min
- **URL:** https://grafana.com/docs/grafana/latest/administration/provisioning/
- **Type:** Official Docs
- **Summary:** The reference for `/etc/grafana/provisioning/{datasources,dashboards,alerting,plugins}` YAML manifests. Pay attention to `editable: false`, `allowUiUpdates: false`, `prune: true`, `foldersFromFilesStructure`, and how dashboard providers differ from data source providers. This is the baseline "GitOps for Grafana" pattern that predates Git Sync.

### 3. Variables & Templating (Official Docs) ⏱️ 15 min
- **URL:** https://grafana.com/docs/grafana/latest/dashboards/variables/
- **Type:** Official Docs
- **Summary:** Variable types (query, custom, interval, datasource, text box, constant, ad-hoc filters), chained/cascading variables, multi-value formatting (`${var:pipe}`, `${var:regex}`, `${var:csv}`), and repeating rows/panels. Skim — you already know PromQL, so focus on the Grafana-specific mechanics around `$__rate_interval`, `$__range`, and how multi-value variables inject into label matchers.

### 4. Transform Data (Official Docs) ⏱️ 15 min
- **URL:** https://grafana.com/docs/grafana/latest/panels-visualizations/query-transform-data/transform-data/
- **Type:** Official Docs
- **Summary:** Transformations are the Grafana-specific superpower you can't replicate in PromQL — Reduce, Merge, Organize Fields, Join by field, Labels to fields, Config from query, Partition by values. Understand the pipeline model (each transform's output feeds the next) and why transformations let one query power multiple panel types. Critical for building stat/table panels without spawning extra queries.

### 5. Grafana Dashboards Are Now Powered by Scenes (Grafana Labs Blog) ⏱️ 15 min
- **URL:** https://grafana.com/blog/grafana-dashboards-are-now-powered-by-scenes-big-changes-same-ui/
- **Type:** Blog (Official)
- **Summary:** Explains the architectural shift from the legacy AngularJS/React dashboard renderer to the Scenes library — what changed under the hood, why the same UI now performs better, and what Scenes enables (dynamic dashboards, app plugins sharing the dashboard engine). Context you need to understand the v2 schema.

### 6. Grafana 13 Release -- Dynamic Dashboards GA, Git Sync GA, Advisor, Assistant (Grafana Labs Blog) ⏱️ 15 min

- **URL:** [https://grafana.com/blog/grafana-13-release-all-the-latest-features/](https://grafana.com/blog/grafana-13-release-all-the-latest-features/)
- **Type:** Blog (Official)
- **Summary:** The April 21, 2026 Grafana 13 launch post (replaces the dead Grafana 12 dynamic-dashboards preview URL). Dynamic Dashboards (tabs, nested rows, responsive layout) went **GA** with automatic v1 -> v2 schema migration on open. 2-way **Git Sync went GA** across GitHub / GitLab / Bitbucket / Git with GitHub App auth and PR-based dashboard review -- the successor to file-based provisioning for cloud-native shops. **Grafana Advisor** (new, GA) runs automated health checks on data sources, plugins, SSO config, and misconfigurations with AI-assisted remediation. **Grafana Assistant** is now available in OSS and Enterprise (previously Cloud-only). Plus: dashboard layout templates for DORA/USE/RED methodologies (the "blank page" killer), updated Gauge panel GA, service-topology mapping, and Alloy's new OpenTelemetry Engine mode (standard OTel Collector YAML). The single resource that turns "Grafana 12+" knowledge into "current as of GrafanaCON 2026" knowledge -- this is what keeps senior-level answers from reading as stale.

### 7. Alertmanager vs Grafana Alerting — When to Use Each ⏱️ 15 min
- **URL:** https://alexandre-vazquez.com/alertmanager-vs-grafana-alerting/
- **Type:** Article
- **Summary:** Clear side-by-side comparison: architecture (Grafana Alerting couples dashboards + alerts + SQL-backed state vs. Alertmanager/Mimir-AM's file-configured, horizontally scalable design), multi-datasource alerting, data-source-managed vs. Grafana-managed rules, and the common hybrid pattern (Alertmanager for Prometheus-native, Grafana Alerting for everything else like CloudWatch/Loki/SQL). Builds directly on your Alertmanager deep dive from Apr 23.

### 8. Grafana Terraform Provider + Dashboards-as-Code (Official Docs) ⏱️ 15 min
- **URL:** https://grafana.com/docs/grafana/latest/as-code/infrastructure-as-code/terraform/
- **Type:** Official Docs
- **Summary:** The production IaC path: `grafana_folder`, `grafana_data_source`, `grafana_dashboard` (with `config_json`), `grafana_folder_permission`, `grafana_team`, `grafana_rule_group`, `grafana_contact_point`, `grafana_notification_policy`. Shows how to thread folder IDs into dashboards and how to manage the full stack declaratively across OSS and Cloud. Pair mentally with the Grafonnet/Jsonnet escape hatch for dashboards you don't want to hand-write as JSON.

## Study Tips

- **Don't re-read PromQL or Prometheus.** Every time a resource dips into query syntax, skim — your mental model is already solid. Lock in instead on Grafana-specific mechanics: variable interpolation formats, transformation pipelines, folder/permission boundaries, and the Scenes-vs-legacy architectural split.
- **Keep a "where does this live?" map open while reading.** For every feature, ask: is it stored in the Grafana SQL DB (users, teams, dashboards, alert rules, silences), on disk (provisioning YAML, dashboard JSON files), or in the data source (Prometheus recording rules, Mimir ruler)? This is the single most useful frame for understanding production Grafana topologies.
- **Mentally place each alerting feature on the Grafana-managed vs data-source-managed axis.** You already know Prometheus Alertmanager deeply — so Grafana Alerting's value prop is "alerting on non-Prometheus sources" and "dashboard-coupled alerts." Keep that framing while reading resource #7.
- **Open a throwaway Grafana OSS container (`docker run -p 3000:3000 grafana/grafana`) and poke at variables, transformations, and provisioning as you read.** 10 minutes of hands-on beats 30 minutes of passive reading for dashboard mechanics.

## Next Steps

- **Loki & Tempo fundamentals** — logs and traces as first-class Grafana data sources; this is the natural next topic in the observability phase.
- **Grafana Mimir architecture** — long-term Prometheus storage, multi-tenant ruler, horizontally scalable Alertmanager.
- **Grafana Operator / Git Sync (Grafana 12+)** — Kubernetes-native dashboard and alert CRD management, the successor to file-based provisioning for cloud-native shops.
- **Grafonnet / cdk8s-grafana / Grizzly** — programmatic dashboard authoring for teams that have outgrown hand-edited JSON.
- **SLO tooling inside Grafana** — the Grafana SLO plugin (Grafana Cloud) and the Sloth/Pyrra ecosystem for SLO-as-code that compiles to Prometheus recording+alerting rules.
