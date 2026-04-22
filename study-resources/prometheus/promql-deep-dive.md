# PromQL Deep Dive — 2-Hour Study Plan

**Total Time:** ~2 hours (120 min)
**Created:** 2026-04-21
**Difficulty:** Intermediate / Advanced

## Overview

This plan sharpens your PromQL fluency after yesterday's Prometheus Fundamentals deep dive (`2026/april/prometheus/2026-04-20-prometheus-fundamentals.md`). It skips "what is a counter" and focuses on the query language itself: the instant vs range vector mental model, vector matching, the `rate()` family's extrapolation math, `histogram_quantile` on classic vs native histograms, subqueries vs recording rules, and the RED / USE / SLI idioms that turn raw series into answers.

By the end you should be able to look at any PromQL expression, predict whether it returns an instant or range vector, know which labels flow through, and spot the classic footguns (sum-then-rate, rate over the wrong window, quantile on per-instance buckets).

## Resources

### 1. Prometheus Official Docs — Querying Basics ⏱️ 20 min
- **URL:** https://prometheus.io/docs/prometheus/latest/querying/basics/
- **Type:** Official Docs
- **Summary:** The canonical reference for selector syntax, the four value types (scalar / string / instant vector / range vector), label matchers including the important empty-matcher and `__name__` semantics, plus the `offset` and `@` modifiers. Read this even if you think you know it — the empty-matcher and `@` rules are easy to misremember and this page is the ground truth.

### 2. Prometheus Official Docs — Operators (Vector Matching) ⏱️ 15 min
- **URL:** https://prometheus.io/docs/prometheus/latest/querying/operators/
- **Type:** Official Docs
- **Summary:** The formal rules for arithmetic, comparison, and logical/set operators (`and`, `or`, `unless`), plus vector matching with `on` / `ignoring` and many-to-one joins via `group_left` / `group_right`. Skim the aggregation section (`by` / `without`) — you'll use `without (instance)` constantly in real queries.

### 3. Grafana Labs — PromQL Vector Matching Explained ⏱️ 15 min
- **URL:** https://grafana.com/blog/2024/12/13/promql-vector-matching-what-it-is-and-how-it-affects-your-prometheus-queries/
- **Type:** Blog
- **Summary:** A 2024 walkthrough of the single most error-prone area of PromQL: matching two vectors whose label sets don't line up. Makes `group_left` / `group_right` visual and shows the "multiple matches for labels" error you will absolutely see in production. Pairs perfectly with the Operators doc above.

### 4. Robust Perception — Rate Then Sum, Never Sum Then Rate ⏱️ 10 min
- **URL:** https://www.robustperception.io/rate-then-sum-never-sum-then-rate/
- **Type:** Blog (Brian Brazil, Prometheus core maintainer)
- **Summary:** Short but load-bearing. Explains why `sum(rate(x[5m]))` is correct and `rate(sum(x)[5m:])` is a time bomb — a restart of any instance turns the summed counter into a fake reset and `rate()` treats it as a spike. Internalize this; it is the #1 PromQL mistake in code review.

### 5. PromLabs — How Exactly Does PromQL Calculate Rates? ⏱️ 20 min
- **URL:** https://promlabs.com/blog/2021/01/29/how-exactly-does-promql-calculate-rates/
- **Type:** Blog (Julius Volz, Prometheus co-creator)
- **Summary:** The definitive explanation of `rate()` vs `irate()` vs `increase()` — including counter reset handling, the extrapolation-to-window-boundaries rule, why `rate()` is almost always what you want, and why `irate()` is only appropriate for high-resolution graphs of fast-moving counters. Read this and you'll never write `irate()` for alerts again.

### 6. Prometheus Best Practices — Histograms and Summaries ⏱️ 15 min
- **URL:** https://prometheus.io/docs/practices/histograms/
- **Type:** Official Docs
- **Summary:** How `histogram_quantile()` actually works — why you must `sum by (le, ...)` across instances *before* feeding buckets into `histogram_quantile`, the linear-interpolation approximation inside each bucket, and the differences when operating on native histograms (where aggregation is "just addition" and you also get `histogram_count`, `histogram_sum`, `histogram_fraction`). Builds directly on yesterday's histogram fundamentals.

### 7. Prometheus Official Docs — Recording Rules (Best Practices) ⏱️ 15 min
- **URL:** https://prometheus.io/docs/practices/rules/
- **Type:** Official Docs
- **Summary:** The `level:metric:operations` naming convention, when to pre-aggregate vs query ad-hoc, and the "always use `without`, not `by`" guidance so you keep `job` and other discriminator labels. Complements the subquery discussion — recording rules are almost always the right answer when you're tempted to write `[1h:1m]`.

### 8. Robust Perception — Common Query Patterns in PromQL ⏱️ 10 min
- **URL:** https://www.robustperception.io/common-query-patterns-in-promql/
- **Type:** Blog
- **Summary:** A compact cookbook: error ratios, ratio of summaries (`rate(sum)/rate(count)`), percent-change over time, and the RED-method shape. Short, high density — treat this as the "patterns you'll copy-paste for the rest of your career" page.

### 9. PromLabs — PromQL Cheat Sheet (bookmark, don't read linearly) ⏱️ 10 min
- **URL:** https://promlabs.com/promql-cheat-sheet/
- **Type:** Reference
- **Summary:** The best single-page PromQL reference on the internet. Skim it so you know what's there — then keep the tab open forever. Notable sections: `label_replace` / `label_join` (how to rewrite labels mid-query), `absent()` / `absent_over_time()` for alerting on missing series, and the `@` modifier syntax.

## Study Tips

- **Build the vector-type habit.** Every time you read an expression, say out loud whether each sub-expression returns an instant vector, range vector, or scalar. `rate()` takes a range vector and returns an instant vector; `rate(foo[5m])[10m:1m]` is a subquery that wraps it back into a range vector. Most PromQL errors are type errors in disguise.
- **Actually run the queries.** Open Prometheus or a Grafana Explore pane and paste expressions from the resources against real data. Toggle "Table" vs "Graph" views — tables force you to see the label sets that the vector-matching rules are operating on.
- **Practice the "sum first or rate first?" drill.** For any counter query, ask: am I applying `rate()` per-series (correct) or across an already-summed series (wrong)? The Robust Perception post makes this instinctive.
- **Don't skip the histogram section even if you feel comfortable with counters.** `histogram_quantile` + classic-histogram aggregation is the most asked PromQL interview question after `rate` vs `irate`.

## Next Steps

- **Alerting rules (Day 3 of the week):** PromQL expressions become alert bodies. Learn `for:`, the `ALERTS` / `ALERTS_FOR_STATE` series, and how `absent()` / `absent_over_time()` catch missing data.
- **Grafana dashboarding with PromQL:** variables (`label_values()`), `$__rate_interval`, panel repetition, and the difference between instant and range queries in Grafana's query editor.
- **Query performance:** cardinality analysis via `topk(10, count by (__name__)({__name__=~".+"}))`, the `--query.max-samples` knob, and when to offload heavy queries to Thanos/Mimir query frontends.
- **SLO math:** burn-rate alerts (multi-window, multi-burn-rate), the Sloth generator, and error-budget queries that compose the patterns you practiced today.
