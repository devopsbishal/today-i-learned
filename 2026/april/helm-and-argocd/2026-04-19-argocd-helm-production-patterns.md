# ArgoCD + Helm in Production -- Where the Two Tools Actually Meet

> Yesterday closed out the **franchise empire** view of ArgoCD: App-of-Apps, ApplicationSets, multi-cluster, AppProjects, RBAC, secrets. Three days ago closed out the Helm side: subcharts, hooks, library charts, OCI registries, Helmfile. Both halves are now in your head as separate tools. Today is the day they collide. Production GitOps is not "Helm OR ArgoCD" -- it is **Helm chart authored by an app team, OCI-published by CI, referenced by an ArgoCD Application, layered with environment-specific values from a separate config repo, image-tag promoted by Image Updater on a PR, rolled out progressively by Argo Rollouts with metric gating, and gated through dev -> stg -> prod by a promotion workflow**. Every previous concept becomes a Lego brick; today is the assembly diagram.
>
> The kitchen analogy carries the day. A **Helm chart is a recipe** in the corporate cookbook -- pancakes, ingredients, ratios, oven temp. The **values.yaml is the spice level** the recipe defaults to (mild for everyone). The **per-environment overrides** are how each restaurant adapts the recipe -- the prod kitchen turns up the heat, the dev kitchen swaps in cheap ingredients to test, the staging kitchen mirrors prod but with smaller portions. **ArgoCD is the head chef** standing in every kitchen, recipe in hand, comparing the plate to the picture and forcing the cook to redo it if it's wrong. The **multi-source Application** is the recipe living in the corporate cookbook (OCI registry) while the per-restaurant spice card lives on the kitchen wall (config Git repo) -- one recipe, many spice cards, no fork. The **Image Updater** is a robot inventory clerk who watches the freight dock for new ingredient shipments and, when a fresher batch arrives, walks over to the spice card and updates the "use ingredient v1.4.7" line so the chef sees a change at the next inspection. **Argo Rollouts** is the practice of bringing a new dish to one table first, watching the diners' faces, then to the next two tables, then the rest of the room -- with the maitre d' empowered to yank the dish back to the kitchen if anyone winces. **Promotion dev -> stg -> prod** is the corporate process for moving a new recipe up from the test kitchen to the flagship restaurant: the test kitchen autoupdates from every shipment, staging accepts changes only via an approved transfer slip, prod accepts changes only after a senior manager signs the slip in person.
>
> This doc walks through it in order: the three Helm-chart-source patterns and when each one is right, the values precedence chain that decides which spice card wins, the multi-source pattern that decouples chart from values, the repo-structure decisions (mono vs multi, folder vs branch per env), the Image Updater write-back loop (`argocd` vs `git` methods, semver/digest/latest/name strategies, the helm-specific annotations), end-to-end promotion patterns including PR-based and Kargo-based, Argo Rollouts as the canary/blue-green layer with AnalysisTemplates and metric gating, and the production gotchas that bite the first time you wire all of it together -- Helm hooks reinterpreted as sync hooks, release-ownership conflicts when both `helm` CLI and ArgoCD touch the same workload, mutating-webhook drift, CRD ordering, and why secrets in `values.yaml` is the worst architecture mistake you can make on day one. By the end you can defend a complete production GitOps design from chart authorship to canary promotion with metric-gated rollback.

---

