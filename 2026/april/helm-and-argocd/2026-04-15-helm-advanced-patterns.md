# Helm Advanced Patterns -- Composing, Hooking, Distributing, Orchestrating, and Testing Charts at Scale

> Two days ago you internalized the **blueprint factory** mental model -- chart = blueprint, values = parameter sheet, template engine = CAD, release = the building. That model is enough to ship one workload from one chart, the way most Helm tutorials end. Real platforms do not look like that. A real platform chart bundles your application **plus** a Postgres subchart, a Redis subchart, a shared library of common helpers, a database-migration Job that must run before the new pods come up, a smoke-test Job that runs after, an OCI registry to publish the whole thing, and a Helmfile that orchestrates dev/staging/prod with one command. Each of those is a Helm subsystem in its own right, and each has a sharp edge that only shows up the third or fourth time you use it.
>
> The analogy that carries today's material is a **construction firm with a project-management division**. The single-blueprint architect from Day 1 still exists, but now they sit inside a larger firm. **Subcharts are sub-contractors** -- the architect's blueprint depends on a plumbing firm's blueprint and an electrical firm's blueprint, and the parent specs flow downward (the parent says "use copper pipe everywhere, paint everything beige") but the sub-contractors cannot read the parent's private notes. **Global values are the firm-wide style guide** -- the only document every sub-contractor automatically receives. **Hooks are the stage crew** for a play: pre-install hooks set the stage before the actors arrive (run database migrations, seed secrets), post-install hooks take down the scenery and check the lights worked (smoke tests). **Library charts are the firm's shared CAD-block library** -- standard door, standard window, standard label set -- pulled into every blueprint but never built on its own. **OCI registries are Docker Hub for blueprints**: the modern, signed, versioned distribution channel that has replaced the old HTTP filing cabinet. **Helmfile is docker-compose for Helm releases**: one declarative file that says "in dev install these eight charts with these values, in prod install nine charts with those values, and reconcile both with a single command." And **chart testing** -- `helm test`, `ct`, `helm-unittest` -- is the firm's QA department, with one team that walks finished buildings to check the doors open (`helm test`), one that lints every blueprint before it leaves the office (`ct`), and one that mathematically verifies "if the spec says 10 floors, the drawings will show 10 floors" without ever pouring concrete (`helm-unittest`).
>
> This doc walks through all of that, in order: dependencies and global values; the full hook lifecycle; library charts; OCI distribution and ECR specifics; Helmfile orchestration; and the three-layer testing stack. It assumes you already hold the Day 1 mental model. By the end you should be able to design a multi-chart platform release, write a safe DB-migration hook, publish a signed chart to ECR, orchestrate dev/staging/prod with Helmfile, and choose between `helm test`, `ct`, and `helm-unittest` for any given testing job.

---

**Date**: 2026-04-15
**Topic Area**: kubernetes
**Difficulty**: Intermediate-to-Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Subchart (dependency) | A sub-contractor (plumbing firm) the parent firm hires | A chart declared in `Chart.yaml` `dependencies:`; pulled into `charts/` by `helm dependency update`; values nest under the dep's name (or alias) |
| `helm dependency update` vs `build` | "Re-bid the contract" vs "use the bids already on file" | `update` re-resolves SemVer ranges and refreshes `Chart.lock`; `build` reinstalls the exact tarballs from an existing `Chart.lock` |
| `condition:` | A switch in the parent contract that turns a sub-contractor on/off | `condition: postgres.enabled` -- subchart loaded only when that value is `true`; a comma-separated list of paths in the single `condition:` string is evaluated first-found-wins |
| `tags:` | A category label that toggles a group of sub-contractors at once | `--set tags.observability=true` enables every dep tagged `observability` |
| `alias:` | Hiring the same plumbing firm for two separate floors under different names | Lets you depend on the same chart twice (two Redis instances `cache` and `queue`); values nest under the alias, not the original name |
| `import-values:` | Sub-contractor publishes a number the parent re-uses on the cover page | Surfaces a subchart's value into the parent's namespace so other templates can read it cleanly |
| `global:` | The firm-wide style guide every sub-contractor automatically gets | The ONLY values block that propagates from parent to all subcharts; standard pattern for image registry, pull secrets, env name |
| Value scoping rule | The parent has master keys to every sub-contractor's office, but no sub-contractor can see another's notes | Parent overrides subchart values via `<subchart>:` blocks; subcharts cannot see parent's non-`global` values; sibling subcharts cannot see each other |
| Hooks | The stage crew that runs before/after the play | Annotated K8s resources (`helm.sh/hook: pre-install`); run at lifecycle moments; **NOT tracked as part of the release** (gotcha) |
| Hook types | Crew shifts: setup, teardown, intermission checks | `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`, `test` |
| `helm.sh/hook-weight` | Stage-crew call sheet ordering | **Quoted string** holding an integer (e.g. `"-10"`), sorted ascending (negative-to-positive); ties broken alphabetically by resource name; sort happens within each hook phase |
| `helm.sh/hook-delete-policy` | "When do we strike the set?" | `before-hook-creation` (default), `hook-succeeded`, `hook-failed`; combine to control orphans |
| Library chart (`type: library`) | Firm's shared CAD-block library -- doors, windows, label patterns | `Chart.yaml` `type: library`; defines reusable templates only; cannot be installed standalone; depended on by application charts |
| OCI registry distribution | Docker Hub for blueprints | `helm push oci://...`, `helm pull oci://...`; charts stored as OCI artifacts; default modern model (Helm 3.8+ GA) |
| Classic HTTP repo | The old paper filing cabinet down the hall | `index.yaml` + `.tgz` files on an HTTP server; `helm repo add`; still supported, deprecated in spirit |
| Helmfile | docker-compose for Helm releases | `helmfile.yaml` declares many releases across environments; `helmfile apply` reconciles everything in one command |
| `helmfile diff` / `apply` / `sync` / `destroy` | "Show the diff," "make it match," "upgrade unconditionally," "tear it all down" | Four lifecycle commands wrapping `helm` operations across all releases. `apply` skips unchanged releases (diff-first); `sync` runs `helm upgrade --install` on every release regardless of diff -- Helm itself still merges safely |
| `environments:` (Helmfile) | Job-site profile (dev site / prod site) the firm switches between | Per-env value files and template variables selected via `helmfile -e prod` |
| `helm test` | QA walks the finished building and checks the doors open | Runs Pods/Jobs annotated `helm.sh/hook: test` against a deployed release; smoke-tests cluster-side reachability |
| `ct` (chart-testing) | The pre-shipping QA on every blueprint before it leaves the office | CLI for `lint` + `install` in CI; auto-detects changed charts via git; runs `helm install` against ephemeral kind/k3s clusters |
| `helm-unittest` (plugin) | The mathematical "if the spec says 10 floors, drawings show 10 floors" check | BDD-style YAML unit tests asserting rendered template output without a cluster |
| Umbrella chart | A general contractor whose only job is to wire sub-contractors together | A chart with little or no templates of its own, mostly `dependencies:`, used to bundle many components into one release |
| Post-renderer | An external editor who marks up the drawings just before delivery | An external binary (typically `kustomize build`) that processes Helm's rendered output before it reaches the cluster; `--post-renderer ./script.sh` |
| `lookup` function | Walking the construction site and measuring an existing wall | Reads live cluster state at render time; returns empty during `helm template` (no cluster); used to preserve auto-generated Secrets across upgrades |

---

## The Construction Firm Framework

Day 1 had one architect with one blueprint. Today, that architect works inside a firm:

1. **The General Contractor (Umbrella / Parent Chart)** -- A chart whose primary job is to bundle other charts together. Often has minimal `templates/` of its own. Owns the merged values, propagates `global:`, gates subcharts via `condition:`/`tags:`.

2. **The Sub-contractors (Subcharts / Dependencies)** -- Independently authored charts pulled in via `Chart.yaml` `dependencies:`. They live under `charts/` after `helm dependency update`. They can be remote charts (Bitnami Postgres, the Prometheus stack) or local sibling directories.

