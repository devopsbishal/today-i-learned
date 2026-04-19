# ArgoCD Advanced -- Multi-App, Multi-Cluster, Multi-Tenant

> Yesterday you internalized the **site-foreman** mental model: one Git repo, one Application CRD, one foreman on one construction site, reconciling continuously. That model gets you through the first cluster and the first dozen Applications. It falls apart the moment the firm expands. Now there are **four regions**, **twelve clusters**, **six product teams** sharing the hub, a **platform team** who needs to bootstrap 40 add-ons into every new cluster the day it's provisioned, a **security team** who wants each team's blast radius strictly fenced, a **CI team** who needs a throwaway preview environment for every pull request, and a **compliance team** who wants Slack alerts on every failed sync in production. You cannot hand-write 500 Application manifests. You cannot give six teams cluster-admin and hope for the best. You cannot paste plaintext Postgres passwords into Git no matter how "private" the repo is.
>
> Today's material is how ArgoCD scales from one site to a construction empire. The analogy extends naturally: the single-foreman model becomes a **franchise with a home office and regional operations**. **App of Apps** is the franchise parent company -- one master work order that spawns many child work orders at once, which is how you bootstrap an entire cluster's worth of add-ons from a single `kubectl apply`. **ApplicationSets** are the **mail-merge machine** sitting on top -- one template, one spreadsheet of parameters (the generator), and the controller stamps out N Applications with the template filled in per row. **AppProjects** are **corporate departments** inside the firm -- each department has its own vendor list (allowed Git repos), its own approved job sites (allowed destinations), its own budget (RBAC roles), and its own org chart (OIDC group-to-role mapping). **Sharding the Application Controller** is **splitting a library's card catalog across multiple librarians** -- each owns a slice of the alphabet, and the trick is choosing an algorithm that doesn't force every librarian to rebuild their slice from scratch whenever a new colleague joins. **Secrets management** is the art of putting **locked diplomatic pouches inside the plain manila folder** that rides in the Git repo -- Sealed Secrets, External Secrets Operator, and SOPS are three different pouch designs for three different trust models. **Notifications** are the **foreman's dispatch radio** -- sync failures broadcast to #alerts, prod health-degraded pages the on-call via PagerDuty, routine deploys trickle into #deploys, each message templated per channel so the listener gets the right level of detail.
>
> This doc walks through all of it, in order: the App of Apps bootstrapping pattern and where it breaks down, the seven ApplicationSet generators with runnable YAML for each, Go-template rendering and the hotfix escape hatches (`ignoreApplicationDifferences`, preserve-and-rebuild, generator-routing), progressive rollouts with `RollingSync`, multi-cluster registration and hub-and-spoke topology, Application Controller sharding, AppProjects as tenant boundaries with RBAC and JWT tokens, the three secret-management strategies and when to reach for each, the notifications controller triggers-templates-services-subscriptions model, and advanced resource-hook patterns layered into ApplicationSet-templated apps. It assumes you already hold yesterday's fundamentals -- Application CRD, sync policies, sync status vs health, waves/hooks, and the `helm template` bridge. By the end you should be able to architect the full GitOps control plane for a multi-team, multi-cluster platform and defend your choices for every moving piece.

---

**Date**: 2026-04-17
**Topic Area**: kubernetes
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| App of Apps | Franchise parent company with regional franchisees | One root Application points at a directory of child Application manifests; `kubectl apply` the root, everything else cascades |
| Root Application's directory | The franchise HQ's "all locations" binder | A Git folder containing N child Application YAMLs; the root uses `source.directory.recurse: true` to pick them all up |
| Cascading delete | Close the parent, close every franchisee | `finalizers: [resources-finalizer.argocd.argoproj.io]` on the root Application so `kubectl delete app root` cleans up every child |
| Nested App of Apps | HQ -> regional office -> individual franchisees | A root App points at child Apps that are themselves App-of-Apps; organizational boundary between platform team and product teams |
| ApplicationSet | Mail-merge template + a spreadsheet | One `ApplicationSet` CR with `template:` (Application skeleton) + `generators:` (rows); the controller stamps out one Application per row |
| List generator | A hand-maintained guest list | Static list of parameter objects; the simplest generator |
| Cluster generator | Mail-merge spreadsheet whose rows are the cluster Secret registry | One Application per registered cluster (one row); columns include `name`, `server`, labels/annotations from the Secret |
| Git generator (directories) | "One work order per subfolder of the monorepo" | One Application per subdirectory matching a glob (`apps/*`) |
| Git generator (files) | "One work order per config sheet on my desk" | One Application per YAML/JSON file matching a glob; the file's top-level keys become template parameters |
| Matrix generator | SQL cross join of two generators | Cartesian product: every combination of (cluster x git-directory) becomes an Application |
| Merge generator | SQL LEFT JOIN of generators on a key | Combine generators by `mergeKeys`; second generator overrides fields from the first |
| SCM Provider generator | "Onboard every repo in our GitHub org automatically" | Discovers repos in GitHub/GitLab/Bitbucket orgs by filter; one Application per discovered repo |
| Pull Request generator | Ephemeral preview environments | One Application per open PR; auto-deleted on merge/close |
| Template | The Application skeleton that gets filled in | `spec.template.spec` -- identical to a regular Application spec with `{{ .param }}` placeholders |
| `templatePatch` | Per-element amendment to the boilerplate | String-valued YAML patch applied after template render for advanced per-element overrides |
| `goTemplate: true` | Upgrade from Mad Libs to a full templating engine | Switches from legacy fasttemplate (`{{ .param }}` only) to real Go templates with sprig, conditionals, range |
| `goTemplateOptions: ["missingkey=error"]` | "Fail loud on typos instead of silently writing empty string" | Undefined variables raise a render error instead of rendering as empty |
| `preservedFields` | "Don't overwrite annotations set by other controllers" | Metadata-only preservation: `annotations` and `labels` on generated Applications are left alone even when template disagrees. Spec-field preservation is NOT supported |
| `ignoreApplicationDifferences` | "Ignore specific JSON paths on generated Applications" | Controller-level ignore rules on specific paths in generated Applications (newer ApplicationSet feature); the real escape hatch for spec-level on-call overrides |
| `RollingSync` strategy | Deploy ring-by-ring instead of all-at-once | Progressive rollout of an ApplicationSet across labeled waves of Applications |
| Cluster Secret | The spoke's kubeconfig locked in HQ's safe | A Secret in the `argocd` namespace labeled `argocd.argoproj.io/secret-type: cluster`, containing the target cluster's API URL, CA, and auth |
| `in-cluster` | The foreman's own office | Shortcut pointing at `https://kubernetes.default.svc`; the cluster ArgoCD itself runs in |
| Hub and spoke | One HQ overseeing many branch offices | One ArgoCD install (hub) reconciles workloads into N other clusters (spokes), each registered via a Cluster Secret |
| Standalone per cluster | Every office runs its own independent ops team | One ArgoCD install per cluster; simpler blast radius, no cross-cluster visibility |
| Application Controller sharding | Splitting a library's card catalog across multiple librarians | Multiple controller replicas, each owning a subset of clusters (slices of the catalog); tunable via `controller.sharding.algorithm` |
| `legacy` sharding | Reshuffle every librarian's slice whenever a new librarian joins | `hash(cluster UID) mod replicas`; reshuffles on replica count change -- caches rebuild from scratch |
| `round-robin` sharding | Deal clusters out in order at startup, but reshuffle most of them on scale | Distributes clusters evenly in registration order; also reshuffles on scale, only predictable for small static setups |
| `consistent-hashing` | Move only ~1/N of the cards to the new librarian; existing ones keep their assignments | Consistent hash ring with bounded loads; minimizes cache rebuilds when replicas scale. The only algorithm safe for production |
| AppProject | A corporate department with its own budget and vendor list | The multi-tenant boundary: `sourceRepos`, `destinations`, allowed resource kinds, RBAC roles, JWT tokens |
| Casbin policy.csv | The corporate rulebook in spreadsheet form | `p, subject, resource, action, object, effect` lines; `g,` lines map OIDC groups to roles |
| Global vs project RBAC | Company-wide policy vs department-only policy | Global lives in `argocd-rbac-cm`; project-scoped lives in the AppProject `spec.roles[].policies` |
| Project JWT token | A corporate keycard for a robot | `argocd proj role create-token` -- a bearer token scoped to a specific project role, used by CI |
| Dex | The HR department that federates with every ID system | Bundled OIDC proxy with connectors for GitHub, SAML, LDAP, OIDC |
| Direct OIDC | Skip HR, talk straight to corporate ID | `argocd-cm` `oidc.config` pointing at Okta/Auth0/Keycloak/Google; simpler than Dex when the IdP speaks OIDC natively |
| g: group mapping | "Everyone in this Active Directory group is a manager" | `g, <oidc-group>, role:<role-name>` in policy.csv |
| Sealed Secrets | Tamper-evident locked diplomatic pouch | `kubeseal` encrypts with the in-cluster controller's public key; only that controller can decrypt; ciphertext safe in Git |
| External Secrets Operator (ESO) | Just-in-time vending machine -- the operator walks up to the vault with an IAM badge and fetches fresh every refresh | `ExternalSecret` + `SecretStore` CRDs pull live values from AWS Secrets Manager / Vault / GCP-SM / Azure KV and drop them into native Kubernetes Secrets; the app never knows a vault exists |
| SOPS | Sealed envelopes inside the manila folder | Per-file YAML encryption with age or KMS keys; `.sops.yaml` defines rules; decryption in ArgoCD via KSOPS plugin or helm-secrets |
| `notifications-cm` | The Rolodex of external services | ConfigMap with `service.slack`, `service.email`, etc. credentials and endpoints |
| Trigger | A named condition on the Application | CEL-like expression like `app.status.sync.status == 'OutOfSync'`; triggers fire templates |
| Template | The message body | Go text/template with access to `.app`, `.context`; generates Slack blocks, email bodies, JSON payloads |
| Subscription | Which service gets which trigger | Annotation on Application/AppProject: `notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts-prod` |
| `preserveResourcesOnDeletion` | "If you delete the template, DON'T delete the customers" | ApplicationSet flag that stops cascading deletes when the ApplicationSet itself is removed |

---

## The Franchise Framework

Yesterday you had one foreman on one site. Today the firm is a franchise:

1. **The Parent Company (App of Apps)** -- One root Application owns the bootstrapping of many child Applications. The root is a "work order to create more work orders." `kubectl apply -f root.yaml` is the entire day-one deployment.

2. **The Mail-Merge Machine (ApplicationSet)** -- Takes a template and a generator and stamps out N Applications. Scales where App-of-Apps scales badly. One declarative source of truth for "every app in every cluster."

3. **The Regional Offices (Managed Clusters)** -- Each registered via a Cluster Secret in the hub's `argocd` namespace. The hub reconciles into every spoke through that Secret's kubeconfig. The controller shards clusters across replicas so memory and watch load are distributed.

4. **The Departments (AppProjects)** -- Multi-tenant fences. Each Project restricts what Git repos its Applications may source from, what clusters/namespaces they may deploy to, what resource kinds they may touch, and who (via OIDC groups or JWT tokens) may sync them.

5. **The Vault (Secrets Strategy)** -- Three designs for getting sensitive values into the cluster without committing them plaintext to Git. Sealed Secrets, External Secrets Operator, or SOPS -- each with a different trust boundary.

6. **The Dispatch Radio (Notifications Controller)** -- The last-mile pipe from Application events to Slack, PagerDuty, email, webhooks. Triggers watch the Application object, templates render per-service message bodies, subscriptions route to recipients. One event can fan out to multiple channels, each at its own detail level.

```
THE FRANCHISE -- ARGOCD ADVANCED OVERVIEW
================================================================================

                        GIT (single source of truth)
                        +------------------------------+
                        | infra/                       |
                        |   bootstrap/                 |
                        |     root-app.yaml (App-of-Apps)|
                        |     apps/                    |
                        |       cert-manager.yaml      |
                        |       external-dns.yaml      |
                        |       monitoring.yaml        |
                        |   appsets/                   |
                        |     team-a-apps.yaml (AppSet)|
                        |     platform-addons.yaml     |
                        |   sealed/ or sops/ or eso/   |
                        +---------+--------------------+
                                  | pulled by repo-server
                                  v
  +------------- HUB CLUSTER (argocd namespace) --------------------------+
  |                                                                      |
  |  argocd-server      repo-server     application-controller (sharded) |
  |  applicationset-controller  notifications-controller  Dex            |
  |                                                                      |
  |  cluster-secrets:                                                    |
  |    in-cluster, prod-us-east-1, prod-eu-west-1, dev-us-east-1, ...   |
  +-----------+--------------+--------------+---------------+------------+
              |              |              |               |
              v              v              v               v
         +-------+        +-------+     +-------+       +-------+
         |prod-us|        |prod-eu|     |stg-us |       |dev-us |
         |east-1 |        |west-1 |     |east-1 |       |east-1 |
         | spoke |        | spoke |     | spoke |       | spoke |
         +-------+        +-------+     +-------+       +-------+

  PER CLUSTER: just a ServiceAccount + Secret with kubeconfig;
               no ArgoCD pods run on the spokes.

  NOTIFICATIONS                     TENANT BOUNDARIES
  -------------                     -----------------
  Slack, PagerDuty, email           AppProject per team:
  triggered by app.status events    - platform, team-a, team-b
                                    - scoped sourceRepos + destinations
                                    - OIDC group -> role mapping
                                    - JWT tokens for CI
```

