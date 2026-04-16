# ArgoCD Fundamentals -- The GitOps Model and the Site-Foreman Analogy

> Three days ago you internalized Helm's **blueprint factory**: `helm install` stamps a chart against a parameter sheet, applies the rendered manifests, and records a revision Secret. That model explains how one human, typing one command, shapes one cluster. It breaks the moment there are **twenty clusters**, **eight engineers**, and **a compliance team who wants every change traced to a Git commit**. The imperative Helm flow cannot answer "what is actually running?" without inspecting every cluster, cannot answer "who changed this?" without audit trawling, cannot revert a manual `kubectl edit` that a panicked on-call did at 03:00, and cannot guarantee that the dev cluster and the prod cluster were actually configured the same way. You need an operating model, not another CLI.
>
> That operating model is **GitOps**, and ArgoCD is its best-known Kubernetes implementation. The analogy that carries today's material is a **construction site with a Git-reading foreman**. Helm on its own is the architect who faxes fresh blueprints to the site whenever they feel like it -- push-based, ad-hoc, no single source of truth for "what's supposed to be on this site right now." GitOps replaces the fax with a **blueprint vault in Git** that the foreman continuously consults. The **architectural blueprint** is the Git repository: versioned, immutable, declarative. The **site foreman** is the ArgoCD Application Controller: they stand on the site all day, periodically re-reading the blueprint from the vault, looking at what is actually built, and making the site match. If someone shows up in the middle of the night and moves a wall, the foreman notices at the next inspection pass and moves it back. If the architect commits a new blueprint, the foreman picks it up and rebuilds to spec. If the architect adds a door labeled "wave 1" and a doorknob labeled "wave 2," the foreman installs the door first, waits until it is standing and healthy, *then* installs the doorknob. Nothing gets built unless the blueprint says so. Nothing stays on site unless the blueprint says so.
>
> That is the whole GitOps mental model in one sentence: **Git is the desired state, the cluster is the observed state, and ArgoCD is the reconciliation loop that makes them match -- continuously, declaratively, and auditably.** Everything else in this doc -- the seven ArgoCD components, the Application CRD, sync policies, sync waves, hooks, health checks, the fact that ArgoCD runs `helm template` and not `helm install` -- is machinery in service of that one idea. You already know Helm deeply; today's job is to plug every Helm concept you hold into its ArgoCD counterpart, and to notice what ArgoCD removes from the Helm world (release Secrets, `helm history`, `helm rollback`, working `lookup`) in exchange for what it adds (continuous reconciliation, drift detection, a declarative Application CRD, multi-cluster orchestration).
>
> This doc walks through all of it, in order: the four CNCF OpenGitOps principles, ArgoCD's seven-component architecture, the Application CRD as the central abstraction, sync status vs health status, the three sync-policy toggles (auto-sync / prune / self-heal) and the per-resource sync options, the Helm-specific integration -- including the critical bridge that ArgoCD uses `helm template` + `kubectl apply` rather than `helm install` -- sync waves and hooks for deployment ordering, built-in and custom Lua health checks, and the gotchas that bite practitioners the third or fourth time they touch the tool.

---

**Date**: 2026-04-16
**Topic Area**: kubernetes
**Difficulty**: Intermediate-to-Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| GitOps | Blueprint vault + site foreman who reads the vault and keeps the site matching | An operational model defined by four CNCF principles: **Declarative**, **Versioned and Immutable**, **Pulled Automatically**, **Continuously Reconciled** |
| Push CD (Jenkins, classic CI/CD) | Architect faxes fresh blueprints at random | CI pipeline holds cluster credentials and runs `kubectl apply` / `helm upgrade` from outside the cluster |
| Pull CD (ArgoCD, Flux) | Foreman stands on site and pulls blueprints from the vault | Controller runs **inside** the cluster, watches Git, applies changes; cluster never exposes credentials outward |
| ArgoCD Application (CRD) | A specific site's "what should be built here" work order | `Application` custom resource: links a Git `source` + chart/path to a `destination` (cluster + namespace) with a `syncPolicy` and `project` |
| ArgoCD Project | A portfolio of sites the firm is allowed to work on | `AppProject` CRD; scopes allowed Git repos, destinations, and resource kinds for a set of Applications -- the RBAC boundary |
| API Server | The foreman's radio to HQ (UI, CLI, webhooks) | gRPC/REST gateway; serves `argocd` CLI, web UI, and SSO auth; never runs reconciliation itself |
| Repo Server | The blueprint photocopy room | Clones Git, runs `helm template` / `kustomize build` / plain YAML to produce rendered manifests; **never touches the cluster** |
| Application Controller | The site foreman | The reconciliation loop: diffs desired (from repo server) vs live (from cluster), drives sync, runs health checks, triggers self-heal |
| Redis | The foreman's clipboard | Caches rendered manifests and cluster state; not durable storage -- loss means re-render, not data loss |
| Dex | The HR desk that verifies IDs | OIDC/SAML SSO federation (GitHub, Google, Okta, Entra ID); optional -- ArgoCD has a local `admin` user as well |
| ApplicationSet Controller | The "one work order per site" template machine | Generates many Applications from one template + a generator (Git / cluster / matrix / list / pull request) |
| Notification Controller | The foreman's walkie-talkie to Slack/email | Emits notifications on sync/health events to Slack, email, webhooks |
| `source` | The vault URL + shelf + edition | `repoURL`, `path`, `targetRevision` (branch/tag/SHA), plus `helm:` or `kustomize:` config |
| `destination` | The physical site | `server` (API server URL or `in-cluster`) + `namespace` |
| `targetRevision` | "Build from the blueprint as of commit X" | Branch name, tag, or SHA; **HEAD** means "whatever the branch points to right now" (classic GitOps foot-gun) |
| Sync status | Does the site match the blueprint? | `Synced` or `OutOfSync` -- comparison of **rendered manifests** to live cluster, NOT Helm release revisions |
| Health status | Are the built things actually standing? | `Healthy`, `Progressing`, `Degraded`, `Suspended`, `Missing`, `Unknown` -- a per-resource and aggregate assessment |
| Target state | What the blueprint says | Result of `helm template` / `kustomize build` on the commit at `targetRevision` |
| Live state | What is actually on site | What the Kubernetes API returns for the Application's managed resources |
| Refresh | Foreman glances at the blueprint vault | Updates the cached target/live state; does NOT apply changes |
| Sync | Foreman picks up hammer and makes site match blueprint | Reconciliation action: apply the diff to the cluster |
| `syncPolicy.automated` | Foreman is always on site, not waiting to be called | Enables the reconciliation loop; without it, syncs only run on manual trigger |
| `syncPolicy.automated.prune` | Foreman tears down walls the blueprint no longer specifies | Deletes cluster resources that are no longer in Git |
| `syncPolicy.automated.selfHeal` | Foreman puts walls back after vandalism | Reverts out-of-band `kubectl edit` changes; fires after a controller-level timeout (default ~5s, tunable via `--self-heal-timeout-seconds` on the Application Controller or `controller.self.heal.timeout.seconds` in `argocd-cmd-params-cm`) |
| `syncPolicy.automated.allowEmpty` | "It's okay if the blueprint is blank this one time" | Safety flag -- when `false` (default), an empty Git commit does NOT trigger pruning of everything |
| `syncOptions: Replace=true` | Replace the wall wholesale instead of patching it | `kubectl replace` semantics instead of `kubectl apply`; required for large ConfigMaps/CRDs exceeding the last-applied-configuration annotation limit |
| `syncOptions: ServerSideApply=true` | Use the new apply API with proper field ownership | Kubernetes server-side apply; avoids the 256KB annotation limit and produces cleaner field ownership |
| `syncOptions: CreateNamespace=true` | Foreman digs the foundation before building | Creates `destination.namespace` if it doesn't exist, with Application-managed labels |
| `syncOptions: PruneLast=true` | Demolish last, after new construction is standing | Defers pruning until after all new/updated resources have applied |
| Sync wave | Call-sheet ordering for carpenters | `argocd.argoproj.io/sync-wave: "5"` annotation -- lowest number first; a wave waits for ALL resources in the previous wave to be Healthy before advancing |
| Sync phase | Stage curtains: before show / show / after show | `PreSync`, `Sync`, `PostSync` (+ `SyncFail` on errors); hooks are scheduled into phases; main resources live in `Sync` |
| Hook | Guest contractor (migration crew, smoke-test crew) who shows up for one phase | Resource annotated `argocd.argoproj.io/hook: PreSync` (usually a `Job`); participates in wave ordering and health-gating unlike Helm hooks |
| Hook delete policy | "When do we send the contractor home?" | `HookSucceeded`, `HookFailed`, `BeforeHookCreation` -- when unset, ArgoCD applies `BeforeHookCreation` implicitly (prior hook replaced on next sync); add `HookSucceeded` to clean up immediately on success |
| Health check (built-in) | Site inspector's standard checklist | Shipped checks for Deployment, StatefulSet, Service, Ingress, PVC, Job, etc.; inspect `.status` fields |
| Custom health (Lua) | Custom inspection rubric for a bespoke structure | Lua script in `argocd-cm` ConfigMap under `resource.customizations.health.<group>_<kind>`; returns `{status, message}` |
| `helm template` vs `helm install` | Print the drawings vs break ground | **ArgoCD uses `helm template`** -- no release Secrets, no `helm history`, no `helm rollback`, `lookup` returns empty |
| Git history + ArgoCD rollback | The blueprint vault's version log replaces the construction journal | ArgoCD stores the last N synced revisions; rollback is `git revert` + re-sync, not `helm rollback` |
| Resource tracking | How the foreman marks walls as "built by me" | Annotation (`argocd.argoproj.io/tracking-id`) is the default; label mode (`app.kubernetes.io/instance`) is legacy |

---

## The Site-Foreman Framework

Every ArgoCD concept maps to one of five roles on the GitOps construction site:

1. **The Blueprint Vault (Git repository)** -- The single source of truth. Contains Helm charts, Kustomize overlays, or plain YAML. Versioned, immutable, auditable. Every change is a commit. Every rollback is a `git revert`.

2. **The Work Order (Application CRD)** -- A Kubernetes custom resource in the ArgoCD namespace that pairs a vault location (`source`) with a construction site (`destination`) and a policy for how aggressively the foreman should enforce the blueprint (`syncPolicy`).

3. **The Foreman (Application Controller)** -- The reconciliation loop running inside the cluster. Pulls the blueprint from the vault (via the Repo Server), compares to the live cluster, and makes them match. Never asked "what should I build?" -- always answers by re-reading Git.

4. **The Supporting Crew (API Server, Repo Server, Redis, Dex, ApplicationSet Controller, Notification Controller)** -- Six more components, each with one focused job. The API Server is the radio to HQ (UI, CLI, SSO). The Repo Server is the photocopy room (clones Git, runs `helm template`). Redis is the clipboard (cache). Dex is ID verification. ApplicationSet is a work-order template machine. Notifications is the walkie-talkie to Slack.

5. **The Sync Machinery (phases, waves, hooks, health checks)** -- The rules the foreman follows when actually applying the blueprint: install in wave order, wait for each wave's resources to be healthy before advancing, run PreSync hooks for migrations, run PostSync hooks for smoke tests, reverse the wave order when tearing down.

