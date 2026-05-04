# SLOs, Error Budgets & Production Observability — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-05-01
**Difficulty:** Advanced
**Audience:** Senior/Staff DevOps & Platform engineers — assumes Prometheus, PromQL, Alertmanager, multi-burn-rate basics, OpenTelemetry, and unified observability are already understood.

## Overview

This plan caps the observability arc by zooming out from "what signals do I emit" to "what reliability promises do I make, and how do I run an organization around them." The reading is biased toward the Google SRE Workbook, the Hidalgo / Majors / Fong-Jones canon, and the OSS SLO tooling ecosystem (Sloth, Pyrra, OpenSLO). After completing it you should be able to design SLIs from user journeys, defend an SLO target on a cost-of-reliability curve, write an error-budget policy that has organizational teeth, articulate why 14.4/6/3/1 are the canonical burn rates, and pick the right SLO tool for a given platform.

---

## Resources

### 1. Google SRE Workbook — Chapter 2: Implementing SLOs ⏱️ 30 min — MUST READ
- **URL:** https://sre.google/workbook/implementing-slos/
- **Type:** Official Docs (canonical text)
- **Why this matters:** This is the source-of-truth chapter for SLI specification vs implementation, the difference between "what you want to measure" and "what your monitoring system can actually express," and the user-journey-first approach to picking SLIs. The "Continuous Improvement" section at the end is the missing piece most teams skip — SLOs are not a one-time setting, they are a feedback loop.
- **Focus areas:** "Specifying SLIs and SLOs" (skim), "SLIs for different types of services" (request-driven, data processing, storage — read carefully — this is the section your prior burn-rate doc did not cover), "Choosing an appropriate time window," and "Continuous Improvement of SLO Targets."

### 2. Google SRE Workbook — Chapter 5: Alerting on SLOs (advanced re-read) ⏱️ 20 min — MUST READ
- **URL:** https://sre.google/workbook/alerting-on-slos/
- **Type:** Official Docs
- **Why this matters:** You already covered the 14.4/6/3/1 table in the Apr 23 Alertmanager doc, so this re-read is targeted: focus on the *derivation* — the six iterations the chapter walks through (1. Target Error Rate >= SLO, 2. Increased Alert Window, 3. Incrementing Alert Duration, 4. Alert on Burn Rate, 5. Multiple Burn Rate Alerts, 6. Multi-Window Multi-Burn-Rate). Understanding *why* iterations 1-5 fail builds the mental model needed to defend the design choice in interviews, and to know when you can deviate from it.
- **Focus areas:** Skip the basic burn-rate definition; focus on the precision/recall/detection-time/reset-time tradeoff table and the explicit math behind 14.4 (2% budget burn in 1 hour) and 6 (5% budget burn in 6 hours).

### 3. Google SRE Workbook — Chapter 4: Error Budget Policy ⏱️ 15 min — MUST READ
- **URL:** https://sre.google/workbook/error-budget-policy/
- **Type:** Official Docs
- **Why this matters:** The error-budget *policy* is the part that turns SLOs from a dashboard into an organizational lever. This chapter shows the actual template Google uses — who signs it, what triggers a feature freeze, what the escalation path is when the dev team disputes an SRE-proposed freeze, and what counts as a "release" under the policy. This is the highest-signal artifact for Staff/Principal interviews because it tests whether you can scale reliability culture beyond a single team.
- **Focus areas:** "Goals and Non-Goals," the specific consequence ladder, and the section on policy disagreements (this is where most real-world programs collapse).

### 4. Sloth Documentation — Architecture & Spec ⏱️ 15 min
- **URL:** https://sloth.dev/introduction/architecture/
- **Type:** Official Docs
- **Why this matters:** Sloth is the most-deployed OSS Prometheus SLO generator. Reading the architecture page shows you exactly which recording rules and alert rules get generated from a YAML spec — this is the cleanest way to internalize how a single SLO declaration expands into ~10 PrometheusRule artifacts. Pair it with a quick scan of the GitHub README for plugin / SLI source patterns.
- **Focus areas:** The generated rule taxonomy (SLI rules across windows, metadata rules, multi-window multi-burn-rate alert rules) and the SLOTH-spec YAML structure.

### 5. Pyrra — Project README & Live Demo ⏱️ 10 min
- **URL:** https://github.com/pyrra-dev/pyrra
- **Demo:** https://demo.pyrra.dev/
- **Type:** Official Docs (OSS project)
- **Why this matters:** Pyrra is the Kubernetes-native alternative to Sloth — it ships a CRD (`ServiceLevelObjective`) plus a controller plus a UI. Compare it mentally with Sloth: Sloth = CLI generator, Pyrra = operator-pattern with live UI. The 10-minute live demo lets you click through actual SLO objects and see error-budget burndown rendered without you wiring anything up. Knowing both lets you answer "why did you pick X?" in design interviews.
- **Focus areas:** The CRD schema, the `--config-map-mode` fallback for non-Operator clusters, and how it interacts with the Prometheus Operator's PrometheusRule reconciliation.