3. **The Shared CAD-Block Library (Library Chart)** -- A chart of pure helpers (`define` blocks). Marked `type: library`. Cannot be installed; can only be depended on by application charts that `include` its templates.

4. **The Stage Crew (Hooks)** -- Resources that run at specific lifecycle moments outside the main release content: DB migrations before the new app starts, smoke tests after install, cleanup Jobs before delete. **Not tracked as release resources.**

5. **The Distribution Channel (OCI Registry or HTTP Repo)** -- Where packaged charts live for others to pull. OCI registries (ECR, GHCR, Harbor, Docker Hub) are the modern default; HTTP `index.yaml` repos are the legacy form.

6. **The Project Management Division (Helmfile)** -- One file orchestrating many releases across many environments. The thing Helm itself deliberately does not do.

7. **The QA Department (Testing Stack)** -- Three teams with three different jobs: `helm test` (does the deployed building actually work?), `ct` (does this blueprint pass our lint/install bar in CI?), `helm-unittest` (does the rendered output match the spec, mathematically, without building anything?).

```
THE CONSTRUCTION FIRM -- HELM ADVANCED OVERVIEW
================================================================================

  GENERAL CONTRACTOR (umbrella chart)
  +------------------------------------------+
  |  Chart.yaml                              |
  |    dependencies:                         |
  |      - name: postgres   (sub-contractor) |
  |      - name: redis      (sub-contractor) |
  |      - name: common     (library chart)  |
  |  values.yaml                             |
  |    global: { imageRegistry, env, ... } --+--+ propagates DOWN to ALL subs
  |    postgres: { ... }       ---------------+
  |    redis: { auth.enabled: false } --------+
  |    cache: { ... }      (alias of redis) --+
  |  templates/                                |
  |    deployment.yaml (uses common.* helpers) |
  |  hooks:                                    |
  |    pre-upgrade Job (DB migrate, weight -10)|
  |    post-install Job (seed admin user, w 0) |
  |    test/ Pod (curl /healthz, hook: test)   |
  +------------------------------------------+
                |
                | helm dependency update / build
                v
            charts/
              postgres-15.2.tgz
              redis-18.6.1.tgz
              common-1.4.0.tgz   (library; templates only)

  PUBLISHING                       ORCHESTRATION                  TESTING
  ----------                       -------------                  -------
  helm package                     helmfile.yaml                  ct lint+install (CI)
  helm push oci://ECR              releases:[platform,monitoring] helm-unittest (no cluster)
  helm pull oci://ECR/chart        environments:[dev,staging,prod] helm test (post-deploy)
                                   helmfile apply -e prod
```

---

## Part 1: Subcharts and Dependencies -- Hiring Sub-contractors

### Declaring Dependencies in `Chart.yaml`

```yaml
apiVersion: v2
name: platform
description: Payments platform -- API + Postgres + Redis (cache & queue)
type: application
version: 2.1.0
appVersion: "2.8.1"

dependencies:
  - name: postgresql
    version: "~15.5.0"                           # SemVer range; ~15.5.0 means ">=15.5.0 <15.6.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled                # Subchart loaded only if .Values.postgresql.enabled is true
    tags:
      - storage

  - name: redis
    version: "18.6.1"
    repository: "https://charts.bitnami.com/bitnami"
    alias: cache                                 # Mounts subchart values under .Values.cache (NOT .Values.redis)
    condition: cache.enabled
    tags:
      - storage

  - name: redis                                  # Same chart, second time -- alias makes it a separate release
    version: "18.6.1"
    repository: "https://charts.bitnami.com/bitnami"
    alias: queue
    condition: queue.enabled
    tags:
      - storage

  - name: common                                 # Library chart (templates only, cannot install standalone)
    version: "1.4.0"
    repository: "oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts"
    # No condition -- always included; library charts are free at runtime

  - name: shared-cache                           # Local sibling subchart (no repository)
    version: "0.1.0"
    repository: "file://../shared-cache"
    import-values:                               # Surface a child value into the parent's namespace
      - child: service.port
        parent: cachePort                        # Parent templates can now read .Values.cachePort
```

### `helm dependency update` vs `helm dependency build`

```bash
# Re-resolve SemVer ranges, fetch newest matching versions, refresh Chart.lock
helm dependency update ./platform

# Use the EXACT versions in Chart.lock (don't re-resolve); fast and reproducible
helm dependency build ./platform
```

The mental model: `update` is "re-bid every contract from scratch," `build` is "use the bids we already filed." **CI should run `build`, not `update`** -- otherwise a new patch version of a Bitnami chart sneaks into your build silently the day Bitnami publishes it. Commit `Chart.lock` to git for the same reason `package-lock.json` is committed in Node projects: pinning is a security and reproducibility concern.

There is also a third command, `helm dependency list ./mychart`, which prints the resolved dependency tree (name, version, repository, status) without fetching anything -- handy for sanity-checking what `Chart.lock` says before a build.

**What's in `Chart.lock`**: the file records each dependency's resolved concrete version (e.g., `15.5.4`, not the range `~15.5.0`), the repository URL, and a content `digest:` (sha256 of the tarball). It also stamps a `generated:` timestamp and a top-level `digest:` covering the lockfile itself. Any manual edit or version drift in `Chart.yaml` invalidates the lockfile and forces a re-`update`.

The `charts/` directory after `helm dependency build`:

```
platform/
├── Chart.yaml
├── Chart.lock                  # Resolved exact versions + checksums
├── values.yaml
├── templates/
└── charts/
    ├── postgresql-15.5.4.tgz
    ├── redis-18.6.1.tgz
    ├── redis-18.6.1.tgz        # The same chart again -- it's an alias, but file is identical
    ├── common-1.4.0.tgz
    └── shared-cache/           # Local subchart copied as a directory, not tarballed
```

### Conditions vs Tags -- Two Switches with Different Shapes

A **condition** is a per-dependency switch tied to a single boolean values path:

```yaml
# In Chart.yaml
- name: postgresql
  condition: postgresql.enabled

# In values.yaml
postgresql:
  enabled: false                # Subchart skipped entirely; no resources rendered
```

A **tag** is a group switch -- multiple deps tagged the same can be flipped together:

```yaml
# In Chart.yaml
- name: postgresql
  tags: [storage, persistence]
- name: redis
  tags: [storage, cache]
- name: prometheus
  tags: [observability]

# In values.yaml or via --set
tags:
  storage: true                 # Enables postgresql AND redis
  observability: false          # Disables prometheus
```

**The resolution rule when both exist**: **if a `condition:` field is present on the dep and the path it points to resolves in values, the condition alone decides -- `tags:` are never consulted for that dep.** Tags are evaluated **only** when the dep has no condition, or when every path in the condition's comma-separated list is missing from values. Within the condition string itself, Helm walks the paths left-to-right and uses the first one it finds. The practical implication: you cannot disable a dep with `tags.storage=false` if its `condition: postgresql.enabled` is `true`. Pick one mechanism per dep and stick with it.

### Value Scoping -- The Master-Key Rule

This is the core mental model people get wrong:

**Parent values can override any subchart value, by nesting under the subchart's name (or alias).** Subchart values **cannot** read parent values. Sibling subcharts **cannot** read each other.

```yaml
# Parent values.yaml -- the parent is the master-key holder
replicaCount: 5                          # Parent's own value

postgresql:                              # Overrides INTO the postgresql subchart
  auth:
    database: payments
    username: payments_user
    existingSecret: payments-db-secret
  primary:
    persistence:
      size: 50Gi

cache:                                   # Overrides INTO the redis subchart aliased as `cache`
  auth:
    enabled: false
  master:
    persistence:
      size: 8Gi

queue:                                   # Overrides INTO the redis subchart aliased as `queue`
  auth:
    enabled: true
  master:
    persistence:
      size: 16Gi
```

A template inside the `postgresql` subchart that says `{{ .Values.replicaCount }}` will **NOT** see the parent's `5` -- it sees only its own values namespace. The only way to share data downward is `global:` (next section).

A template inside the `redis` subchart cannot read `.Values.postgresql.auth.database` -- subcharts are siblings and isolated. If you need cross-subchart values, surface them via `global:` or via `import-values:` from one subchart into the parent.