```
THE GITOPS CONSTRUCTION SITE -- ARGOCD OVERVIEW
================================================================================

  BLUEPRINT VAULT                 ARGOCD (in-cluster)                SITE
  ---------------                 -------------------                ----

  +--------------+
  | Git repo     |
  |  charts/api/ |
  |  values/     |                +-------------------+
  |    dev.yaml  |<---clone-------|  Repo Server      |
  |    prod.yaml |                |  (photocopy room) |
  |  Chart.yaml  |                |                   |
  +--------------+                |  helm template    |
                                  |  kustomize build  |
                                  +-------+-----------+
                                          |
                                 rendered | manifests (YAML)
                                          v
                                  +-------------------+           +------------+
                                  | Application       |<--diff--->| Kubernetes |
                                  | Controller        |           | API Server |
                                  | (the foreman)     |           +------------+
                                  |                   |                 |
                                  | reconcile loop    |---kubectl--+    |
                                  |  every 3 min      |    apply   |    |
                                  |  + webhook pings  |            v    v
                                  +---+---+-----------+           +-------------+
                                      |   ^                        | Deployments|
                                      |   |                        | Services   |
                                      v   | watch live state       | Ingresses  |
                                  +--------------+                  | ConfigMaps |
                                  | Redis        |                  | ...        |
                                  | (clipboard)  |                  +-------------+
                                  +--------------+

  USER INTERFACES                 EXTRAS
  ---------------                 ------
  +--------------+                +----------------+
  | argocd CLI   |---grpc-------->| API Server     |<--OIDC/SAML--+ Dex (HR)
  | Web UI       |                | (radio to HQ)  |              +--> GitHub,
  | Webhooks     |                |                |                   Okta, etc.
  +--------------+                +----------------+
                                  +----------------+
                                  | ApplicationSet |  (template machine: 1 spec
                                  | Controller     |   -> N Applications, driven
                                  +----------------+    by Git/Cluster/List
                                                        generators)
                                  +----------------+
                                  | Notification   |--->  Slack / email / webhook
                                  | Controller     |
                                  +----------------+
```

---

## Part 1: The Four OpenGitOps Principles