### 6. OpenSLO Specification ⏱️ 10 min
- **URL:** https://github.com/OpenSLO/OpenSLO
- **Spec home:** https://openslo.com/
- **Type:** Open Specification
- **Why this matters:** OpenSLO is the vendor-neutral YAML schema (think OpenTelemetry but for SLOs) backed by Nobl9, Sumo Logic, Dynatrace, and others. It matters strategically because it answers the question: "How do I avoid SLO lock-in if I move from Sloth to Nobl9 to Grafana SLO?" Skim the spec to see the `Service`, `SLI`, `SLO`, and `AlertPolicy` resource kinds — these mirror Pyrra/Sloth's mental model and let you reason about portability.
- **Focus areas:** The four kinds (Service, SLI, SLO, AlertPolicy), and the "ratio" vs "threshold" SLI patterns.

### 7. Grafana Labs Blog — Implementing Multi-Window, Multi-Burn-Rate Alerts ⏱️ 15 min
- **URL:** https://grafana.com/blog/how-to-implement-multi-window-multi-burn-rate-alerts-with-grafana-cloud/
- **Type:** Vendor Blog (high-quality, no signup required)
- **Why this matters:** You already understand the math, so use this to see a *concrete production implementation* with PromQL recording rules and alert thresholds laid out side-by-side. The blog also covers severity routing — fast-burn pages on-call immediately, slow-burn opens a ticket — which is the routing pattern your Alertmanager doc set up but didn't ground in SLO semantics.
- **Focus areas:** The exact PromQL expressions, the severity-vs-burn-rate mapping, and how Grafana's SLO plugin generates the same rules declaratively.

### 8. Charity Majors / Liz Fong-Jones — Observability Maturity Model ⏱️ 15 min — MUST READ
- **Primary URL (free, no signup):** https://www.honeycomb.io/blog/observability-maturity-model
- **Secondary URL (free book, email signup):** https://www.honeycomb.io/observability-engineering-oreilly-book — Honeycomb hands out the full O'Reilly book as a complimentary PDF; the Maturity Model is Chapter 21
- **O'Reilly (paywalled, for reference):** https://www.oreilly.com/library/view/observability-engineering/9781492076438/ch21.html
- **Type:** Authoritative blog post + free ebook + paywalled book chapter
- **Why this matters:** This is the one resource that *zooms out* from SLOs to "what does mature observability look like organizationally?" It defines five capability dimensions (respond to system failure with resilience, deliver high-quality code, manage complexity and technical debt, release on predictable cadence, understand user behavior) and ties each back to observable outcomes. For Staff/Principal interviews this is the framework you cite when asked "How would you assess the observability maturity of an org you just joined?"
- **Focus areas:** The five capability dimensions, the "creating a mature observability practice is not linear" caveat, and the linkage between SLOs and the "respond to system failure" capability.
- **Note:** A previously-circulated `erenow.org` mirror of this chapter has been hijacked into a spam-redirect loader page — do not use it. The Honeycomb blog post above covers the same framework directly from the authors, and the free PDF download is the cleanest way to get the full book chapter.

### 9. SoundCloud Engineering — Alerting on SLOs Like Pros ⏱️ 10 min — supplementary war story
- **URL:** https://developers.soundcloud.com/blog/alerting-on-slos/
- **Type:** Engineering Blog (production case study)
- **Why this matters:** A real production team's walkthrough of how they implemented MWMBR alerting and the gotchas they hit — recording rule explosion, low-traffic SLO noise, and what they did when alert volume spiked after launch. Worth reading as the practical companion to the SRE Workbook's theoretical treatment.
- **Focus areas:** The recording-rule layering they ended up with, and their handling of the "low traffic = noisy SLO" problem.

---

## Study Tips

- **Read the SRE Workbook chapters in this order: 2 -> 5 -> 4.** Specification before alerting before policy mirrors how you'd actually roll out an SLO program: define what good looks like, instrument detection, then attach organizational consequences.
- **Take notes in the form "What would I disagree with at $previous_company?"** The Workbook presents an idealized version; real orgs deviate. Capturing your deviations now is the rehearsal for the "tell me about a time you implemented SLOs" interview question.
- **For Sloth and Pyrra, read the YAML specs side-by-side.** The mental exercise of translating a Sloth spec into a Pyrra CRD (and noticing what each tool exposes that the other doesn't — Sloth's plugin SLIs vs Pyrra's UI) is more valuable than reading either doc twice.
- **Skip the basics.** If a section starts with "An SLO is..." or "An error budget is 100% minus...", skim to the next subhead. Your Apr 23 Alertmanager doc already covered this ground; this session is for the layer above.

## Next Steps

After this 2-hour session, the highest-leverage follow-ups are:
1. **Implement** — Take the SLO from the kube-prometheus-stack you set up in the prior arc (e.g., API server availability), express it as a Sloth YAML spec, generate the rules, and verify the alerts fire in a chaos test.
2. **Read** — Alex Hidalgo's *Implementing Service Level Objectives* (Chapters 4, 5, 8 specifically). The book extends the Workbook's request-driven examples into batch, streaming, ML pipeline, and infrastructure-as-a-product SLIs that the Workbook only hints at. Available via O'Reilly subscription or library.
3. **Compare** — Spend 30 min benchmarking Grafana SLO plugin (Enterprise) vs Nobl9 vs Pyrra against the OpenSLO spec for portability — useful for the "build vs buy" decision in a real platform-team interview.
4. **Tie it back** — Revisit the unified observability doc (Apr 30) and trace one SLO breach end-to-end through metrics -> trace -> log correlation. The SLO breach is the *trigger* the unified pillar story was always building toward.