**Date**: 2026-04-19
**Topic Area**: argocd
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Helm chart in ArgoCD Application | Recipe in the head chef's binder | `spec.source` points at a chart; ArgoCD runs `helm template` and applies the rendered manifests |
| Pattern 1: chart + values in same Git repo | Recipe and spice card stapled together | `repoURL` Git, `path: charts/myapp`, `helm.valueFiles: [values-prod.yaml]` -- simplest, fine for first-party apps |
| Pattern 2: chart from Helm/OCI repo, values inline | Buy a published cookbook, write notes in the margin | `repoURL: oci://...`, `chart: kube-prometheus-stack`, `helm.values:` inline -- fine for low-touch upstream charts |
| Pattern 3: multi-source (chart + values from different repos) | Cookbook on the shelf, spice card on the wall | Two `sources:`, one for chart (OCI/Git) and one with `ref: values` for the config repo; `valueFiles: [$values/envs/prod.yaml]` |
| `valueFiles` | Ordered list of spice cards | Files merged left-to-right; later wins; `--ignore-missing-value-files` lets per-env files be optional |
| `values` (inline string) | Sticky note on top of the spice card | Raw YAML string in the Application spec; legacy escape-hatch, prefer `valuesObject` |
| `valuesObject` (structured) | Typed override sheet | Structured YAML map under `helm.valuesObject:`; better than `values:` because it parses, not strings |
| `parameters` (Helm `--set`) | A single ingredient cross-out on the spice card | One-key overrides, highest precedence except `fileParameters`; how Image Updater writes image tag back |
| Values precedence chain | "Whose pen wins on the spice card?" | chart `values.yaml` < `valueFiles[]` (later file in the list wins) < (`values` OR `valuesObject` -- mutually exclusive; `valuesObject` overrides `values` if both set) < `parameters` / `fileParameters` (highest, both feed Helm `--set`/`--set-file`) |
| `$values` reference | "Look on the kitchen wall, not in the binder" | `valueFiles: [$values/envs/prod/values.yaml]` -- references the multi-source ref named `values` |
| Mono-repo (app code + manifests in one) | One cookbook for everything | App developers and deployment configs side by side; simpler workflow, weaker boundaries |
| Multi-repo (app repo + config/manifests repo) | Code in the engineering vault, deployment configs on the chef's shelf | The production default; CI pushes images to OCI, then opens PRs against the config repo |
| Folder-per-environment | One section per restaurant in the cookbook | `envs/dev/`, `envs/stg/`, `envs/prod/` directories in the same branch; the modern default |
| Branch-per-environment | Anti-pattern: separate cookbook editions per restaurant | `dev`, `staging`, `prod` Git branches; promotion = merge; bad merge conflicts, drift, audit pain |
| Repo-per-environment | Anti-pattern: separate cookbook per restaurant | Three repos (`config-dev`, `config-stg`, `config-prod`); maximum drift, no diffability |
| ArgoCD Image Updater | Inventory robot that watches the freight dock | Sidecar (or separate Deployment) that polls registries; on new tag, writes back the new tag so ArgoCD picks it up |
| `image-list` annotation | The robot's watch list | `image-list: alias=registry/repo:tagPattern` on the Application; one entry per image |
| `helm.image-name` / `helm.image-tag` | "Where on the spice card to write the new ingredient" | Maps logical image alias to the values-keys the chart actually reads |
| Update strategy `semver` | "Only accept newer minor/patch versions of v1.x" | Constraint like `1.x.x`; only newer matching tags promote |
| Update strategy `digest` | "Same tag, watch the SHA" | Mutable tag (`latest`) where you only care about digest changes |
| Update strategy `latest` (renamed `newest-build`) | "Whatever was built most recently, regardless of name" | By image creation time; useful for SHA tags |
| Update strategy `name` (renamed `alphabetical`) | "Pick the alphabetically last tag" | For calver (`2026-04-19-1234`) where alpha-sort is chronological |
| Write-back `argocd` (default) | Robot scribbles on the chef's printout | Mutates live `Application` via the ArgoCD API; lost on Application recreation, no Git audit |
| Write-back `git` (production) | Robot opens a Git commit | Commits an override file (`.argocd-source-<appname>.yaml` or a Helm valuesfile patch) back to the tracked branch; drift-proof, audited |
| `write-back-target` | "Which spice card to scribble on" | `helmvalues:./values-prod.yaml` to amend a values file directly, or default to the `.argocd-source-<app>.yaml` overlay |
| `git-branch` | Which kitchen wall the robot writes to | Branch the Application tracks; if missing, can specify a write branch + auto-PR pattern |
| Promotion: dev -> stg -> prod | Test kitchen -> staging -> flagship | Image Updater auto-promotes in dev; PR moves the new tag to stg values file; manual approval moves it to prod |
| Pinned tags per environment | Different spice level per restaurant | Each env's values file pins its own image tag; promotion is just bumping that pin |
| Kargo | Corporate transfer-slip system | Akuity's CRD-driven promotion engine -- models environments as a graph, automates PR generation between stages |
| GitOps Promoter | Open-source Kargo alternative | CNCF-aligned project for Argo CD-native PR-based promotion |
| Argo Rollouts | Bringing the dish to one table first | Replaces `Deployment` with a `Rollout` CRD; supports canary and blue-green strategies natively |
| Canary strategy | Serve new dish to one table, then two, then the room | Weighted traffic shift through `setWeight`/`pause` steps; integrates with NGINX/ALB/Istio/Linkerd for real traffic control |
| Blue-green strategy | Two kitchens; switch the menu pointer | `activeService` + `previewService`; promote = flip the selector; supports `prePromotionAnalysis` |
| AnalysisTemplate | The maitre d's diner-reaction checklist | Reusable metric-query definition (Prometheus, Datadog, web, job) with `successCondition`/`failureCondition` |
| AnalysisRun | One actual diner-reaction reading | An instance of an AnalysisTemplate kicked off mid-rollout; failure aborts the rollout |
| Background analysis | Continuous diner-watching while serving | Runs alongside steps, doesn't block them |
| Inline analysis | Pause and check before serving the next table | Blocks the next step until success |
| `prePromotionAnalysis` (blue-green) | Sniff-test the new dish before announcing it | Runs against `previewService` before traffic flips |
| Helm hooks via ArgoCD | Recipe steps reinterpreted as kitchen rituals | ArgoCD remaps `helm.sh/hook: pre-install` -> `argocd.argoproj.io/hook: PreSync`; runs on every sync, not just upgrade |
| `hook-delete-policy` collapse | "When the comma list of phases gets ignored" | `helm.sh/hook: post-upgrade,post-delete` -- ArgoCD picks one and silently drops the other |
| Release-ownership conflict | Two chefs trying to plate the same dish | Both `helm install` AND ArgoCD managing the same workload -- duplicate label conflicts and ping-pong reconciliation |
| Mutating webhook drift | Sous-chef adds garnish the recipe didn't ask for | Sidecar injectors / Linkerd / Istio change the live spec; ArgoCD sees diff; fix with `ignoreDifferences` |
| CRD ordering with sync waves | Build the oven before baking | CRDs in `wave: -10`, operator deployments in `wave: -5`, custom resources in `wave: 0` |
| Secrets in `values.yaml` | Writing the safe combination on the spice card | Plaintext secrets in Git; the architectural mistake -- always use ESO/Sealed Secrets/SOPS |

---

## The Production Assembly Diagram

Every ArgoCD + Helm production stack is the same five Lego bricks snapped together. Once you know the bricks, every "how does Company X do GitOps?" answer becomes a permutation:

1. **The Recipe Source (Helm chart, somewhere)** -- One of three places: same repo as the values, an external Helm/OCI registry with values inline, or an external chart with values from a separate config repo via multi-source.

2. **The Spice Cards (per-environment values)** -- A hierarchy of values files (`common.yaml` -> `region.yaml` -> `env.yaml` -> `cluster.yaml`) merged left-to-right by `valueFiles`, with the per-env file pinning that environment's image tag. The hierarchy lives in the **config repo**, separate from the **app repo**.

3. **The Inventory Robot (Image Updater)** -- Watches container registries, on a new matching tag commits the new tag to the dev environment's values file in Git. ArgoCD then syncs dev. Production tag bumps come via PR from dev's values file.

4. **The Service Brigade (Argo Rollouts)** -- Inside the cluster, the workload is a `Rollout` CRD instead of a `Deployment`. Argo Rollouts shifts traffic stepwise (canary) or kitchen-wholesale (blue-green), with AnalysisTemplates polling Prometheus to make automated promote/abort decisions.