### Global Values -- The Firm-Wide Style Guide

`global:` is the **only** values block that propagates to every subchart's namespace automatically. Every subchart can reference `.Values.global.X`, and any value the parent puts under `global:` is visible everywhere.

```yaml
# Parent values.yaml
global:
  imageRegistry: 123456789.dkr.ecr.us-east-1.amazonaws.com
  imagePullSecrets:
    - name: ecr-pull-secret
  storageClass: gp3
  environment: prod
  commonLabels:
    team: payments
    cost-center: cc-4421
```

Inside the parent's templates: `{{ .Values.global.imageRegistry }}`. Inside the `postgresql` subchart's templates: also `{{ .Values.global.imageRegistry }}`. Bitnami charts are designed to honor `global.imageRegistry`, `global.imagePullSecrets`, and `global.storageClass` out of the box -- which is why setting these once in the umbrella's `global:` block reconfigures the entire bundle in one place.

**The catch -- the "parent must declare" rule**: subcharts only see global values that the **parent chart has declared in its own `values.yaml`**. Merely passing `--set global.foo=bar` at install time, or supplying a `values-prod.yaml` that sets `global.foo`, is **not enough on its own**. The parent's committed `values.yaml` has to include a `global:` block with `foo:` in it (even as an empty string or default) for the merge machinery to propagate it downward. Without that declaration, subchart references to `.Values.global.foo` silently render as `<no value>`, which Go's template engine emits without any error.

This is the #1 "why didn't my global get picked up?" support ticket on chart-author repos, and it is asked in interviews precisely because it tests whether you understand Helm's merge semantics rather than just "globals propagate."

**Production pattern**: declare every `global.*` key the chart actually uses in the parent's `values.yaml`, even with empty or default values -- both as documentation and as the literal trigger for subchart propagation. Pair this with `{{ required "global.foo must be set" .Values.global.foo }}` (or a sensible `| default`) in any template that depends on a global, and optionally with a `values.schema.json` that enforces the shape -- so misplaced keys, typos, and forgotten overrides fail loudly at install time rather than rendering as `<no value>` labels on production pods.

### `import-values` -- Surfacing Subchart Values Upward

The reverse direction (subchart -> parent) is opt-in and explicit:

`import-values:` has **two forms**, with different requirements:

```yaml
# In parent Chart.yaml
dependencies:
  - name: postgresql
    version: "~15.5.0"
    repository: "https://charts.bitnami.com/bitnami"
    import-values:
      # FORM 1 -- explicit child/parent mapping (parent-driven; works against any subchart)
      - child: primary.service.ports.postgresql       # Subchart path
        parent: postgresPort                          # Becomes .Values.postgresPort in parent

      # FORM 2 -- shorthand (requires the child to opt in)
      - data
```

**Form 1 (explicit)** is parent-driven and works against any subchart, whether or not the subchart author anticipated imports.

**Form 2 (shorthand)** requires the **subchart** to define a top-level `exports:` block in its own `values.yaml`:

```yaml
# charts/postgresql/values.yaml -- child-side opt-in
exports:
  data:
    dbPort: 5432
    dbName: postgres
```

`- data` in the parent then pulls the entire contents of `exports.data` into the parent's namespace -- `.Values.dbPort` and `.Values.dbName` become readable at the parent's top level. The key insight: **shorthand imports need child-side declaration**; if the subchart has no `exports:` block, `- data` silently imports nothing.

Use `import-values` when the parent needs to wire one subchart's output into another's input (e.g., the API's ConfigMap needs the Postgres port the Postgres subchart chose). Prefer Form 1 for third-party charts you don't control; Form 2 is for charts you or your team author where you want a clean published surface.

### The Subchart Indentation Trap

The single most common subchart-override bug:

```yaml
# WRONG -- this sets a top-level postgresql.auth, not the subchart's auth
postgresql.auth.database: payments

# WRONG -- flat keys don't merge into nested subchart values
postgresql.auth:
  database: payments

# CORRECT -- proper nested override into the postgresql subchart's namespace
postgresql:
  auth:
    database: payments
```

The dot-notation in `--set` works (`--set postgresql.auth.database=payments`), but in YAML you must nest properly. This is one of the highest-frequency tickets on chart-author repos.

---

## Part 2: Hooks -- The Stage Crew

### What a Hook Is, Precisely

A hook is **a regular Kubernetes resource manifest with a special annotation** that makes Helm apply it at a specific lifecycle moment instead of as part of the main release content. The most common hook resources are `Job` (for one-shot tasks like migrations), `Pod` (for tests), `ConfigMap`/`Secret` (for hook-time data), and occasionally `ServiceAccount`/`Role`/`RoleBinding` (the RBAC the hook Job needs).

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "platform.fullname" . }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: {{ include "platform.serviceAccountName" . }}
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          command: ["alembic", "upgrade", "head"]
          envFrom:
            - secretRef:
                name: {{ include "platform.dbSecretName" . }}
```

### The Full Hook Catalog

| Hook | Fires at | Common use |
|------|---------|-----------|
| `pre-install` | Before any chart resource is rendered & applied | Create namespace, seed Secrets, validate environment |
| `post-install` | After all main resources are **applied**; fires after readiness ONLY if `--wait` is set | Smoke test, register with external system, seed admin user (use `--wait` if the hook depends on pods being ready) |
| `pre-upgrade` | Before any new manifest is applied during upgrade | **DB schema migrations** (the canonical hook use case) |
| `post-upgrade` | After upgrade applies and resources are ready | Post-migration reindex, cache warming, alert "deploy complete" |
| `pre-delete` | Before any resource of the release is deleted | Drain queues, decommission external integrations |
| `post-delete` | After all release resources are deleted | Clean up out-of-cluster state (DNS, S3 buckets, IAM roles) |
| `pre-rollback` | Before rollback applies the prior revision | Reverse a forward-only migration if you have one |
| `post-rollback` | After rollback applies | Notify, verify health on the rolled-back version |
| `test` | On `helm test <release>` (NOT during install/upgrade) | Smoke tests, integration sanity checks |

A single resource can have multiple hook phases: `"helm.sh/hook": pre-install,pre-upgrade` runs the same Job for both lifecycle events -- the standard pattern for migrations.

### Hook Weights -- The Call Sheet

Within a single hook phase, resources are sorted by `helm.sh/hook-weight` **ascending** (negative first, then zero, then positive). Ties break **alphabetically by resource name**.

```yaml
# Three pre-install hooks, executed in this order:
# 1. namespace-bootstrap      (weight -20)
# 2. db-migrate               (weight -10)
# 3. seed-admin-user          (weight 0, no annotation given)
```

The mental model: think of weights like Kubernetes init containers, but for hooks. Lower weight = runs sooner. Use negative numbers for "infrastructure" hooks (namespaces, secrets), zero/positive for application logic.

**Gotcha**: the weight annotation value must be a **quoted string** -- `"-10"`, not `-10`. YAML will parse an unquoted `-10` as an integer, and Helm rejects non-string annotation values. This is one of the most common first-hook bugs.

**Helm waits for each hook resource to reach ready/complete state before moving to the next weight bucket.** A failed hook (Job's backoffLimit exhausted) aborts the install/upgrade.

### Hook Delete Policies -- "When Do We Strike the Set?"

Default behavior: **hook resources are deleted before the same hook is re-created, but otherwise persist forever.** A `pre-upgrade` Job from upgrade #5 lingers in the namespace until upgrade #6 runs and Helm deletes the old one to make room.

```yaml
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Three policies, comma-separated:

| Policy | Meaning |
|--------|--------|
| `before-hook-creation` (default) | Delete the previous hook resource before creating a new one with the same name |
| `hook-succeeded` | Delete the hook resource as soon as it succeeds |
| `hook-failed` | Delete the hook resource as soon as it fails |

The recommended production combination for migration Jobs is `before-hook-creation,hook-succeeded`: clean up successful hooks immediately to keep the namespace tidy, but leave failed hooks in place so you can `kubectl logs` them and figure out why they failed.

### **Hooks Are NOT Release Resources** -- The Big Gotcha