---

## Part 1: App of Apps -- The Bootstrapping Pattern

### The Problem

Day zero for a new cluster: you need cert-manager, external-dns, aws-load-balancer-controller, Karpenter, the Prometheus stack, an ingress controller, ArgoCD itself (bootstrapped from outside), and a handful of platform operators. Applying 40 separate `kubectl apply -f ...` commands out of order is how clusters get stuck. You want **one command** that brings up everything, in the right order, with the right health gating.

### The Mechanic

**App of Apps** is a single **root Application** whose Git source points at a **directory full of other Application manifests**. When ArgoCD syncs the root, it applies every child Application manifest it finds in that directory. Each child Application then reconciles its own workload. You have reduced "bootstrap 40 components" to "bootstrap 1 component that knows about 40 components."

```yaml
# root-app.yaml -- the franchise parent
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # Cascading delete: tear down all children too
spec:
  project: platform
  source:
    repoURL: https://github.com/acme/infra.git
    path: bootstrap/apps                         # Directory full of child Application manifests
    targetRevision: main
    directory:
      recurse: true                              # Walk subdirectories
      jsonnet: {}                                # Enable jsonnet if any *.jsonnet files
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd                            # Children are applied into argocd namespace (they ARE Applications)
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```
bootstrap/apps/
  cert-manager.yaml          # Application pointing at the cert-manager chart
  external-dns.yaml          # Application pointing at external-dns
  ingress-nginx.yaml         # Application pointing at ingress-nginx
  karpenter.yaml             # Application pointing at karpenter
  kube-prometheus-stack.yaml # Application pointing at kube-prometheus
```

Each child looks like a normal Application:

```yaml
# bootstrap/apps/cert-manager.yaml -- one franchisee
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-20"        # Bootstrap ordering: cert-manager goes first
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.14.3
    helm:
      valuesObject:
        installCRDs: true
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

### Bootstrap Ordering with Sync Waves

Children are themselves resources from the root Application's perspective. That means **sync waves on child Application manifests order their creation from the root's sync**. Annotate children with increasing waves:

```
sync-wave: -20   cert-manager           (needs CRDs up first)
sync-wave: -15   external-dns
sync-wave: -10   ingress-nginx          (depends on cert-manager for TLS certs)
sync-wave: -5    aws-load-balancer-controller
sync-wave:   0   kube-prometheus-stack  (default)
sync-wave:  10   team-a-apps            (workloads last)
```

The root Application's Sync phase applies children in wave order, waiting for each wave to reach Healthy before advancing. Since each child Application's initial health is "Progressing until its own child resources are Healthy," this gives you **natural bootstrap ordering** without a CI script.

### Cascading Delete

The `resources-finalizer.argocd.argoproj.io` finalizer on the **root** Application tells ArgoCD to delete the root's managed resources (the child Applications) when the root is deleted. Without the finalizer, `kubectl delete app root` leaves all children behind as orphaned Applications that keep reconciling their workloads. With the finalizer, the delete cascades: root -> children -> child workloads. This is the one-command teardown of the entire cluster's platform layer.

**Gotcha**: the child Applications also need their own `resources-finalizer.argocd.argoproj.io` so their deletion cascades to their managed workloads. Otherwise you get: root deleted, children deleted, but cert-manager pods still running because nothing told them to go away. Always set the finalizer on every Application, root or child.

### Nested App of Apps -- Platform vs Product Separation

The classic organizational layout:

```
root-app (platform team's entry point)
  |
  +-- platform-addons (App-of-Apps owned by platform team)
  |     +-- cert-manager
  |     +-- external-dns
  |     +-- ingress-nginx
  |     +-- karpenter
  |
  +-- team-a-apps (App-of-Apps owned by Team A)
  |     +-- team-a-api
  |     +-- team-a-worker
  |     +-- team-a-db
  |
  +-- team-b-apps (App-of-Apps owned by Team B)
        +-- team-b-web
        +-- team-b-scheduler
```

The outer root belongs to the platform team. The inner `team-a-apps` and `team-b-apps` are themselves root Applications whose Git path is owned by the respective product team. This gives you:

- **Organizational separation.** Product teams don't need write on platform's Git path.
- **Independent Git cadences.** Team A can restructure their own app layout without asking platform.
- **Scoped blast radius.** A broken child in Team A doesn't prevent Team B's reconciliation.
- **Delegated onboarding.** Adding a new team = adding one more child Application in the root, pointing at the team's Git directory.

### The Drift Cascade Gotcha

**When a child Application's spec is edited by a human directly in the cluster, the root Application reconciles it back.** The root's Git source says "child cert-manager Application should look like *this*," and any drift on the child triggers a self-heal that reverts the manual edit. This is correct behavior, but it surprises teams who expect the child to be "their" Application: any manual change to a child is a three-way race between (the child Application's own self-heal reconciling into its workload) and (the root Application's self-heal reconciling the child itself). Always edit Git, never the cluster.

### Where App of Apps Breaks Down

The pattern scales cleanly up to about 50-100 Applications. Past that, three problems compound:

1. **One Git directory becomes unreadable.** 200 Application YAMLs in one folder is a bad code review experience.
2. **Adding a new cluster means duplicating the whole tree.** If you want the same 40 platform add-ons on three new clusters, you copy-paste the tree three times and patch the `destination.server` field in 120 places.
3. **No templating.** Every child is a static YAML file. If the only difference between prod-us-east-1 and prod-eu-west-1 is the cluster destination and one value, you still have two full files.

The answer is the second pattern: **ApplicationSet**.

---

## Part 2: ApplicationSet -- The Mail-Merge Machine

> **Why this pattern exists**: with hand-written Applications, config scales as **apps x clusters** (multiplicative) -- 10 apps across 5 clusters is 50 YAML files to maintain. With a Cluster generator + Git directory generator (Matrix), config scales as **apps + clusters** (additive) -- one file per app, one Secret per cluster. Adding a cluster is "register a Secret"; adding an app is "drop a folder." Nothing else rewrites. That shift from multiplicative to additive is the entire reason ApplicationSet exists.

### The Mental Model

An **ApplicationSet** is a CR with two top-level pieces:

- **`template:`** -- a single Application skeleton with `{{ .placeholder }}` variables.
- **`generators:`** -- producers of parameter sets (rows). The controller runs each generator, renders the template once per row, and creates/updates the resulting Application in the `argocd` namespace.

Think mail-merge. One form letter ("Dear {{ .firstName }}"), one CSV spreadsheet (firstName, lastName, address columns, thousand rows), one command, thousand letters produced.

The ApplicationSet Controller is the machine. It watches its own `ApplicationSet` CR **and** watches its generator inputs (Git refs, cluster Secrets, SCM APIs), and it reconciles the set of generated Applications to match the product of (template x generators). When a generator input changes (new PR opened, new cluster registered, new directory added), the ApplicationSet Controller generates, updates, or prunes Applications automatically.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-a-apps
  namespace: argocd
spec:
  goTemplate: true                               # Use full Go templates, not legacy fasttemplate
  goTemplateOptions: ["missingkey=error"]        # Fail loud on typos

  generators:
    - list:
        elements:
          - app: api
            image: acme/api:2.8.2
          - app: worker
            image: acme/worker:2.8.2
          - app: web
            image: acme/web:2.8.2

  template:                                      # The Application skeleton
    metadata:
      name: team-a-{{ .app }}
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: team-a
      source:
        repoURL: https://github.com/acme/team-a.git
        path: charts/{{ .app }}
        targetRevision: main
        helm:
          valuesObject:
            image: "{{ .image }}"
      destination:
        server: https://kubernetes.default.svc
        namespace: team-a
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true]
```

This ApplicationSet yields three Applications: `team-a-api`, `team-a-worker`, `team-a-web`. Adding a fourth app is one extra list element, not one extra YAML file.

### The Seven Generators

#### 1. List Generator -- Hand-Maintained Roster

Simplest generator. Good for small static sets where no other generator's inputs exist.

```yaml
generators:
  - list:
      elements:
        - cluster: prod-us-east-1
          url: https://EABC123.gr7.us-east-1.eks.amazonaws.com
          env: prod
        - cluster: prod-eu-west-1
          url: https://EDEF456.gr7.eu-west-1.eks.amazonaws.com
          env: prod
        - cluster: staging
          url: https://EGHI789.gr7.us-east-1.eks.amazonaws.com
          env: staging
```

Use when: the list is short, stable, and humans manage it directly. Anti-pattern: anything dynamic (new clusters, new PRs, new git directories) -- you'll end up maintaining the list by hand and drifting.

#### 2. Cluster Generator -- Every Registered Cluster

**Same mail-merge frame**: the spreadsheet is the cluster Secret registry in the `argocd` namespace. Each labeled cluster Secret is one row, with columns for `name`, `server`, and whatever labels/annotations you attached at registration. The hub doesn't travel anywhere -- it templates a new Application per row and hands each to the Application Controller, which talks to the matching cluster via the Secret's kubeconfig.

Iterates over the cluster Secrets in the `argocd` namespace, one Application per matching cluster.

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          environment: prod                      # Only cluster Secrets labeled environment=prod
      values:                                    # Extra per-cluster values (surfaced as {{ .values.region }})
        region: "{{ index .metadata.labels \"region\" }}"

template:
  metadata:
    name: platform-addons-{{ .name }}            # .name is the Cluster Secret's name
  spec:
    project: platform
    source:
      repoURL: https://github.com/acme/infra.git
      path: platform-addons
      targetRevision: main
      helm:
        valuesObject:
          clusterName: "{{ .name }}"
          region: "{{ .values.region }}"
    destination:
      server: "{{ .server }}"                    # .server is the Cluster Secret's API URL
      namespace: platform
```

Available variables: `{{ .name }}`, `{{ .server }}`, `{{ .metadata.labels.* }}`, `{{ .metadata.annotations.* }}`, plus any `values:` you define. This is **the** generator for "deploy this add-on to every prod cluster" patterns.

**Gotcha**: the magic label `argocd.argoproj.io/secret-type: cluster` is required on cluster Secrets. The `in-cluster` entry (the ArgoCD home cluster itself) **does not have a Secret by default** -- ArgoCD auto-generates a synthetic cluster entry for `https://kubernetes.default.svc`. To make the Cluster generator see it, either select by label (add a real Secret with labels matching the synthetic in-cluster endpoint) or include a `selector: {}` empty selector which matches all clusters including the synthetic in-cluster one.

#### 3. Git Generator -- Directories

One Application per subdirectory of a Git path.

```yaml
generators:
  - git:
      repoURL: https://github.com/acme/team-a.git
      revision: main
      directories:
        - path: apps/*                           # Glob: one Application per apps/<subdir>
        - path: apps/experimental/*
          exclude: true                          # Skip these
```

Available variables: `{{ .path.path }}` (full path), `{{ .path.basename }}` (just the leaf name), `{{ .path.basenameNormalized }}` (DNS-safe version of basename).

```yaml
template:
  metadata:
    name: "{{ .path.basenameNormalized }}"
  spec:
    source:
      repoURL: https://github.com/acme/team-a.git
      path: "{{ .path.path }}"                   # e.g. apps/api, apps/worker
      targetRevision: main
    destination:
      server: https://kubernetes.default.svc
      namespace: team-a
```

Use when: the monorepo's directory layout is the app inventory. Adding a new app = creating a new directory, no ApplicationSet edit required.

#### 4. Git Generator -- Files

One Application per YAML or JSON file matching a glob; the file's top-level keys become template parameters.

```yaml
generators:
  - git:
      repoURL: https://github.com/acme/infra.git
      revision: main
      files:
        - path: environments/*/config.yaml       # Each environments/<env>/config.yaml renders one Application
```

Example `environments/prod/config.yaml`:

```yaml
env: prod
cluster: prod-us-east-1
replicaCount: 10
image:
  tag: "2.8.2"
```

Template reads `{{ .env }}`, `{{ .cluster }}`, `{{ .replicaCount }}`, `{{ .image.tag }}`. Use when each "row" of the mail merge needs rich structured data, not just flat strings.

#### 5. Matrix Generator -- SQL Cross Join

Cartesian product of two generators. The workhorse for multi-tenant platforms where you want "every app in every cluster."

