# Helm Fundamentals -- The Parameterized Blueprint Factory Analogy

> You spent all of last year building EKS clusters (nine deep dives in December 2025) -- VPC CNI, IAM roles for service accounts, node autoscaling, security, observability, upgrades. You know how to run Kubernetes. You spent most of January and February internalizing Terraform (fourteen docs) -- modules, state, workspaces, variables, functions, project structure. You know how to parameterize and reuse infrastructure code. But one gap remains in the workflow: **how do you actually ship application workloads into the cluster?** You could `kubectl apply -f` a pile of YAML files and call it a day -- and for a prototype that is fine. But real applications need the same three things Terraform needs: **templating** (so the same Deployment ships to dev with 1 replica and prod with 20), **versioning** (so you can roll forward and backward deterministically), and **packaging** (so you can distribute the whole thing as an artifact). Kubernetes itself gives you none of those. The manifest system is deliberately declarative and unopinionated; it expects something above it to generate the YAML. That "something above" is **Helm**, the de facto Kubernetes package manager.
>
> The analogy that genuinely illuminates Helm -- not the shallow "Helm is like apt for Kubernetes" pitch -- is a **parameterized blueprint factory**. Imagine an architect who designs a building template: a blueprint with parameters for floor count, window type, exterior color, and HVAC capacity. The blueprint (**the chart**) is a reusable design, not a building. The parameter sheet (**values.yaml**) captures defaults -- 3 floors, double-pane windows, beige. When the construction firm actually breaks ground, they fill in the parameter sheet with site-specific overrides ("this site wants 10 floors, triple-pane windows, charcoal") and feed it to the blueprint factory, which stamps out a concrete set of construction drawings (**rendered Kubernetes manifests**). The physical building that stands on the lot is the **release** -- a named, versioned, stateful instance of the blueprint applied to a specific site. You can demolish the building (uninstall), renovate it with a new parameter sheet (upgrade), or undo the last renovation and restore the previous blueprint (rollback) because the factory keeps a **history of every construction revision** ever stamped.
>
> Critically, the factory does not care how many buildings use the same blueprint -- twenty buildings on twenty lots all sharing "Office-Tower-v3" blueprint are twenty independent releases, each with its own parameter overrides, revision history, and physical existence. The **chart repository** is the architectural firm's catalog -- a shelf of blueprints indexed by name and version that anyone can pull from. The **template engine** is the CAD software that reads the blueprint file, substitutes parameters, and emits the actual drawings. Once you hold this mental model firmly -- **chart = blueprint, values = parameter sheet, template engine = CAD, release = actual building, repository = catalog** -- every Helm command and every confusing behavior falls into place. This doc walks through chart anatomy, the Go template engine, named templates, values precedence, the release lifecycle, and the three-way debugging toolbelt -- the foundation you need before OCI registries (tomorrow) and ArgoCD-driven GitOps (next week).

---

**Date**: 2026-04-13
**Topic Area**: kubernetes
**Difficulty**: Intermediate

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Chart | The blueprint -- a directory of templates + default parameter sheet | Reusable, versioned package; contains `Chart.yaml`, `values.yaml`, `templates/`, `charts/`, optional `crds/` |
| Release | The actual building standing on a lot | Named, stateful instance of a chart applied to a cluster/namespace; has its own revision history |
| Repository | The architectural firm's catalog | HTTP server (or OCI registry) with `index.yaml` + chart tarballs; `helm repo add` to subscribe |
| `Chart.yaml` | Cover page of the blueprint (title, version, author) | Required metadata: `apiVersion: v2`, `name`, `version` (chart SemVer), `appVersion` (app SemVer), `dependencies` |
| `values.yaml` | Default parameter sheet shipped with the blueprint | Chart-shipped defaults; lowest precedence in the value merge chain |
| `templates/` | Pages of the blueprint with blanks to fill in | Go-template YAML files rendered against values; `_` prefixed files are partials, not manifests |
| `_helpers.tpl` | Reusable design motifs (window style, door hardware) | Named templates via `define`; included into manifests with `include`; shared labels, selectors, image refs |
| `charts/` | Shared modules from the firm's library (vendor blueprints pulled in alongside yours) | Subchart tarballs; resolved via `helm dependency update` from `Chart.yaml` dependencies block; parent overrides nest under the dep name (`redis:` block) |
| `crds/` | Raw structural steel that the CAD cannot template | Install-only CRDs; NEVER re-applied on upgrade; no templating; use `templates/` with `helm.sh/hook` for managed CRDs |
| Go template engine | The CAD software that stamps the blueprint with parameters | Sprig-extended Go text/template; pipelines (`\|`), flow control (`if`/`with`/`range`), 100+ functions across string/list/dict/date/crypto/network categories |
| Built-in objects | Site information sheet the CAD reads automatically | `.Release`, `.Values`, `.Chart`, `.Files`, `.Capabilities`, `.Template` -- always available, never declared |
| `define` + `include` | Reusable motif block + "use this motif here" instruction | `define` creates a named template; `include` invokes it as a pipeable expression (string-returning) |
| `template` vs `include` | Motif stamped inline (no piping) vs motif copied into a variable (pipeable) | `template` is an action (cannot be piped); `include` returns a string and can feed `\| nindent 4`, `\| toYaml` |
| `required` | "This parameter MUST be filled in -- do not ship otherwise" | `required "message" .Values.x` -- fails render with a human-readable error if value is empty/null |
| `default` | Pre-filled example value on the parameter sheet | `.Values.x \| default "foo"` -- falls back when value is empty/null/missing |
| `toYaml` + `nindent` | Emit a nested structure with correct indentation | `.Values.resources \| toYaml \| nindent 12` -- the standard pattern for dumping maps into manifests |
| `tpl` | Run the CAD a second time on a string that itself contains blueprint markup | Evaluates a string as a template against the current scope; used for values that contain `{{ }}` placeholders |
| `lookup` | Walk the construction site and measure an existing wall | Reads live cluster objects at render time; **returns empty `dict` during `helm template` (no cluster)**; useful for ConfigMaps, Secrets, existing state |
| Values precedence | Two merge phases, rightmost wins within each | Phase 1 (chart): subchart defaults < parent `values.yaml` (overrides via namespaced keys). Phase 2 (user): `-f` files (left to right) < `--set` (left to right, beats `-f`) |
| `helm install` | Break ground and erect the building | Creates a new release; renders templates, applies manifests, records revision 1 |
| `helm upgrade` | Renovate with an amended parameter sheet | Renders with new values, diffs against current release, applies changes, records next revision |
| `helm rollback` | Revert to the blueprint used two renovations ago | Re-applies a prior stored revision; becomes a new revision itself (not a history rewind) |
| Release history | The building's construction and renovation log | Stored as Secrets of type `helm.sh/release.v1` in the release's namespace (Helm 3) |
| `helm lint` | Blueprint plan check -- spelling, missing cover page, YAML sanity | Structural/best-practice checker; no rendering against a cluster |
| `helm template` | Run CAD and print the drawings to a PDF (no construction) | Full local rendering; no API server contact; `lookup` returns empty |
| `helm install --dry-run=server --debug` | Hand drawings to city planning for review before building | Server-aware simulation; catches conflicting resources, auth errors, CRD validation, admission webhooks. Bare `--dry-run` since Helm 3.13 is client-side only |
| `helm get` family | Pull as-built drawings from city archives | `helm get values/manifest/notes/hooks <release>` -- inspect what is actually deployed |
| `helm diff` (plugin) | Markup overlay on existing drawings | Third-party but ubiquitous; shows the diff between current cluster state and a proposed upgrade |