5. **The Transfer-Slip System (promotion workflow)** -- Either hand-rolled PRs (`bump prod values to image v1.4.7`), an automation tool like Kargo or GitOps Promoter, or ApplicationSet generators that pick up new env values automatically. The thread tying it all together is "every promotion is a Git commit" -- nothing is promoted by clicking a button in a UI.

```
THE PRODUCTION ASSEMBLY -- ARGOCD + HELM IN ANGER
================================================================================

       APP REPO (engineering)              CONFIG REPO (platform)
       +-------------------+                +------------------------------+
       |  src/             |                |  apps/                       |
       |  Dockerfile       |                |    myapp/                    |
       |  charts/myapp/    |                |      base/values.yaml        |
       |    Chart.yaml     |                |      envs/                   |
       |    values.yaml    |                |        dev/values.yaml       |
       |    templates/     |                |        stg/values.yaml       |
       +---------+---------+                |        prod/values.yaml      |
                 |                          |  appsets/                    |
        CI builds & pushes                  |    myapp-appset.yaml         |
                 |                          +-------------+----------------+
                 v                                        |
       +-------------------+                              | pulled by ArgoCD
       |  OCI Registry     |<----- chart push (helm push) | repo-server
       |  (ECR/GHCR/Harbor)|<----- image push (docker)    |
       |                   |                              |
       |  myapp:1.4.7      |                              |
       |  charts/myapp-2.1 |                              |
       +-------+-----------+                              |
               |                                          |
               | watched by                               |
               |                                          v
       +-------+----------+                  +----------------------------+
       | Image Updater    |  commits new     |  ArgoCD Application(s)     |
       | (poll ECR every  |  tag PR to ----> |  - chart from OCI          |
       |  ~2 min)         |  dev/values.yaml |  - values from config repo |
       +------------------+                  |  - multi-source            |
                                             +-------------+--------------+
                                                           |
                                              helm template| + apply
                                                           v
                                             +----------------------------+
                                             | Cluster (per env)          |
                                             |   Rollout (NOT Deployment) |
                                             |   Service (active+preview) |
                                             |   AnalysisTemplate         |
                                             +-------------+--------------+
                                                           |
                                                           v
                                             +----------------------------+
                                             | Argo Rollouts controller   |
                                             |   step 10% -> analysis     |
                                             |   step 50% -> analysis     |
                                             |   step 100% -> promote     |
                                             |   (or abort + rollback)    |
                                             +----------------------------+

  PROMOTION FLOW:
    dev:  Image Updater commits to envs/dev/values.yaml -> auto-sync, auto-rollout
    stg:  PR copies dev's image tag into envs/stg/values.yaml -> auto-sync after merge
    prod: PR copies stg's image tag into envs/prod/values.yaml -> manual review + sync
```

---

## Part 1: The Three Helm-Chart-Source Patterns

Every ArgoCD Application that uses Helm picks one of three patterns. The decision drives everything downstream -- the repo layout, the promotion workflow, who can change what. Get this wrong and the rest of the design wobbles.

### Pattern 1 -- Chart and Values in the Same Git Repo

The recipe and the spice card stapled together. The simplest pattern, and the right answer for **first-party apps your team owns**.

```yaml
# Pattern 1 -- chart-and-values in one Git repo
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/acme/myapp.git
    path: charts/myapp                    # the chart directory in the repo
    targetRevision: main
    helm:
      valueFiles:
        - values.yaml                     # base values from chart dir
        - ../../envs/prod/values.yaml     # per-env override (same repo)
      parameters:
        - name: image.tag
          value: "v1.4.7"                 # pinned per env
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Use when**: your team owns both the chart and the application. Your app code, Dockerfile, chart, and per-env values all live in one repo. CI pushes images to a registry and ArgoCD watches the same repo for both chart and values changes.

**Don't use when**: the chart is upstream (`ingress-nginx`, `kube-prometheus-stack`) and you'd be forking it just to layer values, or you have many tenants who shouldn't see each other's values.

### Pattern 2 -- Chart from Helm/OCI Registry, Values Inline in Application

You buy a published cookbook and write your notes in the margin. The right answer for **upstream charts where you change very few values**.

```yaml
# Pattern 2 -- external chart, values inline
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://charts.jetstack.io   # classic HTTP Helm repo
    chart: cert-manager                   # chart name in that repo
    targetRevision: v1.14.3               # pinned chart version
    helm:
      releaseName: cert-manager
      valuesObject:                       # structured values (preferred over `values:` string)
        installCRDs: true
        global:
          leaderElection:
            namespace: cert-manager
        prometheus:
          enabled: true
          servicemonitor:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true              # cert-manager CRDs are large, avoid 256KB annotation limit
```

For OCI registries (Helm 3.8+), the source becomes:

```yaml
spec:
  source:
    # Note: NO `oci://` scheme prefix -- ArgoCD strips it. Just the registry host.
    # The chart name goes in the `chart:` field, not appended to repoURL.
    # The repository must be registered in argocd-cm (or as an argoproj.io/v1alpha1 Repository)
    # with type: helm and enableOCI: true.
    repoURL: registry-1.docker.io/bitnamicharts
    chart: postgresql
    targetRevision: 16.2.5
    helm:
      valuesObject: { ... }
```

**Use when**: the chart is upstream and you're tweaking <20 keys. Your customizations are short enough to inline without losing readability.

**Don't use when**: your `valuesObject:` block is hundreds of lines (now you can't diff it sensibly), or different envs need different values (you'd duplicate the whole Application across envs).

### Pattern 3 -- Multi-Source: Chart from One Repo, Values from Another

The cookbook on the shelf, the spice card on the wall. The **production answer** for any heavily-customized upstream chart.

This pattern, available in Argo CD 2.6+, lets one Application reference **multiple sources**: one for the chart and one (or more) Git repos that supply the value files. The trick is the `ref:` field, which names a source so other sources can reference it.

```yaml
# Pattern 3 -- multi-source: external chart + values from your own config repo
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: platform
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 58.2.1
      helm:
        # Critical: the valueFiles entries reference the OTHER source by ref name
        valueFiles:
          - $values/envs/common/values.yaml         # shared base
          - $values/envs/prod/values.yaml           # env-specific
          - $values/envs/prod/clusters/use1.yaml    # cluster-specific (last wins)
    - repoURL: https://github.com/acme/gitops-config.git
      targetRevision: main
      ref: values                                   # <-- the named ref Pattern 3 hinges on
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**The mechanic**: ArgoCD pulls the chart from the upstream registry AND clones your config repo. The `$values/...` paths are resolved relative to the source named `values`. You never fork the upstream chart.