This is the single most-bitten production trap with hooks:

> A hook resource is **owned by the hook annotation, not by the release**. `helm uninstall` does NOT delete hook resources unless you have set `hook-delete-policy: hook-succeeded` (which already deleted them) or `before-hook-creation` (which only deletes "before next creation," and there is no next creation on uninstall).

Concretely:

- A `pre-install` Job that ran successfully on revision 1 sits in the cluster forever after `helm uninstall` unless you set delete policies. It accumulates.
- `helm rollback` does NOT re-run a `pre-upgrade` hook or automatically reverse its side effects. It only runs hooks specifically annotated `pre-rollback` / `post-rollback`. In principle you could author a separate `pre-rollback` Job that runs a reverse migration -- but in practice no one does, because keeping inverse migrations in sync with forward ones is operationally miserable. The accepted pattern is **expand-and-contract**: every migration is backward-compatible with the previous app version during its deploy window, so `helm rollback` is always safe without custom reverse hooks. Forward-only migrations paired with `helm rollback` are a classic source of "rollback succeeded but the app is broken because the DB schema is now ahead of the old code."
- `helm get hooks <release>` shows the hook **definitions** in the release Secret, not whether the hook resources currently exist in the cluster.

**Production rules**:

1. **Always set `hook-delete-policy: before-hook-creation,hook-succeeded`** on Job hooks unless you have a specific reason to keep them.
2. **Forward-only migrations need rollback strategy outside Helm.** If your migrations are expand-then-contract, document that rolling back across a contract migration is a manual operation -- not a `helm rollback` away.
3. **`pre-delete` hooks need to be safe to re-run.** A user can always `helm uninstall` then reinstall without state in between.

### Hook Timeline for Install / Upgrade / Rollback

```
INSTALL:
  CRDs in crds/ (no templating)
    -> pre-install hooks (sorted by weight, executed in order)
    -> main resources (rendered, applied; Helm only waits for readiness if --wait is set)
    -> post-install hooks (sorted by weight)
        # WITHOUT --wait: fires as soon as kubectl apply returns, BEFORE pods are ready
        # WITH --wait: fires after all main resources reach ready state
        # Canonical smoke-test-in-post-install pattern REQUIRES --wait (or helmDefaults.wait: true)
    -> NOTES.txt printed

UPGRADE:
    -> pre-upgrade hooks (sorted by weight)
        # CRITICAL: these see the CURRENT live cluster state, NOT the new manifests yet
    -> main resources (3-way diff: live, last-applied, new) applied
    -> post-upgrade hooks (sorted by weight)
    -> NOTES.txt printed

ROLLBACK:
    -> pre-rollback hooks (sorted by weight)
    -> main resources from the target revision applied
    -> post-rollback hooks (sorted by weight)

UNINSTALL:
    -> pre-delete hooks (sorted by weight)
    -> main release resources deleted
    -> post-delete hooks (sorted by weight)
    -> release Secret deleted (revision history gone)

helm test <release>:
    -> test hooks executed against existing release (Pods/Jobs)
    -> NOT triggered automatically by install/upgrade
```

**The single most common production bug**: a `pre-upgrade` migration Job that needs the new ConfigMap or Secret to read DB credentials -- but the new ConfigMap doesn't exist yet because the hook runs BEFORE the main resources. Fix: either reference a Secret that already exists from a prior revision, or split the migration into two steps (deploy the Secret as a normal resource, then run the migration as `post-upgrade`).

### Common Hook Patterns

**DB migrations** (the canonical pattern):