---

## The Blueprint Factory Framework

Every Helm concept maps to one of five roles in an architectural firm:

1. **The Blueprint (Chart)** -- A directory on disk. A `Chart.yaml` cover page, a `values.yaml` parameter sheet, a stack of templated drawings in `templates/`, shared library modules in `charts/`, and optional structural steel in `crds/`. Reusable, versioned, distributable.

2. **The CAD Software (Template Engine)** -- Go text/template extended with Sprig. It reads the blueprint files, substitutes parameters, evaluates flow control (`if`/`with`/`range`), resolves named templates, and emits concrete YAML.

3. **The Parameter Sheet Stack (Values)** -- Layered configuration. Chart defaults at the bottom. Parent-chart overrides next. User-supplied `-f` files above that. `--set` flags on top. Rightmost layer in each category wins.

4. **The Building (Release)** -- What physically exists in the cluster. A named, namespaced entity with a revision history. Install creates it, upgrade renovates it, rollback reverts it, uninstall demolishes it. The same chart can produce many independent releases.

5. **The Catalog (Repository)** -- Where blueprints live when they are not in your working directory. Traditional HTTP repos with an `index.yaml`, or (since Helm 3.8 GA) OCI registries like ECR, GHCR, and Docker Hub.

```
THE BLUEPRINT FACTORY -- HELM OVERVIEW
==============================================================================

  CATALOG                        FACTORY                        CONSTRUCTION SITE
  -------                        -------                        -----------------

  ┌──────────────┐              ┌──────────────────────────┐
  │ Chart Repo   │              │  THE TEMPLATE ENGINE     │
  │ (index.yaml) │──pull───────▶│  (Go template + Sprig)   │
  │  chart-1.tgz │              │                          │
  │  chart-2.tgz │              │  Inputs:                 │
  │  chart-3.tgz │              │   - templates/*.yaml     │       ┌──────────┐
  └──────────────┘              │   - values (merged)      │──────▶│ Rendered │
                                │   - built-in objects     │       │ Manifests│
  ┌──────────────┐              │       .Release           │       │  (YAML)  │
  │ OCI Registry │──pull───────▶│       .Values            │       └────┬─────┘
  │ (ECR, GHCR)  │              │       .Chart             │            │
  │  oci://...   │              │       .Files             │            │ kubectl
  └──────────────┘              │       .Capabilities      │            │ apply
                                │                          │            ▼
                                │  Output: concrete YAML   │       ┌──────────┐
                                └────────────┬─────────────┘       │ Kubernetes│
                                             │                     │  Cluster  │
  ┌────────────────────┐                     │                     └────┬─────┘
  │ VALUES (2 phases)  │                     │                          │
  │                    │─merged into─────────┘                          │
  │ Phase 2 (user):    │                                                │
  │  --set (LTR wins)  │                           ┌────────────────────┤
  │  -f files (LTR)    │                           ▼  release state     │
  │ Phase 1 (chart):   │              ┌──────────────────────┐  read on │
  │  parent values     │              │  RELEASE (stateful)  │  upgrade │
  │  subchart defaults │              │   name, namespace    │ /rollback│
  └────────────────────┘              │   revision 1, 2, 3...│◀─────────┘
                                      │   stored as Secrets  │   helm history
                                      │   helm.sh/release.v1 │   helm rollback
                                      └──────────────────────┘
```

---

## Part 1: Chart Anatomy -- What Actually Lives in the Directory

`helm create mychart` scaffolds the canonical layout. Every production chart you will read follows this shape:

```
mychart/
├── Chart.yaml              # Blueprint cover page (required)
├── values.yaml             # Default parameter sheet (strongly recommended)
├── values.schema.json      # JSON Schema for values validation (optional)
├── .helmignore             # Like .gitignore for `helm package` -- excludes from .tgz
├── charts/                 # Subchart dependencies (resolved via `helm dep update`)
│   └── redis-18.6.1.tgz    # A subchart tarball pulled in from its repo
├── crds/                   # Raw CRDs -- install-only, never upgraded
│   └── myresource-crd.yaml
├── templates/              # The templated drawings
│   ├── _helpers.tpl        # Named templates (partials), leading underscore
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── configmap.yaml
│   ├── NOTES.txt           # Post-install message rendered to user
│   └── tests/              # `helm test` workloads
│       └── test-connection.yaml
├── README.md               # Human docs (not rendered)
└── LICENSE                 # License file (not rendered)
```

The rules the engine applies to `templates/`:

- Files whose basename starts with `_` are **never rendered as manifests** -- they exist only to define reusable named templates (`_helpers.tpl`, `_labels.tpl`, etc.).
- Files whose basename starts with `.` are ignored entirely (dotfiles, editor backups).
- `NOTES.txt` is rendered but not applied to the cluster -- it prints to stdout after install/upgrade.
- Everything else in `templates/` is rendered, parsed as YAML, and applied. Multiple YAML documents can live in one file separated by `---`.

**`values.schema.json`** is an optional JSON Schema file at the chart root that validates the merged values before any template renders. If the user supplies an invalid value (wrong type, missing required field, value out of range), `helm install`/`helm upgrade` fails fast with a schema error rather than failing halfway through a render with a cryptic template error. Highly recommended for any chart you publish externally.

**`.helmignore`** works exactly like `.gitignore` but for `helm package` -- it controls which files are excluded from the produced `.tgz` tarball. Use it to keep test fixtures, local secrets, editor cruft, and CI configs out of distributed charts.

**`templates/tests/`** holds workloads annotated with `helm.sh/hook: test`. Run them with `helm test <release>` after install or upgrade to smoke-test the deployed application (e.g., a Pod that curls the Service and exits 0 on success). Full hook lifecycle is tomorrow's topic.

### `Chart.yaml` -- The Blueprint Cover Page

```yaml
apiVersion: v2                  # v2 is Helm 3; v1 is legacy Helm 2 (do not use)
name: payments-api              # Chart name (lowercase, kebab-case)
description: Payments service Deployment, HPA, Service, Ingress
type: application               # "application" or "library" (library = shareable helpers only, not installable)
version: 1.4.2                  # CHART SemVer -- bump on any chart change
appVersion: "2.8.1"             # APP SemVer -- version of the software being packaged (quoted string)
kubeVersion: ">= 1.28.0-0"      # Enforced at install/upgrade time; blocks install on mismatched clusters
icon: https://example.com/icon.png

dependencies:                   # Subcharts (resolved via `helm dependency update`)
  - name: redis
    version: "~18.6.0"          # SemVer range; ~ means "18.6.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled    # Pulled in only when .Values.redis.enabled is true
    alias: cache                # Mount subchart values under .Values.cache instead of .Values.redis
    import-values:              # Surface specific subchart values into the parent
      - child: service.port
        parent: redisPort

maintainers:
  - name: Platform Team
    email: platform@example.com
```

**The `version` vs `appVersion` distinction is one of the most confused points in Helm**: `version` is the chart's own SemVer -- bumped when you change the template, add a dependency, or tweak default values. `appVersion` is the version of the containerized application the chart deploys. You can release chart `1.5.0` that still deploys app `2.8.1` because you only changed labels. Conversely, you can release chart `1.4.3` that deploys app `2.9.0` because the chart did not structurally change but the image tag did.