**Critical gotchas**:
- The `ref:` source has **no `chart:` and no `path:`** -- it exists purely so other sources can reference it.
- `valueFiles` from the `$values` source are **read-only** to ArgoCD; you can't write back to them via Image Updater unless the chart-source itself is also Git (more on this in Part 4).
- The Application UI shows multiple sources; sync ordering is the union of all sources, not sequenced.
- A common confusion: `$values` is the **literal** `ref` name you chose. If your ref is `ref: configrepo`, you'd write `$configrepo/...`.

**Use when**: the chart is upstream and your customization sprawls across hundreds of lines per env. This is the canonical pattern Kostis Kapelonis (Codefresh) describes for production GitOps.

### Decision Tree

| Question | Pattern |
|----------|---------|
| Do you own the chart? | Pattern 1 (same repo) |
| Upstream chart, <20 lines of customization, same across envs? | Pattern 2 (inline values) |
| Upstream chart, heavy customization or multi-env? | Pattern 3 (multi-source) |
| First-party chart but config team is different from app team? | Pattern 3 (multi-source) |

---

## Part 2: Per-Environment Values -- Precedence and Hierarchies

Once you've chosen a pattern, the next question is "how do per-environment differences get layered in?" The answer depends on Helm's value precedence chain, which ArgoCD inherits faithfully:

```
chart's own values.yaml          (lowest precedence)
  < helm.valueFiles[]            (last file listed wins per key)
  < helm.values  OR  helm.valuesObject
                                 (MUTUALLY EXCLUSIVE -- if both are set,
                                  valuesObject takes precedence and `values`
                                  is effectively replaced, NOT merged on top)
  < helm.parameters[]  /  helm.fileParameters[]
                                 (highest precedence; both equivalent to
                                  `helm install --set` / `--set-file`)
```

Two non-obvious rules to internalize:

1. **`valueFiles` last-wins**: in `valueFiles: [a.yaml, b.yaml]`, `b.yaml` overrides `a.yaml`. Same as Helm's `-f` flag semantics.
2. **`values` and `valuesObject` are NOT layered**: they are mutually exclusive sources. If you set both, `valuesObject` wins entirely -- the `values` block is dropped, not merged. Always pick one. (`valuesObject` is preferred because it's structured YAML the API server validates; `values` is a raw string.)

Almost every "why isn't my prod override winning?" debugging session traces back to this chain -- usually because someone set `parameters` in dev and then layered `valueFiles` in prod, not realizing `parameters` always beat files.

### The Common -> Region -> Env -> Cluster Hierarchy

The Codefresh / Kapelonis pattern is a four-layer hierarchy of values files, merged in order:

```yaml
# In your config repo:
envs/
  common/values.yaml            # globals: monitoring labels, default resources
  regions/
    us-east-1/values.yaml       # region-specific: storage class, AZ list
    eu-west-1/values.yaml
  dev/values.yaml               # env-specific: replicas: 1, debug: true
  stg/values.yaml               # env-specific: replicas: 2, profiling: true
  prod/values.yaml              # env-specific: replicas: 6, alerts: true
  clusters/
    use1-prod/values.yaml       # cluster-specific: node pool, custom DNS
    euw1-prod/values.yaml
```

```yaml
# Application picks the relevant slice via valueFiles
helm:
  valueFiles:
    - $values/envs/common/values.yaml
    - $values/envs/regions/us-east-1/values.yaml
    - $values/envs/prod/values.yaml
    - $values/envs/clusters/use1-prod/values.yaml   # last one wins
  ignoreMissingValueFiles: true                     # cluster file is optional
```

The `ignoreMissingValueFiles: true` flag (Argo CD 2.4+) is what makes this pattern composable -- not every cluster needs a cluster-specific override file, so it's optional and skipped silently if absent.

### `values` vs `valuesObject` -- Always Use `valuesObject`

Two ways to inline values into an Application:

```yaml
# Legacy: a raw YAML string. Awkward, error-prone (indentation in a string).
helm:
  values: |
    image:
      tag: v1.4.7
    replicas: 3

# Modern: a structured YAML map. Preferred. Validates and merges properly.
helm:
  valuesObject:
    image:
      tag: v1.4.7
    replicas: 3
```

Always pick `valuesObject` -- it parses, type-checks, diffs cleanly in Git, and survives `kubectl apply` cleanly. The `values` string variant exists for backwards compatibility and is best avoided.

### `parameters` -- The `--set` Equivalent (and Why Image Updater Loves It)

```yaml
helm:
  parameters:
    - name: image.tag
      value: "v1.4.7"
    - name: image.repository
      value: "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp"
    - name: ingress.enabled
      value: "true"
      forceString: true                    # treat "true" as string, not bool (rare)
```

`parameters` map directly to `helm template --set key=value`. Their highest-precedence position in the chain is the reason **Image Updater's default `argocd` write-back method writes to `parameters`** -- it can override one key (the image tag) without touching the rest of your values stack. This is also the reason it's drift-prone: if you reapply the Application from Git, Image Updater's parameter override evaporates. Hence the `git` write-back method, covered in Part 4.

---

## Part 3: Repo Structure -- The Architectural Choice You Make Once

Repo structure is the GitOps decision you *cannot* refactor later without breaking every sync. Get it right on day one. The trade-offs cluster around two axes: **mono vs multi-repo** and **how environments are encoded**.

### App Repo vs Config Repo Separation

The strong production default is **two repos**: an **app repo** (source code, Dockerfile, chart) and a **config repo** (per-env values, Application manifests, ApplicationSets). Why split?

| Concern | Mono-repo | Multi-repo (app + config) |
|---------|-----------|---------------------------|
| CI permissions | App devs need write to deploy configs | App devs only push to app repo; CI bot writes to config repo |
| Merge velocity | Every deploy bump is a code-repo PR | Config-repo PRs don't trigger CI test suites |
| Rollback | Revert a code commit also reverts deploy config (bad) | Revert a config PR only rolls back deploy (good) |
| Audit | "What deployed to prod last week?" requires sifting code commits | Config repo log = deployment log |
| ArgoCD scope | Watches a busy repo with lots of irrelevant churn | Watches a small focused repo, faster repo-server |
| Image Updater write-back | Commits image tag bumps into the code repo | Commits cleanly into the config repo |

The exception is **small teams or single-tenant apps** where the overhead of two repos isn't worth it. For platform teams managing multiple apps across multiple envs, two repos is non-negotiable.

### Folder-per-Environment (Default) vs Branch-per-Environment (Anti-Pattern)

Two ways to encode environments in the config repo:

**Folder-per-environment (right):**
```
config-repo/
  envs/
    dev/values.yaml
    stg/values.yaml
    prod/values.yaml
```
- Single branch (`main`).
- Promotion = PR that copies new tag from `dev/values.yaml` to `stg/values.yaml`.
- Dead-simple diffability: `git diff envs/stg envs/prod` shows you exactly the env-to-env delta.
- Pairs naturally with ApplicationSet Git generators that walk subdirectories.

**Branch-per-environment (wrong):**
```
config-repo/
  values.yaml      <-- on branches: dev, staging, prod
```
- Three branches that diverge over time.
- Promotion = `git merge dev -> staging -> prod`, which is a recipe for merge conflicts and accidental drift.
- Reverting prod requires a force-push or revert commit on the prod branch.
- "What did stg look like a week ago?" is hard to answer without a checkout.

The Akuity GitOps best-practices whitepaper, the Codefresh blog series, and every practitioner write-up converge on the same recommendation: **folders, not branches**.

### Repo-per-Environment (Worse Anti-Pattern)

Three separate Git repos: `config-dev`, `config-staging`, `config-prod`. Promotion is now a *cross-repo copy* with no atomic diff. Drift accumulates silently. Tooling fights you. Avoid unconditionally.

---

## Part 4: ArgoCD Image Updater -- Closing the CI/CD Loop

CI builds an image, pushes it to ECR, and... what? In a non-GitOps world, CI also runs `kubectl set image` or `helm upgrade` to bump the running deployment. That breaks GitOps -- now Git no longer reflects what's running.

**ArgoCD Image Updater** is the GitOps-correct closing of the loop. It runs as a separate controller in your ArgoCD namespace, polls container registries on a configurable interval (~2 min default), and when a new tag matching a configured strategy appears, it either mutates the live Application (default `argocd` method) or commits an updated values override back to Git (the `git` method, the only correct production choice).

### Annotations -- The DSL on the Application

Image Updater is configured per-Application via annotations. The minimum useful set for a Helm chart:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
  annotations:
    # 1. The watch list. Format: alias=registry/repo:tagPattern
    argocd-image-updater.argoproj.io/image-list: |
      myapp=123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:~1.4

    # 2. Map alias -> values keys (Helm-specific)
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/myapp.helm.image-tag:  image.tag

    # 3. Update strategy
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v?\d+\.\d+\.\d+$

    # 4. Write-back: GIT method (production answer)
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: helmvalues:./envs/dev/values.yaml
    argocd-image-updater.argoproj.io/git-branch: main

    # 5. Pull policy
    argocd-image-updater.argoproj.io/pull-policy: Always
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 58.2.1
      helm:
        valueFiles:
          - $values/envs/dev/values.yaml
    - repoURL: https://github.com/acme/gitops-config.git
      targetRevision: main
      ref: values
  destination: { server: https://kubernetes.default.svc, namespace: monitoring }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

### The Two Write-Back Methods

| Method | What it does | When to use |
|--------|-------------|-------------|
| `argocd` (default) | Mutates the live `Application` resource via the ArgoCD API, writing the new tag into `spec.source.helm.parameters` | Quick demos, dev clusters where audit doesn't matter; **never in prod** |
| `git` | Commits a values-file or `.argocd-source-<appname>.yaml` patch back to the tracked branch | Production. Drift-proof, audit-trailed, survives Image Updater outage |

The `argocd` method's failure mode is brutal: if anyone reapplies the Application from Git (e.g., the App-of-Apps root re-syncs), the parameter override Image Updater wrote is **wiped** because Git doesn't have it. The deployment silently rolls back to whatever tag is in Git. The `git` method avoids this by making Git the source of truth Image Updater modifies.

### Update Strategies

| Strategy | Picks | Use case |
|----------|-------|----------|
| `semver` (constraint-based) | Highest semver matching constraint (e.g., `~1.4`, `1.x.x`) | Production: only newer minor/patch within a major |
| `digest` | Same tag, watching for SHA changes | Mutable tags like `latest` or `main` |
| `latest` (newer name: `newest-build`) | Tag with the latest creation timestamp at the registry | Quick prototype; SHA-named tags |
| `name` (newer name: `alphabetical`) | Tag that sorts last alphabetically | Calver tags like `2026-04-19-1234` |

Note: in Image Updater 0.13+, `latest` was renamed to `newest-build` and `name` to `alphabetical` to better describe what they actually do. The old names still work as aliases.

### The Git Write-Back Workflow

When `write-back-method: git` is set, here's what happens on each detection:

1. Image Updater pulls the latest matching tag from the registry.
2. It clones the config repo, checks out the `git-branch`.
3. It writes either:
   - A patch to the `write-back-target` values file (e.g., `envs/dev/values.yaml`), updating the `image.tag` key in place, **or**
   - A `.argocd-source-<appname>.yaml` overlay at the repo root if no `write-back-target` is set.
4. It commits with a message like `build: automatic update of myapp` and pushes.
5. ArgoCD's webhook (or the next refresh) sees the commit, syncs, and the rollout begins.

You can also configure `git-write-branch` to commit to a *different* branch, which is the key trick for **PR-based promotion**: Image Updater pushes to a `image-updates/dev` branch, a CI job opens a PR against `main`, your team reviews and merges. This is the production-safe pattern for prod environments.

---

## Part 5: Promotion Workflows -- dev -> stg -> prod

You now have charts, environment values, and image automation. The remaining question is "how does a new image actually walk from dev through staging to prod?" Three patterns, in increasing order of maturity.

### Pattern A: Image Updater in Dev Only, Manual PRs to Promote

The simplest and most common production pattern.

- Dev Application: Image Updater enabled, semver strategy `~1.x`, write-back `git` to `envs/dev/values.yaml`. Auto-syncs on every CI build.
- Staging Application: **No Image Updater**. Tag pinned in `envs/stg/values.yaml`. Promotion is a PR: developer or release engineer copies `image.tag` from `envs/dev/values.yaml` to `envs/stg/values.yaml`, opens a PR, reviewer approves, merge triggers ArgoCD auto-sync to staging.
- Prod Application: **No Image Updater, no auto-sync** (manual sync via UI / API after PR merge). Promotion is the same PR flow but with required CODEOWNERS approval, a passing tests-on-staging gate, and a release-window check.

```yaml
# envs/dev/values.yaml -- updated by Image Updater
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
  tag: v1.4.7-dev-build-1247

# envs/stg/values.yaml -- updated by hand-rolled PR
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
  tag: v1.4.7                  # released tag, not a -dev build

# envs/prod/values.yaml -- updated by approved PR after staging soak
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
  tag: v1.4.6                  # always trailing stg by ~24-48h
```

### Pattern B: Sync Waves Within a Single Promotion

For a release that touches multiple Applications (db migration + app + cache warm), use **sync waves** on the prod Application to ensure ordering:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"        # main app deploys at wave 0

# Hooks within the chart:
# - Database migration Job: sync-wave: "-5", hook: PreSync
# - Cache warmer Job: sync-wave: "5", hook: PostSync
```

### Pattern C: Kargo or GitOps Promoter

When hand-rolled PR promotion gets tedious, two CRD-driven engines automate it:

- **Kargo** (Akuity, the original ArgoCD team) -- models environments as a graph of `Stage` resources, automates `Freight` (a bundle of artifacts) promotion between stages with verification gates. Generates the PRs for you.
- **GitOps Promoter** -- the open-source CNCF-aligned alternative, similar concept, designed specifically for Argo CD environments.

You don't need either on day one -- hand-rolled PRs are fine for 5-10 services. Reach for them when promotion overhead starts dominating engineering time.

---

## Part 6: Argo Rollouts -- Progressive Delivery as the Final Mile

ArgoCD answers "what to deploy and where." It does **not** answer "how to deploy safely." A vanilla Kubernetes `Deployment` does a basic rolling update -- it shifts replicas one at a time without observing whether the new pods are actually serving traffic correctly.

**Argo Rollouts** is the missing layer: a CRD that **replaces** `Deployment` (does not wrap it) and adds canary, blue-green, and metric-gated automation. ArgoCD syncs the `Rollout` manifest like any other resource; the Argo Rollouts controller then drives the progressive traffic shift inside the cluster.

### The Rollout CRD -- A Drop-In Deployment Replacement

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  namespace: prod
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
  template:                              # IDENTICAL to Deployment.spec.template
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.4.7
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: 100m, memory: 128Mi }
  strategy:
    canary:
      canaryService: myapp-canary        # Service for canary pods
      stableService: myapp-stable        # Service for stable pods
      trafficRouting:
        nginx:
          stableIngress: myapp-ingress   # NGINX-managed traffic split
      steps:
        - setWeight: 10                  # 10% to canary
        - pause: { duration: 2m }        # observe for 2 minutes
        - analysis:                      # gate the next step on metric check
            templates:
              - templateName: success-rate
        - setWeight: 30
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 60
        - pause: { duration: 5m }
        - setWeight: 100                 # full rollout
```

Critical points:
- The `template:` block is **identical** to a Deployment's pod template; migration is a kind change plus the `strategy:` block.
- HPA, PDB, NetworkPolicy, and Service configs work unchanged because the Rollout exposes the same selectors.
- ArgoCD treats the Rollout like any other CRD -- syncs it, watches it, reports its health (Argo Rollouts ships a `Rollout` health check Lua script ArgoCD picks up automatically).

### Blue-Green Strategy

```yaml
spec:
  strategy:
    blueGreen:
      activeService: myapp-active        # production traffic
      previewService: myapp-preview      # smoke-test traffic
      autoPromotionEnabled: false        # require manual promotion
      prePromotionAnalysis:
        templates:
          - templateName: smoke-tests
      postPromotionAnalysis:
        templates:
          - templateName: success-rate
```

- New ReplicaSet spins up behind `previewService`.
- `prePromotionAnalysis` runs against the preview before flipping traffic.
- Manual or automated promotion flips `activeService` selectors to the new RS.
- `postPromotionAnalysis` runs against active traffic; failure triggers automatic rollback.

### AnalysisTemplate -- Metric-Gated Promotion

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: prod
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 30s
      count: 5                           # 5 readings = 2.5 min
      successCondition: result[0] >= 0.95
      failureLimit: 2                    # 2 bad readings = abort
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              status!~"5.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              service="{{args.service-name}}"
            }[2m]))