```yaml
generators:
  - matrix:
      generators:
        - clusters:                              # Generator A: every prod cluster
            selector:
              matchLabels: { environment: prod }
        - git:                                   # Generator B: every app directory
            repoURL: https://github.com/acme/platform-addons.git
            revision: main
            directories:
              - path: addons/*

template:
  metadata:
    name: "{{ .path.basenameNormalized }}-{{ .name }}"   # e.g. cert-manager-prod-us-east-1
  spec:
    project: platform
    source:
      repoURL: https://github.com/acme/platform-addons.git
      path: "{{ .path.path }}"
      targetRevision: main
    destination:
      server: "{{ .server }}"
      namespace: platform
```

If generator A produces 3 clusters and generator B produces 10 add-ons, the matrix produces **30 Applications**. The cardinality can blow up fast -- a 3x3 matrix of matrixes is 81 Applications, most of which you probably didn't mean. Review matrix outputs in a dry run (`argocd appset generate`) before applying.

**Gotcha -- parameter name collisions**: if both generators emit a field called `name`, the second silently overrides the first. Use `values:` blocks in each generator to remap fields into unambiguous keys (`{{ .values.clusterName }}` + `{{ .values.addonName }}`).

#### 6. Merge Generator -- SQL LEFT JOIN on a Key

Combines generators by matching on one or more `mergeKeys`. The second generator's fields override the first's for matching rows.

```yaml
generators:
  - merge:
      mergeKeys:
        - server                                 # Join on the cluster URL
      generators:
        - clusters: {}                           # Generator A: all clusters (baseline)
        - list:                                  # Generator B: per-cluster overrides
            elements:
              - server: https://EABC.eks.amazonaws.com
                replicaCount: "20"               # Override just for this cluster
              - server: https://EDEF.eks.amazonaws.com
                replicaCount: "5"
```

Every cluster gets an Application; clusters listed in generator B get the override values merged in. Use when the baseline is "all clusters with defaults" and a few clusters need bespoke overrides without duplicating the whole list.

#### 7. SCM Provider Generator -- Auto-Discover Repos

Queries GitHub/GitLab/Bitbucket/Gitea for repos matching filters. One Application per discovered repo.

```yaml
generators:
  - scmProvider:
      github:
        organization: acme                       # Every repo in the acme org
        allBranches: false
        tokenRef:
          secretName: github-token
          key: token
      filters:
        - repositoryMatch: ^service-.*           # Only repos named service-*
        - pathsExist: [charts/Chart.yaml]        # And only if they contain a Helm chart
```

The generator re-queries on its `requeueAfterSeconds` interval (default 30m). Use when the organization follows a convention like "every service repo has a charts/ folder" and you want ArgoCD to auto-adopt new services.

**Gotcha**: GitHub API rate limits. A large org with many repos and a 30-minute requeue is fine; a 1-minute requeue will hit rate limits. Use a GitHub App token (5000 req/hour per install) for larger orgs, not a personal access token (5000 req/hour for your entire user).

#### 8. Pull Request Generator -- Ephemeral Previews

One Application per open PR. Auto-deleted on merge or close.

```yaml
generators:
  - pullRequest:
      github:
        owner: acme
        repo: payments-api
        tokenRef:
          secretName: github-token
          key: token
        labels:                                  # Only PRs with this label
          - preview
      requeueAfterSeconds: 60

template:
  metadata:
    name: "preview-pr-{{ .number }}"             # .number = PR number
    labels:
      preview: "true"
  spec:
    project: previews
    source:
      repoURL: https://github.com/acme/payments-api.git
      path: charts/payments-api
      targetRevision: "{{ .head_sha }}"          # Deploy the PR's exact SHA
      helm:
        valuesObject:
          image:
            tag: "pr-{{ .number }}"
          ingress:
            host: "pr-{{ .number }}.preview.acme.com"
    destination:
      server: https://kubernetes.default.svc
      namespace: "preview-pr-{{ .number }}"
    syncPolicy:
      automated: { prune: true, selfHeal: true }
      syncOptions: [CreateNamespace=true]
```

Available variables (with `goTemplate: true`): `.number` (PR number as int), `.branch` (source branch name), `.branch_slug` (DNS-safe version of branch name, useful for hostnames), `.head_sha` (full commit SHA of the PR head), `.head_short_sha` (short SHA), `.head_short_sha_7` (7-character SHA), `.labels` (array of label strings). Note that `.author` is NOT a standard field -- don't rely on it; if you need the PR author, query the SCM API out-of-band or use the SCM Provider generator's filters. The preview stack is created when the PR opens and destroyed when it closes -- the exact ephemeral-environment pattern teams used to build with custom Jenkins jobs.

**Gotcha**: deletion on PR close depends on (a) the Application having `resources-finalizer.argocd.argoproj.io` so its managed resources are cleaned up, and (b) the ApplicationSet **not** having `preserveResourcesOnDeletion: true`. With that flag on, the Application is deleted but its workloads stay, forever. Combine it with (c) the Kubernetes namespace being unique per PR (`preview-pr-{{ .number }}`) so a stuck Finalizer on one preview doesn't block another.

### Templates -- `goTemplate` and `templatePatch`

**Legacy fasttemplate** (default when `goTemplate` is unset) is simple string substitution: `{{ .image }}` becomes the value verbatim. No conditionals, no loops, no functions. It fails silently when you typo a variable -- `{{ .imge }}` renders as empty string.

**Go templates** (`goTemplate: true`) unlock the full Go `text/template` language plus sprig functions:

```yaml
template:
  spec:
    source:
      helm:
        valuesObject:
          image:
            tag: "{{ .image.tag | default \"latest\" }}"
          replicaCount: "{{ if eq .env \"prod\" }}10{{ else }}2{{ end }}"
          regionLabel: "{{ .region | upper }}"
```

**Always set `goTemplate: true` for new ApplicationSets.** It's strictly more powerful. The default has been flipped in some distributions but the project default for backward compatibility is still the legacy engine; be explicit.

**Always set `goTemplateOptions: ["missingkey=error"]`.** Without it, undefined variables render as empty string and you ship a broken Application with `targetRevision: ""`. With it, the ApplicationSet fails loudly and nothing is generated until you fix the typo.

**`templatePatch`** is a second-stage string patch applied after template render, useful for per-element overrides that can't be expressed in a single template:

```yaml
template:
  spec:
    source:
      targetRevision: main                        # Default
  # templatePatch is a YAML string that merges into the rendered template
  templatePatch: |
    {{- if eq .env "prod" }}
    spec:
      source:
        targetRevision: "{{ .prodPinnedSha }}"
    {{- end }}
```

`templatePatch` is powerful but harder to read than a conditional `valuesObject`. Reach for it only when plain Go templates inside the main template can't express the shape.

### `preservedFields` -- Metadata Only, Not Spec

By design, an ApplicationSet **overwrites** generated Applications on every reconcile. `preservedFields` exists to exempt specific **metadata** entries from that overwrite -- useful when another controller writes annotations or labels onto the generated Applications and you don't want the ApplicationSet reconcile to wipe them out.

**What it actually preserves**: `annotations` and `labels` in `metadata`. Nothing else. There is **no mechanism** to preserve arbitrary `spec.*` field paths like `spec.source.targetRevision`.

```yaml
spec:
  goTemplate: true
  preservedFields:
    annotations:
      - argocd-image-updater.argoproj.io/image-list       # Written by argocd-image-updater
      - notifications.argoproj.io/subscribe.on-sync-failed.slack  # Written by on-call tooling
    labels:
      - team.acme.com/owner                               # Set by external labeling jobs
```

The honest use case is **protecting metadata set by other controllers** from being overwritten on reconcile -- image-updater annotations, notifications subscriptions added out-of-band, labels from cost-allocation tooling. It is **not** a hotfix mechanism.

### The Real Hotfix Escape Hatches

The common misconception: "I'll use `preservedFields` to pin a hotfix `targetRevision` on one generated Application." This does not work -- `spec` paths are not preservable. The three real options:

**Option 1 -- `ignoreApplicationDifferences` (recommended, newer feature)**

The ApplicationSet controller supports declaring specific JSON paths in generated Applications that should be ignored during reconciliation. On-call edits to those paths stick until removed:

```yaml
spec:
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/source/targetRevision        # Let on-call pin revisions
        - /spec/source/helm/valuesObject/image/tag
      # Optionally scope by name/labels of the generated Application
      name: preview-pr-42
```

**Option 2 -- Delete + preserve + rebuild**

```bash
# Stop the ApplicationSet from cascading the delete to its children
kubectl patch applicationset team-a-apps -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"preserveResourcesOnDeletion":true}}}'

# Delete the ApplicationSet; its Applications stay alive
kubectl delete applicationset team-a-apps -n argocd

# Now edit the orphaned Application directly for your hotfix
kubectl edit application team-a-api -n argocd

# Later, rebuild the ApplicationSet; it will re-adopt matching Applications
```

**Option 3 -- Route the hotfix through a generator input**

If the generator is a List or a Git files generator, edit the generator's source of truth (the list element or the config file) to pin the SHA. The change propagates through the template on the next reconcile. This keeps the hotfix declarative and in Git:

```yaml
# List generator holding per-app pinned revisions
generators:
  - list:
      elements:
        - app: api
          revision: main
        - app: worker
          revision: abc123def        # Pinned SHA for hotfix; revert by changing back to "main"
        - app: web
          revision: main
```

Of these, `ignoreApplicationDifferences` is the closest to "preserve spec fields" as a controller-native feature. Option 2 is a last resort during incidents. Option 3 is the GitOps-pure answer and the one to design for from day one.

### `applicationsSync` -- Controller Policy for Create/Update/Delete

`preserveResourcesOnDeletion` is **per-ApplicationSet** and only controls what happens when the ApplicationSet itself is deleted. There is a second, broader knob: the ApplicationSet controller's **policy** for how it's allowed to mutate generated Applications on any reconcile.

Set globally via the `--policy` flag on the `argocd-applicationset-controller` Deployment, or overridden per-ApplicationSet via the `argocd.argoproj.io/applicationset-refresh` family of annotations (newer releases expose it as `applicationsSync` on the AppSet spec):

```yaml
# In the applicationset-controller Deployment
spec:
  template:
    spec:
      containers:
        - name: argocd-applicationset-controller
          args:
            - --policy=create-update       # or create-only, create-delete, sync
```

```yaml
# Per-ApplicationSet override
spec:
  syncPolicy:
    applicationsSync: create-update        # Overrides the controller default for this AppSet
    preserveResourcesOnDeletion: false
```

The four policies:

| Policy | Create new Apps? | Update existing Apps? | Delete Apps no longer in generator output? |
|--------|------------------|-----------------------|----|
| `sync` (default) | yes | yes | yes |
| `create-update` | yes | yes | **no** -- Apps that fall out of the generator are left alone |
| `create-delete` | yes | **no** -- existing Apps are never overwritten | yes |
| `create-only` | yes | **no** | **no** -- the AppSet bootstraps Applications and never touches them again |

**When to pick what**:
- `sync` (default) for the normal mail-merge model -- generator is the source of truth, Applications mirror it exactly.
- `create-update` during migrations where you're moving Applications between AppSets and don't want the old AppSet deleting things the new one is about to own.
- `create-delete` when humans edit generated Applications after creation and you want those edits to stick without blocking cleanup.
- `create-only` when the ApplicationSet is a one-shot bootstrapper (e.g., seed preview environments that evolve independently).

Pair with `preserveResourcesOnDeletion`: `applicationsSync` controls reconcile-time behavior, `preserveResourcesOnDeletion` controls AppSet-deletion-time behavior. They're orthogonal.

### Progressive Rollouts -- `RollingSync` Strategy

By default, an ApplicationSet change reconciles **every generated Application in parallel**. For 30 Applications across 3 prod clusters, that means all 30 get the change simultaneously. Not always what you want.

`RollingSync` (still beta as of ArgoCD 2.x) lets you progress through labeled waves:

```yaml
spec:
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: env
              operator: In
              values: [dev]                      # Wave 1: dev only
        - matchExpressions:
            - key: env
              operator: In
              values: [staging]                  # Wave 2: staging, only after dev is Healthy
        - matchExpressions:
            - key: env
              operator: In
              values: [prod]
              maxUpdate: 25%                     # Wave 3: prod, 25% at a time
```

The controller applies changes to Applications matching step 1's labels, waits for them all to be Healthy, then advances to step 2, and so on. Combined with cluster labels on the Cluster generator, this is the "ring deployment" pattern: dev canary first, staging next, then prod-small-region, then prod-large-region. The Applications themselves still reconcile their workloads as usual -- `RollingSync` only controls the pace at which the ApplicationSet rolls its own template changes out to them.

---

## Part 3: Multi-Cluster -- Registration, Topology, Sharding

### Registering a Cluster -- What `argocd cluster add` Actually Does

Running `argocd cluster add my-context` from the hub does four things on the target cluster and one on the hub:

1. **Creates a ServiceAccount** named `argocd-manager` in the `kube-system` namespace of the target.
2. **Creates a ClusterRoleBinding** binding that ServiceAccount to `cluster-admin` (or a narrower role with `--service-account` and custom ClusterRole).
3. **Extracts the ServiceAccount's token**.
4. **Builds a kubeconfig** using the target cluster's API server URL, CA bundle, and the extracted token.
5. **Creates a Secret** in the **hub's** `argocd` namespace labeled `argocd.argoproj.io/secret-type: cluster`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-prod-us-east-1
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    environment: prod                            # Custom labels for Cluster generator selectors
    region: us-east-1