### `values.yaml` -- The Default Parameter Sheet

```yaml
replicaCount: 2

image:
  repository: ghcr.io/example/payments-api
  tag: ""                       # Empty -> fallback to .Chart.AppVersion via `default`
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: alb
  annotations: {}
  hosts:
    - host: payments.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

serviceAccount:
  create: true
  annotations: {}
  name: ""                      # Empty -> auto-generate via template helper

# Subchart values nest under the subchart name
redis:
  enabled: false
  auth:
    enabled: true
```

**Best practice**: camelCase keys (`replicaCount`, not `replica_count`). Flat structures where possible but nest when semantic grouping helps (`image.repository`, `resources.requests.cpu`). Always provide sensible defaults so `helm install mychart` works out of the box; require only what truly has no reasonable default (image repository, hostname, secrets).

---

## Part 2: The Template Engine -- Go Templates + Sprig

### Built-In Objects -- The Site Information Sheet

Every template has access to these without declaration:

| Object | Contents | Example |
|--------|----------|---------|
| `.Release` | Install-time metadata | `.Release.Name`, `.Release.Namespace`, `.Release.Revision`, `.Release.IsInstall`, `.Release.IsUpgrade`, `.Release.Service` (always `Helm`) |
| `.Values` | Merged values (chart defaults <- -f files <- --set) | `.Values.image.repository` |
| `.Chart` | `Chart.yaml` fields | `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion` |
| `.Files` | Access to non-template files in the chart | `.Files.Get "config.toml"` (reads from chart **root**, not `templates/`), `.Files.Glob "dashboards/*.json"` |
| `.Capabilities` | Cluster/Helm capabilities | `.Capabilities.KubeVersion.Version`, `.Capabilities.APIVersions.Has "networking.k8s.io/v1"` |
| `.Template` | The template itself | `.Template.Name`, `.Template.BasePath` |

### Pipelines and Flow Control

Go templates use `{{ ... }}` for expressions. The pipe (`|`) chains functions left-to-right:

```yaml
# Pipeline: take .Values.image.tag, fall back to .Chart.AppVersion if empty, then quote it
image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion | quote }}

# Whitespace control: {{- strips leading whitespace, -}} strips trailing
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- end }}

# `with` rebinds the scope (the dot) -- useful for deep paths
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 4 }}
{{- end }}

# `range` iterates
{{- range $i, $host := .Values.ingress.hosts }}
  - host: {{ $host.host | quote }}
{{- end }}
```

**Critical whitespace rule**: YAML is indentation-sensitive and Go templates preserve whitespace by default. The `-` inside `{{-` or `-}}` strips whitespace (including newlines) on that side. Miss a `-` and you end up with blank lines or broken indentation that YAML refuses to parse. `helm template | kubectl apply --dry-run=client -f -` is how you catch these in CI.

### The Functions That Matter in Production

```yaml
# `default` -- fallback when the value is empty/null/missing
replicas: {{ .Values.replicaCount | default 1 }}

# `required` -- fail render with a human-readable error
repository: {{ required "A valid .Values.image.repository is required!" .Values.image.repository }}

# `toYaml` + `nindent` -- emit a map/list with proper indentation
# `nindent N` = newline + N spaces; `indent N` = just N spaces (no leading newline)
resources:
  {{- toYaml .Values.resources | nindent 2 }}

# `tpl` -- evaluate a string as a template against the current scope
# Used when a value itself contains template markup
annotations:
  config-hash: {{ tpl .Values.app.annotationTemplate . }}

# `lookup` -- read live cluster state (returns empty dict in `helm template`)
# There is no flag to populate `lookup` results during `helm template` (unlike
# `--api-versions` for Capabilities). If chart behavior depends on `lookup`,
# test with `helm install --dry-run=server --debug` against a real cluster.
{{- $existing := lookup "v1" "Secret" .Release.Namespace "db-credentials" }}
{{- if $existing }}
# Use existing secret's data
{{- end }}

# `include` -- call a named template, return its output as a string (pipeable)
metadata:
  labels:
    {{- include "payments-api.labels" . | nindent 4 }}

# `quote` / `squote` -- wrap in double/single quotes, useful for string values that YAML might misinterpret
pullPolicy: {{ .Values.image.pullPolicy | quote }}

# `trunc` + `trimSuffix` -- the canonical pattern for Kubernetes name length limits.
# 63 is the DNS-1123 label limit for Kubernetes resource names. Truncating can
# leave a trailing hyphen which makes the name DNS-invalid, so `trimSuffix "-"`
# is mandatory after every `trunc 63`. This is not optional cleanup -- it is
# correctness.
name: {{ .Release.Name | trunc 63 | trimSuffix "-" }}
```

---

## Part 3: Named Templates and `_helpers.tpl`

Named templates are reusable fragments defined with `define` and invoked with `include` (or `template`). By convention they live in `templates/_helpers.tpl` -- the leading underscore tells Helm not to render the file as a manifest.

### A Realistic `_helpers.tpl`

```gotemplate
{{/*
Expand the name of the chart.
Used as the base name for all generated resources.
*/}}
{{- define "payments-api.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Fully qualified app name.
Truncated to 63 chars because some Kubernetes name fields are limited to this.
Uses the release name unless fullnameOverride is set, or unless release name already contains chart name.
*/}}
{{- define "payments-api.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}

{{/*
Chart label -- "chart-name-chart-version"
*/}}
{{- define "payments-api.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Common labels applied to every resource.
These MUST be stable across upgrades for label selectors to keep matching.
*/}}
{{- define "payments-api.labels" -}}
helm.sh/chart: {{ include "payments-api.chart" . }}
{{ include "payments-api.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{/*
Selector labels -- MUST be immutable across upgrades.
A Deployment's selector cannot change once created; if these labels change,
`helm upgrade` will fail with a Forbidden error.
Never add release revision, image tag, or any mutable value here.
*/}}
{{- define "payments-api.selectorLabels" -}}
app.kubernetes.io/name: {{ include "payments-api.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{/*
Service account name.
*/}}
{{- define "payments-api.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
{{- default (include "payments-api.fullname" .) .Values.serviceAccount.name -}}
{{- else -}}
{{- default "default" .Values.serviceAccount.name -}}
{{- end -}}
{{- end -}}
```

### Using the Helpers in a Deployment

```gotemplate
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "payments-api.fullname" . }}
  labels:
    {{- include "payments-api.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount | default 2 }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "payments-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        # Roll pods when the ConfigMap changes -- classic pattern
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "payments-api.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "payments-api.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          env:
            - name: DATABASE_URL
              value: {{ required "A valid .Values.database.url is required!" .Values.database.url | quote }}
            - name: LOG_LEVEL
              value: {{ .Values.logLevel | default "info" | quote }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### `include` vs `template` -- Why `include` Always Wins

Both `include` and `template` invoke a named template. The difference is the return type:

- **`template "name" .`** is an **action** -- it writes output directly to the template stream and returns nothing. You cannot pipe it to another function.
- **`include "name" .`** is a **function** -- it returns the rendered output as a **string**, which can then be piped to `indent`, `nindent`, `toYaml`, `sha256sum`, etc.

```gotemplate
# BROKEN -- `template` cannot be piped, so `nindent 4` applies to nothing
metadata:
  labels:
    {{ template "payments-api.labels" . | nindent 4 }}