```

Key concepts:
- Four provider types: `prometheus`, `datadog`, `web` (any HTTP endpoint with a JSON-path expression), `job` (run a Job and inspect its exit code).
- `successCondition` and `failureCondition` are CEL-like expressions over the result.
- `failureLimit` of N means "abort after N consecutive failures" -- protects against transient blips.
- `inline` analysis blocks the next step; `background` analysis runs continuously alongside steps.
- On failure, Argo Rollouts **automatically aborts** -- routes 100% of traffic back to the stable RS and marks the Rollout `Degraded`.

### Traffic Routing Integrations

Argo Rollouts can drive real traffic shifting via:
- NGINX ingress controller (`stableIngress` annotation manipulation)
- AWS ALB (target group weights)
- Istio (VirtualService weight manipulation)
- Linkerd (TrafficSplit CRD)
- SMI (Service Mesh Interface)

Without a traffic-routing integration, Argo Rollouts falls back to **replica-based weighting** -- if `setWeight: 30` and replicas: 10, three pods run the canary version. That's coarse but works for many workloads.

---

## Part 7: Production Gotchas -- The Bugs That Bite First

Every team that puts ArgoCD + Helm into production hits the same six gotchas. Internalize them once.

### Gotcha 1: Helm Hooks Are Reinterpreted as ArgoCD Sync Hooks

This is the single biggest one. Because ArgoCD runs `helm template` (not `helm install`), any `helm.sh/hook: post-upgrade` annotation gets remapped to `argocd.argoproj.io/hook: PostSync` -- which fires **on every successful sync**, not just on actual chart upgrades.

The mapping table:
| Helm annotation | ArgoCD interpretation | Fires when |
|----------------|----------------------|-----------|
| `helm.sh/hook: pre-install`, `pre-upgrade` | `argocd.argoproj.io/hook: PreSync` | Before every sync |
| `helm.sh/hook: post-install`, `post-upgrade` | `argocd.argoproj.io/hook: PostSync` | After every successful sync |
| `helm.sh/hook: post-delete` | `argocd.argoproj.io/hook: PostDelete` (Argo CD 2.10+) | When Application is deleted; **version-dependent -- pre-2.10 silently drops it** |
| `helm.sh/hook: pre-rollback`, `post-rollback`, `pre-delete`, `test`, `test-success`, `test-failure` | **NOT mapped -- silently dropped** | Never. ArgoCD doesn't have rollback or test phases. |
| Multi-phase comma list e.g. `helm.sh/hook: post-install,post-upgrade` | Mapped to one ArgoCD phase (`PostSync`); duplicate phases collapse cleanly | OK for same-phase lists |
| Cross-phase comma list e.g. `helm.sh/hook: post-upgrade,post-delete` | **Only one phase mapped; the other is silently dropped** | Hard-to-debug bug |

**Do this, not that** -- if you need a Job that runs both after upgrade AND after delete, write **two separate Job manifests**, each with its own single-phase annotation. Never a comma list across phases.

The implication: a non-idempotent post-upgrade Job (DB migration, license activation, third-party API call) will fire on every drift reconciliation -- including the ones triggered by `selfHeal`. Fix with one of:

1. Make the Job idempotent (check for existing state before doing the work).
2. Use ArgoCD's native `argocd.argoproj.io/hook: Sync` with `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded`.
3. Use a Helm chart **without** hooks for the install path, and run migrations as a separate ArgoCD Application that you sync explicitly.

```yaml
# Migration Job done right in an ArgoCD-managed chart
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
    argocd.argoproj.io/sync-wave: "-5"
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myapp:{{ .Values.image.tag }}
          command: ["./migrate.sh", "--idempotent"]   # Critical: idempotent