Before any tool, **GitOps is a principled operational model**, codified by the CNCF-chartered OpenGitOps project (https://opengitops.dev/). The principles are tool-agnostic: ArgoCD implements them, Flux implements them, a sufficiently disciplined shell script could implement them. Missing any one means you are not doing GitOps -- you are doing "CI/CD that happens to use Git."

### Principle 1: Declarative

**The desired state of the system is expressed declaratively.**

Blueprints say *what* the building should look like, not *how* to build it. You commit a `Deployment` manifest that declares "3 replicas of image X" -- you do not commit a bash script that says `kubectl scale deployment api --replicas=3`. Declarative state is diffable, idempotent, and replayable. Imperative scripts are none of those.

Helm charts and Kustomize overlays are declarative. Bash wrappers around `kubectl patch` are not. Terraform is declarative. `aws cli` one-liners in a pipeline are not.

### Principle 2: Versioned and Immutable

**The desired state is stored in a way that enforces immutability, versioning, and retains a complete version history.**

Every change is a commit. You cannot alter history without forcing a push. You can trace "why is production running image 2.8.3?" to a specific commit, a specific PR, a specific reviewer, a specific hour on a specific Tuesday. This is what "Git is the audit log" means -- there is no separate audit log to fall out of sync with the cluster.

The contrast: a ClickOps workflow where someone runs `helm upgrade --set image.tag=2.8.3` on their laptop leaves zero trace in Git. The cluster drifts from what the repo shows. Nobody knows.

### Principle 3: Pulled Automatically

**Software agents automatically pull the desired state declarations from the source.**

The cluster goes to Git, not the other way around. Three benefits, each important:

1. **No outbound credentials.** Your CI system does not need cluster-admin tokens. The cluster has Git read credentials, nothing more. This cleanly removes one of the worst security footguns in Jenkins-style CD (long-lived cluster kubeconfigs stored in CI secrets).
2. **Firewall-friendly.** The cluster only needs outbound HTTPS to Git. Inbound ports stay closed. This matters in regulated environments where inbound control-plane access from CI runners is prohibited.
3. **Pulls survive CI outages.** If your CI platform goes down, nothing happens to deployments -- the controller keeps reconciling from Git independently.

This is the defining difference between **pull-based CD** (ArgoCD, Flux) and **push-based CD** (Jenkins, GitHub Actions running `kubectl apply`). Push-based CD violates Principle 3 by definition.

### Principle 4: Continuously Reconciled

**Software agents continuously observe actual system state and attempt to apply the desired state.**

Deploying once is easy. Staying deployed is the hard part. Humans run `kubectl edit` at 03:00 during incidents. Admission controllers inject sidecars that vanish on the next re-apply. Cloud providers occasionally rehydrate resources. Continuous reconciliation means the controller is **always running**, not just at deploy time -- the site foreman is on the site 24/7, not just showing up at kickoff.

ArgoCD's default reconciliation interval is 3 minutes (configurable via `timeout.reconciliation` in the `argocd-cm` ConfigMap). Webhooks from Git let it react faster to commits. **Self-heal** (covered later) is the specific feature that reverts out-of-band cluster changes -- the operational teeth of Principle 4.

### Push vs Pull -- The Comparison

| Dimension | Push CD (Jenkins, GitHub Actions `kubectl`) | Pull CD (ArgoCD, Flux) |
|-----------|---------------------------------------------|------------------------|
| Who holds cluster credentials? | CI platform (long-lived tokens) | Nobody outside the cluster |
| Who initiates deploys? | CI pipeline on Git push | Controller inside the cluster |
| What happens on CI outage? | Deploys stop | Reconciliation continues |
| How is drift detected? | Not detected | Detected on every reconcile loop |
| How is drift corrected? | Next manual deploy | `selfHeal: true` auto-reverts |
| Multi-cluster fan-out? | Separate pipeline per cluster | One ApplicationSet, N clusters |
| Audit trail? | CI logs (separate system) | Git history + ArgoCD events |
| Secrets in Git? | Discouraged | Forbidden (same discouragement, enforced by the model) |

The principles do not outlaw CI; CI still builds images, runs tests, lints Helm charts, and writes back new image tags to Git. CI stops at Git. **CD (cluster reconciliation) starts from Git.** This is "the CI/CD boundary lives at the Git repo," which is the practical one-line summary of GitOps that sticks once you've held the mental model for a while.

---

## Part 2: ArgoCD Architecture -- The Seven Components

ArgoCD is not one process. It is seven components running as Deployments/StatefulSets in the `argocd` namespace, each with a narrow job. Understanding who does what answers questions like "why didn't my sync pick up the new commit?" or "why did the UI time out?" -- those are different components failing.

### The Components

```
+----------------------------------------------------------------------------+
|                        argocd namespace                                    |
|                                                                            |
|  +-----------------+     +------------------+     +----------------------+ |
|  | API Server      |     | Repo Server      |     | Application          | |
|  | (gRPC/REST)     |---->| (clones Git,     |<----| Controller           | |
|  |                 |     |  helm template,  |     | (reconcile loop,     | |
|  | - UI/CLI auth   |     |  kustomize build)|     |  compares desired    | |
|  | - exposes APIs  |     |                  |     |  vs live, triggers   | |
|  | - SSO (via Dex) |     | NEVER talks to   |     |  sync, self-heal)    | |
|  +--------+--------+     | target clusters  |     +----------+-----------+ |
|           |              +------------------+                |             |
|           | auth                                             | apply       |
|           v                                                  v             |
|  +-----------------+                                +----------------------+|
|  | Dex (optional)  |                                | Kubernetes API       ||
|  | OIDC/SAML SSO   |                                | (target cluster)     ||
|  +-----------------+                                +----------------------+|
|                                                                            |
|  +-----------------+     +------------------+     +----------------------+ |
|  | Redis           |     | ApplicationSet   |     | Notification         | |
|  | (cache)         |     | Controller       |     | Controller           | |
|  | - manifest cache|     | - generators     |     | - Slack / email /    | |
|  | - cluster info  |     |   (git, cluster, |     |   webhook triggers   | |
|  +-----------------+     |    matrix, list, |     +----------------------+ |
|                          |    PR)           |                              |
|                          +------------------+                              |
+----------------------------------------------------------------------------+
```

**1. API Server (argocd-server)**

The gRPC and REST gateway. Serves the `argocd` CLI, the web UI, webhook endpoints from Git providers, and the SSO auth flow (delegated to Dex or a direct OIDC provider). **Stateless** -- scales horizontally. Does not run reconciliation; does not talk to target clusters directly. Think "the foreman's radio to HQ" -- it is how humans and automation interact with ArgoCD, not how ArgoCD interacts with clusters.

**2. Repo Server (argocd-repo-server)**

The "photocopy room." Clones Git repositories, resolves Helm chart dependencies, and runs one of:

- `helm template` (for Helm sources)
- `kustomize build` (for Kustomize sources)
- plain YAML rendering (for directory sources)
- a custom config management plugin (CMP) for anything else

Emits rendered manifests to the Application Controller via gRPC. **Never touches the cluster.** The Repo Server is pure rendering -- it has no `kubeconfig`, no credentials for the target cluster. This is a security design choice: a compromised Repo Server cannot apply changes; it can only produce manifests a controller may or may not apply.

The `argocd-repo-server` is also the component that caches rendered manifests in Redis (default TTL 24 hours) -- so re-renders are cheap when Git has not changed.

**3. Application Controller (argocd-application-controller)**

The foreman. The heart of ArgoCD. A StatefulSet (runs as a replicated statefulset to allow sharding by cluster for scale). The reconciliation loop does:

1. Refresh: ask the Repo Server for rendered manifests at `targetRevision`.
2. Observe: list all resources the Application owns from the target cluster.
3. Diff: compute the JSON-patch difference between desired and live.
4. Decide: if there is a diff and `syncPolicy.automated` is set, trigger a sync.
5. Apply: `kubectl apply` (or `replace`, or server-side apply) the desired manifests in wave/phase order.
6. Health-check: evaluate each resource's health using built-in or custom (Lua) checks.
7. Report status back to the Application CRD and optionally emit notifications.

For scale, the Application Controller supports **sharding**: each replica owns a subset of clusters, distributed by hash. At large scale (hundreds of clusters) you also have to tune `--repo-server-timeout-seconds` and cache sizes, but the basic architecture is "one foreman per shard of sites."

**4. Redis**

Cache, not a database. Stores rendered manifest cache (so repeated syncs of the same commit don't re-run `helm template`), cluster state cache (so the UI doesn't re-list all namespaces on every refresh), and session state for UI users. **Losing Redis is not data loss** -- the Application Controller and Repo Server rebuild the cache. Losing Redis mid-reconcile causes slow first-reconcile-after-recovery and nothing worse.

ArgoCD ships with a single Redis pod by default. Production at scale uses `redis-ha` (Sentinel-based) or externally-managed Redis (ElastiCache).

**5. Dex (optional)**

OIDC/SAML federation proxy. Bridges ArgoCD's auth model to GitHub, Google, Okta, Entra ID, LDAP, etc. If your identity provider speaks OIDC directly, you can skip Dex and point ArgoCD's API server at the IdP -- Dex exists specifically for non-OIDC providers (SAML, LDAP, GitHub's classic OAuth flow) and for unified multi-IdP setups. Many production deployments run without Dex.

The local `admin` user (bootstrap credentials in the `argocd-initial-admin-secret` Secret) works without Dex and is how you first log in.

**6. ApplicationSet Controller (argocd-applicationset-controller)**

A template machine for Applications. You write one `ApplicationSet` manifest with a template (looks like an Application) and one or more **generators** that produce parameter sets. The controller renders the template against each parameter set and creates/updates an Application per parameter set.

Generators include:

- **List generator**: an explicit list of parameter sets (env: dev / staging / prod).
- **Cluster generator**: one Application per cluster registered with ArgoCD.
- **Git generator**: one Application per directory (or per file matching a glob) in a Git repo.
- **Matrix generator**: Cartesian product of two other generators (env x cluster).
- **Pull request generator**: one Application per open PR in a Git repo (for preview environments).

ApplicationSets are Day 4's topic (tomorrow's doc) -- mentioned here because it is a core architectural component, but we won't go deep today.

**7. Notification Controller (argocd-notifications-controller)**

Emits notifications based on Application events (synced, health-changed, failed) to Slack, email, webhooks, Telegram, etc. Configured via the `argocd-notifications-cm` ConfigMap with a trigger-template-subscription model. Optional -- many teams skip it in favor of alerting on the Prometheus metrics ArgoCD exposes.

### Why the Repo Server Never Talks to the Cluster

This is a subtle but important design point. The Repo Server **only reads Git** and **only emits rendered YAML**. It holds no `kubeconfig` for target clusters. The Application Controller holds cluster credentials and does all `kubectl apply` work.

Why split them? Three reasons:

1. **Blast radius.** A compromised Repo Server can at worst serve malicious manifests the Controller may apply. It cannot bypass the Controller and write to the cluster directly.
2. **Scaling axis.** Rendering (CPU-bound, memory-bound on big charts) and reconciliation (network-bound, API-server-bound) scale differently. Splitting them lets you scale Repo Server pods independently when you have many charts, and Application Controller replicas when you have many clusters.
3. **Template-engine upgrades.** Bumping Helm or Kustomize versions means bumping only the Repo Server image. The Controller is unaffected.

This is also why, when you see "HeLM template output is wrong" bugs in ArgoCD, you `kubectl exec` into the Repo Server, not the Controller, to debug them.

---

## Part 3: The Application CRD -- The Central Abstraction

The **Application** custom resource is the central object you interact with in ArgoCD. It pairs a Git source with a cluster destination and a sync policy. Everything in the UI, the CLI, the notifications -- it is all a view of Application CRDs.

### A Minimal Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api-prod
  namespace: argocd                        # Applications live in the ArgoCD namespace
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # On delete, also delete the Application's managed resources
spec:
  project: platform                        # AppProject scoping (RBAC + allowed destinations)

  source:                                  # WHERE the blueprint lives
    repoURL: https://github.com/acme/infra.git
    path: charts/payments-api              # Path within the repo
    targetRevision: main                   # Branch / tag / SHA; HEAD-of-branch is a classic foot-gun

  destination:                             # WHERE to apply it
    server: https://kubernetes.default.svc # "in-cluster" or an external cluster API URL
    namespace: payments

  syncPolicy:
    automated:                             # Enables the reconciliation loop
      prune: true                          # Delete resources removed from Git
      selfHeal: true                       # Revert out-of-band cluster edits
      allowEmpty: false                    # Safety: empty Git = refuse to prune everything
    syncOptions:
      - CreateNamespace=true               # Create the destination namespace if missing
      - PruneLast=true                     # Prune after new resources are applied
      - ServerSideApply=true               # Use server-side apply (avoids 256KB last-applied annotation limit)
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

The four top-level spec fields -- `project`, `source`, `destination`, `syncPolicy` -- are the skeleton. Everything else is configuration of those.

### `source` Deep Dive -- Helm-Specific Fields

For Helm sources (the most common case given your Helm background), the `source.helm` block is where the action is:

```yaml
source:
  repoURL: https://github.com/acme/infra.git
  path: charts/payments-api
  targetRevision: v1.4.2

  helm:
    # Release name (default: Application name)
    releaseName: payments-api

    # Chart value files (relative to the chart path)
    valueFiles:
      - values.yaml                        # Chart defaults (always loaded if you list it)
      - values-prod.yaml                   # Per-env overrides

    # Inline values as a YAML string -- the OLDER form
    values: |
      replicaCount: 10
      image:
        tag: "2.8.2"

    # Inline values as a proper YAML object -- the MODERN form (v2.6+)
    valuesObject:
      replicaCount: 10
      image:
        tag: "2.8.2"
      database:
        url: postgres://prod-db.internal:5432/payments
      resources:
        requests:
          cpu: 250m
          memory: 256Mi

    # Per-key overrides, semantically equivalent to `--set key=value`
    parameters:
      - name: image.tag
        value: "2.8.2"
      - name: replicaCount
        value: "10"
        forceString: true                  # Keep as string, don't coerce to int

    # Reference values in files (e.g., binary data, Docker configs)
    fileParameters:
      - name: dockerConfig
        path: files/dockerconfig.json

    # Pass repo credentials to Helm when pulling chart dependencies from protected repos
    passCredentials: false

    # Use --skip-crds when rendering (for charts that ship CRDs you manage separately)
    skipCrds: false

    # Use --include-crds (the opposite)
    ignoreMissingValueFiles: false
```

**Values precedence in ArgoCD's Helm integration** (lowest to highest, highest wins):

1. Chart's own `values.yaml` (always loaded)
2. `valueFiles` (left to right; rightmost wins)
3. `values` (inline YAML string)
4. `valuesObject` (inline YAML object -- **preferred over `values`**)
5. `parameters` (individual key overrides)
6. `fileParameters` (file-based overrides, usually binary data)

This is **the Helm precedence you already know**, mapped into CRD fields. The one surprise for Helm natives: `parameters` (not `--set`) sits at the top, overriding everything. If your Application has both `valuesObject.replicaCount: 10` and `parameters: [{name: replicaCount, value: "20"}]`, the rendered manifests get `replicaCount: 20`.

**Why `valuesObject` beats `values`**: `values` is a YAML-as-a-string field -- you write `values: |` followed by a YAML block. IDE autocomplete does nothing. Typos in indentation produce cryptic render errors. `valuesObject` is a proper YAML object, validated as you type by any Kubernetes-aware editor, and works with JSON Schema. Always prefer `valuesObject` for new Applications; `values` is legacy.

### `destination` Deep Dive

```yaml
destination:
  server: https://kubernetes.default.svc   # Target cluster API URL; "in-cluster" for ArgoCD's home cluster
  # OR, preferred: reference by name
  name: prod-us-east-1                     # Cluster name (must be registered via `argocd cluster add`)
  namespace: payments
```

Two mutually exclusive ways to name a cluster: **`server`** (API URL) or **`name`** (friendly name, resolved via registered cluster secrets in `argocd` namespace). Use `name` in production -- it is stable across cluster rebuilds where the URL might change.

`namespace` is the default namespace for resources that don't explicitly set one. Resources with their own `metadata.namespace` keep it. Cluster-scoped resources (ClusterRoles, CRDs) ignore `destination.namespace`.

### `project` and `AppProject`

Every Application belongs to an `AppProject` (default: the auto-created `default` project with no restrictions). Projects scope what an Application is allowed to do:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: Platform team applications

  # Allowed source repositories
  sourceRepos:
    - https://github.com/acme/infra.git
    - oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts/*

  # Allowed destinations
  destinations:
    - namespace: 'payments*'
      server: https://kubernetes.default.svc
    - namespace: 'monitoring'
      server: https://kubernetes.default.svc

  # Cluster-scoped resources this project may touch
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: rbac.authorization.k8s.io
      kind: ClusterRole

  # Namespaced resources this project may touch (default: everything)
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'

  # RBAC roles for this project (who can sync, who can override, etc.)
  roles:
    - name: platform-deploy
      policies:
        - p, proj:platform:platform-deploy, applications, sync, platform/*, allow
        - p, proj:platform:platform-deploy, applications, override, platform/*, allow
      groups:
        - acme:platform-team                  # OIDC group claim
```

Projects are the RBAC boundary in ArgoCD. A compromised Application (e.g., attacker gets write on a Git repo) cannot deploy to namespaces or clusters outside what its project allows. Day 4's doc covers Projects in depth; for today, know that every Application has a `project` and the `default` project is permissive.

---

## Part 4: Sync Status and Health Status

Two orthogonal pieces of status live on every Application, and conflating them is the #1 source of confusion for Helm-native users.

### Sync Status: "Does the cluster match Git?"

| Status | Meaning |
|--------|---------|
| `Synced` | Every rendered manifest equals the live resource (by spec-diff) |
| `OutOfSync` | At least one rendered manifest differs from the live resource (or is missing) |
| `Unknown` | ArgoCD cannot determine -- usually during initial refresh or repo server errors |

Sync status is computed by diffing **rendered manifests** (output of `helm template`) against **live cluster state** (what the Kubernetes API returns). It is *not* a Helm release revision comparison -- there are no Helm release revisions at all in ArgoCD's world. If you `kubectl edit deployment api` and change `replicas: 3` to `replicas: 10`, the Application goes `OutOfSync` immediately because the rendered manifest says 3 and the cluster says 10, even though Git hasn't changed.

### Health Status: "Are the deployed things actually running correctly?"

| Status | Meaning |
|--------|---------|
| `Healthy` | Resource is up and serving (`Deployment` has `updatedReplicas == replicas`, Pods Ready, Services have endpoints, etc.) |
| `Progressing` | Currently rolling -- Deployment rollout in progress, Pods starting |
| `Degraded` | Something is wrong: replicas unavailable, probes failing, Ingress has no LB |
| `Suspended` | Intentionally paused -- e.g., `Deployment.spec.paused: true`, `CronJob.spec.suspend: true` |
| `Missing` | Resource declared but not found in cluster (brief -- usually only during initial sync) |
| `Unknown` | No health check defined for this kind |

Health is per-resource; the Application's aggregate health is the **worst** health across all its resources (one Degraded resource makes the Application Degraded).

### The Two-Axis Matrix

```
                  Healthy          Degraded
              +----------------+-----------------+
   Synced     | Everything     | Cluster matches |
              | is fine        | Git, but pods   |
              |                | are crashing    |
              +----------------+-----------------+
   OutOfSync  | New commit     | Drift detected  |
              | pending sync   | AND things are  |
              |                | broken          |
              +----------------+-----------------+
```

Each cell is a different operational situation. **Synced + Degraded** is "Git is right but the app is broken" -- rollback in Git or fix the bug. **OutOfSync + Healthy** is "new commit waiting to deploy, current deploy is fine" -- normal state during rollout windows. Read both axes together; neither alone tells you what is happening.

### Refresh vs Sync -- A Helm-Native's Most Common Confusion

- **Refresh** updates the cached target state (re-fetches Git, re-runs `helm template`) and re-diffs against live. **It does not apply anything.** It is "look at the current blueprint and say whether we match." Triggered automatically every 3 minutes, on webhooks, or manually via `argocd app get --refresh` / "Refresh" button.
- **Sync** actually applies the diff. **It is the only operation that changes the cluster.** Triggered automatically (if `syncPolicy.automated` is set) or manually via `argocd app sync` / "Sync" button.

The classic mistake: "the Application shows OutOfSync, I clicked Refresh, nothing happened." Refresh doesn't apply. You need to Sync (or enable auto-sync).

---

## Part 5: Sync Policies -- The Three Toggles and Their Teeth

Sync policies live in `syncPolicy.automated` and control *when* ArgoCD applies changes without human intervention.

### `automated` -- Enable the Reconciliation Loop

```yaml
syncPolicy:
  automated: {}    # Just enabling the block is enough to turn on auto-sync
```

Without `automated`, syncs only happen when a human triggers them (UI button, `argocd app sync`, or Git webhook with manual confirmation). This is **manual sync mode** -- fine for sensitive environments (prod with change-approval workflows) where you want Git-as-source-of-truth but explicit human gate before applying.

With `automated`, the Application Controller applies any detected diff automatically. This is the default for most non-prod environments.

### `automated.prune` -- Delete Resources Removed from Git

```yaml
syncPolicy:
  automated:
    prune: true
```

Default is `false`. Without `prune`, deleting a Deployment from Git does **not** delete it from the cluster -- ArgoCD reports `OutOfSync` (the cluster has something Git doesn't) but won't remove it. With `prune: true`, ArgoCD deletes resources that existed in a previous sync but are no longer in the current Git state.

**Gotcha**: per-resource sync-option `Prune=false` on specific resources overrides the Application-level `prune: true`. Useful for "auto-sync everything but never delete the PVC even if it disappears from Git."

### `automated.selfHeal` -- Revert Manual Cluster Changes

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

Default is `false`. Without self-heal, ArgoCD only syncs when Git changes. With self-heal, ArgoCD also syncs when the **live cluster** drifts from Git -- someone `kubectl edit`-s a Deployment, ArgoCD notices on the next reconcile, and reverts.

**Drift detection vs self-heal.** These are two separate things that are easy to conflate. ArgoCD **always** detects drift on every reconcile regardless of self-heal setting (that's how `OutOfSync` status is computed). Self-heal is specifically about *correcting* the drift automatically -- without it, ArgoCD flags the drift but waits for a human to hit Sync.

**The self-heal timer is controller-level configurable.** When drift is detected and `selfHeal: true`, ArgoCD waits ~5 seconds by default before applying the correction. This window exists so legitimate controllers (HPA scaling, rolling updates) that *intentionally* modify fields don't trigger tug-of-war loops. The window is tunable via:

- `--self-heal-timeout-seconds` flag on the `argocd-application-controller` process
- `controller.self.heal.timeout.seconds` key in the `argocd-cmd-params-cm` ConfigMap (note: `argocd-cmd-params-cm`, not `argocd-cm` -- controller flags live separately)

Recent ArgoCD versions also add exponential backoff for repeated self-heal corrections (`--self-heal-backoff-factor`, `--self-heal-backoff-cap-seconds`) so a pathological tug-of-war with another controller doesn't burn CPU.

**You cannot disable self-heal per-sync.** It is an Application-level setting. If you need "self-heal for everything except this one Deployment's `replicas` field," use `spec.ignoreDifferences` (next section) -- that is the canonical HPA coexistence pattern. Removing `replicas` from the chart also works but breaks initial deploy when no HPA exists yet.

### `automated.allowEmpty` -- The "Empty Repo Safety"

```yaml
syncPolicy:
  automated:
    prune: true
    allowEmpty: false     # DEFAULT; keep this on production
```

Default is `false`, which is what you want. If Git becomes empty (someone accidentally `rm -rf`-s a chart directory, or a bad refactor commits an empty manifest set), ArgoCD **refuses to prune** the entire Application. `OutOfSync` shows the missing resources, but the cluster stays up. Flip `allowEmpty: true` only when you genuinely want "empty Git means empty cluster" -- almost never.

This is the single most important safety toggle. If your Git workflow ever produces a zero-manifest Application (e.g., via ApplicationSet generator misconfiguration), `allowEmpty: false` is the seat belt that keeps production from vanishing.

### `ignoreDifferences` -- Coexisting with Other Controllers

The foreman has one weakness: if something else legitimately writes to a field ArgoCD manages (HPA updates `.spec.replicas`, a mutating webhook injects sidecars, Cilium annotates Services), every reconcile sees drift and self-heal fights to revert it. The canonical fix is **not** to disable self-heal or remove the field from the chart -- it is to tell ArgoCD "this specific field is none of your business":

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api-prod
spec:
  # ... source, destination, syncPolicy as before ...
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas              # HPA manages this
    - group: ""
      kind: Service
      jsonPointers:
        - /spec/clusterIP             # set by API server, not by Git
    - group: apps
      kind: Deployment
      jqPathExpressions:
        - '.spec.template.spec.containers[] | select(.name == "istio-proxy")'
      # sidecar injected by mutating webhook; ignore the whole istio-proxy container
    - group: apps
      kind: Deployment
      managedFieldsManagers:
        - kube-controller-manager
      # ignore ANY field owned by kube-controller-manager per server-side apply field ownership
```

`jsonPointers` uses RFC 6901 JSON Pointer syntax (`/spec/replicas`). `jqPathExpressions` is more expressive for array filtering (ignore a specific sidecar container without hardcoding its index). `managedFieldsManagers` ties directly into Kubernetes server-side apply field ownership -- tell ArgoCD to ignore any field whose current owner is the named manager.

**The HPA coexistence pattern (most common use):**

1. Keep `replicas: 3` in your chart so initial deploy has a sensible starting count when no HPA exists yet.
2. Deploy the HPA in the same chart with its own sync wave (after the Deployment).
3. Add `ignoreDifferences` on `/spec/replicas` to the Application so HPA can scale freely without ArgoCD reverting it.
4. **Add `RespectIgnoreDifferences=true` to `syncPolicy.syncOptions`** -- this is the load-bearing step most guides miss.

Without step 3, every HPA scaling event triggers drift detection and self-heal fights the HPA. With step 3 *but not* step 4, `ignoreDifferences` only affects the **diff computation** (the UI shows Synced even if the field differs), but the sync operation itself still applies the full manifest. The first time any *other* field triggers a sync (a real Git change, drift on a sibling resource), the apply rewrites `.spec.replicas` back to the chart value and restarts the thrashing cycle. `RespectIgnoreDifferences=true` makes the sync phase itself skip the ignored fields:

```yaml
spec:
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - RespectIgnoreDifferences=true     # <-- THIS is what makes the sync respect ignoreDifferences
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

With all four steps, ArgoCD sees `.spec.replicas` as "not mine to diff **or** apply" and the two controllers peacefully co-manage the Deployment.

**When to reach for `ignoreDifferences`:**

- HPA / VPA managing replica or resource fields
- Istio / Linkerd sidecar injection (mutating webhooks)
- External secrets operator writing back to `.data` on Secrets
- AWS Load Balancer Controller annotating Services
- Cilium / other CNIs annotating Services or Pods
- Any field whose true owner is not your chart

### Per-Resource Sync Options

Beyond `syncPolicy.syncOptions` (Application-level) and `syncPolicy.automated` (auto-sync toggles), individual resources can carry annotations that modify sync behavior:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  annotations:
    argocd.argoproj.io/sync-options: Replace=true,ServerSideApply=true,Prune=false
    argocd.argoproj.io/sync-wave: "2"
```

The most useful sync options:

| Option | Meaning | When to use |
|--------|---------|-------------|
| `Replace=true` | Use `kubectl replace` instead of `kubectl apply` | Large ConfigMaps/CRDs that exceed the 256KB `kubectl.kubernetes.io/last-applied-configuration` annotation limit; resources with schema that `apply` cannot merge cleanly |
| `ServerSideApply=true` | Use server-side apply (Kubernetes feature) | Same large-resource use case as `Replace`, but cleaner -- Kubernetes tracks field ownership; preferred for CRDs |
| `CreateNamespace=true` | Application-level only: create `destination.namespace` | Always set it on new Applications unless the namespace is managed by another Application |
| `PruneLast=true` | Defer pruning until after all new/updated resources apply | Deployments that depend on resources being replaced (not co-existing) with new ones |
| `Prune=false` | Never prune this resource | PVCs, manually-imported resources, anything with data you don't want ArgoCD to delete |
| `Validate=false` | Skip schema validation | CRDs that install their own validation webhook; occasionally necessary during bootstrap |
| `SkipDryRunOnMissingResource=true` | Skip the server-side dry-run if the resource doesn't exist | Required for CRDs that create themselves (operators that bootstrap their own kinds) |
| `RespectIgnoreDifferences=true` | Make the sync phase skip fields listed in `spec.ignoreDifferences` | **Required for `ignoreDifferences` to actually work end-to-end.** Without it, `ignoreDifferences` only affects diff display; sync still rewrites the fields. Always pair with `ignoreDifferences` for HPA/VPA/webhook coexistence |

**`ServerSideApply=true`** is increasingly the default for any Application with large ConfigMaps, Prometheus rules, or CRD-heavy charts. The `last-applied-configuration` annotation has a 256KB hard limit; blow past it and syncs fail with `metadata.annotations: Too long`. Server-side apply sidesteps this entirely and produces cleaner field-ownership semantics.

### Decision Framework: Auto-Sync vs Manual Sync

```
Environment has change-approval workflows (prod, regulated)     -> manual sync
Environment is ephemeral (dev, PR preview)                      -> auto-sync + prune + self-heal
Application owns stateful data (database, queue with messages)  -> auto-sync, prune=false per-PVC
Application is multi-tenant shared (ingress controller, CNI)    -> manual sync, extra review
Application is frequently drifted by humans (dev cluster)       -> self-heal ON to enforce Git
Application is deliberately shared-write (HPA manages replicas) -> delegate the mutable fields,
                                                                    self-heal still ON for the rest
```

**Critical insight on "manual sync as approval gate":** many teams reach for manual sync thinking it's the way to gate production changes, but this conflates two orthogonal properties. **Human approval belongs at the Git layer (branch protection + required reviewers + CODEOWNERS), not at the sync layer.** Turning off auto-sync to create an "approval gate" *also* turns off drift correction -- you lose Principle 4 (Continuous Reconciliation) in exchange for a gate you could have built at the PR level. The more sophisticated production pattern is:

- **Auto-sync + self-heal ON in every environment** (including prod), so drift is always corrected
- **Branch protection + required reviewers** on the Git repo for changes that land in prod paths
- **GitHub Environments / protection rules** for any automated image-tag updates (CI promotes images by writing back to Git, which is the gated step)

With this pattern, CI is gated, Git is gated, but reconciliation runs continuously. You get PR-style approval *and* continuous drift correction. Manual sync should be reserved for genuinely dormant environments (DR clusters that stay cold until failover) or highly-regulated scenarios where a human must physically click Sync as the change-ticket execution record.

A related pattern for dev iteration speed: rather than loosening sync policy in dev ("disable self-heal so engineers can `kubectl edit` freely" is a trap -- see Gotcha section), provision **per-PR preview namespaces via ApplicationSet Pull Request generator**. Every engineer gets a personal short-lived environment per PR, faster than any `kubectl edit`-based workflow, and fully GitOps-compliant.

### Decision Framework: Self-Heal On or Off

```
Does the team expect to kubectl-edit in emergencies?
  YES -> self-heal OFF in that env, or per-resource sync-options to exempt fields
  NO  -> self-heal ON

Does another controller (HPA, VPA) legitimately modify fields in your chart?
  YES -> remove those fields from the chart; let the other controller own them; self-heal ON
  NO  -> self-heal ON

Is the cluster a regulated env where drift must be auto-corrected for compliance?
  YES -> self-heal ON, always
```

The general prescription: **turn self-heal on everywhere you can, and design your charts so ArgoCD never owns fields that other controllers want.** Self-heal off is a crutch for charts with bad ownership boundaries.

---

## Part 6: Helm Integration -- The Critical Bridge

This section is where your existing Helm knowledge has to be partially unlearned. **ArgoCD does not run `helm install` or `helm upgrade`.** It runs `helm template` + `kubectl apply`. Internalizing the consequences is the single biggest mental-model shift from Helm-native to ArgoCD-native.

### What ArgoCD Actually Does with a Helm Source

When an Application has a Helm source:

1. Repo Server clones the Git repo at `targetRevision`.
2. Repo Server runs `helm dependency build` (if `Chart.lock` exists) to pull subchart tarballs.
3. Repo Server runs `helm template <releaseName> <chartPath> -f values.yaml -f values-prod.yaml --set image.tag=2.8.2 ...` -- the exact precedence chain you know from Helm.
4. The template output is a multi-document YAML stream -- a bunch of Deployments, Services, ConfigMaps, etc.
5. Repo Server returns this YAML stream to the Application Controller.
6. Application Controller diffs each resource against the live cluster.
7. Application Controller applies the diff via `kubectl apply` (or `replace` / server-side apply, depending on sync options).

**What is conspicuously missing**: `helm install`. `helm upgrade`. Helm release Secrets. `helm history`. `helm rollback`. Any Helm-side state whatsoever.

### The Helm -> ArgoCD Concept Map

This table is the one to burn into memory:

| Helm world | ArgoCD world | Notes |
|------------|--------------|-------|
| `helm install` | Create Application + first sync | |
| `helm upgrade` | Commit to Git + sync | |
| `helm upgrade --install` | Create-or-update Application (idempotent) | |
| `helm uninstall` | Delete Application (with `resources-finalizer.argocd.argoproj.io`) | |
| `helm rollback <rev>` | `git revert <commit>` + sync | ArgoCD stores last N synced revisions for UI rollback, but the canonical rollback is Git |
| `helm history` | Git log + ArgoCD sync history tab | Git is the real history |
| `helm list` | `argocd app list` or the UI | |
| `helm get values` | Inline `valuesObject` in the Application CRD | You already know what values are applied -- they're in Git |
| `helm get manifest` | `argocd app manifests` | Shows rendered YAML from the last sync |
| `helm install --dry-run=server` | `argocd app diff` or "Refresh" button | |
| `helm diff upgrade` (plugin) | `argocd app diff` | Built in |
| `helm.sh/hook` (Helm hook) | `argocd.argoproj.io/hook` (ArgoCD hook) | Different semantics -- ArgoCD hooks participate in wave/health gating |
| `helm.sh/hook-weight` | `argocd.argoproj.io/sync-wave` | Different syntax, same intent |
| `helm.sh/hook-delete-policy` | `argocd.argoproj.io/hook-delete-policy` | Same idea; ArgoCD has slightly different value names (`HookSucceeded` not `hook-succeeded`) |
| Helm release Secret | Nothing -- no such thing in ArgoCD | Revisions live in Git |
| `lookup` function | Returns empty dict always | No cluster access during `helm template` rendering in Repo Server |
| `.Release.IsInstall` / `.IsUpgrade` | Always `IsInstall=true` in `helm template` | The Repo Server has no concept of "upgrade" |

### What You Lose

Being explicit about what disappears when you move from `helm install` to ArgoCD:

1. **No `helm history`**. ArgoCD stores the last ~10 synced revisions in its own history tab (Application-level), but `kubectl get secret` in the release namespace returns no `sh.helm.release.v1` Secrets. If you have observability tooling that keys off release Secrets, it stops working.
2. **No `helm rollback`**. The Helm CLI cannot roll back an ArgoCD-deployed release because there are no release Secrets to roll back from. You rollback via `git revert <commit> && git push` + sync. ArgoCD also offers a UI/CLI rollback (`argocd app rollback`) that re-syncs from a previous commit ArgoCD recorded -- but that is ArgoCD's rollback, not Helm's.
3. **`lookup` returns empty**. The function that reads live cluster state during template rendering -- used in production charts to preserve auto-generated Secrets across upgrades -- always returns an empty `dict` inside ArgoCD because the Repo Server has no cluster credentials. If your chart relies on `lookup` for idempotent Secret generation, it regenerates on every sync.
4. **`.Release.IsUpgrade` is never true**. The Repo Server renders with `helm template`, which always sets `IsInstall=true, IsUpgrade=false`. Any template logic that branches on these values always takes the install branch. Usually harmless; occasionally surprising.
5. **No Helm hooks fire**. `helm.sh/hook: pre-install` annotations are silently ignored by ArgoCD -- the Repo Server renders them as regular manifests but doesn't treat them as hooks. You need to rewrite them as **ArgoCD hooks** (see Part 7).
6. **No `helm test`**. The `templates/tests/` Pods annotated `helm.sh/hook: test` are rendered as regular resources (or as ArgoCD PostSync hooks if you re-annotate them).

### What You Gain

In exchange:

1. **Continuous reconciliation**. A human `kubectl edit`-ing a field owned by the chart gets reverted automatically (with self-heal). `helm upgrade` cannot do this -- it only runs when invoked.
2. **Multi-cluster fan-out via ApplicationSets**. One manifest, N clusters, zero script sprawl.
3. **Drift detection on the UI**. You can *see* what differs between Git and each cluster at a glance.
4. **No long-lived cluster credentials in CI**. The pull model eliminates the entire class of "CI secrets leaked" incidents.
5. **Integration with webhook-driven Git flows**. Commits trigger syncs within seconds, not on the next CI run.

### A Realistic Application with Helm Source

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://github.com/acme/infra.git
    path: charts/payments-api
    targetRevision: main

    helm:
      releaseName: payments-api
      valueFiles:
        - values.yaml                      # Chart defaults
        - values-prod.yaml                 # Prod overrides
      valuesObject:                        # Inline overrides (highest-precedence except `parameters`)
        image:
          tag: "2.8.2"
        autoscaling:
          enabled: true
          maxReplicas: 50
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
      parameters:                          # Per-deploy override from CI (highest precedence)
        - name: image.tag
          value: "2.8.2"

  destination:
    name: prod-us-east-1                   # Cluster registered via `argocd cluster add`
    namespace: payments

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### The Per-Environment Values Pattern

The standard pattern for shipping the same chart to dev/staging/prod:

```
repo/
├── charts/payments-api/               # The chart (committed to Git)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── apps/
│   ├── payments-api-dev.yaml          # Application for dev
│   ├── payments-api-staging.yaml      # Application for staging
│   └── payments-api-prod.yaml         # Application for prod
└── values/
    ├── payments-api/
    │   ├── dev.yaml
    │   ├── staging.yaml
    │   └── prod.yaml
```

Each Application points at the same chart path with a different values file:

```yaml
# apps/payments-api-prod.yaml
spec:
  source:
    repoURL: https://github.com/acme/infra.git
    path: charts/payments-api
    targetRevision: main
    helm:
      valueFiles:
        - values.yaml                    # Chart default
        - ../../values/payments-api/prod.yaml   # Env-specific override (relative to path)
```

Note: `valueFiles` paths can be relative to the chart path (with `..`) -- this is how you reference per-env values that live outside the chart directory. For values files in a *separate* Git repo, you need the multi-source Application feature (covered in tomorrow's Advanced doc).

---

## Part 7: Sync Waves and Hooks -- The Ordering System

Real deployments need ordering. The database schema migration must run before the new app code. The PodSecurityPolicy must exist before the Deployment tries to use it. The smoke test must run *after* the rollout completes. ArgoCD's ordering system is **sync phases + sync waves + hooks** -- three concepts layered together.

### Phases -- The Stage Curtains

Every sync runs in up to four phases, in strict order:

```
PreSync  ->  Sync  ->  PostSync
                  \
                   `-> SyncFail  (only if Sync failed)
```

- **PreSync**: runs hooks annotated `argocd.argoproj.io/hook: PreSync`. Only hooks live here -- no main resources. Classic use: DB migrations.
- **Sync**: the main phase -- all regular resources and any hooks annotated `argocd.argoproj.io/hook: Sync` (unusual).
- **PostSync**: runs hooks annotated `argocd.argoproj.io/hook: PostSync` after the Sync phase completes and all Sync-phase resources are Healthy. Classic use: smoke tests, Slack notifications, cache warming.
- **SyncFail**: runs hooks annotated `argocd.argoproj.io/hook: SyncFail` if the Sync phase fails. Classic use: rollback notifications, cleanup.

Phases are strict -- a phase does not start until the previous phase is completely finished (all resources/hooks in that phase are Healthy).

### Waves -- Ordered Call Sheet Within a Phase

Within a phase, resources are ordered by the `argocd.argoproj.io/sync-wave` annotation (an integer, can be negative). **Lower number = applied first.** Within the same wave number, ArgoCD uses a built-in kind-based order (Namespaces first, then RBAC, then ConfigMaps/Secrets, then workloads -- similar to Helm's install order).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-api-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"     # Default wave if not set
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  annotations:
    argocd.argoproj.io/sync-wave: "1"     # Applied after wave 0 is Healthy
---
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  annotations:
    argocd.argoproj.io/sync-wave: "2"     # Applied after wave 1 is Healthy
```

**The health-gating rule**: ArgoCD applies all resources in wave N, waits for them all to be `Healthy`, then applies wave N+1. A Deployment in wave 1 must reach Ready before the Ingress in wave 2 gets applied.

**Reverse order on deletion**: when an Application is deleted (or a resource is pruned), ArgoCD deletes in **reverse wave order** -- highest first. This matches the classic "tear down services before workloads before ConfigMaps" intuition.

### Hooks -- Guest Contractors

A **hook** is a resource with the `argocd.argoproj.io/hook` annotation that ArgoCD schedules into a specific phase rather than treating as a regular managed resource. Typically a `Job`, but can be any kind.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: payments-api-db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "-1"                          # Can order multiple PreSync hooks too
    argocd.argoproj.io/hook-delete-policy: HookSucceeded,BeforeHookCreation
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: payments-api
      containers:
        - name: migrate
          image: acme/payments-api:2.8.2
          command: ["alembic", "upgrade", "head"]
          envFrom:
            - secretRef:
                name: payments-api-db
```

**Hook delete policies** (comma-separated list):

| Policy | Meaning |
|--------|---------|
| `BeforeHookCreation` | Delete the previous hook before creating a new one with the same name (default if omitted) |
| `HookSucceeded` | Delete the hook resource as soon as it succeeds |
| `HookFailed` | Delete the hook resource as soon as it fails |

The recommended combo for migration Jobs: **`HookSucceeded,BeforeHookCreation`** -- clean up successful runs, keep failures around for debugging, ensure no stale hook resource blocks the next sync.

### ArgoCD Hooks vs Helm Hooks -- The Critical Distinction

You know Helm hooks from two days ago. ArgoCD hooks are superficially similar but operationally different:

| Dimension | Helm hook | ArgoCD hook |
|-----------|-----------|-------------|
| Annotation | `helm.sh/hook: pre-install` | `argocd.argoproj.io/hook: PreSync` |
| Phase names | `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`, `test` | `PreSync`, `Sync`, `PostSync`, `SyncFail` |
| Ordering | `helm.sh/hook-weight` (integer, ascending) | `argocd.argoproj.io/sync-wave` (integer, ascending) |
| Delete policy | `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded` (kebab-case) | `argocd.argoproj.io/hook-delete-policy: HookSucceeded,BeforeHookCreation` (PascalCase) |
| Participates in wave gating? | No -- fires and forgets, next phase proceeds regardless | **Yes** -- hook must reach Healthy (Job complete) before phase advances |
| Participates in health assessment? | No | Yes -- a failed hook marks the Application Degraded |
| Visible in UI? | No (it's just a Helm Secret detail) | Yes -- hooks show up as first-class Application resources |
| Runs on `helm template`? | No (never rendered) | **Yes** -- ArgoCD renders the annotation and schedules the resource into its phase |

**When both annotations are on the same resource** (you ported a Helm chart to ArgoCD without stripping its hook annotations): ArgoCD reads `argocd.argoproj.io/hook` first. If absent, it reads `helm.sh/hook` and applies a translation (e.g., `pre-install` -> `PreSync`). Relying on this translation is fragile; when migrating charts to ArgoCD, re-annotate hooks explicitly with the ArgoCD annotations.

### Phase Boundaries Are Absolute -- Sync Waves Don't Cross Phases

A subtle but load-bearing rule: **sync-wave annotations only order resources within their own phase.** The PreSync phase runs to completion before the Sync phase begins, regardless of wave numbers in the Sync phase. Likewise, the Sync phase completes (including all its waves) before PostSync begins.

This creates a production gotcha that often bites teams who have already internalized sync waves:

> **Scenario:** You have a `DatabaseClaim` CRD in the Sync phase at `sync-wave: "0"` that provisions an RDS instance and populates a Secret. You also have a PreSync hook that runs `alembic upgrade head` for DB migrations, which needs to connect to the database using that Secret.
>
> **What happens:** The PreSync hook runs *before* any Sync-phase resource -- before the DatabaseClaim is even applied, let alone provisioned. The migration Job tries to read a Secret that doesn't exist and fails. No sync-wave number can fix this, because the PreSync phase and Sync phase are separate phases, not ordered waves of a single phase.
>
> **The fix requires rethinking the phase boundary:**
>
> 1. **Move the migration out of PreSync and into the Sync phase** as a Job with a later `sync-wave` than the DatabaseClaim. This forfeits "migrations run before anything else" semantics but works if your Deployment wave is later still.
> 2. **Split into two Applications** with App-of-Apps ordering: the first Application provisions database infrastructure, the second deploys the app (and its PreSync migration hook). The outer App-of-Apps orchestrates.
> 3. **Use an `initContainer`** on the Deployment itself that waits for the Secret to exist, eliminating the need for a PreSync migration at all.

The principle: **PreSync hooks are for work that needs to happen before *any* application resource exists (e.g., validate credentials, fetch a maintenance mode banner).** If your "PreSync" work depends on a Sync-phase resource, it's not PreSync -- it's just "early Sync," and the correct mechanism is a Job in the Sync phase with a low wave number.

### When Do You Need a Hook vs Just a Sync Wave?

This decision matters in practice. The rule:

- **Use a sync wave annotation** on a regular resource when the resource is a **permanent** part of the Application (stays in the cluster after sync). ConfigMap, Deployment, Service, Ingress -- everything the app is made of.
- **Use a hook** when the resource is **transient** -- it exists to run a one-shot task (migration, smoke test, cache warm-up) and then disappears. Typically a `Job`.

```
Permanent resource, needs ordering     -> sync-wave annotation
Transient one-shot task (Job/Pod)      -> hook annotation (PreSync / PostSync)
                                          (and usually also a sync-wave within the phase)
```

**Common confusion**: "can I just put a Job in wave -1 instead of using a PreSync hook?" Technically yes -- if the Job has `sync-wave: -1` and no hook annotation, it gets applied in the Sync phase at wave -1 before other Sync resources. But:

1. The Job becomes a permanent managed resource. Pruning semantics apply. Self-heal may re-create it.
2. Hook delete policies do not apply; you need your own cleanup.
3. In the UI, it looks like application state, not a transient hook.

For migrations specifically, use **PreSync hooks with delete policies**, not sync-wave-annotated Jobs. The hook model is purpose-built for this.

### A Realistic Ordered Deployment

A payments-api chart with proper wave/hook ordering:

```yaml
# Wave -1 (PreSync phase): DB migration (hook)
apiVersion: batch/v1
kind: Job
metadata:
  name: payments-db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook-delete-policy: HookSucceeded,BeforeHookCreation
spec: {...}
---
# Wave 0 (Sync phase): config and secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-api-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---
# Wave 1 (Sync phase): workload -- waits for wave 0 to be Healthy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  annotations:
    argocd.argoproj.io/sync-wave: "1"
---
# Wave 2 (Sync phase): service -- waits for Deployment to be Healthy
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  annotations:
    argocd.argoproj.io/sync-wave: "2"
---
# Wave 3 (Sync phase): ingress -- waits for service to be Healthy
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payments-api
  annotations:
    argocd.argoproj.io/sync-wave: "3"
---
# PostSync phase: smoke test (hook)
apiVersion: batch/v1
kind: Job
metadata:
  name: payments-smoketest
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec: {...}
```

Sync order: db-migrate -> ConfigMap -> Deployment (waits for Ready) -> Service (waits for endpoints) -> Ingress (waits for LB) -> smoketest. A failure at any stage pauses the rest.

Delete order reverses: smoketest is already gone (HookSucceeded), then Ingress, Service, Deployment, ConfigMap. The PreSync migration hook is already gone too.

### Hook Delete Policies Explained in Detail

Hooks, unlike regular Sync-phase resources, are **not** part of the Application's managed resource set -- they don't show up in the "resources owned by this Application" diff. They are one-shot Jobs/Pods that ArgoCD schedules into a phase, runs to completion, and then leaves in the namespace subject to delete-policy rules.

**Important correction to a common misconception:** ArgoCD's implicit default when no `hook-delete-policy` annotation is set is **`BeforeHookCreation`**, not "no cleanup at all." The previous hook is deleted at the start of the next sync that creates a hook with the same name. The *most recent* hook persists in the namespace until superseded. This is the same default Helm uses for its hooks, so the behavior is familiar.

**`BeforeHookCreation`** (implicit default): before creating a new hook resource with the same name, delete the previous one. Keeps the namespace from accumulating stale hooks as syncs progress. Leaves the most recent hook in place after it completes so you can inspect `kubectl logs`.

**`HookSucceeded`**: delete the hook resource immediately after it reaches Healthy/Succeeded. Cleanest option; removes all trace of a successful hook right away without waiting for the next sync.

**`HookFailed`**: delete the hook resource immediately after it fails. Usually NOT what you want -- failed hooks are what you need to `kubectl logs` for debugging. Only reach for this when your monitoring already ingested the failure event and the hook resource itself is noise.

The production combo: **`HookSucceeded,BeforeHookCreation`** -- successful hooks vanish immediately (your namespace stays clean), failed hooks stick around for debugging (no `HookFailed`), and on the next sync any stale failure gets overwritten by `BeforeHookCreation`. This explicit combo is recommended because it is self-documenting: a reader of the YAML knows exactly when the hook gets cleaned up without having to remember the implicit default.

---

## Part 8: Health Checks -- The Wave-Gating Mechanism

Sync waves only work because ArgoCD can determine when a resource is "Healthy." Without health assessment, waves would be "apply and move on immediately," and the whole ordering system collapses.

### Built-in Health Checks

ArgoCD ships with built-in health assessments for standard Kubernetes resources:

| Kind | Healthy when... |
|------|-----------------|
| `Deployment` | `.status.updatedReplicas == .spec.replicas` and all Pods Ready |
| `StatefulSet` | All replicas updated, ready, and matching desired |
| `DaemonSet` | All desired Pods scheduled and ready on all nodes |
| `Service` (ClusterIP) | Always Healthy once created |
| `Service` (LoadBalancer) | Healthy once `.status.loadBalancer.ingress` is populated |
| `Ingress` | Healthy once `.status.loadBalancer.ingress` is populated |
| `PersistentVolumeClaim` | `.status.phase == Bound` |
| `Pod` | Phase `Running` with all containers Ready, or `Succeeded` for run-to-completion |
| `Job` | `.status.succeeded >= .spec.completions` (Healthy), or `.status.failed > backoffLimit` (Degraded) |
| `HorizontalPodAutoscaler` | AbleToScale condition true, ScalingActive condition true |
| Custom Resource with no health check defined | `Unknown` |

The full list is in ArgoCD's source code at `argoproj/gitops-engine/pkg/health/`. The built-ins cover ~95% of production resources out of the box.

### Custom Health Checks (Lua)

For CRDs or custom behaviors, you write Lua scripts. They live in the `argocd-cm` ConfigMap under `data.resource.customizations.health.<group>_<kind>` -- ArgoCD loads them at controller startup and when the ConfigMap is updated. (Note: `ignoreDifferences` on the Application spec is for **diff exclusion**, not health checks -- don't confuse the two.)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
data:
  # Key format: resource.customizations.health.<group>_<kind>
  resource.customizations.health.cert-manager.io_Certificate: |
    hs = {}
    if obj.status ~= nil and obj.status.conditions ~= nil then
      for i, condition in ipairs(obj.status.conditions) do
        if condition.type == "Ready" and condition.status == "True" then
          hs.status = "Healthy"
          hs.message = condition.message
          return hs
        end
        if condition.type == "Ready" and condition.status == "False" then
          hs.status = "Degraded"
          hs.message = condition.message
          return hs
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for certificate to be issued"
    return hs
```

Script contract:

- Receives the resource as a global variable `obj` (the parsed YAML).
- Returns a table with **two keys**: `status` (string) and `message` (string).
- Valid status values: `"Healthy"`, `"Progressing"`, `"Degraded"`, `"Suspended"`, `"Missing"`, `"Unknown"`.

The key naming convention: `resource.customizations.health.<apiGroup>_<Kind>`. For `core/v1` resources, use `""` for the group (but you almost never need to override built-ins). For namespaced CRDs, use the full CRD group (`cert-manager.io`, `monitoring.coreos.com`, etc.).

### Why Health Matters for Sync Waves

Health is the **gating signal** that tells ArgoCD when to move from wave N to wave N+1. A resource with `Unknown` health (no check defined) is effectively treated as "Healthy immediately" -- which means it does **not** block wave progression. This is one of the subtle gotchas:

> If your CRD has no custom health check and the Application Controller cannot assess it, sync waves move past it as if it were Healthy. A Deployment in wave 1 that depends on a CRD in wave 0 being ready may start before the CRD controller has actually provisioned the backing resources.

**If you add a CRD to wave 0 and a Deployment to wave 1 that depends on the CRD's reconciliation having finished**, you need a custom health check for the CRD that returns `Progressing` until the CRD controller reports ready. Otherwise, the wave gate is cosmetic.

### Health Check for a Custom Resource -- Full Example

Scenario: you are using a custom `DatabaseClaim` CRD that provisions an RDS instance and exposes a Secret with credentials. Your Deployment in wave 1 mounts the Secret; your DatabaseClaim in wave 0 provisions it. You need wave 0 to gate on the provisioning completing.

```yaml
# argocd-cm ConfigMap, in argocd namespace
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.platform.acme.io_DatabaseClaim: |
    hs = {}
    if obj.status == nil then
      hs.status = "Progressing"
      hs.message = "DatabaseClaim has no status yet"
      return hs
    end

    if obj.status.phase == "Ready" and obj.status.secretName ~= nil then
      hs.status = "Healthy"
      hs.message = "Database ready, Secret " .. obj.status.secretName .. " populated"
      return hs
    end

    if obj.status.phase == "Failed" then
      hs.status = "Degraded"
      hs.message = obj.status.message or "DatabaseClaim failed"
      return hs
    end

    hs.status = "Progressing"
    hs.message = "DatabaseClaim phase: " .. (obj.status.phase or "unknown")
    return hs
```

Now the sync wave works: wave 0's DatabaseClaim reports `Progressing` until the operator provisions RDS, then flips to `Healthy`, and only then does wave 1 start applying the Deployment.

### The Application's Aggregate Health

The Application itself has a health status aggregated from its resources:

- **Healthy**: every managed resource is Healthy.
- **Progressing**: at least one resource is Progressing (rollout in flight).
- **Degraded**: at least one resource is Degraded.
- **Missing**: at least one resource is Missing (post-initial-sync, this is usually a prune-pending state).
- **Unknown**: the controller cannot determine (rare; usually during initial refresh).

The aggregate rule is "worst status wins" -- one Degraded resource = Degraded Application.

---

## Part 9: Resource Tracking -- Annotation vs Label Mode

One last piece of plumbing: how does ArgoCD know which resources an Application owns? Two modes:

### Annotation Mode (default, recommended)

ArgoCD stamps `argocd.argoproj.io/tracking-id` on every managed resource:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/tracking-id: payments-api-prod:apps/Deployment:payments/payments-api
```

Format: `<app-name>:<group>/<kind>:<namespace>/<name>`. Includes the full kind and namespace, which means ArgoCD can distinguish a Deployment named `foo` from a StatefulSet named `foo`. Annotations do not participate in label selectors, so they never accidentally match `kubectl get all -l app=foo`.

### Label Mode (legacy)

The old mode used `app.kubernetes.io/instance: payments-api-prod` as a **label**. This is the same label key Helm uses by convention for selectors, which is the whole problem: if your Helm chart sets its own `app.kubernetes.io/instance` value in selectors, ArgoCD's label overwrites it and breaks the chart's own selector logic. Label mode also cannot distinguish between kinds -- two Applications with the same resource name in different kinds collide.

**Always use annotation mode in new installations.** It is the default in every ArgoCD installation newer than 2.5 (late 2022). If you inherit an installation on label mode (`application.instanceLabelKey: app.kubernetes.io/instance` in `argocd-cm`), migrate to annotation mode with a careful plan -- the migration touches every resource ArgoCD manages.

Set via the `argocd-cm` ConfigMap:

```yaml
data:
  application.instanceLabelKey: argocd.argoproj.io/tracking-id   # annotation mode
  # or
  application.instanceLabelKey: app.kubernetes.io/instance       # label mode (legacy)
```

---

## Part 10: Production Gotchas and Interview Traps

**1. `lookup` always returns empty inside ArgoCD.**
This is the biggest mental-model shift for Helm natives. Charts that use `lookup` to preserve auto-generated Secrets across upgrades regenerate the Secret on every sync when deployed via ArgoCD. Either refactor to use a persistent Secret (external-secrets operator, AWS Secrets Manager), or move Secret generation out of the chart entirely.

**2. No Helm release Secrets = no `helm history` or `helm rollback`.**
Tooling that inspects `sh.helm.release.v1.*` Secrets in the release namespace returns nothing when the chart is deployed via ArgoCD. Observability dashboards that count "Helm releases per namespace" read zero. Rollback is `git revert` + sync, not `helm rollback`.

**3. Self-heal timer is controller-level configurable; self-heal itself cannot be disabled per-sync.**
Default ~5 seconds, but tune via `--self-heal-timeout-seconds` on `argocd-application-controller` or `controller.self.heal.timeout.seconds` in `argocd-cmd-params-cm` (not `argocd-cm` -- controller flags live in `argocd-cmd-params-cm`). Recent versions add exponential backoff via `--self-heal-backoff-factor` to avoid tight tug-of-war loops. Self-heal itself is an Application-level toggle -- there is no per-resource disable. If you need "self-heal for everything except this Deployment's `replicas`," use `spec.ignoreDifferences` on the conflicted field (canonical HPA coexistence pattern), not a per-sync bypass.

**4. Sync waves gate on *health*, not *presence*. A missing health check means waves don't wait.**
If a wave-0 CRD has no custom health check defined, ArgoCD treats its "Unknown" health as non-blocking and proceeds to wave 1 immediately. Always pair ordered CRDs with custom Lua health checks. **Related trap:** sync-wave annotations only order resources within their own phase. A PreSync hook that depends on a Sync-phase resource (e.g., a migration Job that reads a Secret populated by a Sync-phase `DatabaseClaim` CRD) will fail regardless of wave numbers, because the entire PreSync phase runs before any Sync-phase resource applies. Fix: move the migration into the Sync phase at a later wave, or split into two Applications with App-of-Apps ordering.

**5. Prune can wipe things if Git becomes empty -- use `allowEmpty: false`.**
Default is `false`, which is safe. If someone flips it to `true` "for flexibility" and a CI step commits an empty manifest set (bad templating, wrong branch merged), the next sync deletes every managed resource in the target namespace. Lock `allowEmpty: false` in production and require a PR review to change it.

**6. `ServerSideApply=true` is effectively required for large ConfigMaps and CRD-heavy charts.**
The `last-applied-configuration` annotation has a 256KB hard limit. Blow past it with a big Prometheus rules ConfigMap or a bulky CRD schema and the sync fails with `metadata.annotations: Too long: must have at most 262144 bytes`. Fix: add `ServerSideApply=true` per-resource or at the Application level.

**7. ArgoCD hooks are NOT part of the managed resource set -- the most recent hook persists until the next sync.**
Common misconception: "hooks leak forever without a delete-policy." Reality: when no `hook-delete-policy` is set, ArgoCD implicitly uses `BeforeHookCreation`, meaning the previous hook is replaced at the start of the next sync that creates a hook with the same name. The *most recent* hook lingers in the namespace until then. This is the same default Helm uses. For cleaner behavior, set **`hook-delete-policy: HookSucceeded,BeforeHookCreation`** explicitly: successful hooks disappear immediately, failed hooks stick around for debugging, and staleness is bounded. The explicit form is recommended for readability -- YAML readers shouldn't have to remember the implicit default.

**8. `targetRevision: HEAD` or a branch name is mutable -- the deployed commit floats.**
`targetRevision: main` means "whatever main points to right now." Every commit to main triggers a sync. For environments where you want reproducible deploys with explicit promotion, use tags (`v1.4.2`) or SHAs, and have a separate promotion process that updates the Application manifest to the new ref. Classic foot-gun: deploying prod from `main` and realizing prod drifted an hour after someone merged an experimental PR.

**9. ArgoCD hooks fire in phase order AND wave order within a phase.**
Multiple PreSync hooks with different `sync-wave` annotations sort within the PreSync phase by wave. A PreSync wave -2 hook runs before a PreSync wave -1 hook, both before any Sync phase resource. You can have the full ordering machinery for pre-migrations without touching the main Sync phase.

**10. Helm hook annotations inside an ArgoCD-managed chart may silently break.**
A chart with `helm.sh/hook: pre-install` on a Job: ArgoCD translates the annotation to its nearest equivalent (`PreSync`) in most cases, but the translation is incomplete -- e.g., `helm.sh/hook: test` has no exact ArgoCD equivalent (the ArgoCD equivalent is PostSync + a different annotation). When migrating a Helm chart to ArgoCD, grep for `helm.sh/hook` and rewrite every annotation to the ArgoCD equivalent.

**11. Resource tracking -- annotation vs label mode.**
Inherited ArgoCD installations may still be on label mode, where `app.kubernetes.io/instance: <app-name>` is the tracking key. This collides with any chart that sets its own `instance` label for Deployment selectors. New installations default to annotation mode. Check `argocd-cm` if you see weird "ArgoCD doesn't recognize its own resource" bugs.

**12. The Application Controller needs cluster-admin (or close) on every managed cluster.**
To apply arbitrary manifests (including ClusterRoles, CRDs, ValidatingWebhookConfigurations), the Application Controller's ServiceAccount needs cluster-admin equivalent on every target cluster. In single-cluster installs, this is local. In multi-cluster installs, it means the credentials stored in `cluster` Secrets in the `argocd` namespace are extremely sensitive. RBAC Projects scope *Applications*, not the controller itself.

**13. Pruning in the wrong order breaks things -- use `PruneLast=true` for finalizer races.**
Default prune order is reverse wave order. But if a Namespace is deleted before its PVCs finish their finalizers, the PVCs hang, the Namespace stays in `Terminating` forever, and the Application is stuck. `PruneLast=true` (Application-level) defers all pruning until after all other sync operations complete, avoiding some race conditions.

**14. Webhooks need the right secret or ArgoCD ignores them.**
Git provider webhooks (GitHub, GitLab, Bitbucket) require a shared secret for signature verification. If the secret in `argocd-secret` doesn't match what's configured in the Git webhook, ArgoCD silently ignores the webhook -- you don't get an error, you just get "syncs only happen on the 3-minute reconcile loop instead of instantly." Easy to miss for a week.

**15. `argocd app sync --force` is NOT a rollback -- it just re-applies.**
The `--force` flag makes `argocd app sync` use `kubectl replace` semantics instead of `kubectl apply`, useful for resources with field-ownership issues. It does NOT roll back to an earlier commit. For rollback, use `argocd app rollback <app> <revision>` (ArgoCD-recorded history) or `git revert` + sync (the canonical Git rollback).

**16. The Application CRD lives in the ArgoCD namespace, but the managed resources don't.**
Applications are created in `argocd` (or wherever ArgoCD is installed). The `destination.namespace` is where managed resources go. These are different namespaces and different RBAC scopes. A user with write on `argocd` namespace can create Applications that deploy to any namespace (bounded by AppProject), which is why Project RBAC matters.

**17. `finalizers: [resources-finalizer.argocd.argoproj.io]` is what makes delete cascade.**
Without this finalizer on the Application, `kubectl delete application payments-api` deletes the Application CRD but leaves all managed resources in the cluster. With the finalizer, deletion cascades: ArgoCD walks the managed resource list and deletes each one in reverse wave order. Forget this and you get "zombie resources" after deleting an Application. Always include the finalizer in production Applications.

**18. Helm chart values changes don't automatically trigger a sync if only the ConfigMap or Deployment content changes at render time.**
ArgoCD's diff is on rendered manifests vs live state. A commit that only changes `values-prod.yaml` produces a new rendered Deployment, which ArgoCD sees as OutOfSync and syncs (good). But if your chart uses `checksum/config: {{ include "..." . | sha256sum }}` to force pod rolls on ConfigMap change, that pattern still works -- the Deployment pod spec changes, ArgoCD syncs the Deployment, Pods roll. The pattern is preserved; it just works via ArgoCD's diff rather than via `helm upgrade`.

**19. `ignoreDifferences` alone is cosmetic -- it needs `RespectIgnoreDifferences=true` to actually work.**
Many tutorials and blog posts show `ignoreDifferences` for HPA coexistence but omit the matching sync option, leading to a subtle failure mode: the UI shows Synced (because the diff ignores the field), but any sync triggered by another field still rewrites the ignored field. HPA scales from 3 to 5, no drift alert fires, but the next Git commit to any other field triggers a sync and `.spec.replicas` gets reset to 3 anyway. The fix is always the pair: `ignoreDifferences` on `Application.spec` **and** `RespectIgnoreDifferences=true` in `syncPolicy.syncOptions`. Without both, you've only fixed the UI.

**20. "Auto-sync with self-heal OFF" in dev is a silent betrayal, not a feature.**
Tempting dev-env pattern: "turn self-heal off so engineers can `kubectl edit` during debugging." Failure mode: engineer's edits stick *until any teammate merges to the dev branch*, at which point auto-sync fires and silently reverts the engineer's work without warning. The non-determinism is the poison -- engineers don't learn "don't `kubectl edit`," they learn "ArgoCD eats my changes unpredictably" and stop trusting the tool. The correct pattern: full GitOps in dev too, with PR-preview namespaces via ApplicationSet Pull Request generator for per-engineer iteration. If you truly need ad-hoc experimentation, carve out a *separate* namespace that ArgoCD does not manage.

**21. Never use the default AppProject. Scope every Application to a Project with a source-repo allowlist and destination-namespace allowlist.**
The default Project allows any Application to deploy anything from any source repo to any namespace, including cluster-scoped resources like `ClusterRoleBinding` and CRDs -- effectively cluster-admin blast radius. Every "default-project" Application is one compromised PR or typo away from "someone just granted themselves cluster-admin by committing a ClusterRoleBinding." The fix is a scoped `AppProject` per team/app with `sourceRepos`, `destinations`, `clusterResourceWhitelist: []` (deny all cluster-scoped by default), and `namespaceResourceWhitelist` restricted to what the app actually needs. This is configuration that should be **the same across dev, staging, and production** -- blast-radius containment is not environment-specific. Your `syncPolicy` can be flawless and your branch protection locked down, and a default-project Application will still let one compromised PR escalate to cluster-admin.

---

## When-to-Use-What -- Decision Frameworks

### Auto-Sync vs Manual Sync

```
Env type                           Recommendation
---------                          --------------
Dev / PR preview / ephemeral      auto-sync + prune + self-heal
Staging / integration              auto-sync + prune + self-heal
Production (standard)              auto-sync + prune + self-heal (+ require PR review on Git)
Production (highly regulated)      manual sync (PR is approval, Sync button is change-ticket execution)
Disaster recovery cluster          manual sync (stays dormant until failover)
Shared infra (CNI, ingress)        manual sync + strict AppProject + two-person Sync
```

### ArgoCD Hooks vs Sync Waves

```
Transient one-shot task (Job)              -> hook (PreSync / PostSync / SyncFail)
Permanent resource needing ordering        -> sync-wave annotation
Ordering multiple transient tasks          -> hooks with sync-wave annotation WITHIN phase
Replacing a Helm chart's helm.sh/hook      -> rewrite as ArgoCD hook, don't rely on translation
```

### ArgoCD Hooks vs Helm Hooks (when both could apply)

```
Chart is deployed ONLY via ArgoCD (pure GitOps shop)
  -> use ArgoCD hooks exclusively; strip helm.sh/hook annotations

Chart must work both standalone (helm install) AND via ArgoCD
  -> keep helm.sh/hook annotations for standalone users
  -> add argocd.argoproj.io/hook annotations for ArgoCD users (both can coexist on same resource)
  -> document that the chart supports both modes

Chart is ArgoCD-only but you forgot to migrate hooks from Helm
  -> ArgoCD translates most helm.sh/hook values, but the translation has edge cases
  -> migrate explicitly; don't rely on implicit translation
```

### When to Use Helm Inside ArgoCD vs Pure Kustomize

```
Chart is a third-party distributable (Prometheus stack, cert-manager, Bitnami)
  -> Helm source in the Application; use helm.valueFiles for env overrides

Chart is your own, heavily parameterized, shared across many apps
  -> Helm source; treat chart as a library

YAML is static, varies only slightly across envs (dev/staging/prod)
  -> Kustomize source; base + overlays

Need to patch a third-party chart you can't fork
  -> Helm source + ArgoCD's Kustomize post-rendering support (multi-source Applications)
```

### Terraform vs ArgoCD for Platform Components

```
Component is tied to the cluster lifecycle (VPC CNI, AWS Load Balancer Controller)
  -> Terraform (it exists because the cluster exists; co-managed with EKS)

Component is a workload with its own release cadence (Prometheus stack, ArgoCD itself)
  -> GitOps: initial bootstrap via Terraform, ongoing upgrades via a bootstrap Application
     (the "App of Apps" pattern, covered tomorrow)

Component has ENV-specific configuration that changes on its own cadence
  -> ArgoCD Application with Helm source, values files per env

Component needs to exist before the cluster is fully usable
  -> Terraform (chicken-and-egg otherwise)
```

---

## Key Takeaways

- **GitOps is four principles, not a tool.** Declarative, Versioned and Immutable, Pulled Automatically, Continuously Reconciled. ArgoCD implements them; Flux implements them; a sufficiently disciplined pipeline could implement them. Missing any principle means you are doing CI/CD that happens to use Git, not GitOps.

- **Pull beats push for CD.** The cluster pulls desired state from Git; CI never holds cluster credentials; the reconciliation continues through CI outages; drift is detected on every loop. Push-based CD (Jenkins running `kubectl apply`) violates Principle 3 and misses all the safety that pull provides.

- **ArgoCD is seven components, each with one focused job.** API Server (radio to HQ), Repo Server (photocopy room -- runs `helm template`), Application Controller (foreman -- reconciliation loop), Redis (clipboard), Dex (HR / SSO), ApplicationSet Controller (template machine), Notification Controller (walkie-talkie). The Repo Server never touches the cluster -- that's the security boundary.

- **The Application CRD is the central abstraction.** `source` (where the blueprint lives) + `destination` (where to build) + `project` (what the portfolio allows) + `syncPolicy` (how aggressively to enforce). Everything else -- the UI, the CLI, the notifications -- is a view of Application CRDs.

- **Sync status and health status are orthogonal.** Sync = "does cluster match Git?". Health = "are the things running correctly?". `Synced + Degraded` is "Git is right but the app is broken." `OutOfSync + Healthy` is "a new commit is waiting to deploy, current state is fine." Read both axes.

- **Auto-sync is three toggles plus a safety and a surgical escape hatch.** `automated` enables the loop. `prune` deletes resources removed from Git. `selfHeal` reverts out-of-band cluster edits (default ~5-second timer, tunable via the Application Controller). `allowEmpty: false` is the safety that keeps an empty Git repo from wiping the cluster. `spec.ignoreDifferences` is the surgical escape hatch for fields legitimately owned by other controllers (HPA replicas, mutating webhook sidecars, service-mesh annotations).

- **ArgoCD runs `helm template`, not `helm install`.** No release Secrets. No `helm history`. No `helm rollback`. `lookup` returns empty. `.Release.IsUpgrade` is always false. In exchange you get continuous reconciliation, drift detection, and multi-cluster fan-out. Internalize the concept map and you stop expecting Helm CLI features that don't exist.

- **`valuesObject` beats `values`.** Use the structured YAML object form for inline values in the Application CRD. Values precedence (lowest to highest): chart defaults -> valueFiles -> values -> valuesObject -> parameters -> fileParameters. `parameters` always wins.

- **Sync waves + phases + hooks are the ordering system.** Three phases (PreSync, Sync, PostSync) + integer waves within each phase + hooks scheduled into phases. Lower wave first; wave N+1 waits for wave N to be Healthy. Deletion reverses wave order. Hooks are transient Jobs/Pods with delete policies; `HookSucceeded,BeforeHookCreation` is the production default.

- **ArgoCD hooks differ from Helm hooks in one critical way: they participate in health-gating.** A Helm hook fires and forgets. An ArgoCD hook must reach Healthy (Job complete) before the phase advances. This is what makes `PreSync` actually gate the Sync phase.

- **Health is the gating signal for waves.** Built-in checks cover Deployment/Service/Ingress/PVC/Job etc. Custom Lua checks for CRDs. A CRD with no health check has `Unknown` health, which does NOT block waves. Always pair ordered CRDs with custom Lua health checks.

- **Always set `finalizers: [resources-finalizer.argocd.argoproj.io]` on production Applications.** Without it, deleting the Application leaves managed resources behind as zombies in the cluster.

- **Resource tracking = annotation mode.** `argocd.argoproj.io/tracking-id` is the default and the right choice. Label mode (`app.kubernetes.io/instance`) is legacy and collides with Helm chart selectors. Check `argocd-cm` if you inherit an installation.

- **The cross-cutting mental flip from Helm-native to ArgoCD-native.** You no longer run `helm install`. You commit YAML to Git. Everything else -- rollback, inspection, drift correction, multi-env promotion -- is Git + ArgoCD, not the Helm CLI. The Helm CLI survives for chart authoring, rendering, testing, and packaging; the deploy surface moves to Git.

---

## Study Resources

Curated reading list for this topic: [`study-resources/kubernetes/argocd-gitops-fundamentals.md`](../../../study-resources/kubernetes/argocd-gitops-fundamentals.md)

**Key references**:
- [OpenGitOps Principles](https://opengitops.dev/) -- The canonical four-principle CNCF standard
- [The New Stack: 4 Core Principles of GitOps](https://thenewstack.io/4-core-principles-of-gitops/) -- The "why" behind each principle with practitioner quotes
- [ArgoCD Core Concepts](https://argo-cd.readthedocs.io/en/stable/core_concepts/) -- Canonical definitions of Application, Project, sync status, health status
- [ArgoCD Understand the Basics](https://argo-cd.readthedocs.io/en/stable/understand_the_basics/) -- Target vs live state, refresh vs sync vocabulary
- [ArgoCD Architecture -- DevOpsCube](https://devopscube.com/argo-cd-architecture/) -- The best single-article explanation of the seven components
- [ArgoCD Helm Integration](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/) -- The critical bridge for Helm natives: `helm template` not `helm install`, valuesObject, parameters, fileParameters
- [ArgoCD Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/) -- `automated`, `prune`, `selfHeal`, `allowEmpty`
- [ArgoCD Sync Options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/) -- Per-resource annotations (`Replace`, `ServerSideApply`, `CreateNamespace`, `PruneLast`)
- [Codefresh: ArgoCD Sync Policies Practical Guide](https://codefresh.io/learn/argo-cd/argocd-sync-policies-a-practical-guide/) -- Concrete YAML and CLI examples
- [ArgoCD Sync Phases and Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) -- Official reference for phases, waves, hooks
- [Codefresh: Understanding ArgoCD Sync Waves](https://codefresh.io/learn/argo-cd/understanding-argo-cd-sync-waves-with-examples/) -- Practical wave examples with DB migration and Slack hook
- [ArgoCD Resource Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/) -- Built-in checks table and custom Lua scripts
- [ArgoCD Resource Tracking](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_tracking/) -- Annotation vs label mode migration

**Cross-references in this repo**:
- [Helm Fundamentals (Apr 13, 2026)](./2026-04-13-helm-fundamentals.md) -- Chart anatomy, template engine, values precedence, install/upgrade/rollback lifecycle; the baseline mental model you plug into ArgoCD
- [Helm Advanced Patterns (Apr 15, 2026)](./2026-04-15-helm-advanced-patterns.md) -- Hooks with weights and delete policies, library charts, OCI registries, Helmfile; the parallel mental model for hook ordering that maps to ArgoCD's wave+phase system
- [EKS Deep Dives -- December 2025](../../../2025/december/eks/) -- The cluster ArgoCD runs in and reconciles into; nine dives covering VPC CNI, IAM roles for service accounts, autoscaling, security, observability, upgrades
- [Terraform Project Structure](../../../2026/january/terraform/2026-02-02-terraform-project-structure.md) -- Parallel mental model: Terraform workspaces per env mirror ArgoCD Applications per env; both express "same spec, different parameters, different destinations"
