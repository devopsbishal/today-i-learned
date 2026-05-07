# GitHub Actions — Custom Actions & Reusable Patterns — 2-Hour Study Plan

**Total Time:** ~2 hours
**Created:** 2026-05-06
**Difficulty:** Intermediate

## Overview

This plan goes beyond yesterday's workflow/expressions/concurrency foundation and dives into the *authoring side* of GitHub Actions: how to build the three custom action types (JavaScript, Docker, composite), how to design reusable workflows that other repos can call, and the production patterns around caching, artifacts, and monorepo path filtering that determine whether a CI pipeline runs in 2 minutes or 20. By the end, you should be able to look at any custom action's `action.yml` and predict its behavior, and confidently choose between a composite action and a `workflow_call` workflow for any reuse scenario.

## Resources

### 1. About custom actions — GitHub Docs (concepts) ⏱️ 12 min
- **URL:** https://docs.github.com/en/actions/concepts/workflows-and-actions/custom-actions
- **Type:** Official Docs
- **Summary:** The conceptual map of the three custom action types (JavaScript, Docker container, composite) — read this first to lock in the mental model of which type runs where (Linux-only for Docker, cross-OS for JS/composite), what `action.yml` actually is, and the trade-offs between them before diving into authoring details.

### 2. Creating a JavaScript action — GitHub Docs ⏱️ 25 min
- **URL:** https://docs.github.com/en/actions/tutorials/creating-a-javascript-action
- **Type:** Official Docs / Tutorial
- **Summary:** Full walkthrough of authoring a Node 20 JavaScript action: the `action.yml` anatomy (inputs with `required`/`default`/`description`, outputs, `runs.using: node20`, `runs.main`), wiring to `@actions/core` and `@actions/github`, the `dist/` bundling requirement (ncc), and branding fields. The reference template for your own first custom action.

### 3. Creating a Docker container action — GitHub Docs ⏱️ 15 min
- **URL:** https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-docker-container-action
- **Type:** Official Docs / Tutorial
- **Summary:** Covers the Dockerfile + `action.yml` (`runs.using: docker`, `runs.image: Dockerfile` or pre-built image) pattern, `args`/`env` mapping from inputs, and the Linux-runner-only constraint. Skim quickly — focus on when you'd reach for a container action over JS (heavy native deps, language other than Node, sealed toolchain).

### 4. Reusing workflow configurations — GitHub Docs ⏱️ 20 min
- **URL:** https://docs.github.com/en/actions/concepts/workflows-and-actions/reusing-workflow-configurations
- **Type:** Official Docs
- **Summary:** The authoritative reference for `workflow_call` reusable workflows: declaring `on.workflow_call.inputs`/`secrets`/`outputs`, referencing them via `inputs.x` and `secrets.X`, returning values via `jobs.<id>.outputs`, and the calling syntax (`uses: org/repo/.github/workflows/x.yml@ref`). Also covers composite-action authoring blocks with `runs.using: composite` and the `shell:` requirement on every step.

### 5. Creating a composite action — GitHub Docs ⏱️ 15 min
- **URL:** https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-composite-action
- **Type:** Official Docs / Tutorial
- **Summary:** The third leg of the custom-actions trio (paired with #2 JS and #3 Docker). Walks through `runs.using: composite` + `runs.steps`, the **mandatory `shell:` on every `run:` step** (the most common authoring bug), how to surface step outputs as action outputs via `${{ steps.<id>.outputs.<name> }}`, mixing `uses:` (other actions) inside composite steps, and the per-step `if:` and `working-directory` semantics. Yesterday already covered the *decision* of composite-vs-reusable-workflow — this is the *how to actually author one* reference you'd otherwise be missing.

### 6. actions/cache README + Dependency caching reference ⏱️ 20 min
- **URL:** https://github.com/actions/cache
- **URL:** https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching
- **Type:** Official Docs
- **Summary:** Two short reads stacked together. The repo README covers `key`/`restore-keys` hierarchy, `hashFiles('**/package-lock.json')` for lockfile-based invalidation, immutable cache rule (a key written once cannot be overwritten — bump a segment to refresh), 10 GB/repo eviction, branch scoping (main vs feature), and the cross-OS pitfall (Linux cache won't restore on Windows). The dependency caching reference shows the *easier* path: `setup-node`/`setup-python`/`setup-java` with `cache: 'npm'|'pip'|'maven'` parameters that wire actions/cache for you.

### 7. actions/upload-artifact v4 — Repo + Changelog ⏱️ 15 min
- **URL:** https://github.com/actions/upload-artifact
- **URL:** https://github.blog/changelog/2023-12-14-github-actions-artifacts-v4-is-now-generally-available/
- **Type:** Official Docs / Changelog
- **Summary:** v4 is a hard break from v3: artifacts are now immutable, scoped to a single job (so multiple jobs can no longer append to one named artifact — split + use `download-artifact@v4` with `pattern:` and `merge-multiple: true`), available for download immediately after upload (no end-of-workflow wait), 500 artifacts/job limit, and v3 and v4 artifacts cannot be mixed. Critical for understanding why upgrades break pipelines and how to restructure fan-out matrices.

### 8. dorny/paths-filter — Selective monorepo CI ⏱️ 15 min
- **URL:** https://github.com/dorny/paths-filter
- **Type:** Marketplace Action / README
- **Summary:** The de-facto tool for per-job path filtering in monorepos because GitHub's built-in `paths:` only filters at the workflow level. Read the README for the filter YAML syntax, the boolean outputs per filter group, and the typical "detect-changes job → downstream jobs gated on `needs.detect.outputs.<filter>`" pattern. Pair with `actions/checkout` `fetch-depth: 0` for accurate diffs on push events.

## Study Tips

- **Build a tiny composite action while reading resource 4** — even 10 lines (checkout + setup-node + npm ci) cements the `runs.using: composite` + per-step `shell:` requirement faster than re-reading docs.
- **For caching, mentally separate two layers**: (a) the manual `actions/cache@v4` with hand-crafted keys, and (b) the `setup-*` actions' built-in `cache:` parameter. Pick (b) by default; reach for (a) only for non-language artifacts (Docker layers, Terraform plugins, custom build outputs).
- **When reading v4 artifact docs, look for the words "immutable" and "scoped to job"** — those two changes drive every breaking-change symptom you'll see when upgrading older workflows.
- **Treat `action.yml` as an API contract**: inputs are function parameters, outputs are return values, branding is the package.json. The same mental model applies whether the runtime is Node, Docker, or composite.

## Next Steps

Tomorrow (Day 3 of Week 6.8) covers GitHub Actions security and hardening: OIDC trust between GitHub and AWS/cloud providers (no long-lived keys), secret scoping, the `permissions:` block and minimal `GITHUB_TOKEN` scopes, third-party action pinning by SHA, and dependency review. After that, the natural progression is to author and publish your own custom action to the Marketplace (semver tagging, `release.yml`, auto-bumping major tag), then explore self-hosted runners and runner groups for compliance-sensitive workloads.