```yaml
"helm.sh/hook": pre-upgrade,pre-install
"helm.sh/hook-weight": "-5"
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

**Smoke test after deploy**:

```yaml
"helm.sh/hook": post-install,post-upgrade
"helm.sh/hook-weight": "10"
"helm.sh/hook-delete-policy": hook-succeeded
```

**`helm test` smoke check** (run on demand via `helm test`):

```yaml
"helm.sh/hook": test
"helm.sh/hook-delete-policy": hook-succeeded
```

**Pre-delete drain** (graceful uninstall):

```yaml
"helm.sh/hook": pre-delete
"helm.sh/hook-weight": "0"
"helm.sh/hook-delete-policy": hook-succeeded
```

---

## Part 3: Library Charts -- The Shared CAD-Block Library

### What a Library Chart Is

A library chart is a chart whose `Chart.yaml` declares `type: library`. It contains **only** named templates (`define` blocks) -- typically in `templates/_helpers.tpl`, `_labels.tpl`, `_deployment.tpl`, etc. It has no `values.yaml` of its own that produces resources, no manifest templates, no main resources.

```yaml
# common/Chart.yaml
apiVersion: v2
name: common
description: Shared template helpers for the platform team's charts
type: library
version: 1.4.0
```

```yaml
# common/templates/_labels.tpl
{{- define "common.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
team: platform
{{- end -}}

{{- define "common.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{- define "common.securityContext" -}}
runAsNonRoot: true
runAsUser: 65532
fsGroup: 65532
seccompProfile:
  type: RuntimeDefault
{{- end -}}
```

### Consuming a Library Chart

An application chart depends on the library chart like any other dependency:

```yaml
# payments-api/Chart.yaml
apiVersion: v2
name: payments-api
type: application
version: 1.0.0
appVersion: "2.8.1"

dependencies:
  - name: common
    version: "1.4.0"
    repository: "oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts"
```

```yaml
# payments-api/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "payments-api.fullname" . }}
  labels:
    {{- include "common.labels" . | nindent 4 }}      # From the library
spec:
  selector:
    matchLabels:
      {{- include "common.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "common.selectorLabels" . | nindent 8 }}
    spec:
      securityContext:
        {{- include "common.securityContext" . | nindent 8 }}
      containers:
        # ...
```

`helm dependency build` pulls the library tarball into `charts/`. The library's `define` blocks become available in the parent's templates via `include`. **No resources from the library are rendered** -- it is pure helper code.

### Why Library Charts Win at Scale

Imagine a platform team running 50 microservice charts, each with its own copy of `_helpers.tpl` defining labels, securityContexts, resource defaults, sidecar specs. When you need to add `cost-center: cc-4421` to every label set, you have 50 PRs to write. With a library chart, you bump `common` from 1.4.0 to 1.5.0, the team bumps the dependency in each chart, done.

The Bitnami `bitnami/common` library chart is the canonical reference implementation -- it ships with helpers for image references, image-pull secrets, capabilities, validation, common annotations, and dozens of other patterns. Most production Bitnami charts depend on it.

### Namespace Your Template Names -- The Silent-Collision Trap

Named templates live in a **flat, global namespace across the entire release** -- including the parent chart, every subchart, and every library chart depended on. A `{{- define "labels" -}}` in your library chart collides with any other `define "labels"` anywhere in the release tree, with "last one wins" semantics that depend on template load order. There is no error, no warning, and the collision is invisible until labels start rendering wrong on one of the subcharts.

**Always namespace your library templates** with the library chart's name as a prefix:

```yaml
# common/templates/_labels.tpl -- GOOD (namespaced)
{{- define "common.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
...
{{- end -}}

# BAD -- will collide with any other `define "labels"` in the release
{{- define "labels" -}}
...
{{- end -}}
```

The Bitnami `bitnami/common` library is the canonical reference -- every helper in it is prefixed (`common.labels`, `common.images.image`, `common.capabilities.kubeVersion`). Follow the same convention: `<chart-name>.<helper-purpose>`.

### Library vs Application -- The Decision

| Question | Library chart | Application chart |
|---------|--------------|-------------------|
| Does it produce K8s resources? | No | Yes |
| Can `helm install` it directly? | No (errors out) | Yes |
| Can it have a `values.yaml`? | Yes (for helper inputs) but defaults are usually the right path | Yes (drives the entire release) |
| Can other charts depend on it? | Yes -- this is its whole purpose | Yes (becomes a subchart) |
| Does it have hooks? | No -- hooks are resources | Yes |

**Heuristic**: extract a library chart the third time you copy-paste the same `_helpers.tpl` into a new application chart. Earlier than that and you over-engineer; later and you've got drift.

---

## Part 4: OCI Registries -- Docker Hub for Charts

### Why OCI

The traditional Helm distribution model is an HTTP server hosting an `index.yaml` file plus chart `.tgz` tarballs. It works, but it duplicates infrastructure most teams already operate -- **container registries**. ECR, GHCR, Harbor, Docker Hub, Azure Container Registry, Quay -- they all already store, version, sign, and authenticate access to OCI artifacts. Helm 3.8 (May 2022, GA) adopted OCI as a first-class distribution mechanism: charts are pushed and pulled as OCI artifacts alongside container images.

### The Commands

```bash
# Authenticate to the registry (one-time per session)
helm registry login -u AWS \
  --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com \
  <<< "$(aws ecr get-login-password --region us-east-1)"

# Package the chart
helm package ./payments-api
# -> payments-api-1.4.2.tgz

# Push to OCI registry
helm push payments-api-1.4.2.tgz \
  oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts

# Pull (you almost always pull directly via install instead)
helm pull oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts/payments-api \
  --version 1.4.2

# Install directly from OCI -- no `helm repo add` needed
helm install payments \
  oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts/payments-api \
  --version 1.4.2 \
  -f values-prod.yaml
```

### OCI vs Classic HTTP Repos -- The Comparison

| Dimension | OCI registry | Classic HTTP repo |
|-----------|-------------|-------------------|
| `helm repo add` needed? | No -- install with full `oci://...` URL | Yes -- repo must be added before search/install |
| Index file | None -- registry API is the index | `index.yaml` regenerated after each upload |
| Authentication | Standard registry auth (Docker config, IAM, OIDC) | HTTP basic auth or unauthenticated; bespoke per host |
| Signing | Cosign, Notary v2 -- standard OCI tooling | `helm package --sign` with PGP; less ecosystem support |
| `--version` flag on install | **Mandatory** -- no "latest" implicit | Optional -- omitting picks the newest in `index.yaml` |
| `helm search repo X` | Not supported -- registry API doesn't expose search | Supported via cached `index.yaml` |
| Tag immutability | Registry-level (ECR, GHCR support immutable tags) | Filesystem-level (replace the tarball, refresh index) |
| Ecosystem direction | Default forward direction (Helm 3.8+ GA) | Legacy; still supported, no new features |

### ECR Specifics -- The AWS Path

```bash
# 1. Create the repo (charts are stored as OCI artifacts in standard ECR repos)
aws ecr create-repository --repository-name charts/payments-api --region us-east-1

# 2. Authenticate (12-hour token)
aws ecr get-login-password --region us-east-1 \
  | helm registry login --username AWS --password-stdin \
      123456789.dkr.ecr.us-east-1.amazonaws.com

# 3. Push
helm push payments-api-1.4.2.tgz \
  oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts

# Resulting artifact location:
# 123456789.dkr.ecr.us-east-1.amazonaws.com/charts/payments-api:1.4.2
```

IAM permissions required for the pushing identity: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`. For pulling: `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`. The same set you use for container images -- which is the whole point.

### Signing & Provenance

For supply-chain integrity, charts can be signed and verified:

- **Classic HTTP repos**: `helm package --sign --key "name" --keyring ~/.gnupg/secring.gpg ./mychart` produces both `mychart-1.4.2.tgz` and `mychart-1.4.2.tgz.prov` (the provenance file, an OpenPGP signature over the chart digest and metadata). Consumers run `helm verify mychart-1.4.2.tgz` or `helm install --verify` to check the signature against their keyring. GPG-based and not widely adopted in practice.
- **OCI registries**: use the standard OCI supply-chain tooling -- **Cosign** (`cosign sign` / `cosign verify` on the OCI artifact digest) or **Notary v2**. This is the same tooling teams already use for container images, which is the point. Charts pushed to ECR/GHCR/Harbor can be cosigned in the same CI step that signs your container images.

One paragraph is enough to know the territory -- most teams either skip signing entirely (bad) or adopt Cosign once they've standardized it for images (better).

### OCI Gotchas

1. **`helm repo add` does not work for OCI URLs.** OCI is "registry-direct" -- you reference the full `oci://...` URL on every install, or you use Helmfile/ArgoCD which abstract this.
2. **OCI charts have no implicit `latest` tag.** Unlike classic HTTP repos (where omitting `--version` picks the newest in `index.yaml`), OCI registries have no index, so Helm cannot pick a version for you. You must pass `--version X.Y.Z` on `helm pull oci://...` / `helm install oci://...` -- or, as an anti-pattern, push a chart under a `latest` tag yourself. Explicit is the right answer.
3. **Authentication is per-registry-host.** `helm registry login` writes to `~/.config/helm/registry/config.json` (or your Docker config). On CI runners without persistent state, you must `helm registry login` in every job.
4. **ECR authorization tokens expire after 12 hours.** The token returned by `aws ecr get-login-password` is valid for 12h; a long-running CI job that re-pulls after expiry fails with `401`. Fix: re-run `aws ecr get-login-password | helm registry login` immediately before each push/pull operation (don't cache the login), or use the `amazon-ecr-credential-helper` / `ecr-login` Docker credential helper so the token is refreshed transparently on every registry call.
5. **Tag immutability is opt-in on ECR.** Enable it on your charts repo to prevent silent overwrites of `1.4.2`.

---

## Part 5: Helmfile -- docker-compose for Helm Releases

### The Problem Helm Itself Doesn't Solve

`helm install` deploys one release. A real platform deploys many: the application, the cache, the message broker, the metrics stack, the ingress controller, the certificate manager, the secrets manager. You don't want to write a bash script with eight `helm upgrade --install` lines and pray they stay in order. You want **declarative orchestration**: "in this environment these are the desired releases, reconcile."

That is Helmfile. The `helmfile.yaml` file declares releases, environments, and shared values, and `helmfile apply` makes the cluster match the file.

### The `helmfile.yaml` Structure

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts
  - name: ecr-charts
    url: 123456789.dkr.ecr.us-east-1.amazonaws.com/charts
    oci: true                              # Helmfile v0.150+ also accepts `type: oci` (the newer spelling)

environments:
  default:
    values:
      - environments/dev.yaml
  staging:
    values:
      - environments/staging.yaml
  prod:
    values:
      - environments/prod.yaml
    secrets:
      - secrets/prod.enc.yaml          # SOPS-encrypted; auto-decrypted by Helmfile

# Helpers shared across releases
helmDefaults:
  atomic: true
  cleanupOnFail: true
  wait: true
  waitForJobs: true
  timeout: 600
  historyMax: 5
  createNamespace: true

releases:
  - name: cert-manager
    namespace: cert-manager
    chart: jetstack/cert-manager
    version: v1.14.5
    set:
      - name: installCRDs
        value: true

  - name: kube-prometheus-stack
    namespace: monitoring
    chart: prometheus-community/kube-prometheus-stack
    version: 58.1.4
    values:
      - values/kube-prometheus-stack/{{ .Environment.Name }}.yaml
    needs:
      - cert-manager/cert-manager       # Install ordering: this release waits for cert-manager

  - name: payments
    namespace: payments
    chart: ecr-charts/payments-api
    version: 1.4.2
    values:
      - values/payments/common.yaml
      - values/payments/{{ .Environment.Name }}.yaml
    set:
      - name: image.tag
        value: {{ .Values.imageTag | default "2.8.1" }}
    needs:
      - monitoring/kube-prometheus-stack
```

Key elements:

- **`repositories:`** -- equivalent to running `helm repo add` for each. OCI repos are flagged with `oci: true` (legacy) or `type: oci` (Helmfile v0.150+); both work during the transition.
- **`environments:`** -- per-env value files and (optional) SOPS-encrypted secret files. Switch with `helmfile -e prod`.
- **`helmDefaults:`** -- options applied to every release unless overridden, equivalent to passing them on every `helm upgrade`.
- **`releases:`** -- the desired list of releases. Each maps directly to a `helm upgrade --install` invocation.
- **`needs:`** -- declares ordering between releases. `payments` waits for `monitoring/kube-prometheus-stack`, which waits for `cert-manager/cert-manager`. Helmfile parallelizes where possible and serializes where `needs:` constrains.

### The Lifecycle Commands

```bash
# Show the diff between desired (helmfile.yaml) and actual (cluster) for ALL releases
helmfile -e prod diff

# Reconcile -- install/upgrade what's missing or out of date
helmfile -e prod apply

# Upgrade unconditionally -- run `helm upgrade --install` on EVERY release,
# regardless of whether it has drifted. Helm itself still does a 3-way merge,
# so unchanged resources aren't touched; this is NOT a force-overwrite of cluster state.
helmfile -e prod sync

# Tear down everything declared in this helmfile
helmfile -e prod destroy

# Limit to a subset of releases via labels or names
helmfile -e prod -l name=payments apply
helmfile -e prod --selector tier=app diff
```

`apply` is the "GitOps-shaped" command -- it runs `helmfile diff` first (via the `helm-diff` plugin) and only runs `helm upgrade --install` on releases whose desired state has actually changed. `sync` unconditionally runs `helm upgrade --install` on every release regardless of diff; Helm's own 3-way merge still protects unchanged resources, so `sync` is not a brute-force overwrite of cluster state, but it is much slower and obscures what changed.

**The one thing `sync` does that `apply` doesn't**: revert manual cluster drift on **chart-owned fields**. If someone runs `kubectl scale deployment api --replicas=10` (chart specifies `replicas: 3`), `apply` sees no Git-level diff and skips the release -- cluster stays at 10. `sync` runs `helm upgrade --install` anyway, which via the 3-way merge snaps `replicas` back to 3. **Caveat**: this only works for fields the chart explicitly owns. If `replicas` is removed from the chart (delegated to an HPA), Helm no longer owns that field and `sync` won't touch it either. Drift revert is bounded to chart-owned fields, not live cluster state in general.

**Recommendation**: use `apply` in CI for speed. Handle drift with a separate cadence -- a scheduled `helmfile diff` or `sync` job, or a GitOps controller (ArgoCD / Flux) that continuously reconciles desired state from git. Don't conflate deploys and drift reconciliation into one command on every merge.

### Environment Templating

Every value in the helmfile is templated against the current `.Environment` context, which lets you do conditional logic on the env name:

```yaml
releases:
  - name: payments
    namespace: payments
    chart: ecr-charts/payments-api
    version: 1.4.2
    values:
      - values/payments/{{ .Environment.Name }}.yaml
    {{- if eq .Environment.Name "prod" }}
    set:
      - name: replicaCount
        value: 20
      - name: autoscaling.maxReplicas
        value: 100
    {{- end }}
```

### When to Use Helmfile vs Umbrella Chart vs ArgoCD ApplicationSets

This is one of the highest-stakes architecture choices in K8s deployment:

| Tool | Strength | Best fit |
|------|----------|---------|
| **Umbrella chart** | Versioned bundle of related charts as a single distributable artifact | "Our platform is a product; users install it as one chart with one version" (e.g., the Prometheus stack, GitLab) |
| **Helmfile** | Declarative orchestration across many releases, fast local iteration | Developer-facing local dev environments, CI deployment scripts, "stand up the whole stack with one command" |
| **ArgoCD ApplicationSets** | GitOps reconciliation in production -- continuous, drift-detecting, multi-cluster | Production multi-env, multi-cluster deployments where the cluster pulls desired state from git |

In practice, the trio is often used together: developers run Helmfile locally for fast iteration; CI runs `helmfile apply` against ephemeral test clusters for integration; ArgoCD ApplicationSets reconcile production from git-committed Application manifests. **Don't pick a winner -- pick the right tool per layer.**

---

## Part 6: Chart Testing -- Three Teams, Three Jobs

### `helm test` -- Post-Deploy Smoke Tests

`helm test <release>` runs Pods/Jobs annotated `helm.sh/hook: test` against an existing release. The tests live in `templates/tests/` by convention and are **not applied** during `helm install` -- only when you explicitly invoke `helm test`.

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "payments-api.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: curl
      image: curlimages/curl:8.6.0
      command:
        - sh
        - -c
        - |
          set -eux
          curl -fsS http://{{ include "payments-api.fullname" . }}/healthz
          curl -fsS http://{{ include "payments-api.fullname" . }}/readyz
```

```bash
helm install payments ./payments-api -f values-prod.yaml
helm test payments
# Pod: payments-api-test-connection
# NAME       LAST DEPLOYED  STATUS    REVISION  TEST SUITE
# payments   2026-04-15 ... deployed  1
#
# Phase:        Succeeded
```

**Use case**: "the deployment finished and resources are running -- can the Service be reached, can the DB be queried, does the health endpoint return 200?" `helm test` is for **post-deploy reachability**, not for testing template correctness.

### `ct` (chart-testing) -- CI Lint + Install

`ct` (https://github.com/helm/chart-testing) is the CLI used in chart-author CI pipelines. It auto-detects which charts in a repo changed since a base ref, then runs `helm lint` and `helm install` on each against an ephemeral cluster (typically kind, k3s, or minikube).

```yaml
# .github/workflows/lint-test.yaml
name: Lint and Test Charts
on: pull_request

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: azure/setup-helm@v4
      - uses: helm/chart-testing-action@v2.6.1
      - name: Run chart-testing (lint)
        run: ct lint --target-branch main --chart-dirs charts
      - uses: helm/kind-action@v1.10.0
      - name: Run chart-testing (install)
        run: ct install --target-branch main --chart-dirs charts
```

```yaml
# ct.yaml -- chart-testing config
chart-dirs:
  - charts
target-branch: main
chart-repos:
  - bitnami=https://charts.bitnami.com/bitnami
helm-extra-args: --timeout 600s
upgrade: true                          # Also test upgrades from the previous version
remote: origin
validate-maintainers: false
```

`ct install` also tests **upgrades**: if the chart's previous version exists in the chart's `Chart.yaml` history (or a previous git tag), `ct` installs the previous version and upgrades to the new one to catch breaking changes.

### `helm-unittest` -- True Unit Tests, No Cluster

`helm-unittest` (https://github.com/helm-unittest/helm-unittest) is a Helm plugin that runs **BDD-style YAML assertions against rendered template output**. No cluster required, no `helm install`, just a deterministic check that "if I set value X, the rendered Deployment has field Y."

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest
helm unittest ./payments-api
```

```yaml
# payments-api/tests/deployment_test.yaml
suite: deployment
templates:
  - templates/deployment.yaml
tests:
  - it: should render a Deployment with the right kind and name
    set:
      image.tag: "2.8.5"
    asserts:
      - isKind:
          of: Deployment
      - equal:
          path: metadata.name
          value: RELEASE-NAME-payments-api
      - equal:
          path: spec.template.spec.containers[0].image
          value: ghcr.io/example/payments-api:2.8.5

  - it: should default tag to chart appVersion when image.tag is empty
    set:
      image.tag: ""
    asserts:
      - matchRegex:
          path: spec.template.spec.containers[0].image
          pattern: "ghcr.io/example/payments-api:.+"

  - it: should set replicaCount from values
    set:
      replicaCount: 7
    asserts:
      - equal:
          path: spec.replicas
          value: 7

  - it: should fail render when database.url is missing
    set:
      database.url: ""
    asserts:
      - failedTemplate:
          errorMessage: "database.url is required"

  - it: should add HPA when autoscaling enabled
    template: templates/hpa.yaml
    set:
      autoscaling.enabled: true
      autoscaling.maxReplicas: 25
    asserts:
      - isKind:
          of: HorizontalPodAutoscaler
      - equal:
          path: spec.maxReplicas
          value: 25
```

Common assertions: `isKind`, `equal`, `notEqual`, `matchRegex`, `contains`, `notContains`, `failedTemplate`, `matchSnapshot`. Snapshots (`matchSnapshot`) record rendered output to disk on first run and fail when subsequent runs differ -- the same pattern as Jest snapshots.

### The Three-Tool Decision

| Tool | Runs against | Catches | Where it lives |
|------|-------------|---------|---------------|
| `helm-unittest` | Rendered templates (no cluster) | "Did setting value X change field Y?" -- spec adherence, regression on render output | Pre-commit, fast CI lane, chart author's local loop |
| `ct lint` + `ct install` | Real ephemeral cluster (kind/k3s in CI) | "Will this chart actually install? Does upgrade from previous version succeed?" | Per-PR CI, before publishing chart |
| `helm test` | Live deployed release | "Is the deployed app reachable? Does the smoke test pass against real cluster?" | Post-deploy in pipelines, on-demand for live releases |

You want all three in a serious chart pipeline: `helm-unittest` for correctness of templating logic, `ct` for "does it install at all," `helm test` for "does the deployed thing work."

---

## Part 7: Two Bonus Patterns You Will Need

### Post-Renderers -- Patching Charts You Cannot Fork

When you depend on an upstream chart that lacks a value you need (no way to add a sidecar, no way to set a particular label, no way to inject a securityContext), forking the chart is the heavy hammer. The lighter touch is a **post-renderer**: an external binary that takes Helm's rendered output on stdin, transforms it, and emits the final manifests on stdout.

The most common post-renderer is `kustomize build`:

```bash
#!/usr/bin/env bash
# kustomize-postrender.sh
cat <&0 > base/all.yaml
kustomize build .
```

```yaml
# kustomization.yaml
resources:
  - base/all.yaml
patches:
  - target:
      kind: Deployment
      name: payments
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/securityContext
        value:
          runAsNonRoot: true
          runAsUser: 65532
```

```bash
helm install payments ./payments-api \
  --post-renderer ./kustomize-postrender.sh
```

Use this whenever you'd otherwise be tempted to fork an upstream chart. ArgoCD natively supports post-renderers via `kustomize` integrations on `Application` resources.

### `lookup` -- Reading Live Cluster State

The `lookup` template function reads existing cluster objects at render time. The classic use case: preserve a Helm-generated Secret across upgrades so a new random password isn't generated each deploy:

```yaml
{{- $existing := lookup "v1" "Secret" .Release.Namespace "payments-app-secret" -}}
{{- $password := "" -}}
{{- if $existing -}}
  {{- $password = index $existing.data "password" | b64dec -}}
{{- else -}}
  {{- $password = randAlphaNum 32 -}}
{{- end -}}
apiVersion: v1
kind: Secret
metadata:
  name: payments-app-secret
type: Opaque
stringData:
  password: {{ $password | quote }}
```

The catch you already met on Day 1: **`lookup` returns an empty `dict` during `helm template`** because there's no cluster to query. So this template renders to a freshly-generated password every time you run `helm template`, but preserves the existing one when you actually run `helm install`/`upgrade`. Plan testing accordingly -- assertions on `lookup`-dependent output need a real cluster.

---

## When-to-Use-What -- The Decision Frameworks

### Subchart vs Library Chart vs Umbrella Chart

```
Need to share TEMPLATE DEFINITIONS across many charts (no resources)
   -> Library chart (type: library)

Need to bundle MANY CHARTS into one installable unit
   -> Umbrella chart (parent with mostly dependencies, few/no own templates)

Need to compose ONE app + its dependencies (DB, cache) as one release
   -> Application chart with subchart dependencies

Need to deploy MANY independent releases together with ordering and per-env config
   -> Helmfile (NOT an umbrella chart -- each release stays independent)
```

### `helm test` vs `ct` vs `helm-unittest`

```
"Did I render the right YAML for this value?"          -> helm-unittest
"Will my chart install on a real cluster?"             -> ct install (CI)
"Did my chart pass lint?"                              -> ct lint or helm lint
"Does the deployed release work end-to-end?"           -> helm test
"Did upgrade from previous version succeed?"           -> ct install (--upgrade)
```

### Helmfile vs Umbrella Chart vs ArgoCD ApplicationSets

```
Distribute as a single product/version artifact        -> Umbrella chart
Local dev / CI orchestration of many releases          -> Helmfile
GitOps reconciliation in production multi-cluster      -> ArgoCD ApplicationSets
Simple single-app deployment                            -> Plain helm install
```

### OCI vs Classic HTTP Repo

```
New chart distribution, modern infra                   -> OCI (default)
Already running ChartMuseum or HTTP repo, no migration -> Classic HTTP
ArgoCD older versions or tooling that lacks OCI        -> Classic HTTP (check tooling support first)
Need helm search repo                                   -> Classic HTTP (OCI doesn't support it)
Want immutable tags, signing, IAM-based auth           -> OCI
```

---

## Production Gotchas and Interview Traps

**1. Hooks are not release resources -- they leak.**
The single most-cited Helm gotcha. A `pre-install` Job sits in the namespace forever after `helm uninstall` unless `hook-delete-policy` cleans it up. Always set `before-hook-creation,hook-succeeded` on every Job hook.

**2. `helm rollback` does NOT re-run hooks (unless they are `pre-rollback`/`post-rollback`).**
Roll back across a forward-only DB migration and the app can be broken on the rolled-back version because the schema is now ahead of the old code. Either make migrations expand-then-contract (always backward-compatible during the deploy window), or document that rollback across schema changes is a manual operation.

**3. Subchart value overrides must nest under the dep name -- or the alias if set.**
`postgresql.auth.database: payments` (flat) silently does nothing in YAML. You must write `postgresql: { auth: { database: payments } }`. With `alias: cache`, you must nest under `cache:`, not `redis:`. Cause #1 of "I set the value and nothing changed."

**4. Subcharts only see `global:` values the PARENT declared in its own `values.yaml`.**
This is Helm's most-asked interview gotcha. Passing `--set global.foo=bar` at install time, or supplying `values-prod.yaml` with a `global.foo`, is **not enough**: the parent chart's committed `values.yaml` must itself contain `global.foo` (even as an empty string) for the merge to propagate it to subcharts. Without that declaration, subchart references to `.Values.global.foo` silently render `<no value>` with no error. Always declare every `global.*` key your chart uses in the parent's `values.yaml`, even with empty defaults -- this is both documentation and the literal mechanism that makes the global visible to subcharts. Pair with `{{ required ... }}` or `| default` in templates, and a `values.schema.json` for schema validation.

**5. `helm dependency update` re-resolves SemVer ranges -- use `build` in CI.**
`update` will silently pick up a new patch version of a Bitnami chart the day Bitnami publishes it. CI must run `helm dependency build` against a committed `Chart.lock` for reproducibility.

**6. `condition:` always overrides `tags:` when present.**
If a dep has both `condition: postgresql.enabled` and `tags: [storage]`, Helm consults only the condition -- the `tags:` block is ignored for that dep. Setting `tags.storage=false` does NOT disable the dep when `postgresql.enabled=true`. Tags are evaluated for a dep only when no condition is set or when every path in the condition is missing from values. Pick one mechanism per dep.

**7. Library charts cannot be installed -- the error is helpful but obscure.**
`helm install foo ./common-library-chart` fails with "library charts are not installable." Yes, this is by design.

**8. OCI charts have no implicit `latest` tag.**
OCI registries have no chart index, so Helm cannot pick a version for you. You must pass `--version X.Y.Z` on `helm pull oci://...` / `helm install oci://...` -- omitting it fails. This bites users migrating from HTTP repos, where `index.yaml` lets Helm resolve a default version silently. Pushing a `latest` tag yourself is an anti-pattern; version explicitly.

**9. ECR auth tokens expire after 12 hours.**
The token from `aws ecr get-login-password` is good for 12h. Long-running CI jobs that re-pull charts mid-run after the token expires fail with `401 Unauthorized`. Fix: re-run `aws ecr get-login-password | helm registry login` immediately before each registry operation instead of logging in once at the start of the pipeline, or install the `amazon-ecr-credential-helper` so Docker/Helm refreshes tokens transparently on every call.

**10. Pre-upgrade hooks see the LIVE cluster state, not the new manifests.**
A `pre-upgrade` migration Job that needs the new ConfigMap to read DB credentials breaks because the new ConfigMap doesn't exist yet -- the hook ran first. Either reference a Secret from the prior revision, or restructure as `post-install,post-upgrade`.

**11. Helmfile `apply` vs `sync` -- pick `apply` in CI.**
`apply` runs `helmfile diff` first and only acts on releases whose desired state has changed. `sync` runs `helm upgrade --install` on every release unconditionally (Helm's own 3-way merge still guards unchanged resources, so it's not a brute-force overwrite -- just slower and noisier). `sync` obscures what actually changed; prefer `apply` in CI and use `sync` only to recover from a drifted `helm-diff` cache.

**12. `helm test` is NOT triggered by install/upgrade -- you must invoke it explicitly.**
Tests live in `templates/tests/` annotated `helm.sh/hook: test`, but `helm install` does not run them. CI must add `helm test <release>` as a separate step.

**13. `helm-unittest` snapshots break on chart-version bumps.**
If your snapshot includes the chart version in `helm.sh/chart` labels, every `Chart.yaml` version bump invalidates every snapshot. Regenerate with `helm unittest -u .` before committing -- but always review the diff to make sure only the version line changed.

**14. Library charts must NOT have a `templates/` file without an underscore prefix.**
A library chart with `templates/foo.yaml` (no leading underscore) tries to render `foo.yaml` as a manifest -- and fails because library charts can't produce resources. Every file in a library chart's `templates/` directory must start with `_`.

**15. Helmfile's `needs:` is per-run, not stateful.**
`needs:` declares ordering within a single `helmfile apply` invocation -- "within this run, install `monitoring/kube-prometheus-stack` before `payments`." It is **not** a cross-run or cross-environment dependency check; Helmfile does not track "does the prerequisite already exist in the target cluster." If you drop a `needs:` dep from the file, previously-installed prerequisites stay behind until you explicitly `helmfile destroy` or remove the release block. Don't confuse `needs:` with a dependency graph like Terraform's -- it's closer to step ordering in a single CI job.

**16. Post-renderers run AFTER Helm renders -- they don't see Helm's templating context.**
The post-renderer receives final YAML on stdin. It cannot access `.Values`, `.Chart`, or any Helm objects. Patches must be expressed in the post-renderer's own language (Kustomize patches, sed scripts, jq).

**17. `lookup` returns an empty `dict` during `helm template` and during the first `helm install`.**
First-install `lookup` of a Secret you intend to preserve finds nothing (the Secret doesn't exist yet) -- so the chart generates a new value. On the second install (if it ever happens), it preserves the existing value. The pattern works only for upgrades, not for cross-install state.

**18. Dependency order in `Chart.yaml` does NOT control install order.**
Subcharts are rendered together with the parent and applied as one batch. If you need ordering, use hooks with weights, or use `needs:` in Helmfile, or put subcharts in separate Helmfile releases.

---

## Key Takeaways

- **Subcharts are sub-contractors with isolated office space.** Parent overrides flow downward via `<subchart>:` blocks. Subcharts cannot read parent values. Sibling subcharts cannot read each other. The only cross-cutting bridge is `global:` (parent declares, all subcharts read), and the explicit upward bridge is `import-values:`.

- **`global:` is the firm-wide style guide.** The single values block that auto-propagates everywhere -- the standard pattern for image registry, pull secrets, environment, common labels. Always declare every `global.*` key your chart uses in the parent's `values.yaml`, even empty, as documentation and as the trigger for subchart visibility.

- **`helm dependency build` (not `update`) in CI.** Commit `Chart.lock` for reproducibility. SemVer ranges silently pull new patch versions otherwise.

- **Hooks are the stage crew, not part of the release.** They persist forever unless `hook-delete-policy` cleans them up. They do NOT re-run on rollback unless they are `pre-rollback`/`post-rollback`. They run at lifecycle moments against the live cluster state at that moment -- which is BEFORE the new manifests for `pre-*` hooks.

- **Always set `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded` on Job hooks.** Default behavior leaves orphans. The combination cleans up on success but leaves failures for debugging.

- **Library charts (`type: library`) are pure helpers.** Cannot install. Templates must all be `_`-prefixed. Used as a dep by application charts. Extract one once you've copy-pasted the same `_helpers.tpl` into a third chart -- the Bitnami `bitnami/common` library is the canonical reference.

- **OCI is the default chart distribution mechanism going forward.** `helm push oci://`, `helm install oci://...` -- no `helm repo add`, no `index.yaml`, standard registry auth and signing. ECR, GHCR, Harbor, Docker Hub, Azure CR all support it. Classic HTTP repos still work but are legacy. Remember `--version` is mandatory on OCI installs.

- **Helmfile is docker-compose for Helm releases.** Declarative orchestration of many releases across many environments with one command. Use `apply` (diff-first) in CI, `sync` (force) only for hard reconciliation. Composes well with ArgoCD: Helmfile for local dev and CI; ArgoCD for production GitOps.

- **Three testing tools, three jobs.** `helm-unittest` for "did the right YAML render?" (no cluster), `ct lint+install` for "does the chart install on a real cluster?" (CI), `helm test` for "does the deployed release actually work?" (post-deploy). A serious chart pipeline runs all three.

- **Umbrella vs Helmfile vs ArgoCD ApplicationSets is not either/or.** Umbrella chart for "this is one product with one version." Helmfile for orchestrating many independent releases at deploy time. ArgoCD ApplicationSets for production GitOps reconciliation across clusters. The same platform often uses all three at different layers.

- **Post-renderers (`--post-renderer`) are the escape hatch when you can't fork an upstream chart.** Pipe Helm's rendered output through Kustomize (or any binary) to add sidecars, securityContexts, labels the chart didn't expose. ArgoCD supports this natively.

- **`lookup` reads live cluster state but is empty during `helm template`.** Useful for preserving auto-generated Secrets across upgrades. The first install always sees an empty result -- pattern only works for upgrades.

---

## Study Resources

Curated reading list for this topic: [`study-resources/helm/helm-advanced-patterns.md`](../../../study-resources/helm/helm-advanced-patterns.md)

**Key references**:
- [Subcharts and Global Values](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/) -- The canonical walkthrough of value scoping and `global:`
- [Charts -- Dependencies](https://helm.sh/docs/topics/charts/#chart-dependencies) -- `dependencies:` block reference, `condition`, `tags`, `alias`, `import-values`
- [Chart Hooks](https://helm.sh/docs/topics/charts_hooks/) -- Full hook taxonomy, weights, delete policies
- [Library Charts](https://helm.sh/docs/topics/library_charts/) -- `type: library`, the `bitnami/common` reference implementation
- [Use OCI-Based Registries](https://helm.sh/docs/topics/registries/) -- Modern chart distribution
- [Pushing a Helm chart to Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html) -- ECR-specific auth and IAM
- [Helmfile Documentation](https://helmfile.readthedocs.io/) -- `helmfile.yaml` structure, environments, lifecycle commands
- [Helmfile Best Practices](https://helmfile.readthedocs.io/en/stable/writing-helmfile/) -- Layout, secrets, ordering
- [Chart Tests (`helm test`)](https://helm.sh/docs/topics/chart_tests/) -- Annotated test pods, hook semantics
- [helm-unittest](https://github.com/helm-unittest/helm-unittest) -- BDD-style YAML unit tests, assertion catalogue
- [chart-testing (`ct`)](https://github.com/helm/chart-testing) -- CI lint + install harness
- [Advanced Helm (post-renderers, `lookup`)](https://helm.sh/docs/topics/advanced/) -- Escape hatches

**Cross-references in this repo**:
- [Helm Fundamentals (Apr 13, 2026)](./2026-04-13-helm-fundamentals.md) -- Day 1 of the Kubernetes deployment week; chart anatomy, template engine, install/upgrade/rollback, debugging triad
- [EKS Deep Dives -- December 2025](../../../2025/december/eks/) -- The cluster Helm ships workloads into
- [Terraform Modules](../../../2026/january/terraform/2026-01-16-terraform-modules.md) -- Parallel mental model: Terraform module composition is to cloud resources what Helm subchart composition is to K8s manifests
- [Terraform Project Structure](../../../2026/january/terraform/2026-02-02-terraform-project-structure.md) -- Per-env values files in Helm mirror per-env `terraform.tfvars`; Helmfile environments mirror Terraform workspaces in spirit