```

### Gotcha 2: Release Ownership Conflicts (Helm CLI vs ArgoCD)

Symptom: every sync logs `OutOfSync` for some resources, even right after a successful sync.

Cause: someone ran `helm install myapp` directly against the cluster *and* added the same release as an ArgoCD Application. Both controllers add their own tracking labels:
- Helm adds `app.kubernetes.io/managed-by: Helm` + `meta.helm.sh/release-name: myapp`
- ArgoCD adds `argocd.argoproj.io/tracking-id: myapp:apps/Deployment:default/myapp`

Now they ping-pong. Fix: pick one. Either uninstall the Helm release (`helm uninstall myapp --keep-history`) and let ArgoCD take over (it'll adopt the resources), or remove the ArgoCD Application. Never both.

### Gotcha 3: Mutating-Webhook Drift

Sidecar injectors (Istio, Linkerd, AWS App Mesh, Vault Agent injector) mutate pods after creation, adding a sidecar container. ArgoCD compares the rendered Helm output (one container) against the live state (two containers) and forever sees `OutOfSync`.

Fix with `ignoreDifferences` in the Application. **Do not** try to use `jsonPointers` with a `-` suffix to mean "all elements" -- in JSON Pointer (RFC 6901) `-` refers only to the index past the end (used for appending), not "every element". For collection-level ignores, use `jqPathExpressions` instead:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      # jsonPointers work for fixed paths; use them for known annotation/label keys
      jsonPointers:
        - /spec/replicas                         # if HPA is owning replicas
      # jqPathExpressions handle "every element matching a predicate"
      jqPathExpressions:
        - '.spec.template.spec.containers[] | select(.name == "istio-proxy")'
        - '.spec.template.metadata.annotations["sidecar.istio.io/status"]'
```