# CORRECT -- `include` returns a string, which `nindent 4` can transform
metadata:
  labels:
    {{- include "payments-api.labels" . | nindent 4 }}
```

In practice, **always use `include`**. The only case for `template` is cosmetic (you genuinely do not need piping), and being consistent saves debugging time.

**Debugging "my labels indented wrong" -- the two-orthogonal-layers frame.** Senior engineers diagnose this class of failure as **two independent layers that both have to be right**:

- **Layer 1: directive choice.** `template` is an action, not a function. `{{ template "x" . | nindent 4 }}` parses fine but `| nindent 4` operates on the empty string the action returns, so indentation is never applied. Fix: swap `template` -> `include`.
- **Layer 2: whitespace coordination.** `nindent N` writes its **own** newline followed by N spaces. If you also have literal leading whitespace on the line before `{{` (e.g., `  {{ include "x" . | nindent 4 }}`), that literal whitespace lands on the line above the `nindent` newline, producing trailing whitespace, blank lines, or doubled indentation depending on context. Fix: chomp with `{{-` to strip the leading whitespace.

Both layers can be wrong simultaneously. Fixing only one (e.g., swapping `template` -> `include` without adding `{{-`) leaves the manifest still misaligned, leading to the "I changed it and nothing happened" debugging spiral. The canonical correct form is **`{{- include "x" . | nindent 4 }}`** -- chomp on the left, use `include`, let `nindent` own the newline + spaces on the right.

### The Scope Trap -- Why You Pass `.` Explicitly

Named templates **do not inherit scope from the caller**. When you write `{{ include "my.helper" . }}`, the `.` argument is explicit: you are passing the current context (the top-level scope containing `.Values`, `.Release`, `.Chart`) to the helper. Forget the `.` and you pass `nil`, and every reference inside the helper to `.Values.whatever` returns `<no value>` or explodes.

```gotemplate
# WRONG -- helper sees nothing
{{ include "payments-api.labels" }}

# WRONG inside a range (. is the loop item, not the root context)
{{- range .Values.ingress.hosts }}
  {{- include "payments-api.labels" . }}   # . here is a single host map!
{{- end }}

# CORRECT inside a range -- pass the root context via $
{{- range .Values.ingress.hosts }}
  - host: {{ .host | quote }}
    {{- include "payments-api.labels" $ | nindent 4 }}   # $ is always the root scope
{{- end }}
```

`$` is a Go-template built-in that always refers to the root scope regardless of how deeply nested you are in `with` or `range` blocks. Keep it in mind whenever you see `<no value>` in rendered output.

---

## Part 4: Values Precedence -- The Stack of Parameter Sheets

When you run `helm install myapp ./chart -f prod.yaml -f prod-eu.yaml --set image.tag=2.8.2 --set replicaCount=20`, Helm performs **two merge phases** with **four sources** total:

```
PHASE 1 -- Chart-side merge (resolved when the chart is loaded)
  (1) Subchart defaults from charts/*/values.yaml         LOWEST
  (2) Parent chart's values.yaml                          may override (1) via
                                                          namespaced keys (redis: ...)

PHASE 2 -- User overrides (applied to Phase 1 result)
  (3) -f files, applied left to right                     rightmost -f wins
  (4) --set / --set-string / --set-file / --set-json,     rightmost wins,
      applied left to right                               beats every -f file
                                                          HIGHEST
```

**The universal rule**: **rightmost wins within each source, and `--set` always beats `-f` regardless of command-line order.**

```bash
# replicaCount resolves to 20 (the --set wins the last merge)
helm install myapp ./chart -f prod.yaml --set replicaCount=20

# If prod.yaml has replicaCount: 10 and prod-eu.yaml has replicaCount: 5,
# final value is 5 (prod-eu.yaml is rightmost among -f files)
helm install myapp ./chart -f prod.yaml -f prod-eu.yaml

# --set overrides BOTH -f files regardless of order
helm install myapp ./chart --set replicaCount=20 -f prod.yaml -f prod-eu.yaml
# Still 20 -- --set always wins the final merge
```

**Subchart values override convention**: To override a subchart's value from the parent, nest it under the subchart name (or alias):

```yaml
# Parent values.yaml
redis:                          # Matches the dependency name (or alias) in Chart.yaml
  auth:
    enabled: false              # Overrides the subchart's auth.enabled default
  replica:
    replicaCount: 3
```

`--set-string`, `--set-file`, and `--set-json` are siblings of `--set` for cases where you need type coercion control (e.g., `--set-string image.tag=2.8` to avoid being parsed as a float).

---

## Part 5: The Release Lifecycle

### `helm install` -- Breaking Ground

```bash
# Install from a local chart directory
helm install payments ./payments-api -n payments --create-namespace

# Install from a remote repo (after `helm repo add bitnami ...`)
helm install my-redis bitnami/redis --version 18.6.1 -f values-prod.yaml

# Dry run (server-aware -- since Helm 3.13 you must explicitly pass =server;
# bare --dry-run is client-side only)
helm install payments ./payments-api --dry-run=server --debug

# Install with explicit release name; otherwise `--generate-name` auto-names it
helm install --generate-name ./payments-api
```

What happens:
1. Helm renders templates against merged values.
2. Validates schemas (`values.schema.json` if present, then CRDs and Kubernetes API schema).
3. Submits manifests in Helm's hard-coded **kind order** (Namespaces first, then NetworkPolicy / ResourceQuota / LimitRange / PodSecurityPolicy, then ServiceAccount / Secret / ConfigMap, then workloads like Deployment / StatefulSet / DaemonSet, then autoscalers and Ingress last). Helm does **not** do dependency analysis -- it sorts by a hard-coded `InstallOrder` list in `kind_sorter.go`. For ordering across kinds outside this list, or for ordering inside a single kind, use chart hooks (tomorrow's topic).
4. Waits for resources to be "ready" if `--wait` is passed (see the readiness section below).
5. Records **revision 1** as a Secret named `sh.helm.release.v1.payments.v1` in the release's namespace.

**Readiness semantics with `--wait` and `--wait-for-jobs`.** By default, `helm install`/`helm upgrade` returns as soon as the API server accepts the manifests -- the resources may not yet be running. `--wait` makes Helm block until Deployments meet `minReady`, Services have endpoints, PVCs are bound, and Pods are Running with their containers Ready. `--wait-for-jobs` extends the wait to include Job completion (otherwise Jobs are considered "ready" once created). For CI/CD pipelines that need to know the deploy actually succeeded before moving on, **always** pass `--wait` (and `--wait-for-jobs` if the chart includes Jobs). Pair with `--timeout 5m` (or longer) so a stuck rollout fails fast.

### `helm upgrade` -- Renovation

```bash
# In-place upgrade with new values
helm upgrade payments ./payments-api -f values-prod.yaml --set image.tag=2.8.2

# Upgrade-or-install (idempotent -- preferred in CI/CD)
helm upgrade --install payments ./payments-api -f values-prod.yaml

# Atomic: rollback automatically if the upgrade fails
helm upgrade --install payments ./payments-api --atomic --timeout 5m

# Reuse previous values, only override what is specified
helm upgrade payments ./payments-api --reuse-values --set image.tag=2.8.3

# Reset to chart defaults, ignoring previous values
helm upgrade payments ./payments-api --reset-values -f new-values.yaml

# Take NEW chart defaults but keep the previous user-supplied overrides on top
# (Helm 3.14+, Feb 2024 -- fixes the long-standing --reuse-values footgun)
helm upgrade payments ./payments-api --reset-then-reuse-values --set image.tag=2.8.3
```

Each successful upgrade records a new revision. The previous revision stays in history (default: last 10 kept; tunable with `--history-max`).

### `helm rollback` -- Undo the Renovation

```bash
helm history payments
# REVISION  UPDATED                   STATUS      CHART            APP VERSION
# 1         Mon Apr 13 09:00:00 2026  superseded  payments-1.4.0   2.8.0
# 2         Mon Apr 13 10:00:00 2026  superseded  payments-1.4.1   2.8.1
# 3         Mon Apr 13 11:00:00 2026  failed      payments-1.4.2   2.8.2
# 4         Mon Apr 13 11:05:00 2026  deployed    payments-1.4.1   2.8.1

# Roll back to a specific revision
helm rollback payments 2

# Roll back to the immediately previous revision
helm rollback payments
```

**A rollback is not a history rewind -- it is a new revision that happens to apply an older manifest set.** The revision counter is **strictly monotonic**: it only ever increases. Concretely, if you are on revision 4 and run `helm rollback payments 2`, the deployed revision becomes **revision 5** (with content copied from revision 2), not revision 2. Revision 4 is marked `superseded`; the counter never decreases. Every operation -- install, upgrade, rollback, even auto-rollback from `--atomic` failure -- appends one row to history.

**The blueprint analogy strains slightly here** -- a physical building does not have a time-travel ledger. The cleaner mental model for revisions is **Git**: the **release is a Git branch**, revisions are **commits on that branch**, and `helm rollback` is `git revert` -- it produces a *new* commit (not a state restoration) whose tree happens to match an earlier one. `helm history` is `git log` for the release branch.

### Release State Storage (Helm 3 vs Helm 2)

Helm 3 stores release state as **Secrets of type `helm.sh/release.v1`** in the release's namespace (switchable to ConfigMaps via `HELM_DRIVER`). Helm 2 used ConfigMaps in `kube-system` with the Tiller server-side component. **Tiller was removed in Helm 3.** Everything now runs client-side with the user's kubeconfig credentials, which eliminates the cluster-wide security hole Tiller represented. If you see references to Tiller, the content is pre-2019 and ignorable.

**Each revision Secret is a full, independent snapshot -- not a pointer or a delta.** The Secret's `release` data field is a **gzipped, base64-encoded JSON payload** containing the entire chart tarball, the merged values used at render time, the rendered manifests, hooks, and status metadata. There is no deduplication across revisions, no "previous revision pointer." This is why rollback is fast and self-contained (Helm reads one Secret and applies its manifests) and why **history storage grows linearly with revision count**.

```bash
# Find release state Secrets
kubectl get secret -n payments -l owner=helm,name=payments
# NAME                            TYPE                DATA   AGE
# sh.helm.release.v1.payments.v1  helm.sh/release.v1  1      2h
# sh.helm.release.v1.payments.v2  helm.sh/release.v1  1      1h
```

**`--history-max` matters in production.** Default history retention is **10 revisions per release**. High-frequency CD pipelines (deploys every commit, multiple releases per cluster) can accumulate thousands of stale revision Secrets in etcd, which bloats etcd snapshots, slows down `kubectl get secret -A`, and (in pathological cases) approaches the 1 MB single-object size limit when a chart is large. Cap growth explicitly:

```bash
helm upgrade --install payments ./payments-api --history-max 5 ...
```

Set this consistently in your CI templates. 5 is usually plenty for rollback purposes; if you need deeper history for audit, store it in your CD system (Argo, Spinnaker, GitHub Actions logs), not in etcd.

### Release Naming Rules

- Lowercase DNS-1123: letters, numbers, hyphens (no underscores, no dots).
- Max 53 characters (so that resources named after the release still fit under the 63-char Kubernetes name limit).
- Unique within a namespace. **Same name in a different namespace is a different release.**

---

## Part 6: Debugging -- Three Tools, Three Different Jobs

### `helm lint` -- Structural Check

```bash
helm lint ./payments-api
helm lint ./payments-api --strict     # Warnings become errors
```

Reads `Chart.yaml` for required fields, validates directory structure, renders templates against default values, checks YAML parses. **No cluster contact.** Use as a pre-commit hook and in CI to catch typos before anything else runs.

### `helm template` -- Local Rendering

```bash
# Render everything to stdout -- no API server contact
helm template payments ./payments-api -f values-prod.yaml

# Render a single file
helm template payments ./payments-api -s templates/deployment.yaml

# Pipe to kubectl for client-side validation
helm template payments ./payments-api | kubectl apply --dry-run=client -f -

# Render with a specific release name and namespace for realistic output
helm template payments ./payments-api --namespace payments --release-name payments
```

Use when: you want to see exactly what YAML will be applied, without any cluster involvement. Essential for CI diffing (`helm template` in a pipeline + `diff` against a baseline). **Caveat**: `lookup` returns an empty `dict` because there is no cluster to query. `.Capabilities.APIVersions.Has` uses a built-in static list, so checks for CRDs your cluster actually has may be inaccurate. Use `--kube-version` and `--api-versions` to simulate a specific cluster.

### `helm install --dry-run=server --debug` -- Server-Aware Simulation

```bash
# CRITICAL: since Helm 3.13 (Oct 2023), bare --dry-run defaults to client-side
# and does NOT contact the API server. To get true server-side simulation
# (admission webhooks, validation, default injection), pass --dry-run=server.
helm install payments ./payments-api -f values-prod.yaml --dry-run=server --debug

# Bare --dry-run (== --dry-run=client) is now equivalent to `helm template`
# with the release name and namespace prefilled.
helm install payments ./payments-api --dry-run --debug
```

Use `--dry-run=server` when: you need the API server to weigh in. It catches things `helm template` and `--dry-run=client` cannot:
- Conflicts with existing resources (e.g., a Service of the same name already exists with a different selector).
- CRD schema validation (admission controllers reject the manifest, mutating webhooks inject sidecars).
- RBAC/auth failures (you lack permission to create Deployments in this namespace).
- Accurate `.Capabilities.APIVersions.Has` for your real cluster.
- `lookup` actually returning data from the live cluster.

The output shows the merged values, the rendered manifests, and the server's response. This is the slowest of the three tools (round trips to the API server) but the most realistic.

### `helm get` -- Inspecting What Is Actually Deployed

Once a release exists in the cluster, the `helm get` family pulls "as-built" information from the stored release Secrets. This is the toolkit for answering "what is actually running, with what values, and what was the rendered manifest?"

```bash
# What values were used? (just the user-supplied overrides)
helm get values payments
# What values were used, INCLUDING chart defaults? (the fully merged set)
helm get values payments --all

# The fully rendered YAML that was applied to the cluster
helm get manifest payments

# The NOTES.txt that was printed
helm get notes payments

# Any hooks (pre-install, post-upgrade, test, etc.) attached to this release
helm get hooks payments

# Everything at once
helm get all payments

# Specific revision
helm get values payments --revision 3
```

Reach for `helm get manifest` whenever you need to compare "what Helm thinks it deployed" to "what is actually in the cluster" -- they can drift if someone `kubectl edit`-ed a resource Helm owns.

### `helm diff` -- The Plugin Everyone Installs

`helm diff` is a third-party plugin (`helm plugin install https://github.com/databus23/helm-diff`) but is effectively standard equipment in production CD pipelines. Unlike `helm template | diff`, it is **cluster-state-aware**:

```bash
# Show what would change vs current cluster state if you ran the upgrade
helm diff upgrade payments ./payments-api -f values-prod.yaml --set image.tag=2.8.3

# Diff between two revisions in history
helm diff revision payments 3 4
```

**Position in the debugging ladder**: `helm diff upgrade` sits explicitly **between `helm template` and `helm install --dry-run=server`**. It fetches the **current rendered manifest from the release Secret** (`helm get manifest` under the hood), renders the **proposed new manifest locally** against your new values, and prints a unified diff. That makes it **cluster-state-aware** for "what would change" -- it catches drift introduced by `kubectl edit`, partial out-of-band changes, and the actual current state of the deployment -- but it does **not** invoke admission webhooks or server-side validation. For "will this upgrade succeed?" you still need `--dry-run=server`. For "what specifically will change in the cluster?" `helm diff upgrade` is the right tool. ArgoCD and Flux essentially run `helm diff` continuously under the hood to drive their reconciliation loops.

### The Decision Rule

```
Syntax/structural issue, want fast feedback           -> helm lint
Value merging / template output review (no cluster)   -> helm template
What WOULD CHANGE vs current cluster state?           -> helm diff upgrade
                                                         (cluster-aware, NO admission)
Will this upgrade SUCCEED? (admission, lookup,        -> helm install --dry-run=server
 webhooks, validation, default injection)                --debug
What is actually running RIGHT NOW?                   -> helm get values --all / manifest
```

In a typical CI pipeline you run lint -> template -> diff (against the current cluster) -> upgrade with `--atomic --wait`.

---

## Part 7: Chart Repositories (Briefly)

The traditional chart distribution model: an HTTP server hosting an `index.yaml` file and one tarball per chart version.

```bash
# Add a repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                            # Refresh the cached index.yaml

# Search
helm search repo redis
helm search repo redis --versions           # Show all versions

# Install from a repo
helm install my-redis bitnami/redis --version 18.6.1

# Inspect a chart before installing
helm show values bitnami/redis > redis-defaults.yaml
helm show readme bitnami/redis

# Pull a chart locally (to vendor it or read sources)
helm pull bitnami/redis --version 18.6.1 --untar
```

Behind the scenes, `index.yaml` is a YAML index of all chart versions available at the repo, each pointing to a `.tgz` tarball at a stable URL. `helm repo update` re-fetches the index. `helm install` downloads the tarball, renders it, and applies it.

### Publishing -- `helm package` and `helm push`

The other half of the lifecycle is publishing your own chart. `helm package` produces the distributable `.tgz` artifact; `helm push` (OCI) ships it to a registry:

```bash
# Bump version in Chart.yaml first, then:
helm package ./payments-api
# -> payments-api-1.4.2.tgz in the current directory

# Push to an OCI registry (Helm 3.8+)
helm push payments-api-1.4.2.tgz oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts

# For traditional HTTP repos: regenerate index.yaml after dropping the .tgz on the server
helm repo index ./repo-dir --url https://charts.example.com
```

Full OCI registry treatment (auth flows, ECR specifics, Cosign signing, Chart.lock dependency pinning) is tomorrow's deep dive; the line above is enough to complete the lifecycle picture today.

**OCI registries (Helm 3.8 GA, May 2022)** are the newer distribution model where charts are pushed to container registries (ECR, GHCR, Docker Hub, Harbor) as OCI artifacts instead of to HTTP repositories. The commands change -- `helm push oci://...`, `helm pull oci://...`, no `helm repo add` needed -- and enterprise features like SigV4 auth on ECR, Cosign signing, and tag immutability come along for the ride.

---

## Part 8: A Complete End-to-End Example

Tying everything together. A minimal but realistic payments-api chart:

**`Chart.yaml`**:
```yaml
apiVersion: v2
name: payments-api
description: Payments API -- Deployment, Service, Ingress, HPA
type: application
version: 1.0.0
appVersion: "2.8.1"
kubeVersion: ">= 1.28.0-0"
```

**`values.yaml`** (defaults):
```yaml
replicaCount: 2

image:
  repository: ghcr.io/example/payments-api
  tag: ""                       # Defaults to .Chart.AppVersion
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

database:
  url: ""                       # REQUIRED -- enforced via `required`

logLevel: info

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

nodeSelector: {}
affinity: {}

serviceAccount:
  create: true
  name: ""
```

**`values-prod.yaml`** (environment override):
```yaml
replicaCount: 10
image:
  tag: "2.8.2"
database:
  url: "postgres://prod-db.internal:5432/payments"
logLevel: warn
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
```

**Install and upgrade**:
```bash
# Initial deployment
helm upgrade --install payments ./payments-api \
  -n payments --create-namespace \
  -f values-prod.yaml \
  --atomic --timeout 5m

# Roll to a new app version without re-editing values files
helm upgrade payments ./payments-api \
  -n payments \
  -f values-prod.yaml \
  --set image.tag=2.8.3 \
  --atomic

# Emergency rollback
helm rollback payments -n payments
```

### Driving Helm from Terraform

The Terraform `helm` provider exposes `helm_release` -- a resource that runs `helm install`/`helm upgrade --install` from inside a Terraform plan. This is the canonical way to install platform-level workloads (cert-manager, external-dns, ingress controllers, the AWS Load Balancer Controller) into an EKS cluster from the same Terraform that builds the cluster:

```hcl
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.13"
    }
  }
}

provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.this.endpoint
    cluster_ca_certificate = base64decode(aws_eks_cluster.this.certificate_authority[0].data)
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args        = ["eks", "get-token", "--cluster-name", aws_eks_cluster.this.name]
    }
  }
}

resource "helm_release" "payments_api" {
  name             = "payments"
  namespace        = "payments"
  create_namespace = true

  # Pull from an OCI registry; can also be a traditional repo + chart name
  chart      = "payments-api"
  repository = "oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts"
  version    = "1.4.2" # Pin chart version explicitly -- never leave floating

  # Lifecycle controls
  atomic           = true # Roll back automatically if upgrade fails
  cleanup_on_fail  = true # Delete partially-created resources on failure
  wait             = true # Block until resources are ready (Deployments, PVCs, Services)
  wait_for_jobs    = true # Also wait for Job completion (charts with migrations)
  timeout          = 600  # 10 minutes; default is 5

  # Bulk values via a templated YAML file (file() reads at plan time)
  values = [
    templatefile("${path.module}/values/payments-prod.yaml.tftpl", {
      database_url = aws_db_instance.payments.endpoint
      log_level    = "warn"
    })
  ]

  # Dynamic per-deploy overrides via `set` / `set_sensitive`
  set {
    name  = "image.tag"
    value = var.payments_image_tag # Bumped by CI per release
  }

  set_sensitive {
    name  = "database.password"
    value = data.aws_secretsmanager_secret_version.db.secret_string
  }

  depends_on = [aws_eks_node_group.workers]
}
```

Two patterns this enables: (1) **platform components** (controllers, CRDs, observability stack) installed declaratively alongside the cluster they run in -- one `terraform apply` brings the whole platform up. (2) **Application releases** where Terraform owns the chart version and base values, and CI bumps `var.payments_image_tag` per deploy. For pure application CD with frequent releases, ArgoCD (next week) is usually a better fit -- Terraform `helm_release` shines for things that change at infrastructure cadence.

### When to Reach for Kustomize Instead

Kustomize is the other major Kubernetes manifest tool, built into `kubectl` (`kubectl apply -k`). The two are often presented as alternatives but solve different problems:

| Dimension | Helm | Kustomize |
|-----------|------|-----------|
| Customization model | Templating (Go templates + values) | Patching (strategic merge + JSON Patch overlays) |
| Engine | Render-then-apply | Patch-then-apply (no template engine) |
| Release tracking | Yes (revisions, history, rollback) | No (kubectl owns the state) |
| Distribution | Yes (chart tarballs, repos, OCI) | No (overlays live in your repo) |
| Subcomponents | Subcharts with namespaced overrides | Bases + overlays via path references |
| Best fit | Distributing reusable packages, lifecycle management | Varying a base YAML set across environments without parameterization |

**Heuristic**: use **Helm** when you are publishing a chart for others to consume, or when you need release lifecycle (history, atomic rollback). Use **Kustomize** when you have a base YAML set you want to vary across environments without inventing parameters. Many teams use both -- Helm for third-party charts (Bitnami, Prometheus stack, ingress controllers) and Kustomize for internal app manifests where the team controls every YAML file. Helm 3 also supports `--post-renderer` to pipe rendered output through Kustomize, which is the bridge pattern.

---

## Critical Gotchas and Interview Traps

**1. `template` vs `include` -- piping and indentation.**
`template "name" .` is an action and cannot be piped. `{{ template "labels" . | nindent 4 }}` applies `nindent 4` to the empty string the action returns, leaving your labels unindented. Always use `include`. This is the single most frequent cause of "my labels are indented wrong" bug reports.

**2. Scope loss inside named templates.**
Named templates do not inherit caller scope. You must pass `.` explicitly: `{{ include "my.helper" . }}`. Inside a `range` loop the `.` is the current item, not the root; use `$` to reach the root scope: `{{ include "my.helper" $ }}`. Forgetting this produces `<no value>` in rendered output with no explicit error.

**3. `--set` always beats `-f`, regardless of command-line order.**
A common confusion is thinking that `helm install ... --set x=1 -f later.yaml` lets `later.yaml` override `--set`. It does not. All `--set` flags win over all `-f` files. The ordering-wins rule applies only **within** each tier (left-to-right among `-f`, left-to-right among `--set`).

**4. `lookup` returns an empty `dict` during `helm template`.**
If your chart uses `lookup "v1" "Secret" ns "name"` to conditionally behave based on existing cluster state, `helm template` renders as if the lookup found nothing. This is expected (no cluster) but means `helm template` output does not reflect real upgrade behavior. **There is no flag to populate `lookup` results during `helm template`** -- unlike `--api-versions`, which lets you simulate `Capabilities`. The only way to test `lookup`-dependent charts is `helm install --dry-run=server --debug` against a real cluster.

**5. `crds/` is install-only -- and the alternatives all have catastrophic failure modes.**
Files in `crds/` are applied once at install time and are **never touched by `helm upgrade`, `helm uninstall`, or any subsequent operation.** This is a deliberate safety measure -- removing a CRD cascade-deletes every CR of that kind across the cluster. `crds/` files are also **not templated** -- they are plain YAML applied verbatim. Three patterns exist for managing CRDs, each with a sharp edge:

1. **`crds/` directory** (default): safe but install-only. CRD upgrades require manual `kubectl apply -f`. Best for stable CRDs that rarely change.
2. **CRDs in `templates/`**: lifecycle-managed by Helm upgrade -- but `helm uninstall` will cascade-delete the CRD and **every CR cluster-wide**. The mitigation is the **`helm.sh/resource-policy: keep`** annotation, which tells Helm "never delete this resource on uninstall." Production charts that put CRDs in `templates/` (cert-manager, Prometheus Operator) **always** pair them with this annotation.
3. **`pre-install` / `pre-upgrade` hook Job that runs `kubectl apply -f crds/`** from inside the chart: a niche middle ground used by some operator charts. Decouples CRD application from template render order and lets one chart manage CRD lifecycle through hooks. Requires the chart's ServiceAccount to have cluster-admin or equivalent CRD-write permissions, which is its own security tradeoff.

Default to `crds/` unless you specifically need lifecycle management, and when you do use `templates/`, **always add `helm.sh/resource-policy: keep`**.

**6. `appVersion` vs `version` confusion.**
`version` is the chart's SemVer. `appVersion` is the version of the packaged software. Bumping the image tag without touching templates means you bump `appVersion` but can leave `version` alone -- though in practice most teams bump `version` on any change for traceability. `helm upgrade` compares chart versions in history, not `appVersion`; if you forget to bump `version`, `helm history` will show identical entries.

**7. Selector labels must never change.**
`spec.selector.matchLabels` in a Deployment is **immutable** after creation. If your `selectorLabels` helper includes a value that varies (revision, image tag, a value that changes between environments), `helm upgrade` fails with a `Forbidden` error and the release gets stuck. Keep selector labels to genuinely stable identifiers (`app.kubernetes.io/name`, `app.kubernetes.io/instance`) and use the separate `labels` helper for everything else.

**8. Release names are DNS-1123 AND max 53 characters.**
53, not 63 -- the 10-char headroom is for Helm-generated suffixes on resources named after the release (e.g., the ServiceAccount `{release-name}-token-xyz`). A 55-character release name installs but fails the first time a resource derives a longer name from it.

**9. Subchart value overrides must nest under the subchart name (or alias).**
To set `.auth.enabled = false` on a `redis` subchart, you put `redis: { auth: { enabled: false } }` in the parent values, NOT `auth: { enabled: false }`. If you used `alias: cache` in `Chart.yaml`, you must nest under `cache:` instead. This three-step mapping (subchart name -> dependency name in `Chart.yaml` -> alias if set) is the #1 reason subchart overrides silently fail.

**10. `helm template` does not respect admission webhooks or CRD schemas.**
If an admission controller mutates or validates manifests (e.g., injecting sidecars, enforcing label conventions, validating custom resource fields), `helm template` output shows none of that. Only `helm install --dry-run=server --debug` sends the manifest through the server-side admission chain. Many "it renders fine locally but fails on apply" bugs come from this gap.

**11. `NOTES.txt` is rendered every time, not just on install.**
`NOTES.txt` is printed after install and after every upgrade. It is a template like any other template and has access to `.Release`, `.Values`, etc. A common gotcha: using `{{ .Release.IsInstall }}` to print different content for install vs upgrade is supported and often necessary.

**12. `helm upgrade --reuse-values` ignores chart-default changes.**
`--reuse-values` merges with the values stored in the previous release Secret, not with the current chart's `values.yaml`. If you add a new default to `values.yaml` in a new chart version, `--reuse-values` will not pick it up. Use `--reset-then-reuse-values` (added in Helm 3.14, Feb 2024) to take the new chart defaults AND the previously-set user overrides.

**13. `required` runs at render time, not install time.**
`{{ required "database.url is required!" .Values.database.url }}` fails during template rendering. This happens in `helm template`, `helm install`, `helm install --dry-run`, AND `helm upgrade`. Good: errors surface early. Bad: if you use `required` in a file that is conditionally rendered (behind `{{ if .Values.foo }}`), the check runs only when the block executes. Put `required` checks in unconditional paths or in `_helpers.tpl` templates called early.

**14. Bare `--dry-run` is client-side only since Helm 3.13.**
This is the gotcha that bites every operator who upgraded Helm in late 2023. Before Helm 3.13 (Oct 2023), `helm install --dry-run` contacted the API server. Since 3.13, **bare `--dry-run` is equivalent to `--dry-run=client`** and does NOT contact the API server -- it is essentially `helm template` with the release name and namespace prefilled. To get true server-side simulation (admission webhooks, validation, default field injection, accurate `lookup`), you must explicitly pass **`--dry-run=server`**. Old runbooks, blog posts, and CI scripts that relied on `--dry-run` for server-side validation now silently skip it.

**15. Releases stuck in `pending-upgrade` or `pending-install`.**
If a `helm upgrade` is killed mid-flight (CI timeout, terminal closed, network blip), the release lands in status `pending-upgrade` or `pending-install` and every subsequent `helm upgrade` fails with "another operation (install/upgrade/rollback) is in progress". Recovery, in order: (1) `helm rollback <release>` to the previous good revision -- usually works; (2) if rollback **itself** fails (common when the prior revision is also invalid under current cluster state -- e.g., admission webhook policies tightened since that revision was deployed, so even the "good" revision now violates them), manually delete the latest release Secret: `kubectl delete secret -n <ns> sh.helm.release.v1.<name>.v<N>` where `<N>` is the stuck revision number. Helm then treats revision N-1 as current state and the next `helm upgrade` proceeds. This is the production last-resort unstick for a **chain of bad revisions** that have all become un-rollback-able. (3) For truly unrecoverable cases, `helm uninstall --no-hooks` and reinstall. This entire sequence is a recurring production ops pain point and a frequent interview question.

---

## Key Takeaways

- **Chart vs release vs repository is the core mental model.** The chart is the blueprint (a directory on disk). The release is the building (a named stateful thing in the cluster). The repository is the catalog (an HTTP or OCI-hosted library of chart versions). A single chart produces many independent releases; releases are unique per (name, namespace) pair.

- **Values flow through two merge phases, four sources.** Phase 1 (chart): subchart defaults < parent `values.yaml` (overrides via namespaced keys). Phase 2 (user): `-f` files left-to-right < `--set` left-to-right. `--set` always beats `-f`. Subchart overrides must nest under the subchart's dependency name or alias.

- **`include` always wins over `template`.** `include` returns a string and can be piped to `nindent`, `toYaml`, `sha256sum`. `template` is an action and cannot. The one weird trick is consistency: every place you invoke a named template, use `include`.

- **Pass `.` explicitly to named templates, and remember `$` for root scope.** Named templates do not inherit scope. Inside `range` and `with`, `.` is rebound to the current item -- reach the root context with `$`.

- **Five-tool debugging belt.** `helm lint` for fast structural feedback. `helm template` for local rendering review and CI diffing. `helm install --dry-run=server --debug` for server-aware simulation (NOT bare `--dry-run` since Helm 3.13). `helm get values/manifest/notes/hooks` for "what is actually deployed". `helm diff upgrade` for "what would change vs reality".

- **`crds/` is install-only and NOT templated.** For lifecycle-managed CRDs, use `templates/` with `helm.sh/hook: crd-install` or a dedicated CRD subchart. For raw bootstrap CRDs, `crds/` is fine -- but remember nothing upgrades them.

- **Selector labels are immutable; common labels are not.** Keep `selectorLabels` to stable values (`app.kubernetes.io/name`, `app.kubernetes.io/instance`). Everything else goes in the broader `labels` helper. A change to selector labels turns `helm upgrade` into a Forbidden error.

- **Release state lives in Secrets in the release's namespace (Helm 3).** No more Tiller. No more cluster-wide server-side component. Helm runs client-side with your kubeconfig credentials.

- **Always use `helm upgrade --install --atomic --wait` in CI.** Idempotent (create or update), automatically rolls back on failure, blocks until resources are actually ready, and bounds execution time with `--timeout`. Add `--wait-for-jobs` if the chart includes Jobs (migrations). The default imperative `helm install` / `helm upgrade` split belongs to interactive sessions, not pipelines.

- **`version` bumps on any chart change; `appVersion` reflects the packaged software version.** Treat them as two independent SemVer streams that happen to correlate.

- **Helm vs Kustomize is "or", not "vs".** Helm for distributable, lifecycle-managed packages (third-party charts, anything you ship). Kustomize for environment overlays of YAML you fully own. Many teams run both, and `helm --post-renderer` lets you pipe Helm output through Kustomize when you need patches on a chart you cannot fork.

---

## Study Resources

Curated reading list for this topic: [`study-resources/kubernetes/helm-fundamentals.md`](../../../study-resources/kubernetes/helm-fundamentals.md)

**Key references**:
- [Helm Quickstart](https://helm.sh/docs/intro/quickstart/) -- 5-minute intro: install, add a repo, install a chart
- [Using Helm](https://helm.sh/docs/intro/using_helm/) -- Chart/release/repository mental model and command surface
- [Charts (Chart.yaml, structure)](https://helm.sh/docs/topics/charts/) -- The canonical reference for chart anatomy
- [Chart Template Guide](https://helm.sh/docs/chart_template_guide/) -- Full template engine walkthrough: built-in objects, pipelines, flow control, named templates
- [Debugging Templates](https://helm.sh/docs/chart_template_guide/debugging/) -- `helm lint` vs `helm template` vs `--dry-run=server --debug`
- [Helm Diff Plugin](https://github.com/databus23/helm-diff) -- The third-party plugin every CD pipeline ends up installing
- [Chart Development Tips and Tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/) -- `tpl`, `required`, `default`, `lookup`, `toYaml`
- [Best Practices: Values](https://helm.sh/docs/chart_best_practices/values/) -- camelCase, flat vs nested, when to require vs default
- [Best Practices: Templates](https://helm.sh/docs/chart_best_practices/templates/) -- Naming, scope, chart-prefixed helpers for subchart safety
- [Advanced Helm Templating (Palark)](https://blog.palark.com/advanced-helm-templating/) -- Real-world `tpl`, `tplvalues.render`, production patterns
- [Chart Repository Guide](https://helm.sh/docs/topics/chart_repository/) -- Traditional HTTP repo + `index.yaml`
- [Registries (OCI)](https://helm.sh/docs/topics/registries/) -- Pointer for Day 2

**Cross-references in this repo**:
- [EKS Deep Dives -- December 2025](../../../2025/december/eks/) -- The cluster Helm ships workloads into; nine dives covering VPC CNI, IAM roles for service accounts, autoscaling, security, observability, upgrades
- [Terraform Modules](../../../2026/january/terraform/2026-01-16-terraform-modules.md) -- Parallel mental model for parameterized reuse at the infrastructure layer; Helm is to Kubernetes manifests what Terraform modules are to cloud resources
- [Terraform Variables & Validation](../../../2026/january/terraform/2026-01-23-terraform-variables-validation.md) -- Compare Helm `required`/`default` with Terraform variable `validation` blocks and `default`
- [Terraform Project Structure](../../../2026/january/terraform/2026-02-02-terraform-project-structure.md) -- Environment-specific values files (`values-dev.yaml`, `values-prod.yaml`) mirror Terraform's `terraform.tfvars` per-environment pattern