type: Opaque
stringData:
  name: prod-us-east-1
  server: https://EABC123.gr7.us-east-1.eks.amazonaws.com
  config: |
    {
      "bearerToken": "eyJhbGciOi...",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "LS0tLS1CRUdJTi..."
      }
    }
```

**The Secret is the source of truth.** You can register clusters by creating the Secret directly from Terraform or a GitOps bootstrap repo -- the `argocd cluster add` CLI is just a convenience wrapper. This matters for GitOps purity: you want cluster registration to be declarative, not "someone ran a CLI once."

**For EKS specifically**, you don't put a long-lived bearer token in the Secret. You use IRSA: the Secret's `config` declares `awsAuthConfig` with a `clusterName` and a `roleARN`, and the ArgoCD application-controller's ServiceAccount has an IAM role that can `eks:DescribeCluster` and assume the target's role. Tokens are minted just-in-time via STS. This is the production-grade cross-account EKS pattern.

### The `in-cluster` Shortcut

Every ArgoCD install has a synthetic "in-cluster" entry pointing at `https://kubernetes.default.svc` -- the cluster ArgoCD itself runs in. You reference it in an Application with `destination.server: https://kubernetes.default.svc` or `destination.name: in-cluster`. No Secret required; it's auto-generated.

### Standalone vs Hub-and-Spoke vs Cluster-Per-Env

Three topologies, each with a different trade-off:

| Topology | Control Plane | Blast Radius | Operational Overhead | Cross-Cluster Visibility |
|----------|--------------|--------------|----------------------|--------------------------|
| **Standalone per cluster** | One ArgoCD per cluster | Smallest -- hub outage affects one cluster | Highest -- N ArgoCDs to upgrade, back up, alert on | None -- each UI shows its own cluster only |
| **Hub and spoke** | One hub; N spokes register | Medium -- hub outage pauses reconciliation for all spokes | Lowest -- one ArgoCD to operate | Full -- one UI, one `argocd app list`, cross-cluster ApplicationSets trivial |
| **Cluster per env** | One ArgoCD per env (dev, stg, prod) | Each env isolated | Medium | Per-env only |

The **hub and spoke** model (one ArgoCD in a dedicated "management" cluster, every workload cluster registered as a spoke) is the most common pattern at scale. The hub holds the kubeconfigs, runs the sharded application-controller, serves the unified UI, and fans out to spokes. The spokes run nothing ArgoCD-specific -- just a ServiceAccount (or IRSA role).

The **blast-radius counter-argument** for standalone: if the hub goes down, all reconciliation across all spokes pauses. Workloads keep running (ArgoCD is a control plane, not a data plane), but no new commits deploy and drift goes unchecked. Hub HA (multiple application-controller replicas, Redis HA, multi-AZ) mitigates this in practice.

The **cluster-per-env** middle ground is sometimes used when compliance insists that "prod ArgoCD must not see dev resources." Each environment is its own hub, reducing blast radius but losing cross-env promotion workflows in a single UI.

### Application Controller Sharding

The application-controller holds the **live state cache for every managed cluster** -- every Pod, Service, ConfigMap, CRD across every spoke. Memory scales with (clusters x resources per cluster). Past ~20-30 clusters on a single controller replica, you'll OOM even on fat nodes.

Sharding splits clusters across controller replicas. Each shard owns a subset of clusters and holds only their cache.

> **Core mental model**: controller replicas aren't stateless -- scaling them moves cluster ownership, and every ownership move is a cold-cache rebuild plus a LIST burst on the spoke. Pick the sharding algorithm that minimizes moves.

**Configuration -- three moving parts**:

```yaml
# argocd-cmd-params-cm ConfigMap
data:
  controller.sharding.algorithm: consistent-hashing   # or "legacy" or "round-robin"
```

```yaml
# argocd-application-controller StatefulSet patch
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: argocd-application-controller
  namespace: argocd
spec:
  replicas: 3                                    # (1) Physical pod count
  template:
    spec:
      containers:
        - name: argocd-application-controller
          env:
            - name: ARGOCD_CONTROLLER_REPLICAS   # (2) Logical shard count (must match .spec.replicas pre-2.8)
              value: "3"
```

Three algorithms:

| Algorithm | How it decides | Stability on scale change |
|-----------|---------------|---|
| `legacy` | `hash(cluster UID) mod replicas` | Poor -- adding a replica reshuffles nearly every cluster |
| `round-robin` | Distribute clusters evenly in registration order | Poor -- also reshuffles most clusters on scale; only predictable for static test setups |
| `consistent-hashing` | Consistent hash ring with bounded loads (Google's CHWBL algorithm) | Best -- adding a replica moves ~1/N clusters, rest keep their assignments |

**`consistent-hashing` (more precisely, `consistent-hashing-with-bounded-loads`) is the only algorithm safe for multi-replica production.** Round-robin's "distribute evenly in registration order" sounds stable but in practice reshuffles most clusters on scale changes, similar to legacy. Only consistent-hashing keeps the majority of cluster-to-shard assignments when replicas change. This matters operationally because shard migration means invalidating the cache and rebuilding the live state from scratch for the migrated clusters -- a stampede on the target API servers if it happens to every cluster at once.

**What the spoke experiences during a rebalance**: sharding discussions usually focus on the hub-side cost (controller memory, cache rebuild), but the real production blast radius is on the spoke. Each cluster reassignment triggers a LIST-then-WATCH across every tracked Kubernetes resource kind on the spoke's API server -- every Pod, Service, ConfigMap, CRD the controller cares about, all at once. Simultaneous rebalances across many clusters can trip **API Priority and Fairness** throttling on the spoke, which backs up kubelet lease renewals, HPA metrics scrapes, and any other controllers running there. This is the real cost of picking `legacy` over `consistent-hashing`: it's not just cache rebuild on the hub, it's a multi-spoke API storm that degrades cluster health while the reshuffle stabilizes.

**The replicas trap (pre-2.8)**: `ARGOCD_CONTROLLER_REPLICAS` is a **logical shard count** the controller reads from its own env var. The StatefulSet's `.spec.replicas` is the **physical pod count**. They **must match**. Set one without the other and you either have pods owning no shards (idle) or shards with no pods (stuck clusters). There is no `controller.replicas` key in `argocd-cmd-params-cm` -- the logical count is env-var-driven on the StatefulSet.

**Dynamic Cluster Distribution (ArgoCD 2.8+)**: the controller can auto-detect the replica count from its own StatefulSet/Deployment and rebalance cluster-to-shard assignments as replicas scale, without needing `ARGOCD_CONTROLLER_REPLICAS` set. Enable with:

```yaml
env:
  - name: ARGOCD_ENABLE_DYNAMIC_CLUSTER_DISTRIBUTION
    value: "true"
```

With Dynamic Cluster Distribution on, scaling the StatefulSet is the only knob -- the controller picks up the new shard count on its own, re-runs the sharding algorithm, and migrates the affected clusters. On 2.8+ this is the recommended mode.

**Migration sequence -- scaling controller replicas safely**:

1. Switch the sharding algorithm to `consistent-hashing` in `argocd-cmd-params-cm` first, restart the StatefulSet, and verify steady state via `argocd admin cluster stats` before touching replica count. Doing the algorithm swap and the replica bump in the same change multiplies the blast radius.
2. Update `ARGOCD_CONTROLLER_REPLICAS` on the StatefulSet to the target count (**or** enable Dynamic Cluster Distribution on 2.8+, which makes this step unnecessary).
3. Scale the StatefulSet `.spec.replicas` to the new count.
4. Monitor controller memory, spoke API latency, and Application health for 10-15 minutes as the affected shards rebuild their caches.

**The pitfall**: on pre-2.8 without Dynamic Cluster Distribution, scaling the StatefulSet without first updating `ARGOCD_CONTROLLER_REPLICAS` results in **silent cluster starvation** -- the new pods compute their shard indices using a modulus the old env var doesn't know about, and they claim zero clusters. Applications on those clusters stop reconciling with no obvious error. Always bump the env var first (or use DCD), then scale.

**When does sharding actually matter**: below ~10 managed clusters, one replica is fine. Between 10 and 30, one fat replica or two smaller ones. Past 30, sharding is mandatory and consistent-hashing is the algorithm of choice. If you're operating under 10 clusters, don't pre-optimize -- a single controller with enough memory is simpler to reason about.

---

## Part 4: AppProjects, RBAC, and SSO

### AppProject as Tenant Boundary

> **AppProject is a default-deny sandbox, but its defaults are asymmetric and its CLI-side tools are not safe-by-default.** `clusterResourceWhitelist: []` means deny-all; `namespaceResourceBlacklist: []` means allow-all. `sourceRepos` is a literal string match (missing the `.git` suffix is rejected even though the repo clones fine manually). `argocd proj role create-token` mints tokens with **no expiry** unless `--expires-in` is passed. Every production AppProject pairs the spec with: a canonical repo-URL convention, surgical whitelists for cluster-scoped resources, mandatory `--expires-in` in any token-minting automation, and signed-commit enforcement for security-sensitive repos.

An AppProject is **not a folder for organizing Applications in the UI**. It is a **security boundary** that restricts what any Application in the project is allowed to do:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Team A's applications

  # Allowed source Git repositories (glob supported)
  sourceRepos:
    - https://github.com/acme/team-a.git
    - https://github.com/acme/shared-charts.git
  # Destinations: which cluster + namespace combinations are allowed
  destinations:
    - server: https://EABC123.gr7.us-east-1.eks.amazonaws.com
      namespace: team-a
    - server: https://EABC123.gr7.us-east-1.eks.amazonaws.com
      namespace: team-a-*                        # Wildcard namespaces
    - name: prod-eu-west-1                       # By name is preferred
      namespace: team-a

  # Cluster-scoped resources this project's Applications may create
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace                            # Allowed: Namespace creation
  clusterResourceBlacklist:
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding                   # Explicit deny

  # Namespaced resources: empty whitelist with blacklist is "everything except the blacklist"
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota

  # Orphaned resource warnings (detection, not enforcement)
  orphanedResources:
    warn: true
    ignore:
      - kind: ConfigMap
        name: kube-root-ca.crt

  # Per-project RBAC roles
  roles:
    - name: deploy
      description: Can sync and override Applications in this project
      policies:
        - p, proj:team-a:deploy, applications, get, team-a/*, allow
        - p, proj:team-a:deploy, applications, sync, team-a/*, allow
        - p, proj:team-a:deploy, applications, action/*, team-a/*, allow
      groups:
        - acme:team-a-devs                       # OIDC group mapped to this role
    - name: ci
      description: JWT token for CI pipeline syncs
      policies:
        - p, proj:team-a:ci, applications, sync, team-a/*, allow
```

The **destination's cluster may be referenced by `server` (API URL) or `name` (friendly label from the cluster Secret)**. Prefer `name` -- API URLs change when clusters are rebuilt; the friendly name can stay stable.

**The whitelist vs blacklist asymmetry (critical):**

> An empty `clusterResourceWhitelist: []` means **deny ALL** cluster-scoped resources.
> An empty `namespaceResourceBlacklist: []` means **allow ALL** namespaced resources.
>
> The cluster-scoped path is allow-list (default deny, explicit allow). The namespaced path is block-list (default allow, explicit deny). This is intentional -- cluster-scoped resources are high-blast-radius (CRDs, ClusterRoleBindings) and default-deny is the safe posture. Namespaced resources are tenant-local and default-allow is the ergonomic one. Internalize the asymmetry: leaving `clusterResourceWhitelist` off the project is **not** the same as leaving `namespaceResourceBlacklist` off.

A team-a Application with an empty/absent `clusterResourceWhitelist` literally cannot create a ClusterRoleBinding or install a CRD even if the manifests say to. Allow specific kinds (Namespace, perhaps) explicitly.

### `syncWindows` -- Time-Based Deploy Gates

Production change-freeze policy declared on the project itself. A window is either `allow` (only deploys within this window) or `deny` (no deploys during this window).

```yaml
spec:
  syncWindows:
    # Block prod deploys on Fridays 17:00 UTC for 48 hours (weekend freeze)
    - kind: deny
      schedule: "0 17 * * 5"
      duration: 48h
      applications:
        - "*"
      clusters:
        - prod-us-east-1
        - prod-eu-west-1
      manualSync: false               # Even manual syncs are blocked during the window

    # Allow prod deploys only during business hours Mon-Fri 09:00-17:00 UTC
    - kind: allow
      schedule: "0 9 * * 1-5"
      duration: 8h
      namespaces:
        - payments
      manualSync: true                # Manual syncs override the implicit deny outside the window
```

Fields:
- **`kind`**: `allow` or `deny`.
- **`schedule`**: cron expression for the window's start.
- **`duration`**: how long the window stays open (e.g., `8h`, `48h`, `30m`).
- **`applications`** / **`namespaces`** / **`clusters`**: scope of the window; any matches. Use `"*"` for all.
- **`manualSync`**: whether a human clicking "Sync" in the UI overrides the window. `false` means even incidents can't bypass without removing the window.

`allow` and `deny` windows can overlap -- the rule is: if any `deny` matches now, sync is denied; otherwise, if any `allow` window exists and none matches now, sync is denied; otherwise, sync is allowed.

### `signatureKeys` -- GPG-Signed Commits Only

For regulated environments where every deploy must come from a signed commit:

```yaml
spec:
  signatureKeys:
    - keyID: ABCD1234EF567890           # GPG key ID (long form)
    - keyID: 1234567890ABCDEF
```

With this set on the project, ArgoCD refuses to sync Applications whose source Git revisions are not signed by one of the listed GPG keys. Requires either global signature verification enabled on the argocd-server (`gpg.argoproj.io/*` flags) or the `gpg.argoproj.io/verification: "true"` annotation on the Application. The public keys corresponding to the keyIDs must be installed via `argocd gpg add` or the `argocd-gpg-keys-cm` ConfigMap.

Real control, not theater -- unsigned or differently-signed commits are rejected at reconcile time with a clear error. Pair with branch protection rules that require signed commits, and you close the loop from developer laptop to cluster.

### Global vs Project-Scoped RBAC

RBAC in ArgoCD uses Casbin, a policy engine that reads a CSV-like file. The global policy lives in the `argocd-rbac-cm` ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly                  # What unmapped users get
  policy.csv: |
    # p = policy, g = group mapping
    # p, <subject>, <resource>, <action>, <object>, <effect>
    p, role:platform-admin, applications, *, */*, allow
    p, role:platform-admin, projects, *, *, allow
    p, role:platform-admin, clusters, *, *, allow
    p, role:platform-admin, repositories, *, *, allow

    p, role:auditor, applications, get, */*, allow
    p, role:auditor, logs, get, */*, allow

    # Map OIDC group claims to roles
    g, acme:platform-team, role:platform-admin
    g, acme:security-audit, role:auditor
  scopes: '[groups, email]'                      # Which OIDC claims to read
```

Policy grammar: `p, <subject>, <resource>, <action>, <object>, <effect>`:

- **subject**: a role (`role:admin`), a built-in role (`role:readonly`, `role:admin`), or a project-scoped role (`proj:team-a:deploy`).
- **resource**: `applications`, `clusters`, `projects`, `repositories`, `certificates`, `accounts`, `gpgkeys`, `logs`, `exec`.
- **action**: `get`, `create`, `update`, `delete`, `sync`, `override`, `action/*` (custom Lua actions).
- **object**: for `applications`, the shape is `<project>/<app-name>`, with `*` wildcards. For `clusters`, it's the cluster URL or name. For `projects`, the project name.
- **effect**: `allow` or `deny`. Deny wins over Allow.

**Built-in roles**: `role:admin` (everything), `role:readonly` (view everything but no mutations). Every non-admin starts from `role:readonly` via `policy.default`, then gets additional grants through group mappings.

### Project-Scoped Roles

Roles inside an AppProject are automatically scoped to that project. The earlier example's `proj:team-a:deploy` role is implicitly only usable on applications in `team-a/*`. This is the **tenant boundary** -- a compromised team-a role cannot touch team-b Applications even if its policies say `applications, *, */*`.

### JWT Tokens for CI

Humans log in via SSO. Robots (CI pipelines) need a bearer token:

```bash
# Create a token for the "ci" role in the "team-a" project
argocd proj role create-token team-a ci --id team-a-ci-gh-actions
# Prints a JWT; store in GitHub Actions secret

# In CI:
argocd app sync team-a-api --auth-token "$ARGOCD_JWT" --server argocd.acme.com --grpc-web
```

The token is a JWT signed by ArgoCD's `argocd-secret`, scoped to the role's policies. **Tokens can have an expiry (`--expires-in 90d`)** -- default is never-expire, which is a bad idea. Rotate by creating a new token, updating the CI secret, then `argocd proj role delete-token team-a ci <id>` for the old one.

### Dex vs Direct OIDC

ArgoCD supports two SSO models:

**Dex** (bundled): a proxy that speaks OIDC to ArgoCD and translates to upstream IdPs that may not speak OIDC (SAML, LDAP, GitHub OAuth classic, Google Workspace). Configured in `argocd-cm`:

```yaml
# argocd-cm ConfigMap
data:
  dex.config: |
    connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: $dex.github.clientID        # References argocd-secret
          clientSecret: $dex.github.clientSecret
          orgs:
            - name: acme
```

**Direct OIDC**: skip Dex entirely; point ArgoCD at an IdP that speaks OIDC natively (Okta, Auth0, Keycloak, Google, Azure AD). Configured in `argocd-cm`:

```yaml
data:
  url: https://argocd.acme.com
  oidc.config: |
    name: Okta
    issuer: https://acme.okta.com
    clientID: $oidc.okta.clientID
    clientSecret: $oidc.okta.clientSecret
    requestedScopes: [openid, profile, email, groups]
    requestedIDTokenClaims:
      groups:
        essential: true
```

**Pick direct OIDC when your IdP speaks OIDC** -- it's simpler, removes the Dex pod, and has fewer moving parts. Pick Dex when you need SAML, LDAP, GitHub classic OAuth, or multi-IdP federation (Dex can chain connectors).

### The Group-to-Role Bridge

The magic of SSO + AppProject RBAC is that your identity provider's group membership becomes your ArgoCD authorization. A user in the `acme:team-a-devs` OIDC group automatically gets the `proj:team-a:deploy` role via `g, acme:team-a-devs, proj:team-a:deploy`. Revoking their team membership in Okta revokes their ArgoCD access at next login. No per-user config in ArgoCD.

**The end-to-end trace of one login**:

1. User clicks "Log in with Okta" on the argocd-server UI.
2. Okta authenticates, returns an ID token containing `"groups": ["acme:platform-admins", "acme:team-a-devs"]`.
3. argocd-server parses the token, extracts the `groups` claim (it knows to look for it because `argocd-cm`'s `oidc.config.requestedScopes` includes `groups` and `scopes: '[groups, email]'` in `argocd-rbac-cm` confirms it).
4. Casbin evaluates the user's effective roles: `g, acme:platform-admins, role:platform-admin` and `g, acme:team-a-devs, proj:team-a:deploy` both match; the user gets both roles.
5. Every subsequent UI click or API call checks the user's roles against `p, <role>, <resource>, <action>, <object>` policies. Matching `allow` lines grant; matching `deny` lines (or no match with default `role:readonly`) deny.

The user's identity is never stored in ArgoCD. Removing them from the Okta group revokes access at next token refresh (typically 1 hour).

---

## Part 5: Secrets in GitOps -- Three Pouches, Three Trust Models

The hard constraint: **plaintext secrets must never sit in Git.** Even in a "private" repo, you end up with secrets in backups, laptops, forks, screenshots, and CI logs. The three canonical solutions encrypt the secret before it enters Git, then decrypt it in-cluster.

> **Read the compliance rule carefully before picking a tool.** A rule that says *"any secret committed to Git must be auditable"* is a **conditional** -- it's triggered only if ciphertext lives in Git. ESO commits nothing to Git, so the precondition is never met and CloudTrail on the upstream vault becomes the audit surface. A rule that says *"all secrets must be inspectable in our Git history"* is a **requirement** -- it categorically excludes ESO and forces Sealed Secrets or SOPS. One flipped word changes the architecture.

### Decision Matrix at a Glance

| Approach | Trust Model | Where the key lives | When to pick |
|----------|-------------|---------------------|--------------|
| **Sealed Secrets** | Public-key sealed; only the destination cluster can unseal | Private key in-cluster; public key distributed to developers | Single cluster, simple ops, no existing secret manager |
| **External Secrets Operator (ESO)** | Secrets live in AWS/Vault/GCP/Azure; cluster pulls at runtime | In the external secret manager | Multi-cloud, multi-cluster, rotation-heavy, existing Secrets Manager investment |
| **SOPS** | Per-file encryption with KMS or age keys | KMS (cloud) or age keys (local) | GitHub-visible encrypted secrets; local dev parity; multi-team per-file keys |

### Sealed Secrets -- Tamper-Evident Diplomatic Pouch

**The analogy**: you have a letter for "the cluster." You put it in a pouch. The pouch has a lock that anyone can seal but only the cluster controller has the key to open. The pouch is **transparent** -- you can see the letter's shape, length, who it's addressed to, even the metadata around it -- but only the cluster can read the ink. The sealed pouch is safe to mail through any unsecure channel, including pasting the ciphertext into a public Git repo.

**How it works**:

1. Install the `sealed-secrets-controller` in the cluster. It generates a private RSA key, stored as a Secret in the `kube-system` (or `sealed-secrets`) namespace. The public key is exposed.
2. Developers fetch the public key: `kubeseal --fetch-cert > pub.pem`.
3. Developer creates a normal Secret YAML locally, then encrypts it: `kubeseal --cert pub.pem -f secret.yaml -o yaml > sealed-secret.yaml`.
4. Commit the `SealedSecret` CR to Git.
5. ArgoCD applies the `SealedSecret` to the cluster. The controller sees it, decrypts with its private key, and creates the corresponding native `Secret` object.

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: payments-db-credentials
  namespace: team-a
spec:
  encryptedData:
    DB_PASSWORD: AgBy8hCF...                     # Ciphertext, safe to commit
  template:
    metadata:
      name: payments-db-credentials
      namespace: team-a
    type: Opaque
```

**Scoping**: each SealedSecret is encrypted with a scope that binds it to a specific destination:

- **strict (default)**: tied to namespace + name. Moving it to a different name or namespace won't decrypt.
- **namespace-wide** (`--scope namespace-wide`): any name in the same namespace.
- **cluster-wide** (`--scope cluster-wide`): any name in any namespace.

Strict is safest. Cluster-wide is convenient but removes the namespace boundary as a safety net.

**Key rotation**: the controller rotates its sealing key every 30 days by default. Old keys are kept for decryption of existing SealedSecrets. To force a rotation, delete the Secret holding the sealing key; the controller regenerates. You must re-seal any new secrets with the new public key, but existing SealedSecrets stay decryptable.

**Disaster recovery -- the critical caveat**: the sealing key Secret is **the entire security model**. Lose it (cluster destroyed, Secret deleted without backup) and every SealedSecret in Git becomes permanent ciphertext. **You must back up the sealing key** (Velero, offsite encrypted storage, something) or you lose every encrypted secret in your Git history. This is the single biggest operational gotcha with Sealed Secrets and why larger installations prefer ESO.

### External Secrets Operator (ESO) -- The Just-In-Time Vending Machine

**The analogy**: ESO is a just-in-time secret vending machine. The app doesn't hold any key. The operator walks up to the vault with an IAM badge (IRSA on EKS, workload identity on GKE, managed identity on AKS), fetches the secret fresh every refresh interval, and drops it into a native Kubernetes Secret that the pod mounts like any other. The app never knows a vault exists -- from its perspective it's reading an ordinary Secret. The operator is the employee with the badge; the pod is the customer who just takes the snack out of the tray.

**CRDs**:

- **SecretStore / ClusterSecretStore**: how to talk to the external provider (AWS region, IAM role via IRSA, Vault endpoint, etc.).
- **ExternalSecret**: which keys to pull from the store and how to shape them into a Kubernetes Secret.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-operator
            namespace: external-secrets          # Uses IRSA
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payments-db-credentials
  namespace: team-a
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payments-db-credentials                # Creates a native Secret with this name
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/payments/db
        property: password
    - secretKey: DB_USERNAME
      remoteRef:
        key: prod/payments/db
        property: username
```

**Providers**: AWS Secrets Manager, AWS Systems Manager Parameter Store, HashiCorp Vault, GCP Secret Manager, Azure Key Vault, Akeyless, Doppler, and a dozen others.

**Template transformations**: ESO can template the final Secret shape, e.g., combine multiple remote values into one connection string.

**Refresh cost**: `refreshInterval: 1m` on 1000 ExternalSecrets = 1000 API calls per minute. AWS Secrets Manager charges per API call. At scale, bump the interval to 1h or 24h and trigger immediate refresh via webhook or manual reconcile when you rotate a secret intentionally.

**Multi-cloud / multi-cluster**: ESO is the clear winner. One Secrets Manager per region, one ClusterSecretStore per cluster, and every cluster gets the same secrets without re-sealing N times for N clusters (the Sealed Secrets pain point).

**The recursive-failure gotcha for emergency-response secrets**: if ESO is down **and** a workload needing the PagerDuty/OpsGenie API key has never successfully fetched it (no live Kubernetes Secret materialized yet), you've built a recursive dependency -- you can't page the ESO on-call because the key was never mounted. Bootstrap alerting and monitoring Secrets at cluster creation time and keep them warm; don't let them materialize only on first workload deploy. The general principle: design ESO integrations for **stale-but-available**, not **missing**. A last-known-good Secret that's a few hours out of date still lets incident response page humans; a Secret that doesn't exist locks you out of your own break-glass.

### SOPS -- Sealed Envelopes in the Folder

**The analogy**: the manila folder (Git repo) is not sealed, but every sensitive document inside it is individually sealed. Different envelopes (different YAML files or different keys within one file) can be sealed to different people. You can still `grep` the folder for structure; you just can't read the secret values.

**How it works**:

1. Define a `.sops.yaml` in the repo root specifying encryption rules.
2. Run `sops -e secret.yaml > secret.enc.yaml` to encrypt.
3. Commit the encrypted file. Only the `data` and `stringData` values are encrypted; keys, metadata, and structure stay readable.
4. In ArgoCD, use a Config Management Plugin (CMP) or `helm-secrets` / `KSOPS` to decrypt at render time.

```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/prod/.*\.yaml
    kms: "arn:aws:kms:us-east-1:111122223333:alias/sops-prod"
  - path_regex: secrets/dev/.*\.yaml
    age: "age1abc...def"                         # age key for dev (local-friendly)
```

```yaml
# secrets/prod/payments-db.yaml (after sops encryption)
apiVersion: v1
kind: Secret
metadata:
  name: payments-db
data:
  password: ENC[AES256_GCM,data:...,iv:...,tag:...]
sops:
  kms:
    - arn: arn:aws:kms:us-east-1:111122223333:alias/sops-prod
  lastmodified: "2026-04-17T10:00:00Z"
  version: 3.8.1
```

**Keys**:

- **age keys**: simple public/private pairs, great for local dev and small teams; no cloud dependency.
- **KMS keys**: cloud HSM-backed, supports IAM-gated decryption, the right choice for production.

A single file can be encrypted to multiple recipients (multiple KMS keys + multiple age keys) -- the team lead can decrypt with their KMS role, the SRE can decrypt with theirs, no shared secret.

**ArgoCD integration**: ArgoCD doesn't support SOPS natively. You need a Config Management Plugin (CMP) image with `sops` and `helm-secrets` or `KSOPS` installed. The repo-server is configured to use the plugin for paths matching `*.sops.yaml`. The plugin decrypts at render time, pipes plaintext YAML to the Application Controller, and the Controller applies normally. The plaintext never hits disk.

**When SOPS wins**: when the Git repo is on GitHub.com (public or org-visible) and you want encrypted secrets visible as diffs during PR review, when your team culture values local reproducibility (`sops -d` on a laptop with the right KMS access reads the same values the cluster does), or when you're already using SOPS outside Kubernetes (Terraform, Ansible) and want one tool across the stack.

---

## Part 6: Notifications -- The Dispatch Radio

**The analogy**: the foreman's dispatch radio. Sync-failure events get broadcast to `#alerts`. Prod health-degraded ALSO pages the on-call engineer via PagerDuty. Routine deploys post a short update to `#deploys`. Each message is templated per channel so the listener gets the right level of detail -- Slack gets a thread, PagerDuty gets severity+runbook, email gets the full context. The dispatch is 1:N (one event, many channels) and per-service templated.

Since ArgoCD 2.0 the notifications controller is bundled; no separate install needed. Four concepts:

- **Services** -- where notifications go (Slack, email, Teams, PagerDuty, webhook, Grafana, GitHub). Configured in `argocd-notifications-cm`.
- **Triggers** -- CEL-like conditions on the Application object. When the condition flips from false to true, the trigger fires.
- **Templates** -- the message body. Go text/template with access to `.app` (the Application object) and `.context` (notifications config).
- **Subscriptions** -- annotations on Application or AppProject binding trigger + template + service + destination.

```yaml
# argocd-notifications-cm ConfigMap
data:
  service.slack: |                                # Service: how to talk to Slack
    token: $slack-token                           # Reference to a key in argocd-notifications-secret
  service.pagerduty: |
    token: $pagerduty-token
  service.webhook.grafana: |
    url: https://grafana.acme.com/api/annotations
    headers:
      - name: Authorization
        value: Bearer $grafana-token

  trigger.on-sync-failed: |                       # Trigger: named condition
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]                     # Template name(s) to render
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.sync.status == 'Synced'
      send: [app-sync-succeeded]

  template.app-sync-failed: |                     # Template: message body
    message: |
      App {{.app.metadata.name}} sync FAILED at {{.app.status.operationState.finishedAt}}.
      Repo: {{.app.spec.source.repoURL}}
      Revision: {{.app.status.operationState.syncResult.revision}}
      Error: {{.app.status.operationState.message}}
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name }} -- sync failed",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color":"#E96D76",
          "fields": [
            { "title": "Project", "value": "{{.app.spec.project}}", "short": true },
            { "title": "Revision", "value": "{{.app.status.operationState.syncResult.revision}}", "short": true }
          ]
        }]
```

**Subscriptions** live on the Application or AppProject:

```yaml
# On the Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts-platform-prod
    notifications.argoproj.io/subscribe.on-health-degraded.slack: alerts-platform-prod
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: deploys-platform

# On the AppProject (inherited by all Applications in the project)
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts-team-a
```

The annotation syntax: `notifications.argoproj.io/subscribe.<trigger>.<service>: <destination>`. Multiple subscriptions stack. AppProject subscriptions are inherited by every Application in the project (but an Application's own subscription overrides for the same trigger+service pair).

**Default triggers shipped with ArgoCD**: `on-deployed`, `on-health-degraded`, `on-sync-failed`, `on-sync-running`, `on-sync-status-unknown`, `on-sync-succeeded`. Custom triggers are just additional entries in the ConfigMap.

**The `groupingKey` trick**: Slack messages with the same `groupingKey` are threaded into the same Slack thread, so repeated sync events for the same Application don't flood #alerts with separate top-level messages. Set it to the Application name.

**Production pattern**: one service per severity tier (slack-deploys for info, slack-alerts for sync failures, pagerduty for health-degraded in prod), subscribe AppProjects to the low-severity streams, subscribe individual critical Applications additionally to PagerDuty. Lean on AppProject inheritance so you're not annotating every Application by hand.

---

## Part 7: Resource Hooks -- Beyond the Fundamentals

Yesterday covered the basic hook model: `argocd.argoproj.io/hook: PreSync` on a Job, delete policies, participation in wave gating. Today's additions focus on hooks **inside ApplicationSet-templated Applications** and **cluster-specific pre-sync work**.

### Pre-Sync Migration on a Specific Cluster Only

The scenario: you have an ApplicationSet generating one Application per prod cluster (via Cluster generator). Each generated Application has a PreSync DB migration hook. But migrations must run **once globally**, not once per cluster -- the database is a cross-region Aurora Global Database and migrations on the secondary region would conflict.

The fix: **gate the hook manifest inside the template on a generator value**, so the hook only renders in the primary cluster's Application:

```yaml
template:
  spec:
    source:
      repoURL: https://github.com/acme/payments-api.git
      path: chart
      helm:
        valuesObject:
          isPrimaryRegion: "{{ .values.isPrimary }}"  # From Cluster generator values
```

Chart's `templates/migrate.yaml`:

```yaml
{{- if eq .Values.isPrimaryRegion "true" }}
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded,BeforeHookCreation
spec: ...
{{- end }}
```

Secondary region Applications render an empty hook set; only the primary runs migrations. This pattern -- **gate the hook at chart render time, not at ArgoCD sync time** -- keeps the logic in Helm rather than fighting ArgoCD.

### Skip Hook for Selective Re-Runs

`argocd.argoproj.io/hook: Skip` on a resource tells ArgoCD to not apply that resource at all during sync. Useful for "this migration Job ran last release; don't re-run it just because the Deployment changed." Combine with a checksum annotation that you increment only when you want migrations to fire.

### Hook Delete Policies -- Keeping Failure Evidence

Default: `BeforeHookCreation` is applied implicitly if you set nothing. The most recent hook persists.

The production pattern for migrations: `HookSucceeded,BeforeHookCreation`:

- **HookSucceeded**: clean up successful hooks immediately -- no stale migration Jobs cluttering the namespace.
- **BeforeHookCreation**: if a migration fails, the failed Job stays until the next sync, where it's replaced (giving you pod logs to debug the failure).

**Anti-pattern**: `HookFailed` alone. This deletes failed hooks immediately, which means you lose the pod logs that would tell you why the migration failed. Keep failed evidence.

### Hooks Inside ApplicationSet-Templated Applications

Hooks in a chart rendered by an ApplicationSet-generated Application behave exactly like hooks in a hand-written Application. No special rules. The only wrinkle: if the ApplicationSet regenerates the Application (because a generator input changed), the Application's next sync re-evaluates hooks. If your `hook-delete-policy: BeforeHookCreation` logic relies on hook persistence between syncs, make sure the ApplicationSet isn't thrashing the Application spec unnecessarily (stabilize generator outputs; avoid non-deterministic values like timestamps in template rendering).

### SyncFail Hook Tied to Cluster Generator Values

When an ApplicationSet uses the Cluster generator, each generated Application knows which cluster it was generated for -- the cluster name comes in as `{{ .name }}` at template time. Bake that value into a SyncFail hook that posts a cluster-specific Slack message:

```yaml
# ApplicationSet with Cluster generator
template:
  spec:
    source:
      helm:
        valuesObject:
          clusterName: "{{ .name }}"           # Lands in the chart's values
          clusterServer: "{{ .server }}"
```

In the Helm chart, a SyncFail hook Job:

```yaml
# chart/templates/syncfail-notify.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: syncfail-notify
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded,BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: notify
          image: curlimages/curl:8.6.0
          command: ["sh", "-c"]
          args:
            - |
              curl -X POST -H 'Content-Type: application/json' \
                -d "{\"text\": \"Sync failed on cluster {{ .Values.clusterName }} for release {{ .Release.Name }}\"}" \
                "$SLACK_WEBHOOK_URL"
          env:
            - name: SLACK_WEBHOOK_URL
              valueFrom:
                secretKeyRef:
                  name: slack-webhook
                  key: url
```

Result: each generated Application carries its own cluster identity into its hook payload, so the Slack message says "Sync failed on `prod-eu-west-1`" rather than a generic "a sync failed." This is the ApplicationSet-aware version of ArgoCD notifications -- useful when you want failure details tied to cluster context before argocd-notifications fires its own trigger.

### PostSync Notifications Interpolating Generator Values

Any generator parameter that reaches the template (via `values:` on Cluster generator, or as keys from a Git files generator) can be passed through to chart values and surfaced in a hook. The PostSync pattern for deploy announcements:

```yaml
template:
  spec:
    source:
      helm:
        valuesObject:
          environment: "{{ .values.environment }}"        # Cluster generator values
          appPath: "{{ .path.basename }}"                  # Git directory generator
          deployTarget: "{{ .name }}-{{ .path.basename }}" # Composed identifier
```

A PostSync Job reads `.Values.environment`, `.Values.appPath`, `.Values.deployTarget` and posts a Slack message like "Deployed `payments-api` to `prod-us-east-1` (env=prod)". The hook runs once per Application, which -- because the ApplicationSet spawned one Application per (cluster x app) pair -- gives you per-target deploy notifications without touching argocd-notifications.

Anti-pattern: stuffing timestamps or other non-deterministic values into generator inputs so they end up in the hook. Every regenerate of the ApplicationSet then thrashes the Application spec and re-triggers the PostSync. Keep generator values stable and derived from Git/cluster state.

### Cross-Wave Interaction: App-of-Apps + ApplicationSet

A common layout: a root App-of-Apps Application manages an ApplicationSet as one of its children. The root's child sync waves order the AppSet's creation relative to other children -- but the AppSet's **downstream Applications** run their sync waves independently.

```
root (App-of-Apps)
  |-- sync-wave: -10  cert-manager                (child Application)
  |-- sync-wave:  -5  ingress-nginx               (child Application)
  |-- sync-wave:   0  team-a-apps (ApplicationSet CR)
                        |-- generated Application: team-a-api
                        |     |-- PreSync: db-migrate  (wave -1)
                        |     |-- Deployment         (wave 0)
                        |     |-- PostSync: warm-cache(wave 1)
                        |-- generated Application: team-a-worker
```

The root's wave 0 only gates **the creation of the ApplicationSet CR itself**. Once the AppSet exists, it generates its own Applications, each of which reconciles its own sync waves against its own chart's resources. The root does not and cannot block on the downstream Applications' internal hook ordering -- those are separate reconciliation loops owned by the Application Controller, not the root.

The practical implication: **don't try to sequence work across the boundary with sync waves.** If team-a-api's migration must run after platform-addons are ready, encode that dependency in platform-addons' health gating (cert-manager doesn't report Healthy until its webhook is ready, ingress-nginx doesn't report Healthy until its IngressClass is accepted), not in sync-wave arithmetic that tries to span both layers.

---

## Part 8: Production Gotchas (Advanced-Patterns Edition)

**1. App-of-Apps child drift cascades through the root.**
A manual edit to a child Application's spec is reverted by the root's self-heal reconciliation, not just by the child's own self-heal. The tug-of-war surfaces as "my hotfix to the child lasted 30 seconds." Treat children as fully Git-managed; never edit them directly.

**2. `preserveResourcesOnDeletion: true` on ApplicationSet means "orphan everything when the AppSet is deleted."**
Tempting flag to prevent cascading deletes during refactors. In practice, it means the managed Applications stay around after the ApplicationSet is gone, still reconciling their workloads, with no controlling parent. You now have orphan Applications that future AppSets can't take ownership of without manual cleanup. Use only during migrations; unset afterward.

**3. `goTemplate: false` silently accepts typos.**
See the Templates section above: without `goTemplate: true` + `goTemplateOptions: ["missingkey=error"]`, a typo renders as empty string and ships a broken Application. Always enable both; this is not optional.

**4. Cluster generator selector + `in-cluster` asymmetry.**
The selector matches cluster Secret `metadata.labels`, not any Application-side label. Two distinct cases bite here: (a) a `selector: {}` (empty) includes the synthetic `in-cluster` entry for `https://kubernetes.default.svc`; (b) any non-empty `matchLabels` **excludes** `in-cluster` because the synthetic entry has no labels to match -- unless you create a real cluster Secret for `https://kubernetes.default.svc` with labels matching the selector. Label cluster Secrets consistently at registration time (environment, region, tier), and if you need the hub in the output, register it as a real Secret.

**5. Sealed Secrets controller sealing key loss = total, permanent loss.**
If the sealing key Secret is deleted and never restored, every SealedSecret in Git becomes permanently unreadable. **Back up the sealing key offsite**. This is by far the most common Sealed Secrets disaster.

**6. ESO `refreshInterval` at scale is expensive.**
AWS Secrets Manager is priced per API call. 1000 ExternalSecrets at `refreshInterval: 1m` = $1,440/month in API calls alone. At scale, use `refreshInterval: 24h` with webhook-triggered refresh for rotation events, not polling.

**7. Destinations by `server` URL break on cluster rebuild.**
When EKS is rebuilt, the API URL changes. Every Application with `destination.server: https://OLD-URL` is now invalid. Use `destination.name: prod-us-east-1` instead -- the friendly name can survive cluster rebuilds by pointing the updated cluster Secret at the new URL under the same name.

**8. RBAC hygiene -- tokens, source-repo match semantics, and destination references.**
Three traps that usually bite together:
- **JWT project tokens never expire by default.** `argocd proj role create-token` without `--expires-in` produces a never-expiring JWT. Leaked tokens stay valid forever. Always set an expiry (90 days is a reasonable default) and rotate via CI.
- **AppProject `sourceRepos` uses exact match unless you glob.** `sourceRepos: [https://github.com/acme/team-a]` does not match `https://github.com/acme/team-a.git` (with the `.git` suffix). Normalize repo URLs at registration or use explicit globbing (`https://github.com/acme/team-a*`).
- **Destination name vs server references are not interchangeable.** A project that lists `{ name: prod-us-east-1 }` does not authorize an Application that specifies `{ server: https://EABC.eks...amazonaws.com }` even if they point at the same cluster. Pick one convention per project and stick to it.

**9. Notification subscriptions on AppProject don't inherit to Applications if the Application has ANY subscription for the same trigger+service pair.**
AppProject subscriptions are inherited unless overridden. If you annotate `team-a` AppProject with `subscribe.on-sync-failed.slack: alerts-team-a`, every Application inherits that -- except for Applications that have their own `subscribe.on-sync-failed.slack: ANY-CHANNEL` annotation, which fully replaces the inherited one for that trigger+service. Watch for partial overrides that silently drop the project-level default.

**10. `ARGOCD_CONTROLLER_REPLICAS` must match the StatefulSet's `.spec.replicas` (pre-2.8).**
The logical shard count lives in the `ARGOCD_CONTROLLER_REPLICAS` env var on the `argocd-application-controller` container, not in `argocd-cmd-params-cm`. It must match the StatefulSet's actual pod count. Mismatch = some clusters go unassigned (shards with no pod) or pods idle (pods with no shard). On ArgoCD 2.8+ with `ARGOCD_ENABLE_DYNAMIC_CLUSTER_DISTRIBUTION=true`, the controller reads replicas from its own StatefulSet automatically and the env var isn't required -- prefer that mode on modern installs.

**11. Matrix generator cardinality explosion.**
A 3-cluster x 10-app x 3-env matrix is **90 Applications**. Innocent generator changes can multiply into thousands of Applications. Use `argocd appset generate` or a dry-run flag to preview output cardinality before applying ApplicationSet changes.

**12. Pull Request generator cleanup fails when namespaces stick.**
On PR merge, the generated Application is removed and its `resources-finalizer.argocd.argoproj.io` cascades deletion. If the preview namespace has any custom finalizer (webhooks, operators) blocking deletion, the Application stays stuck in `Terminating`, the ApplicationSet thinks the Application still exists, and re-opening the PR won't recreate it. Unique per-PR namespaces + a periodic audit of stuck preview namespaces is the mitigation.

**13. SCM Provider generator rate limits GitHub on fast requeue.**
`requeueAfterSeconds: 60` on a GitHub org with 500 repos = 500 API calls per minute, easily blowing the personal access token's rate limit. Use a GitHub App (higher rate limit), increase requeue to 15-30 minutes for production, and subscribe to webhook notifications for near-real-time updates.

**14. `preservedFields` preserves metadata only, not spec.**
`preservedFields` supports `annotations` and `labels` in `metadata`. That's it. Spec-field preservation (e.g., pinning `spec.source.targetRevision` via preservedFields) is **not supported** -- don't architect your hotfix workflow around it. For spec-level on-call overrides, use `ignoreApplicationDifferences` with JSON pointers, or delete-the-AppSet-with-`preserveResourcesOnDeletion`-then-edit-then-rebuild, or route the pin through a generator input (e.g., a List element holding the hotfix SHA).

**15. RollingSync step matching uses labels on the generated Applications, not on the generators.**
A step `matchExpressions: [{key: env, operator: In, values: [prod]}]` matches generated Applications that have a label `env=prod`. Your template must **set that label** on the generated Applications, typically from a generator parameter. Forget to set the label and the step matches nothing, progressing instantly.

**16. Dex connector changes require a Dex pod restart.**
Editing `argocd-cm` to change the Dex OIDC/GitHub connector config does not hot-reload. The Dex pod caches its config at start. `kubectl rollout restart deployment/argocd-dex-server -n argocd` after every connector change.

**17. Sharding changes trigger a full cluster cache rebuild on migrated shards.**
Switching algorithms or changing replica counts reshuffles cluster-to-shard assignments. Every reshuffled cluster has its live state cache rebuilt, which means the controller re-lists every resource on that cluster. At scale, this hammers the target API servers. Make sharding changes in a maintenance window -- and prefer `consistent-hashing` to minimize the blast radius of future changes.

**18. `ignoreDifferences` at the AppProject level propagates to every Application in the project.**
Rarely what you want. Prefer per-Application `spec.ignoreDifferences`. Use project-level only when the exception is a true tenant-wide rule (e.g., "this team's charts always use HPA; ignore `.spec.replicas` on every Deployment").

**19. ApplicationSet Git generators poll on a requeue interval (default 3 min).**
A commit to a watched Git path doesn't spawn or update Applications until the next ApplicationSet poll (`requeueAfterSeconds`, default ~180s). The resulting Application then waits on its own Application-level refresh (~3 min default) before it picks up its own Git content. End-to-end latency: **up to ~6 minutes from commit to live workload**. Mitigation: configure Git webhooks on the argocd-applicationset-controller to trigger immediate ApplicationSet regeneration (reduces the first 3 min), plus webhooks on the argocd-server (reduces the second 3 min). Webhooks collapse both hops to seconds; polling stays as the safety net.

---

## Part 9: Production Terraform

The pragmatic pattern: Terraform bootstraps ArgoCD itself and registers clusters, then ArgoCD (via App-of-Apps) takes over everything else.

### Registering a Spoke Cluster

```hcl
resource "kubernetes_secret" "cluster_prod_us_east_1" {
  metadata {
    name      = "cluster-prod-us-east-1"
    namespace = "argocd"
    labels = {
      "argocd.argoproj.io/secret-type" = "cluster"
      "environment"                    = "prod"
      "region"                         = "us-east-1"
      "tier"                           = "production"
    }
  }
  type = "Opaque"

  # Use string_data (plaintext) not data (which expects base64-encoded values).
  # The Kubernetes provider base64-encodes string_data for you; using `data` here
  # would require you to manually base64 each value.
  string_data = {
    name   = "prod-us-east-1"
    server = module.prod_us_east_1_cluster.cluster_endpoint   # terraform-aws-modules/eks/aws output
    config = jsonencode({
      awsAuthConfig = {
        clusterName = module.prod_us_east_1_cluster.cluster_name
        roleARN     = aws_iam_role.argocd_cross_account.arn
      }
      tlsClientConfig = {
        insecure = false
        caData   = module.prod_us_east_1_cluster.cluster_certificate_authority_data
      }
    })
  }
}
```

The EKS module outputs used above match the standard `terraform-aws-modules/eks/aws` v19+ schema: `cluster_endpoint` (the API URL) and `cluster_certificate_authority_data` (the base64 CA bundle). Older module versions and hand-rolled EKS resources use different names (`endpoint`, `certificate_authority[0].data`) -- always verify against your module's current outputs.

### AppProject

```hcl
resource "kubernetes_manifest" "appproject_team_a" {
  manifest = {
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "AppProject"
    metadata = {
      name      = "team-a"
      namespace = "argocd"
      finalizers = ["resources-finalizer.argocd.argoproj.io"]
    }
    spec = {
      description = "Team A's applications"
      sourceRepos = [
        "https://github.com/acme/team-a.git",
        "https://github.com/acme/shared-charts.git",
      ]
      destinations = [
        { name = "prod-us-east-1", namespace = "team-a" },
        { name = "prod-eu-west-1", namespace = "team-a" },
        { name = "staging",        namespace = "team-a" },
      ]
      clusterResourceWhitelist = [
        { group = "", kind = "Namespace" },
      ]
      namespaceResourceBlacklist = [
        { group = "", kind = "ResourceQuota" },
      ]
      roles = [
        {
          name = "deploy"
          policies = [
            "p, proj:team-a:deploy, applications, sync, team-a/*, allow",
            "p, proj:team-a:deploy, applications, get, team-a/*, allow",
          ]
          groups = ["acme:team-a-devs"]
        }
      ]
    }
  }
}
```

### ApplicationSet with Git Generator

```hcl
resource "kubernetes_manifest" "appset_team_a" {
  manifest = {
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "ApplicationSet"
    metadata   = { name = "team-a-apps", namespace = "argocd" }
    spec = {
      goTemplate        = true
      goTemplateOptions = ["missingkey=error"]
      generators = [{
        git = {
          repoURL  = "https://github.com/acme/team-a.git"
          revision = "main"
          directories = [{ path = "apps/*" }]
        }
      }]
      template = {
        metadata = {
          name       = "team-a-{{ `{{ .path.basenameNormalized }}` }}"
          namespace  = "argocd"
          finalizers = ["resources-finalizer.argocd.argoproj.io"]
        }
        spec = {
          project = "team-a"
          source = {
            repoURL        = "https://github.com/acme/team-a.git"
            path           = "{{ `{{ .path.path }}` }}"
            targetRevision = "main"
          }
          destination = {
            name      = "prod-us-east-1"
            namespace = "team-a"
          }
          syncPolicy = {
            automated   = { prune = true, selfHeal = true }
            syncOptions = ["CreateNamespace=true"]
          }
        }
      }
    }
  }
}
```

Note the `{{ ` ... ` }}` escaping -- Terraform's own templating would eat the double-braces otherwise. Use `yamlencode` or heredoc with proper escaping when the nesting gets ugly.

### Notifications ConfigMap

```hcl
resource "kubernetes_config_map" "argocd_notifications_cm" {
  metadata {
    name      = "argocd-notifications-cm"
    namespace = "argocd"
  }
  data = {
    "service.slack" = yamlencode({
      token = "$slack-token"
    })
    "trigger.on-sync-failed" = yamlencode([{
      when = "app.status.operationState.phase in ['Error', 'Failed']"
      send = ["app-sync-failed"]
    }])
    "template.app-sync-failed" = yamlencode({
      message = "App {{.app.metadata.name}} sync FAILED at {{.app.status.operationState.finishedAt}}."
      slack = {
        attachments = jsonencode([{
          title = "{{.app.metadata.name}} -- sync failed"
          color = "#E96D76"
        }])
      }
    })
  }
}

resource "kubernetes_secret" "argocd_notifications_secret" {
  metadata {
    name      = "argocd-notifications-secret"
    namespace = "argocd"
  }
  data = {
    "slack-token" = var.slack_bot_token
  }
}
```

There's also a dedicated **ArgoCD Terraform provider** (`oboukili/argocd`) that speaks directly to the ArgoCD API rather than applying Kubernetes manifests. Use it for `argocd_application`, `argocd_project`, `argocd_repository`, `argocd_cluster` resources when you want ArgoCD's admission validation on every plan. The trade-off is a second API dependency in your Terraform plan.

**Two-stage bootstrap**: `kubernetes_manifest` resources for ArgoCD CRDs (AppProject, Application, ApplicationSet) require the CRDs to already exist at **plan time** -- the provider looks up the CRD schema during plan, not during apply. This forces a two-stage bootstrap: (1) first `terraform apply` installs ArgoCD (Helm release or raw manifests), which registers the CRDs with the API server; (2) second `terraform apply` creates AppProjects and ApplicationSets. In a single monolithic state, this manifests as "the first apply fails on the manifest resources because CRDs don't exist yet" -- targeted applies (`-target=helm_release.argocd`) or split workspaces are the mitigation. The `oboukili/argocd` provider sidesteps this because it talks to ArgoCD's API rather than the Kubernetes API, but still needs ArgoCD running first.

---

## When-to-Use-What -- Advanced Decision Frameworks

### App of Apps vs ApplicationSet vs Nested

```
Small static set (< 20 Apps), manually curated, one cluster
  -> App of Apps (directory of static Application YAMLs)

Dynamic inventory -- new apps, new clusters, new PRs constantly
  -> ApplicationSet (generator produces the right shape)

Multi-team, per-team ownership of "their apps"
  -> Nested App of Apps (root owned by platform, children owned by teams)
      OR one ApplicationSet per team with Git-directories generator

Platform add-ons across every cluster
  -> ApplicationSet + Cluster generator (one App per cluster)

Platform add-ons across every cluster, one add-on per chart
  -> ApplicationSet + Matrix(Cluster x Git-directories) (one App per cluster-addon pair)

Preview environments per pull request
  -> ApplicationSet + Pull Request generator (one App per open PR)

Every repo in a GitHub org auto-adopted
  -> ApplicationSet + SCM Provider generator
```

### Sealed Secrets vs ESO vs SOPS

```
Single cluster, no existing secret manager, small team
  -> Sealed Secrets (simplest, zero external deps)

Multiple clusters, AWS/Vault already in play, rotation matters
  -> External Secrets Operator (secrets live where they already live)

Public/shared Git repos where encrypted secrets visible in diffs is a feature,
  or existing SOPS usage in Terraform/Ansible
  -> SOPS + KSOPS or helm-secrets (per-file keys, multi-recipient)

Need local developer parity ("sops -d on my laptop reads same values as the cluster")
  -> SOPS with age keys for dev + KMS keys for prod

Absolutely cannot lose secrets to a cluster rebuild
  -> ESO (secrets survive in the external store) OR Sealed Secrets with robust
     sealing-key backup. Never SOPS+age without key backup -- losing age keys
     is permanent.
```

### Standalone vs Hub-and-Spoke vs Cluster-per-Env

```
One cluster, one team
  -> Standalone ArgoCD in the cluster (no multi-cluster complexity needed)

Dozens of clusters, unified ops team
  -> Hub and spoke (one hub, many spokes registered via Cluster Secrets;
     consistent-hashing sharding once past 30 spokes)

Strict compliance boundary between prod and non-prod ("prod hub cannot even
  see dev workloads")
  -> Cluster per env (one hub per env; forfeit cross-env promotion in one UI)

Cross-region active-active where each region must survive the other's loss
  -> Hub per region (multi-hub with overlap); each hub reconciles its own
     region's spokes; Git is the shared source of truth
```

### Dex vs Direct OIDC

```
IdP speaks OIDC natively (Okta, Auth0, Keycloak, Google Workspace, Azure AD)
  -> Direct OIDC via argocd-cm oidc.config; skip Dex, simpler stack

IdP is SAML-only, LDAP, or GitHub classic OAuth
  -> Dex (it's the bridge)

Multi-IdP federation (contractors via one IdP, employees via another)
  -> Dex (chain connectors)

You want the minimum moving parts
  -> Direct OIDC, always, when possible
```

### Controller Sharding Algorithm

```
< 10 clusters
  -> Single replica. Don't shard prematurely.

Any multi-replica production setup (10+ clusters)
  -> consistent-hashing (or consistent-hashing-with-bounded-loads)
     Only algorithm that keeps most clusters on their shards when replicas scale;
     all others force cache rebuilds across the fleet

Predictable test environment with a frozen cluster list
  -> round-robin is acceptable for its in-order determinism; do NOT use in
     production -- it reshuffles most clusters on scale, similar to legacy

Legacy ArgoCD installs stuck on <2.8 for compatibility reasons
  -> "legacy" algorithm only as a bridge; upgrade and switch to consistent-hashing

On ArgoCD 2.8+
  -> Enable ARGOCD_ENABLE_DYNAMIC_CLUSTER_DISTRIBUTION=true so the controller
     auto-detects replica count; skip ARGOCD_CONTROLLER_REPLICAS entirely
```

---

## Key Takeaways

- **App of Apps is the bootstrapping pattern, not the scaling pattern.** One root Application pointing at a directory of children gets a cluster from zero to fully provisioned with one `kubectl apply`. It scales cleanly to ~50 Applications; past that, the lack of templating forces duplication. Nested App-of-Apps solves organizational separation (platform vs product teams); ApplicationSet solves repetition.

- **ApplicationSet is mail-merge for Applications.** One template plus one or more generators produces N Applications, reconciled continuously as generator inputs change. The seven generators -- List, Cluster, Git (directories), Git (files), Matrix, Merge, SCM Provider, Pull Request -- cover every dynamic-Application shape you'll meet. Pick the generator whose cardinality matches the problem.

- **Always `goTemplate: true` with `goTemplateOptions: ["missingkey=error"]`.** The legacy fasttemplate engine silently renders typos as empty strings and ships you broken Applications. The Go-template engine with strict missing-key error surfaces the typo at reconcile time and refuses to generate until it's fixed.

- **Matrix generator cardinality is multiplicative and dangerous.** Three clusters x ten apps x three envs is 90 Applications. Dry-run with `argocd appset generate` before applying anything that combines generators; review the output list, not just the ApplicationSet spec.

- **`preservedFields` is metadata-only (annotations + labels) -- not a hotfix escape hatch.** Spec-level on-call overrides need `ignoreApplicationDifferences` (JSON-pointer ignore rules), or the delete-AppSet-with-`preserveResourcesOnDeletion`-edit-child-rebuild dance, or routing the pin through a generator input. Design your hotfix workflow around those three options, never around pretending `preservedFields` can preserve spec fields. Never leave `preserveResourcesOnDeletion: true` on in production -- it orphans Applications on AppSet deletion.

- **A cluster Secret with the `argocd.argoproj.io/secret-type: cluster` label is the unit of multi-cluster.** Whether registered via `argocd cluster add` or directly via Terraform, the Secret is the source of truth. Labels on that Secret drive Cluster generator selectors. Use `destination.name` not `destination.server` so friendly names survive cluster rebuilds.

- **Hub and spoke scales with controller sharding.** One ArgoCD reconciling into N spokes is the dominant topology past three clusters. The application-controller's memory and API-watch load scale with (clusters x resources); sharding splits that load across replicas. Use `consistent-hashing` for any multi-replica production setup regardless of cluster count -- `round-robin` and `legacy` both force cache rebuilds on scale.

- **The StatefulSet's `.spec.replicas` and `ARGOCD_CONTROLLER_REPLICAS` env var must match (pre-2.8).** The env var configures the logical shard count; the StatefulSet replicas are the physical pods. Mismatch = orphaned shards (stuck clusters) or idle pods. On ArgoCD 2.8+, enable `ARGOCD_ENABLE_DYNAMIC_CLUSTER_DISTRIBUTION=true` and the controller auto-detects replicas -- the only knob becomes scaling the StatefulSet.

- **AppProject is the tenant boundary, not a UI organizer.** One Project per team or trust boundary, not one per environment. The Project restricts `sourceRepos`, `destinations`, resource kinds, and RBAC roles. The default Project is unscoped and should never hold production Applications. `clusterResourceWhitelist: []` is the defensive default (deny all cluster-scoped resources).

- **RBAC is Casbin in the `argocd-rbac-cm` ConfigMap.** Policies grant roles; `g,` lines map OIDC group claims to roles; builtin `role:admin` and `role:readonly` cover baseline; project-scoped roles (`proj:<name>:<role>`) are automatically scoped to their project. JWT tokens for CI should always have an expiry and be rotated.

- **Pick the secret strategy from the trust model, not from popularity.** Sealed Secrets for single-cluster simplicity (and back up the sealing key). ESO for multi-cluster + existing secret manager + rotation. SOPS for per-file encryption with multi-recipient keys and local dev parity. The decision hinges on where your secrets live today and how many clusters need them.

- **ESO's `refreshInterval` is a cost dial, not a UX dial.** One-minute polling on 1000 ExternalSecrets is a budget line item. Default to hours or days and use webhook-driven refresh on rotation events.

- **Notifications are triggers + templates + services + subscriptions.** Triggers are conditions; templates are message bodies; services are destinations; subscriptions are annotations binding them. Subscribe at the AppProject level to inherit across Applications; override at Application level for exceptions.

- **Hooks inside ApplicationSet-generated Applications work exactly like normal hooks -- the gotcha is hook persistence across ApplicationSet regenerations.** Stabilize generator outputs (no timestamps, no non-deterministic values) so the Application spec doesn't thrash and re-trigger hooks.

- **The full advanced stack: one hub ArgoCD, N spokes registered via labeled Cluster Secrets, platform Applications bootstrapped via App-of-Apps, workload Applications generated via ApplicationSet + Matrix(Cluster x Git), AppProjects scoping each team, OIDC group-to-role mapping for humans + JWT tokens for CI, ESO for secrets from AWS Secrets Manager, notifications to Slack on sync failures and PagerDuty on prod health-degraded events.** That stack scales from one cluster to a hundred without reshaping.

---

## Study Resources

Curated reading list for this topic: [`study-resources/kubernetes/argocd-advanced.md`](../../../study-resources/kubernetes/argocd-advanced.md)

**Key references**:
- [ArgoCD Cluster Bootstrapping (App of Apps)](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) -- canonical App-of-Apps reference with recurse, finalizers, sync waves
- [ArgoCD ApplicationSet Generators](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/) -- the seven generators with YAML examples
- [ApplicationSet Go Templates](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/GoTemplate/) -- `goTemplate: true`, sprig functions, missing-key error
- [ApplicationSet Template Overrides](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Template/) -- `templatePatch`, `preservedFields`, progressive rollout strategies
- [InfraCloud: Sharding Clusters Across Application Controller Replicas](https://www.infracloud.io/blogs/sharding-clusters-across-argo-cd-application-controller-replicas/) -- sharding algorithm comparison, operational gotchas
- [AWS EKS Blueprints: Multi-Cluster Hub & Spoke with ArgoCD](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/gitops/gitops-multi-cluster-hub-spoke-argocd/) -- concrete AWS hub-and-spoke with IRSA, cluster registration
- [ArgoCD RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/) -- policy.csv grammar, project roles, JWT tokens
- [ArgoCD User Management](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/) -- Dex vs direct OIDC, group-to-role mapping
- [Codefresh: ArgoCD Secrets -- Two Technical Approaches](https://codefresh.io/learn/argo-cd/argocd-secrets/) -- Sealed Secrets vs ESO vs SOPS head-to-head
- [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) -- kubeseal CLI, scoping modes, key rotation
- [External Secrets Operator](https://external-secrets.io/) -- SecretStore, ClusterSecretStore, providers, refresh semantics
- [Mozilla SOPS](https://github.com/getsops/sops) -- age keys, KMS keys, `.sops.yaml` creation rules
- [ArgoCD Notifications](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/) -- triggers, templates, services, subscription annotations

**Cross-references in this repo**:
- [ArgoCD Fundamentals (Apr 16, 2026)](./2026-04-16-argocd-gitops-fundamentals.md) -- GitOps principles, seven-component architecture, Application CRD, sync policies, waves and hooks, Helm bridge; the prerequisite mental model for everything today
- [Helm Advanced Patterns (Apr 15, 2026)](./2026-04-15-helm-advanced-patterns.md) -- subcharts, global values, hook lifecycle, OCI registries, Helmfile, testing stack; the chart-side counterpart to today's ApplicationSet templating
- [Helm Fundamentals (Apr 13, 2026)](./2026-04-13-helm-fundamentals.md) -- baseline chart anatomy and templating that ApplicationSet layers over
- [EKS Deep Dives -- December 2025](../../../2025/december/eks/) -- the cluster shape your ArgoCD hub runs in and fans out from
- [Terraform Project Structure](../../../2026/january/terraform/2026-02-02-terraform-project-structure.md) -- how to organize the Terraform that bootstraps ArgoCD and registers spokes