For sidecar-injection cases, the cleanest fix is the resource-level annotation `argocd.argoproj.io/compare-options: ServerSideDiff=true` -- with Server-Side Diff, ArgoCD asks the API server what the diff actually is after defaulting/mutating webhooks, and many spurious differences disappear without any `ignoreDifferences` config. (Note: there is no `IgnoreExtraneous` value for `compare-options`; that's not a valid option.)

### Gotcha 4: CRD Ordering With Sync Waves

Helm chart installs an operator (e.g., `kube-prometheus-stack` ships the Prometheus Operator + `Prometheus` CR). ArgoCD applies everything in one shot; the `Prometheus` CR fails because the CRD doesn't exist yet, then succeeds on retry.

Fix with explicit sync waves and the `Replace=true` sync option for CRDs:

```yaml
# In your chart's CRD manifests:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-10"            # CRDs first
    argocd.argoproj.io/sync-options: Replace=true  # CRDs are too big for apply

# Operator deployment:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-5"

# CRs that depend on the operator:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

### Gotcha 5: Secrets in `values.yaml` Are Plaintext in Git

The cardinal sin. `values.yaml` is just YAML in your config repo, public to anyone with read access. Putting `dbPassword: hunter2` there means it's in your Git history forever, even if you later rotate.

Fix with one of (covered yesterday):
- **External Secrets Operator (ESO)** + `ExternalSecret` CRDs that fetch from AWS Secrets Manager / Vault / GCP-SM at sync time. The chart references a Secret name; ESO populates it. **Best general answer.**
- **Sealed Secrets** + `kubeseal` -- encrypt locally with cluster's public key, commit ciphertext.
- **SOPS** (with KSOPS plugin or `helm-secrets`) -- per-file YAML encryption with age or KMS.

In `values.yaml` you reference the **name** of the Secret only:

```yaml
# values.yaml -- safe in Git
db:
  passwordSecretName: myapp-db-password    # populated out-of-band by ESO
```

### Gotcha 6: Image Updater Branch Mismatch

Symptom: Image Updater logs successful commits, but ArgoCD never syncs the new tag.

Cause: `git-branch` doesn't match the Application's `targetRevision`. Image Updater committed to `main`, but the Application is tracking `release-2024-q4`. The commit is invisible to ArgoCD.

Fix: ensure `git-branch` annotation = `targetRevision` of the Application. For prod where you want PR review, use `git-write-branch` to commit to a separate branch and have CI auto-open the PR.

### Bonus: Helm `lookup` Returns Empty in ArgoCD

Helm charts sometimes use `lookup` to read existing resources (e.g., to preserve auto-generated Secrets across upgrades). Under `helm template` (which ArgoCD uses), `lookup` always returns an empty dict. Charts that rely on `lookup` for migration logic break under ArgoCD silently.

Workaround: extract the lookup logic into an init container or pre-sync hook that runs `kubectl get` directly.

---

## Decision Frameworks Recap

### Which Helm-chart-source pattern?

```
Do you own the chart?
  YES -> Pattern 1 (chart + values in same repo)
  NO  -> Are your value overrides < 20 lines AND constant across envs?
           YES -> Pattern 2 (external chart, inline values)
           NO  -> Pattern 3 (multi-source: chart from registry, values from config repo)
```

### Image Updater write-back method?

```
Is this a non-prod throwaway demo?     -> argocd method
Is anyone going to git revert config?  -> git method
Is the env prod or compliance-scoped?  -> git method WITH git-write-branch + PR gate
```

### Argo Rollouts strategy?

```
Workload: stateless web app with metric-friendly health signal -> canary + Prometheus analysis
Workload: monolith with hard cutover semantics                  -> blue-green + manual promote
Workload: low-traffic batch / cron                              -> stay on Deployment, no Rollouts
Workload: stateful (DB, queue)                                  -> blue-green ONLY, very carefully
```

### Promotion automation?

```
1-3 services across 2 envs       -> hand-rolled PRs, no tooling
4-10 services across 3 envs      -> hand-rolled PRs, ApplicationSet generators for new envs
10+ services or strict workflows -> Kargo or GitOps Promoter
```

---

## Key Takeaways

- **`helm template`, not `helm install`** is the foundational fact. Every gotcha is downstream of it. Hooks fire as ArgoCD sync hooks (every sync, not every upgrade), `lookup` returns empty, no release Secrets exist, `helm history` shows nothing for ArgoCD-managed releases.
- **Three chart-source patterns**: same-repo (your apps), external-with-inline-values (low-touch upstream), multi-source (heavy customization of upstream). The choice drives the rest of the architecture; revisit it if you've outgrown your current pattern.
- **Multi-source** (Argo CD 2.6+) is the production answer for upstream charts. The `ref:` field names a values source, and `valueFiles: [$values/...]` references it. No more forking upstream charts to layer values.
- **Values precedence**: chart `values.yaml` < `valueFiles` < `values` < `valuesObject` < `parameters` < `fileParameters`. Memorize it. Always prefer `valuesObject` over `values`.
- **Repo structure**: app repo + config repo, folder-per-environment, single `main` branch. Branch-per-env and repo-per-env are anti-patterns.
- **Image Updater closes the CI/CD loop the GitOps way**. Always use `write-back-method: git` in production. Use `semver` strategy for releases, `digest` for mutable tags. For prod, push to a `git-write-branch` and PR-gate the merge.
- **Argo Rollouts replaces Deployment, doesn't wrap it**. Migration is a kind change plus a `strategy:` block. Use canary for stateless web apps, blue-green for monolith cutovers. Always wire `AnalysisTemplate` with Prometheus or equivalent so failed rollouts auto-abort instead of needing human intervention.
- **Secrets never live in `values.yaml`**. Pick ESO (best default), Sealed Secrets, or SOPS once and stick with it. Reference Secret *names*, not values, in your charts.
- **Mutating webhooks cause forever-OutOfSync**. Use `ignoreDifferences` to mask the controller-injected fields.
- **CRDs sync first, operators second, custom resources third**. Sync waves: `-10`, `-5`, `0`. `Replace=true` for CRDs that exceed the apply annotation limit.
- **Promotion = a Git commit, never a UI button click**. Image Updater for dev, PR for staging, approved PR + manual sync for prod. Reach for Kargo or GitOps Promoter when hand-rolled PRs become the bottleneck.
- **Don't run `helm install` and ArgoCD on the same release**. Pick one. Adopting an existing Helm release into ArgoCD requires `helm uninstall --keep-history` first (or careful label rewriting).

The full GitOps stack now sits in your head end-to-end: Helm authors the recipe, OCI publishes it, the config repo holds the spice cards, ArgoCD enforces the menu, Image Updater watches the freight dock, Argo Rollouts brings the dish to one table first, and the promotion workflow walks each new release through dev, staging, and prod with PRs as the audit trail. From here, "production GitOps" is just a permutation of these bricks. Tomorrow's monitoring stack (Prometheus + Grafana) is the missing observability layer that makes Argo Rollouts' `AnalysisTemplate` actually mean something -- without metrics, the "metric-gated" rollback is theatre. That's where you go next.
