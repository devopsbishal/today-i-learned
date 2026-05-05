# GitHub Actions Core Review & Advanced Workflows -- The Automated Factory Floor

> The last six weeks built the runtime stack: Helm packaged the apps, ArgoCD synced them from Git, Prometheus and Loki and Tempo watched them run, and SLOs gave the org a contract for what "running well" means. **But none of those tools build the artifact in the first place.** Something has to read the developer's `git push`, lint the code, run the tests, build the container, scan it, push it to a registry, and bump an image tag in a config repo so ArgoCD has something new to reconcile. That something is GitHub Actions -- and starting today, the syllabus turns from "how do you operate a deployed system" to "how do you get a change *to* the deployed system safely, repeatably, and quickly." Week 6.8 is the CI/CD week, and Day 1 is the foundation: workflow mechanics, contexts, conditionals, concurrency, reusable workflows, and matrix strategy. Get these wrong and every later topic (OIDC, ECR push, ArgoCD integration, environment gates) inherits the bug.
>
> **The core analogy for the day: a GitHub Actions workflow is an automated factory floor.** A `git push` (or a cron tick, or a button click) is the **work order** that triggers production. The **workflow file** is the assembly-line schedule that says, in order, which stations the part visits and in what dependency. **Jobs** are the stations on the line -- a build station, a test station, a scan station, a deploy station -- and each job runs on its own **runner**, which is a *fresh, sterile workbench* that's spun up empty for that one job and then thrown away. **Steps** are the tasks performed at each station: "checkout the part," "run the lathe," "QA-inspect," "stamp the date." Parts that need to move between stations are **artifacts** (uploaded by one job, downloaded by another) or **outputs** (small bits of metadata passed by string). The **contexts** (`github`, `env`, `vars`, `secrets`, `needs`, `matrix`, `runner`, `inputs`, `steps`, `job`, `strategy`, `jobs`) are the **instruction binders** posted on the wall at each station -- some binders ("the original work order") are at every station, but others ("the previous station's output ticket," "this station's matrix cell coordinates") only exist where the station has been told to look at them. Look in the wrong binder at the wrong station and you read empty pages without an error. **Concurrency groups** are the traffic-control rules at shared loading docks -- one work order at a time on the prod-deploy dock, but multiple PR-validation work orders on the CI dock are fine *as long as the new one cancels the stale one*. **Reusable workflows** are the corporate library of pre-approved assembly-line templates a plant can pull off the shelf instead of designing every line from scratch; **composite actions** are pre-approved *station setups* (the standard "checkout + setup-node + cache + install" four-step bundle) you can drop into any line. The 10-deep nesting limit (raised from 4 to 10 by GitHub in 2024, with a hard cap of 50 unique reusable workflows reachable from any single top-level run) is the rule that you can't stack templates of templates indefinitely -- the corporate library has a deep shelf, but not an infinite one.
>
> Three corollaries this analogy carries that show up in every section below. **(1) Each station is a fresh workbench, not a shared workshop.** This is the single biggest mental shift from Jenkins or CircleCI users: jobs do *not* share filesystem state. If Job A clones the repo and builds a binary, Job B starts on a brand-new VM with nothing on it. To pass anything from A to B you must use **artifacts** (whole files), **outputs** (small strings up to 1 MB), or external storage. Forgetting this is the #1 reason people write 200-step single jobs that run an hour -- they're afraid to split because they can't intuit what crosses station boundaries. **(2) The wrong instruction binder is invisible, not loud.** GitHub Actions silently coerces `${{ matrix.foo }}` to an empty string when you reference `matrix` in a non-matrix job, silently coerces `${{ secrets.X }}` to empty in `if:` filters at the trigger level, and silently coerces `${{ needs.upstream.outputs.tag }}` to `"42"` (a string) when the upstream emitted the integer 42. Every "my workflow runs but does the wrong thing" debugging session ends in "I read the wrong binder at the wrong station." **(3) The traffic controller defaults are tuned for demos, not production.** `cancel-in-progress` defaults to false, `timeout-minutes` to 360, matrix `fail-fast` to true -- override deliberately on every production workflow, or discover the consequences at the worst time.

---

**Date**: 2026-05-05
**Topic Area**: cicd
**Difficulty**: Intermediate -> Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Workflow | An assembly-line schedule | One YAML file in `.github/workflows/`, triggered by an event |
| Job | A station on the line | Independent unit of work; gets its own runner |
| Step | A task at a station | A shell command or a `uses:` action invocation |
| Runner | A fresh sterile workbench | The VM/container that hosts a job; thrown away after |
| Job-per-runner | One workbench per station | Jobs do NOT share filesystem; pass data via artifacts/outputs |
| Action | A pre-built power tool | Reusable unit (`uses: actions/checkout@v4`); JS, Docker, or composite |
| Event trigger | The work order that starts production | `push`, `pull_request`, `schedule`, `workflow_dispatch`, `repository_dispatch`, `workflow_run`, `workflow_call` |
| `workflow_dispatch` | The "manual run" button | Human-triggered with up to 25 typed inputs (Dec 2025 limit) |
| `repository_dispatch` | An external API kicking off a run | POST to `/repos/.../dispatches` with `client_payload` JSON |
| `pull_request_target` | A foot-gun trigger | Runs against base repo with secrets -- dangerous on fork PRs |
| Context | Instruction binder posted at a station | `${{ ... }}` namespace: `github`, `env`, `vars`, `secrets`, etc. |
| `github.ref` vs `github.ref_name` | Long form vs friendly name | `refs/heads/main` vs `main` -- both, different scopes |
| `github.head_ref` | The PR's source branch label | Only set on `pull_request` events; empty on push |
| `env` vs `vars` | Job-scoped variable vs repo-config variable | `env` set in YAML; `vars` set in repo settings (added 2023) |
| `secrets` | Locked binder, station-only | NOT available in `if:` at workflow trigger level |
| `needs` | The shipping manifest from upstream stations | Declares DAG dependency + grants access to upstream outputs |
| Matrix | Cloning the line for parallel parts | `strategy.matrix` cross-product of dimensions |
| `if:` | "Run only if..." filter | Conditional execution; works at job and step level |
| `success()`/`failure()`/`always()`/`cancelled()` | Status check functions | The four conditional foundations; `always()` makes jobs uncancellable |
| `if: !cancelled()` | "Run on success or failure but respect Cancel" | The right replacement for `if: always()` in cleanup jobs |
| Concurrency group | Traffic control at a loading dock | Serializes runs sharing a key |
| `cancel-in-progress: true` | "Stop the stale work order" | For PR CI -- new push cancels in-flight run |
| `cancel-in-progress: false` | "Queue, don't interrupt" | For deploys -- canceling mid-deploy corrupts state |
| `timeout-minutes` | The kill-switch on a station | Default 360 is a billing trap; always override |
| `continue-on-error` | "Don't fail the whole line on this step" | Step-level resilience; rare and deliberate |
| Reusable workflow | Corporate library of full assembly lines | `on.workflow_call`; called by `uses: org/repo/.github/workflows/x.yml@sha` |
| Composite action | Pre-built station setup | A bundle of steps; runs on the caller's runner |
| `secrets: inherit` | "Use the parent factory's locked binders" | Caller passes ALL its secrets to a reusable workflow |
| `fromJson` | Decoder for serialized artifacts | Turns a JSON-string output into a real object/array |
| Dynamic matrix | Generated assembly schedule | `strategy.matrix: ${{ fromJson(needs.setup.outputs.matrix) }}` |
| `GITHUB_OUTPUT` | The output ticket file | Replaces deprecated `::set-output::` |
| `GITHUB_ENV` | The job-environment file | Replaces deprecated `::set-env::` |
| Pinning by SHA | Welding a specific tool to the wall | Required for production; tags are mutable |
| 10-deep nesting limit | Library policy on stacked templates | Reusable workflows nest up to 10 levels deep / 50 unique workflows total per top-level run (raised from 4 in 2024) |

---

## The Whole Picture in One Diagram

Before the parts: the entire workflow execution model on one page.

```
THE GITHUB ACTIONS FACTORY FLOOR
================================================================================

EVENT (work order)                         GITHUB SERVERS
+---------------------+                    +----------------------+
| git push            |                    | Workflow scheduler   |
| pull_request opened |  ---webhook--->    | reads .github/       |
| cron tick           |                    | workflows/*.yml      |
| workflow_dispatch   |                    | matches `on:` filter |
| repository_dispatch |                    +----------+-----------+
| workflow_run        |                               |
| workflow_call       |                               v
+---------------------+                    +----------------------+
                                           | Concurrency check    |
                                           | (group + cancel-in-  |
                                           |  progress evaluated) |
                                           +----------+-----------+
                                                      |
                                                      v
                                           +----------------------+
                                           | Job DAG construction |
                                           | (needs: edges)       |
                                           +----------+-----------+
                                                      |
        +---------------------------+-----------------+----------------+
        |                           |                                  |
        v                           v                                  v
+---------------+          +----------------+               +-----------------+
| Job: lint     |          | Job: test      | (matrix)      | Job: build      |
| runner: fresh |          | runs in 6 cells|               | needs: [lint,   |
| ubuntu-latest |          | os x node x py |               |        test]    |
+-------+-------+          +--------+-------+               +--------+--------+
        |                           |                                |
        |  artifacts upload         |  outputs (tag, version)        |  artifacts download
        |  (whole files)            |  (string only, <=1MB)          |
        +---------------+-----------+----------------+---------------+
                        |                            |
                        v                            v
               +-----------------+         +-----------------------+
               | Concurrency     |         | environment: prod     |
               | group: deploy-  |         | required-reviewers    |
               | prod queues 1   |         | gate fires            |
               +--------+--------+         +-----------+-----------+
                        |                              |
                        +-------------+----------------+
                                      v
                            +-------------------+
                            | Job: deploy       |
                            | uses: reusable    |
                            | workflow @SHA     |
                            | secrets: inherit  |
                            +-------------------+

CONTEXT AVAILABILITY (which binders exist where)
================================================================================
                       trigger    workflow   job-level   step-level
                       (on:)      env/vars   if:/env     run/with
github.*               YES        YES        YES         YES
env.*                  NO         YES        YES         YES
vars.*                 YES (some) YES        YES         YES
secrets.*              NO         WORKFLOW   YES         YES
needs.*                NO         NO         YES (if needs:) YES
matrix.*               NO         NO         YES (matrix only) YES (matrix only)
inputs.*               NO         YES (if workflow_dispatch/call) YES YES
runner.*               NO         NO         NO          YES
steps.*                NO         NO         NO          YES (after step runs)
strategy.*             NO         NO         YES (matrix only) YES (matrix only)
job.*                  NO         NO         NO          YES
jobs.*                 NO         NO         workflow_call output only

THE LIFECYCLE OF DATA
================================================================================
within a job: filesystem persists step-to-step  (same workbench, same parts)
across jobs:  filesystem is GONE                (new workbench, sterile)
              -> use upload-artifact / download-artifact for files
              -> use outputs (<= 1MB string per output) for metadata
              -> outputs are ALWAYS strings; numbers/JSON come back as strings
```

Six things to remember as we go:

1. **The runner is a fresh workbench.** Every job starts empty. Anything you want a downstream job to see must be either uploaded as an artifact or set as a job output.
2. **Contexts are scoped by station, and the wrong scope returns empty silently.** If `${{ matrix.os }}` is mysteriously blank, you're in a job without `strategy.matrix`. If `${{ secrets.X }}` is empty in `on.push.if:`, that's by design -- secrets aren't available there.
3. **Defaults are tuned for the demo.** 360-minute timeout, fail-fast=true on matrix, no concurrency, ubuntu-latest pinned to a moving target. Override deliberately on every production workflow.
4. **`always()` makes a job uncancellable.** Cleanup jobs that use `if: always()` will run even when the user clicks "Cancel workflow." Use `if: !cancelled()` instead.
5. **Concurrency cancel-in-progress is the wrong default for deploys.** True for PR CI (saves minutes); false for prod deploys (canceling mid-deploy corrupts state).
6. **Pin actions by SHA in production.** Tags are mutable. The Tj-actions/changed-files supply-chain incident on **March 14-15, 2025 (CVE-2025-30066)** is the canonical reason -- a popular action's tag was repointed at a malicious commit, exfiltrating secrets from approximately 23,000 repos that consumed it during the compromise window.

---

## Part 1: The Workflow / Job / Step Model

A **workflow** is a single YAML file in `.github/workflows/` -- one workflow per file. The file is loaded fresh on every event; you cannot have two `name:` keys at the workflow level pointing at the same workflow. The structure is rigid:

```yaml
name: CI                            # Display name in the UI
on:                                 # Event triggers (the work orders)
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:                        # GITHUB_TOKEN scopes -- default to read-only
  contents: read
  pull-requests: write              # Grant per-job as needed

env:                                # Workflow-level env vars
  REGISTRY: ghcr.io

concurrency:                        # Traffic control
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:                             # Job ID (used in `needs:`)
    name: Lint                      # Display name
    runs-on: ubuntu-latest          # Runner label
    timeout-minutes: 10             # Override the 360-min default
    steps:
      - uses: actions/checkout@a1b2c3d  # Pin by SHA in prod
      - run: npm ci
      - run: npm run lint
```

### Jobs run on isolated runners

Every job gets its **own runner** -- a freshly provisioned Ubuntu/macOS/Windows VM (for GitHub-hosted runners) or container (for self-hosted ephemeral runners). When the job ends, that runner is destroyed. **Two jobs in the same workflow do not share a filesystem**, even if they run on the same runner *label*. This is the single most common confusion for engineers coming from Jenkins (which has shared workspaces by default) or CircleCI (which has `persist_to_workspace`).

The factory analogy is exact: each station has its own workbench, sterilized between work orders. A part you want at the next station must be **physically moved** -- which in GitHub Actions means one of three things:

- **Artifacts** (`actions/upload-artifact` / `actions/download-artifact`): whole files or directories. Persisted in GitHub-managed storage for the workflow run; default retention 90 days, configurable per workflow up to 400 days. Use for build outputs, test reports, coverage files, container image tarballs.
- **Outputs** (`echo "key=val" >> $GITHUB_OUTPUT` -> `${{ steps.X.outputs.Y }}` -> `jobs.<id>.outputs.<name>` -> `${{ needs.<id>.outputs.<name> }}`): small string values, up to 1 MB per individual output with a 50 MB cumulative ceiling across all step outputs in a single job. Use for metadata: image tag, version string, "did anything change in the pkg directory" boolean, JSON-encoded matrix.
- **External storage** (S3, ECR, GHCR, a database): for large mutable state that survives across workflow runs, not just across jobs in one run.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tag }}        # Job output declared here
    steps:
      - uses: actions/checkout@a1b2c3d
      - id: meta
        run: |
          TAG="$(git rev-parse --short HEAD)"
          echo "tag=${TAG}" >> $GITHUB_OUTPUT          # Step output set here
      - run: docker build -t app:${{ steps.meta.outputs.tag }} .
      - run: docker save app:${{ steps.meta.outputs.tag }} -o image.tar
      - uses: actions/upload-artifact@d1e2f3a
        with:
          name: image
          path: image.tar
          retention-days: 7

  deploy:
    needs: build                                       # DAG edge
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@b2c3d4e
        with:
          name: image
      - run: docker load -i image.tar
      - run: |
          # Note the four-step path: step.output -> job.output -> needs.X.outputs -> here
          echo "Deploying tag ${{ needs.build.outputs.image-tag }}"
```

Two gotchas to internalize from this snippet:
1. The image tag is a **string** when it comes back through `needs.build.outputs.image-tag`. If you echoed `42` (an integer) into `$GITHUB_OUTPUT`, the downstream sees the four-character string `"42"`, not a number. Comparisons with `==` work because YAML expressions string-coerce, but arithmetic doesn't.
2. Both upload and download must reference the same **artifact name** (here, `image`). A typo means the download silently produces an empty directory and the next step fails with "image.tar: no such file."

### Steps share state within a job

Steps in the same job run sequentially on the same runner. Filesystem changes from one step persist to the next (until the runner dies at end-of-job). Environment variables set with `echo "FOO=bar" >> $GITHUB_ENV` in step 1 are visible to step 3 as `$FOO` and as `${{ env.FOO }}`. PATH additions via `$GITHUB_PATH` work the same way.

This is *not* the case for plain shell exports: `export FOO=bar` in step 1 dies when the shell exits, so step 2 sees nothing. Anything you want to survive past a single `run:` block must go through the env file.

---

## Part 2: Event Triggers -- The Work Orders

The `on:` block is what kicks the workflow off. Eight production-relevant triggers:

### `push` -- the bread-and-butter trigger

```yaml
on:
  push:
    branches: [main, 'release/**']
    branches-ignore: [renovate/**]
    paths: ['src/**', 'package.json']
    paths-ignore: ['docs/**', '*.md']
    tags: ['v*']
```

Filters are AND-ed within a key (must match `branches` AND `paths`) and OR-ed across patterns within a key (matches `main` OR `release/**`). **Important**: a single trigger cannot use both `branches` and `tags` simultaneously -- a tag push doesn't have a branch context, so the workflow is split into two `push:` blocks or `branches: [...]` is dropped when `tags:` is present.

### `pull_request` and `pull_request_target` -- the dangerous twins

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]
    paths-ignore: ['*.md']
```

`pull_request` runs against the **PR's merge ref** (a hypothetical merge of the PR head into the base) and -- critically -- **does not have access to secrets when the PR comes from a fork**. This is the security boundary: an attacker can fork your repo, modify your CI workflow to `cat $AWS_SECRET_ACCESS_KEY`, and open a PR. If the workflow ran with secrets, every fork PR would be a credential-exfiltration vector. So GitHub disables secrets for fork-PR workflows.

`pull_request_target` is the escape hatch -- it runs against the **base repo's workflow file** (so the PR can't modify it) and **does have access to secrets**. The legitimate use cases are: auto-labeling, welcome-comment bots, security-scan jobs that need a token. The illegitimate (and dangerous) use case is: building/testing the PR's code with secrets present. **Doing this on a fork PR runs untrusted code with your prod credentials.** The Dependabot-related supply-chain hardening that GitHub rolled out in 2021-2022 was largely about closing this hole; the foot-gun is still wide open if you write a `pull_request_target` workflow that does `actions/checkout@... ref: ${{ github.event.pull_request.head.sha }}`. And the danger isn't just `run:` blocks -- `npm install` (postinstall hooks), `pip install -e .` (executes setup.py), `docker build` (arbitrary Dockerfile), and even some linters (ESLint configs that execute JS, jest configs) all run attacker-controlled code if you check out the PR head with secrets in scope. The rule: if you check out the PR head under `pull_request_target`, assume the PR can do anything your workflow's identity can.

The mental model: `pull_request` = "test this contributor's code, no secrets, safe"; `pull_request_target` = "label/comment on the PR using my credentials, but never execute the PR's code." If you need both -- run tests on PR code AND need a secret for a private package registry -- the production answer is the **two-workflow split**: a `pull_request` workflow that runs untrusted code without secrets, and a separate `pull_request_target` workflow that handles labeling/commenting with secrets but runs only base-repo code.

### `schedule` -- the cron tick

```yaml
on:
  schedule:
    - cron: '0 9 * * 1-5'   # 9 AM UTC Mon-Fri
```

Five-field POSIX cron, **always UTC**. Runs only on the **default branch** -- there's no way to schedule a workflow that exists on a non-main branch. GitHub explicitly warns that scheduled workflows can be **delayed during high load** (especially on the hour and half-hour boundaries -- avoid `0 *` and `30 *` if punctuality matters; offset to `:07` or `:23`). And: scheduled workflows on **repositories with no activity for 60 days** are automatically disabled. **Any commit** to the repo wakes them back up -- you do not need `gh workflow enable` for the inactivity case. `gh workflow enable` is the right tool only when the workflow itself was explicitly disabled (a separate state from repo inactivity, set via the UI or by `gh workflow disable`).

### `workflow_dispatch` -- the manual button (with typed inputs)

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: environment        # Special type: only accepts configured environments
      version:
        description: 'Version to deploy'
        required: true
        type: string
        default: 'latest'
      skip_tests:
        description: 'Skip the test stage'
        required: false
        type: boolean
        default: false
      log_level:
        description: 'Log verbosity'
        required: true
        type: choice
        options: [debug, info, warn, error]
        default: info
      replicas:
        description: 'Replica count'
        required: true
        type: number
        default: 3
```

As of December 2025, the limit is **25 inputs per workflow** (raised from 10 in mid-2023). Five input types: `string`, `boolean`, `number`, `choice` (a dropdown), and `environment` (auto-populated with the repo's configured environments). Access them as `${{ inputs.environment }}` (or `${{ github.event.inputs.environment }}` -- the older form, still works). **Booleans are tricky**: `${{ inputs.skip_tests }}` delivers the string `'true'` or `'false'`, not an actual boolean. Use `if: ${{ inputs.skip_tests }}` (truthy coercion handles both forms) -- never `if: ${{ inputs.skip_tests == false }}` (compares string to boolean, always false).

`workflow_dispatch` runs the workflow file **on the branch you trigger from** -- which is invisible in the UI but exposed via `github.ref`. A dispatch run from a feature branch runs that branch's workflow file, not main's. Useful for testing changes; surprising for ops folks who expect prod scripts to come from the prod branch.

### `repository_dispatch` -- the external API trigger

```yaml
on:
  repository_dispatch:
    types: [deploy-from-jenkins, security-scan-completed]
```

Triggered by `POST /repos/<owner>/<repo>/dispatches` with a JSON body:
```json
{ "event_type": "deploy-from-jenkins",
  "client_payload": { "version": "1.2.3", "env": "staging" } }
```

The payload is available as `${{ github.event.client_payload.version }}`. **Repo-scoped, not branch-scoped** -- it runs the workflow file from the default branch. Use cases: an external CI system (Jenkins, GitLab) finishes a build and asks GHA to deploy; a webhook from a third-party (Sentry, Datadog) opens an incident-response workflow; a Lambda kicks off a nightly sync.

### `workflow_run` -- chaining workflows

```yaml
on:
  workflow_run:
    workflows: ['CI']
    types: [completed]
    branches: [main]
```

Runs after another workflow completes. Useful for the "build separately from deploy" pattern -- the Build workflow runs on every push, and a Deploy workflow listens for the build to succeed and then handles the actual rollout. The catch: `workflow_run` triggered runs **don't show up as PR checks**, and the listening workflow runs from the default branch, not the branch the upstream workflow ran on. Often the cleaner pattern is multi-job within one workflow with `needs:` rather than two workflows linked by `workflow_run`.

### `workflow_call` -- the reusable-workflow trigger

Covered in detail in Part 8. Mentioned here for completeness: a workflow with `on: workflow_call` is callable by another workflow, not directly by a Git event.

---

## Part 3: The Eight (Twelve) Contexts -- Instruction Binders

This is THE source of "expression evaluates to empty" bugs. The contexts officially numbered as eight (`github`, `env`, `vars`, `secrets`, `needs`, `matrix`, `runner`, `inputs`) plus four more (`steps`, `job`, `strategy`, `jobs`) are the namespaces you read inside `${{ ... }}`. Each is available in some scopes and silently empty in others.

### Availability matrix (the canonical reference)

| Context | `on:` (trigger filters) | `env:`/`defaults:` (workflow) | `concurrency:` (workflow) | `if:`/`env:` (job) | `if:`/`env:`/`run:`/`with:` (step) | `name:` |
|---------|-----|-----|-----|-----|-----|-----|
| `github.*` | NO | YES | YES | YES | YES | YES |
| `env.*` | NO | NO | NO | YES | YES | YES |
| `vars.*` | YES (limited) | YES | YES | YES | YES | YES |
| `secrets.*` | NO | NO (workflow level only after declaration) | NO | YES | YES | NO |
| `needs.*` | NO | NO | NO | YES (job has `needs:`) | YES (same constraint) | YES |
| `matrix.*` | NO | NO | NO | YES (matrix job only) | YES (matrix job only) | YES |
| `inputs.*` | NO | YES (if `workflow_dispatch`/`workflow_call`) | YES | YES | YES | YES |
| `runner.*` | NO | NO | NO | NO | YES | NO |
| `steps.*` | NO | NO | NO | NO | YES (after step runs) | NO |
| `job.*` | NO | NO | NO | NO | YES | NO |
| `strategy.*` | NO | NO | NO | YES (matrix) | YES (matrix) | YES |
| `jobs.*` | NO | NO | NO | NO | NO | `workflow_call` outputs only |

Three rules that explain 80% of context bugs:

1. **`secrets` are not available in trigger filters.** `if: ${{ secrets.DEPLOY_KEY != '' }}` at the workflow level (`on.push.if:`) doesn't exist -- there is no such field. Even if there were, secrets wouldn't be evaluated there. Move secret-presence checks to job-level `if:`.
2. **`matrix` only exists inside a matrix job.** Referencing `matrix.os` from a non-matrix job returns empty. There's no error, no warning, just a silently empty string -- which often results in "deploy to environment ''" running with no env at all, which then matches *everything* in your environment-specific code paths.
3. **`steps.<id>.outputs.<name>` only exists after that step has run.** Referencing it in an earlier step's `if:` returns empty. If the step is conditionally skipped, its outputs are also empty. Use `if: ${{ steps.X.outcome == 'success' }}` to gate on whether a prior step ran AND succeeded.

### `github.ref` vs `github.ref_name` vs `github.head_ref` vs `github.base_ref`

The four ref-related fields, each subtly different:

| Field | On `push` to main | On PR from feature->main | On `pull_request_target` from fork:feat -> base:main |
|-------|------|------|------|
| `github.ref` | `refs/heads/main` | `refs/pull/42/merge` | `refs/pull/42/merge` |
| `github.ref_name` | `main` | `42/merge` | `42/merge` |
| `github.head_ref` | `''` | `feature` | `feature` |
| `github.base_ref` | `''` | `main` | `main` |
| `github.sha` | tip of main | merge-commit SHA | merge-commit SHA |

Implications:
- For PR checks where you want to display the **source branch name**, use `github.head_ref`, not `github.ref_name` (which gives you `42/merge`).
- For "is this main?" checks, use `github.ref == 'refs/heads/main'` (full form) or `github.ref_name == 'main'` (short form). Both work; pick one project-wide for consistency.
- `github.head_ref` is empty on push events. `if: github.head_ref == ''` is a working "is this a push, not a PR?" check, though `github.event_name == 'push'` is clearer.
- The `github.event` namespace is the raw webhook payload. `github.event_name` is `push`, `pull_request`, etc. `github.event.action` is the **subtype** -- `opened`, `synchronize`, `closed` for PRs; not present for push.

### `env` vs `vars` -- the 2023 split

For years GitHub Actions had two confusing concepts under one name: workflow-defined env vars (set in YAML) and repository configuration variables (set in repo settings, were stored as secrets). In 2023 GitHub split them:

- **`env`**: workflow/job/step env vars set in YAML. Lexically scoped, immediate; written by `env:` blocks or by `echo "FOO=bar" >> $GITHUB_ENV`.
- **`vars`**: repository-, organization-, or environment-level **configuration variables** (non-secret), set in repo settings. Read-only from the workflow's perspective. Use `${{ vars.AWS_REGION }}` for non-secret config that varies between environments.

The rule of thumb: **use `secrets` for things that must not appear in logs, `vars` for non-secret config that varies between repos/environments, `env` for workflow-internal values you compute on the fly.**

### `secrets` -- locked binders, station-only

Secrets are scoped to repo, organization, or environment. Access is `${{ secrets.MY_SECRET }}`. Critical rules:

- **Not available in trigger filters or `on:` `if:`.** Move secret-dependent gates to job level.
- **Auto-redacted in logs.** Anything matching a secret value is replaced with `***`. This works on prefix matches too -- if your secret is `abc123` and a log line is `Token: abc1234567`, you'll see `Token: ***4567` because the redactor matched the prefix.
- **Not passed to fork PRs by default** (the `pull_request_target` boundary).
- **`GITHUB_TOKEN` is auto-generated per workflow run** and expires when the run ends. It's an installation token scoped by the `permissions:` block.

A common pattern: gate a step on whether a secret is present.
```yaml
- name: Push to ECR
  if: ${{ env.AWS_ROLE_TO_ASSUME != '' }}    # Read via env, not secrets
  env:
    AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
  run: |
    aws ecr get-login-password ...
```
Reading `secrets.X` directly inside `if:` works at the **job level**, but you can't tell from the log whether the secret was empty or the comparison failed -- the redactor turns it into `***`. The env-pivot pattern makes the gate visible.

### `needs.<job>.outputs.<name>` -- the upstream shipping manifest

Set up via:
```yaml
jobs:
  upstream:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tag }}     # Map step output -> job output
      changed: ${{ steps.diff.outputs.changed }}
    steps:
      - id: meta
        run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
      - id: diff
        run: echo "changed=true" >> $GITHUB_OUTPUT

  downstream:
    needs: upstream
    runs-on: ubuntu-latest
    if: ${{ needs.upstream.outputs.changed == 'true' }}    # String comparison!
    steps:
      - run: echo "Deploying ${{ needs.upstream.outputs.tag }}"
```

Two gotchas:
1. **All outputs are strings.** Even if you echo `42` or a JSON object. Comparisons must use string semantics: `'true'`, `'false'`, `'42'`. JSON outputs require `${{ fromJson(needs.X.outputs.matrix) }}` to deserialize.
2. **If the upstream job is skipped, its outputs are empty.** A downstream job conditioned on `needs.upstream.outputs.tag` will see an empty string. Combined with `if: success()` (which fails when needs is skipped), this often surprises -- the downstream gets skipped because of the success() check, but if you'd written `if: !failure()` you'd run with empty inputs.

### `matrix` -- only inside a matrix job

`matrix.os`, `matrix.node`, etc., only resolve inside a job that has `strategy.matrix`. From a downstream job:
```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: [20, 22]
    runs-on: ${{ matrix.os }}                     # Works
    steps:
      - run: node --version
        env:
          NODE_VERSION: ${{ matrix.node }}        # Works

  summary:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ matrix.os }}"               # EMPTY -- not a matrix job
```

---

## Part 4: Conditional Execution -- `if:` and the Status Check Trifecta

Every job and every step can carry an `if:`. The expression is evaluated at scheduling time (job-level) or at step-evaluation time (step-level). Four built-in **status check functions** drive most non-trivial conditionals:

| Function | Returns true when |
|---|---|
| `success()` | All upstream `needs:` succeeded (default if `if:` omitted) |
| `failure()` | Any upstream `needs:` failed (transitively) |
| `cancelled()` | The workflow run was cancelled |
| `always()` | ALWAYS -- including on cancel |

**All four functions evaluate against the DAG state of the upstream `needs:`, not the workflow as a whole.** A job with no `needs:` only sees the workflow-level cancel state; `success()` and `failure()` evaluate trivially. A job with `needs: [a, b]` evaluates the union of `a`'s and `b`'s outcomes (and their transitive `needs:`). The transitivity matters: if `a` was skipped because `c` (which `a` needed) failed, then a downstream `if: failure()` on a job that needs `b` (which transitively needed `c`) sees the chain failure too.

**The `always()` foot-gun**: a job with `if: always()` is *uncancellable*. A user clicking "Cancel workflow" cannot stop it. For a 2-minute notify job that's fine. For a 30-minute integration test you want to skip on cancel, `always()` is wrong. The correct pattern for "run regardless of upstream success/failure but respect Cancel" is **`if: !cancelled()`** (boolean-not on the function).

**The `success()` skipped-needs trap**: `success()` returns *false* if any upstream `needs:` job was *skipped*, even though nothing failed. So a downstream cleanup job with the implicit `if: success()` will be skipped when an upstream conditional was false. The fix is `if: !failure()` (or more explicit `if: !failure() && !cancelled()`), which reads as "run unless something actually failed."

The truth table for a downstream job, depending on how you write the conditional:

| Upstream state | (no `if:`) | `if: success()` | `if: failure()` | `if: always()` | `if: !cancelled()` | `if: !failure()` |
|----|----|----|----|----|----|----|
| Success | run | run | skip | run | run | run |
| Failure | skip | skip | run | run | run | skip |
| Skipped | skip | skip | skip | run | run | run |
| Cancelled | skip | skip | skip | **run** | skip | skip |

The `if: !cancelled() && (success() || failure())` combo is what most teams want for "always run cleanup, but respect Cancel."

### `${{ }}` in `if:` -- when you need it and when you don't

Both work:
```yaml
if: github.ref == 'refs/heads/main'
if: ${{ github.ref == 'refs/heads/main' }}
```

But this **does not work**:
```yaml
if: always() && github.event_name == 'push'    # WRONG -- parsed as literal
```

When you mix a status function with another expression, the parser needs the explicit `${{ }}`:
```yaml
if: ${{ always() && github.event_name == 'push' }}    # Correct
```

The rule: **bare `if:` works for simple status functions or simple `context.field op value` comparisons, but the moment you combine them or use parentheses, wrap with `${{ }}`.** When in doubt, always wrap -- it never hurts.

### Operator precedence

GitHub Actions expressions support `&&`, `||`, `!`, `==`, `!=`, `>`, `<`, `>=`, `<=`. Precedence (high to low): `!`, `< <= > >=`, `== !=`, `&&`, `||`. Parens are mandatory if you want anything other than left-to-right evaluation of `&&`/`||`:

```yaml
if: ${{ (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/main' }}
```

Without the parens around the OR, `&&` binds tighter and you get a different truth table -- a classic source of "the workflow ran on a tag push when I thought it wouldn't" surprises.

---

## Part 5: Concurrency Groups -- Traffic Control at Shared Docks

Without `concurrency:`, every workflow event gets its own run, all in parallel. For most CI this is fine. For two scenarios it's actively dangerous: **stale PR runs eating runner minutes** and **concurrent deploys corrupting state**.

### Pattern 1: PR auto-cancel (cancel-in-progress: true)

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

When a developer pushes commits 1, 2, 3 to a PR in quick succession, the runs for commits 1 and 2 are cancelled the moment runs 2 and 3 start. Only the latest run reaches completion. Saves runner minutes (CI runs are expensive at scale) and gives the developer feedback only on the latest commit (which is what they care about).

The `group` key is a string -- here, `ci-CI-refs/pull/42/merge`. It must include `${{ github.ref }}` (or `github.head_ref`) to **scope per-PR** -- without it, your entire org's CI cancels itself any time anyone pushes anywhere.

### Pattern 2: Deploy serialization (cancel-in-progress: false)

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

Two simultaneous prod deploys would be a disaster -- one's `kubectl apply` racing the other's, image tags flipping mid-rollout, ArgoCD seeing alternating manifests. So they serialize: deploy A runs to completion, deploy B waits in the queue. **Crucially `cancel-in-progress: false`** -- if you set it to `true`, deploy B *cancels* deploy A halfway through, leaving the cluster in a partial-rollout state.

GitHub's queue depth is **1** -- only one waiting run is held. If a third deploy is requested while A is running and B is queued, **B is evicted** and the third becomes the new queued. This is the right behavior for production deploys (always run the most recent intent) but a surprise the first time you see it.

### Pattern 3: Per-environment serialization

```yaml
concurrency:
  group: deploy-${{ inputs.environment }}     # staging or production
  cancel-in-progress: false
```

Now staging and production have independent queues. A 30-minute staging deploy doesn't block a hotfix to production.

### Pattern 4: Per-region matrix-aware deploys

```yaml
concurrency:
  group: deploy-${{ matrix.region }}-${{ inputs.environment }}
  cancel-in-progress: false
```

Each region serializes independently. us-east-1 deploys can run while eu-west-1 is mid-rollout, but two us-east-1 deploys queue.

### Decision framework: cancel-in-progress true vs false

| Use `true` | Use `false` |
|---|---|
| PR validation runs (CI for unmerged code) | Production deploys |
| Lint/test on every push to a feature branch | Staging deploys (still want serialization) |
| Documentation builds | Database migrations |
| Preview-environment teardown | Anything that mutates external state |

**The wrong-direction pitfalls are symmetric and severe**:
- `cancel-in-progress: true` on a deploy: a re-run mid-rollout cancels production deploy halfway, you ship half the manifests, and ArgoCD sync state is now wedged between two image tags.
- `cancel-in-progress: false` on PR CI: a developer pushes 5 commits to fix a typo, and now 5 CI runs serialize in the queue, costing you 5x the runner minutes for the one result you cared about.

The mental check: **"if I cancel this run halfway through, does anything outside the workflow remember the partial work?"** If yes (deploy, migration, push to registry), use `false`. If no (test, lint, build to artifact), use `true`.

---

## Part 6: `needs` and the DAG -- Job Dependencies and Outputs

`needs:` declares a dependency edge in the job DAG. Two things happen:
1. The job doesn't start until all `needs:` are complete.
2. The job gains read access to `needs.<job>.outputs.*`, `needs.<job>.result`, `needs.<job>.outcome`.

### Fan-out and fan-in

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.list.outputs.services }}
    steps:
      - id: list
        run: echo 'services=["auth","api","web"]' >> $GITHUB_OUTPUT

  test:                                              # fan-out
    needs: setup
    strategy:
      matrix:
        service: ${{ fromJson(needs.setup.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
      - run: pytest tests/${{ matrix.service }}

  package:                                           # fan-in (needs ALL test cells)
    needs: [setup, test]
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/package-all.sh

  deploy:                                            # serial after fan-in
    needs: package
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: ./scripts/deploy.sh
```

This is a **dynamic matrix**: `setup` produces a JSON list of services, `test` consumes it via `fromJson` to expand the matrix at runtime. The number and names of test cells are determined by the upstream job, not by the YAML. This is the right pattern for "test only the changed services in a monorepo" -- a setup job runs `git diff` against the merge-base, emits `["auth"]` if only auth changed, and the test matrix shrinks to one cell.

### `needs.<job>.result` vs `needs.<job>.outcome`

Two related but different fields:
- `result`: the **final** outcome including the effect of `continue-on-error`. If a step had `continue-on-error: true` and failed, but the job otherwise succeeded, `result` is `success`.
- `outcome`: the **raw** outcome before `continue-on-error` is applied.

For most cases, `result` is what you want.

### Outputs are strings -- always

```yaml
upstream:
  outputs:
    count: ${{ steps.s.outputs.count }}
  steps:
    - id: s
      run: echo "count=42" >> $GITHUB_OUTPUT

downstream:
  needs: upstream
  steps:
    # CORRECT: string comparison or arithmetic via shell
    - if: ${{ needs.upstream.outputs.count == '42' }}
      run: echo "match"
    - run: |
        N=${{ needs.upstream.outputs.count }}
        if [ "$N" -gt 10 ]; then echo big; fi
```

Numeric and JSON outputs round-trip through string serialization. JSON requires `fromJson`:
```yaml
- run: echo "${{ fromJson(needs.upstream.outputs.config).key }}"
```

---

## Part 7: Timeouts and Continue-on-Error

### `timeout-minutes` -- default 360 is a billing trap

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30        # ALWAYS override
    steps:
      - run: ./flaky-test.sh
        timeout-minutes: 5     # Step-level too
```

The default job timeout is **360 minutes** (6 hours). A stuck job (network hang, infinite loop, deadlocked test) burns 6 hours of billed runner time before GitHub kills it. On a $7/hour large runner, that's $42 per stuck job. The fix is trivial -- override `timeout-minutes` on every job to a realistic ceiling (15 for a fast lint, 30 for a normal build, 60 for an integration test). For repos with many workflows, set defaults at the workflow level:

```yaml
jobs:
  build:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    # ...
  test:
    timeout-minutes: 45
    runs-on: ubuntu-latest
    # ...
```

(There is no workflow-level `timeout-minutes` default; it must be specified per job.)

### `continue-on-error` -- step-level resilience

```yaml
- name: Optional flake
  continue-on-error: true
  run: ./sometimes-fails.sh
- name: Always-required step
  run: echo "this still runs"
```

`continue-on-error: true` on a step means: if this step fails, the job doesn't fail -- subsequent steps still run, and the job's `result` is `success`. Use sparingly. Common legitimate uses: optional pre-warm caches, advisory linters whose findings shouldn't block, vendor dashboards whose API is flaky. **Anti-pattern**: wrapping every step in `continue-on-error: true` to silence failures -- the workflow always passes and tells you nothing.

A matrix job can also set `continue-on-error` on individual cells:

```yaml
strategy:
  fail-fast: false
  matrix:
    node: [20, 22]
    include:
      - node: 24
        experimental: true
jobs:
  test:
    continue-on-error: ${{ matrix.experimental == true }}
    # Node 24 cell can fail without failing the matrix
```

---

## Part 7.5: Production Scaffolding -- The GITHUB_TOKEN, Runners, Environments, and Defaults

A handful of facilities don't deserve their own chapter but show up in every production workflow and are missing from most tutorials. Knowing them is the difference between a workflow that *works* and a workflow that *belongs in production*.

### `GITHUB_TOKEN` and the `permissions:` block

Every workflow run gets an auto-generated **`GITHUB_TOKEN`** -- an installation token scoped to the repo, valid only for the life of the run, that's available as `${{ secrets.GITHUB_TOKEN }}` (and used implicitly by many actions). Its scope -- what API calls it's allowed to make -- is controlled by a `permissions:` block:

```yaml
permissions:
  contents: read           # checkout, read repo files
  pull-requests: write     # comment on PRs
  id-token: write          # OIDC token for cloud auth
```

Three rules to internalize:

1. **The default depends on repo settings.** GitHub flipped the default for **new repositories** to read-only across all scopes in early 2023 (Settings -> Actions -> "Workflow permissions" -> "Read repository contents and packages permissions"). Older repos may still be set to "Read and write permissions" -- the legacy default. Always check the org-level setting; never assume a default.
2. **A `permissions:` block REPLACES the default entirely; it does not merge.** Workflow-level `permissions:` overrides the repo default; job-level `permissions:` overrides the workflow-level. If you write `permissions: { contents: read }` and forget `pull-requests: write`, your `gh pr comment` step gets HTTP 403. The fix is enumerate everything you need; don't expect the missing scopes to fall back to anything.
3. **`permissions: {}` is valid and means NO permissions** -- not even `contents: read`. Useful for jobs that should literally do nothing privileged (e.g., a notify job calling an external Slack webhook). The least-privilege production pattern: declare `permissions: {}` (or `contents: read`) at the workflow level, then opt-in to the minimum at each job:

```yaml
permissions:
  contents: read           # workflow default

jobs:
  build:
    permissions:
      contents: read
      packages: write      # to push to GHCR
      id-token: write      # OIDC for AWS
  notify:
    permissions: {}        # nothing -- only calls external webhook
```

For OIDC specifically -- which Day 5 of this week covers in depth -- you need `id-token: write` to mint a token that AWS / GCP / Azure can verify against a trust policy. This is the modern alternative to long-lived `secrets.AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` pairs. For now, treat any `id-token: write` you see in this doc's examples as "OIDC plumbing -- the deep dive comes Friday."

### GitHub-hosted vs self-hosted runners

| | GitHub-hosted | Self-hosted |
|---|---|---|
| Provisioning | GitHub spins up a fresh VM per job | You register persistent or ephemeral runners |
| Cost model | Per-minute billing (free tier on public repos + paid) | Your infra bill (usually cheaper at scale) |
| Image | Opaque, pre-installed software (Node, Python, gh CLI, etc.) | You build the image; you own patching |
| Network | No static IP; fronted by GitHub's egress range | Your VPC; static IP available; can talk to private services |
| Hardware | Fixed CPU/RAM tiers; some `larger` runners on paid plans | Any hardware -- including GPUs, ARM, FPGAs |
| Security boundary | GitHub owns isolation | You own isolation |
| Setup overhead | Zero | Registration tokens, autoscaling, image baking |

**The single most important security rule for self-hosted runners**: GitHub's docs explicitly warn against running self-hosted runners on **public repositories**. A fork PR can submit a workflow change that runs on your runner -- and a *persistent* self-hosted runner means the attacker now has shell access to a machine inside your network, with whatever IAM credentials are mounted on it. The mitigations: **ephemeral self-hosted runners** (one job per runner, runner destroyed after), or restrict self-hosted runners to **private repos only**. Most production self-hosted setups use the actions-runner-controller (ARC) Helm chart on EKS, which spins up an ephemeral runner pod per job.

Self-hosted is the right answer when you need: GPU workloads, ARM-native builds, large monorepo build caches that benefit from persistent disk, network access to private services, or cost optimization at hundreds of CI minutes/day. Otherwise, hosted is simpler and the cost is rarely the problem people imagine.

**Pre-installed software vs `setup-*` actions.** GitHub-hosted runners come with Node, Python, Go, Java, Docker, gh, jq, and dozens of other tools pre-installed -- and the versions shift when GitHub bumps the runner image (which they do roughly monthly). The production rule: **always use `actions/setup-node`, `actions/setup-python`, etc. with an explicit version**, even if the version matches what's pre-installed. Otherwise your CI's behavior changes when GitHub updates the image, and you'll get a surprise breakage on a Tuesday with no commit attached.

### Environment protection rules

The `environment:` key on a job triggers GitHub's **deployment protection rules** -- the production gate that turns "this code merged" into "this code is allowed to deploy to prod":

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com    # Optional, shown in run UI as a link
    steps:
      - run: ./deploy.sh
```

What you configure in the UI under Settings -> Environments -> production:

- **Required reviewers** (1-6 named users or teams). The job pauses with status "waiting for review" until an approver clicks Approve. The approver **cannot be the actor who triggered the run** (no self-approving).
- **Wait timer** (0-43200 minutes, max 30 days). Forces a delay between approval and execution. Useful for change-management windows ("approved Monday, deploys Wednesday at 9 AM EST").
- **Deployment branches** (allowlist of branches/tags that can deploy here). A workflow run from a feature branch with `environment: production` is **rejected at job-scheduling time** if the branch isn't in the allowlist. This is the "only main can deploy to prod" enforcement mechanism.
- **Environment-scoped secrets and variables.** Override repo-level. The `prod` environment can have its own `AWS_ROLE_TO_ASSUME` distinct from `staging`'s, with no risk of cross-environment leakage. Combined with branch allowlists, this is the production-grade isolation between staging and prod credentials.
- **Custom protection rules** (Enterprise). Webhook-based gates that an external system can approve or deny -- ServiceNow change tickets, deployment-window calendars, on-call rotation checks.

The mental model: `environment:` is the bridge between GitHub Actions and your org's change-control process. Required-reviewers + wait-timer + branch allowlist together are what makes "merging to main automatically deploys to prod" *safe* -- because main-merging triggers a deploy that *waits* for explicit human approval and refuses to run from anywhere but main. Without environments, you're either deploying without gates (cowboy) or pushing approval logic into your YAML (fragile).

### `defaults.run` -- shell and working-directory

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./services/api

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: pwd        # /home/runner/work/repo/services/api
      - run: echo "$0"  # bash
```

`defaults.run.shell` and `defaults.run.working-directory` apply to every `run:` step in the workflow (or, if scoped under a job, every step in that job). They eliminate the "every step has `shell: bash` and `working-directory: ./foo`" boilerplate that bloats real workflows. **Asymmetry to remember**: composite actions cannot use `defaults.run` -- every `run:` step in a composite must declare its own `shell:` explicitly. That's the source of "why is my composite action 5x more verbose than the equivalent inline workflow" pain -- the workflow had `defaults.run.shell: bash` doing the work invisibly.

### Workflow-level `if:` doesn't exist

Coming from Jenkins (`pipeline { when { ... } }`) or CircleCI (`when:` at config root), engineers expect a workflow-level `if:` that gates the entire run. **There is no such thing in GitHub Actions.** The closest analogs:

- **`on:` filters** (`branches`, `paths`, `tags`, event types) -- the workflow doesn't run if filters don't match
- **Workflow-level `concurrency:`** -- doesn't gate, but serializes
- **Job-level `if:` on every job** -- gates execution, but the workflow is still scheduled and shows up in the run list (jobs just all skip)
- **Job-level `if:` on the first job** with `needs:`-cascading skips -- effectively gates the rest by short-circuiting the DAG

If you find yourself wanting "skip the entire workflow when X," express it as `on:` filters when X is event-based (branches, paths), or as a job-level `if:` on the first job when X is runtime-computed (e.g., "skip if the PR is in draft state").

---

## Part 8: Reusable Workflows vs Composite Actions

This is the canonical interview question. The short version: **reusable workflows reuse jobs; composite actions reuse steps.** The longer version is a decision framework.

### Reusable workflow -- the corporate library template

Defined as a workflow file with `on: workflow_call`:

```yaml
# .github/workflows/build-and-push.yml (CALLEE)
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      environment:
        required: true
        type: string
    outputs:
      image-tag:
        description: 'Pushed image tag'
        value: ${{ jobs.build.outputs.tag }}
    secrets:
      AWS_ROLE_TO_ASSUME:
        required: true
      ECR_REGISTRY:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write       # OIDC
      contents: read
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@a1b2c3d
      - uses: aws-actions/configure-aws-credentials@b2c3d4e
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: us-east-1
      - id: meta
        run: echo "tag=$(git rev-parse --short HEAD)-${{ inputs.environment }}" >> $GITHUB_OUTPUT
      - run: |
          aws ecr get-login-password ...
          docker build -t ${{ secrets.ECR_REGISTRY }}/${{ inputs.image-name }}:${{ steps.meta.outputs.tag }} .
          docker push ${{ secrets.ECR_REGISTRY }}/${{ inputs.image-name }}:${{ steps.meta.outputs.tag }}
```

Called from another workflow:

```yaml
# .github/workflows/ci.yml (CALLER)
jobs:
  build-staging:
    uses: ./.github/workflows/build-and-push.yml         # Same repo
    with:
      image-name: my-app
      environment: staging
    secrets: inherit                                     # Pass all caller secrets

  build-prod:
    uses: org/platform/.github/workflows/build-and-push.yml@a1b2c3d   # Cross-repo, SHA-pinned
    with:
      image-name: my-app
      environment: production
    secrets:
      AWS_ROLE_TO_ASSUME: ${{ secrets.PROD_AWS_ROLE }}   # Explicit map
      ECR_REGISTRY: ${{ secrets.PROD_ECR_REGISTRY }}

  deploy:
    needs: build-prod
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build-prod.outputs.image-tag }}"
```

Five things to internalize:
1. **The reusable workflow is a whole job (or set of jobs).** It runs on its own runner; the caller does not see the steps as inline steps.
2. **`secrets: inherit`** passes ALL the caller's secrets. **Reliable within a single org.** Cross-org-within-an-enterprise is documented as supported, but in practice org-level secrets often don't propagate (community reports through 2025 still show this is broken) -- so the safe operational rule is **'same org or enumerate explicitly.'** Cross-org `secrets: inherit` to a different organization passes nothing silently; the workflow runs with empty secrets and authentication fails downstream with no error message tying the failure back to inheritance.
3. **Outputs at the workflow level** are mapped from job outputs: `outputs.image-tag.value: ${{ jobs.build.outputs.tag }}`. Two layers of mapping: step output -> job output -> workflow output.
4. **10-deep nesting limit (raised from 4 in 2024).** A top-level workflow can have at most 10 levels of nested reusable-workflow calls, with at most 50 unique reusable workflows reachable across the entire run. Most production setups use 2-3 levels (caller -> reusable -> sometimes sub-reusable); the cap matters mainly when you discover it during refactoring.
5. **Pin by SHA in production**, not `@main`. A buggy or malicious commit to the platform repo otherwise instantly affects every caller.

### Composite action -- the pre-built station setup

Defined as `.github/actions/<name>/action.yml`:

```yaml
# .github/actions/setup-node-with-cache/action.yml
name: 'Setup Node with cache'
description: 'Checkout, install Node, restore cache, run npm ci'
inputs:
  node-version:
    required: false
    default: '20'
runs:
  using: 'composite'
  steps:
    - uses: actions/checkout@a1b2c3d
    - uses: actions/setup-node@b2c3d4e
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    - run: npm ci
      shell: bash                              # MANDATORY in composite
```

Called inline from any job:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: ./.github/actions/setup-node-with-cache    # Local
        with:
          node-version: '22'
      - run: npm test
```

Things to internalize:
1. **Composite actions run on the caller's runner** -- no separate runner spin-up, no separate billing line, the steps appear inline (collapsed under the action name) in the caller's job log.
2. **`shell:` is mandatory** on every `run:` step in a composite. Forgetting it is the #1 composite-action error.
3. **Composite actions have NO `secrets:` block.** You must pass them as `inputs:` in plaintext. **Never pass real secrets this way** -- the input value appears in logs of the action's invocation. The right pattern is to read `secrets.X` in the *caller's* env and let the composite read `$ENV_VAR` indirectly.
4. **Composite actions cannot use `if:` on their internal steps from the outside.** The `if:` you write in the caller's `uses:` invocation only gates whether the entire composite runs.
5. **No `runs-on:`** -- they inherit the caller's runner.

### Decision framework: reusable workflow vs composite action

| Use reusable workflow when... | Use composite action when... |
|---|---|
| You need a different `runs-on:` (different runner labels per call) | You need 3-5 steps glued together that always run on the caller's runner |
| You need to declare and pass `secrets:` cleanly | You need a setup ritual ("checkout + setup-node + cache + install") that's identical across 20 jobs |
| You need a multi-job pipeline (fan-out, fan-in, deps) | You don't need separate runner / separate logs / separate billing |
| You need to use `environment:` with protection rules | You want the steps to appear inline in the caller's job log |
| You want CD-like isolation (build job in one place, deploy job in another) | The unit of reuse is "5 lines of YAML duplicated everywhere" |
| You want the called unit to have its own permissions block | The unit of reuse is shell glue / setup |
| You're sharing across repos in an org platform library | The unit is single-repo-only or cluster of related workflows |

The wrong choice in either direction:
- **Reusable workflow for what should have been composite**: fragments your run UI into 6 tiny jobs, each with its own runner spin-up cost (~15-30s) and its own log group, making "what happened in this CI run" 5x harder to read.
- **Composite for what should have been reusable**: forces secrets through `with:` plaintext (security risk), forbids different runners, can't use `environment:` gates, can't fan out across services.

### Decision framework: matrix vs separate jobs vs reusable-workflow-called-from-loop

You have N similar units of work to run. Three ways:

| Approach | Use when |
|----------|----------|
| **Matrix** (one job, N cells) | Cells are mostly identical; you want shared `if:`, shared timeout, shared concurrency-group, single fan-in |
| **Separate jobs** (N hand-written jobs) | Cells are quite different (different runners, different secrets, different needs DAG); only N <= 4-5 |
| **Reusable workflow called N times** | Cells are large (10+ steps each), you want them to appear as separate runs in the UI, you want different `environment:` gates per call |

**Matrix is the right choice 80% of the time.** It's the cleanest, it scales to dozens of cells, and the `include:`/`exclude:` mechanism handles edge cases. Reach for separate jobs only when matrix can't express the difference; reach for a reusable workflow loop only when each call is heavy enough to deserve its own workflow run.

---

## Part 9: Matrix Strategy -- Cloning the Line for Parallel Parts

`strategy.matrix` is a **Cartesian product** of dimensions, with the cross-product expanded into independent jobs that run in parallel:

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
    runs-on: ${{ matrix.os }}
    steps:
      - run: node --version
```

3 OS × 3 Node = **9 jobs running in parallel**, each with its own runner.

### `include:` -- two distinct meanings (the source of much confusion)

`include:` does **two different things** depending on whether the cell already exists in the cross-product:

1. **Add an extra dimension to an existing cell** (the cell has all the keys from the cross-product PLUS the extra ones from `include`):
   ```yaml
   matrix:
     os: [ubuntu-latest, macos-latest]
     node: [20, 22]
     include:
       - os: ubuntu-latest
         node: 22
         experimental: true     # adds `experimental: true` to that one cell
   ```
   This produces 4 cells total; one of them has the extra `experimental` key.

2. **Append a new combination not in the cross-product**:
   ```yaml
   matrix:
     os: [ubuntu-latest, macos-latest]
     node: [20, 22]
     include:
       - os: ubuntu-latest
         node: 24                # appends a 5th cell -- ubuntu+node24 -- not in product
   ```

The rule: if **all the keys of the include entry overlap with existing matrix keys** and at least one combination already matches, you're augmenting. Otherwise you're appending.

### `exclude:` -- removing combinations

```yaml
matrix:
  os: [ubuntu-latest, macos-latest, windows-latest]
  node: [18, 20, 22]
  exclude:
    - os: windows-latest
      node: 18                  # skip Windows + Node 18 (e.g., known bug)
```

`exclude:` reduces the cross-product. Useful for "skip this known-broken combination" without rewriting the whole matrix.

### `fail-fast` -- the production default to override

**Default is `true`**: the moment any matrix cell fails, all sibling cells are cancelled. For a release pipeline this seems sensible -- "don't waste runner minutes when the build is broken." But for cross-platform CI, it's almost always wrong: a flaky test on macOS cancels the still-passing Linux and Windows runs, hiding whether the failure was platform-specific or universal.

```yaml
strategy:
  fail-fast: false             # The right default for CI matrices
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node: [18, 20, 22]
```

The rule of thumb: **`fail-fast: false` for testing matrices** (you want to see all the failures); **`fail-fast: true` for build matrices** (no point building 6 architectures if one is failing).

### `max-parallel` -- throttling for shared resources

```yaml
strategy:
  max-parallel: 3
  matrix:
    region: [us-east-1, us-west-2, eu-west-1, eu-central-1, ap-southeast-1, ap-northeast-1]
```

Without `max-parallel`, all 6 region cells run in parallel. With `max-parallel: 3`, only 3 run at a time -- essential when each cell hits a shared rate-limited resource (DockerHub pulls, an external API, a single ECR registry).

### Dynamic matrices -- the killer pattern

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.changed.outputs.matrix }}
    steps:
      - uses: actions/checkout@a1b2c3d
        with:
          fetch-depth: 0
      - id: changed
        run: |
          # Find services with changes since main
          CHANGED=$(git diff --name-only origin/main... | awk -F/ '/^services\//{print $2}' | sort -u)
          # Emit as JSON array, [] if empty
          MATRIX=$(echo "$CHANGED" | jq -R -s -c 'split("\n") | map(select(length > 0))')
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT

  test:
    needs: setup
    if: ${{ needs.setup.outputs.services != '[]' }}      # Skip job if empty
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJson(needs.setup.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a1b2c3d
      - run: ./scripts/test-service.sh ${{ matrix.service }}
```

The `setup` job emits a JSON array of changed services; the `test` job's matrix consumes it via `fromJson`. The number and names of cells are determined at runtime. This is the right pattern for monorepos -- only test what changed, with each service in its own parallel cell.

---

## Part 10: 2026 Currency Notes -- What the Old Tutorials Get Wrong

GitHub Actions evolves slowly, but the difference between a 2020 tutorial and a 2026 production workflow is significant:

### `set-output` and `save-state` are deprecated

The old form:
```yaml
- run: echo "::set-output name=tag::$(git rev-parse --short HEAD)"
- run: echo "::save-state name=temp::$RUNNER_TEMP"
- run: echo "::set-env name=FOO::bar"
- run: echo "::add-path::/some/path"
```

The new form (file-based):
```yaml
- run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
- run: echo "temp=$RUNNER_TEMP" >> $GITHUB_STATE
- run: echo "FOO=bar" >> $GITHUB_ENV
- run: echo "/some/path" >> $GITHUB_PATH
```

Why the change: the stdout-parsing form was a log-injection vulnerability -- an attacker who controlled a log line (e.g., a malicious package emitting `::set-output name=admin::true` at install time) could set arbitrary outputs. The file-based form is parsed only from a runner-local file, immune to log injection.

The old form has been emitting deprecation warnings since October 2022. Removal has been postponed multiple times (still working as of late 2025, with louder warnings) but every workflow you write today should use the env-file form.

### Node 20 is the default for JavaScript actions

Node 16 actions have emitted deprecation warnings since late 2023. Node 20 is the current default; Node 22 is supported. If you author a JavaScript action, set `using: 'node20'` in its `action.yml`.

### `secrets: inherit` -- the reusable-workflow shortcut

Added in 2022, this passes ALL caller secrets to a reusable workflow without enumeration:
```yaml
jobs:
  call:
    uses: ./.github/workflows/build.yml
    secrets: inherit
```
Reliable within a single org. Cross-org-within-an-enterprise is documented as supported but unreliable in practice -- treat the safe rule as "same org or enumerate explicitly."

### `vars` context -- the 2023 split

Repository-, organization-, and environment-level **non-secret configuration variables** are now first-class via `${{ vars.X }}`. Previously you'd misuse `secrets` for non-secret configuration, leaking the abstraction.

### `workflow_dispatch` 25-input limit

As of December 2025, workflows can declare up to 25 typed inputs (raised from 10). Five types: `string`, `boolean`, `number`, `choice`, `environment`.

### Pin actions by SHA, not tag

Tags are mutable. The Tj-actions/changed-files supply-chain incident on **March 14-15, 2025 (CVE-2025-30066)** -- a popular third-party action's `@v44` tag was repointed at a malicious commit, exfiltrating GitHub tokens and AWS credentials from approximately 23,000 repos that consumed it during the compromise window -- is the canonical reason. Production workflows pin every `uses:` to a 40-char SHA:

```yaml
- uses: actions/checkout@a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
```

Dependabot can keep SHA-pinned actions updated, raising PRs that bump the SHA and update an inline `# v4.1.2` comment. The combination of SHA-pin + Dependabot is the production answer.

### `runs-on: ubuntu-latest` shifts underneath you

`ubuntu-latest` is a moving target. In Q4 2024 it migrated from Ubuntu 22.04 to Ubuntu 24.04. Workflows pinning to `ubuntu-latest` got new Python, new GCC, new Node defaults, new pre-installed package versions -- often overnight, often with mysterious test breaks. **Pin to a major version** (`ubuntu-24.04`) for production. Update deliberately on a known schedule.

---

## Part 11: Production-Realistic Examples

### Example 1: Multi-job CI workflow with concurrency, matrix, needs/outputs, and conditionals

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    outputs:
      services: ${{ steps.changed.outputs.matrix }}
      any-changed: ${{ steps.changed.outputs.any }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
        with:
          fetch-depth: 0
      - id: changed
        run: |
          if [ "${{ github.event_name }}" = "push" ]; then
            BASE=HEAD~1
          else
            BASE=origin/${{ github.base_ref }}
          fi
          CHANGED=$(git diff --name-only "$BASE"... | awk -F/ '/^services\//{print $2}' | sort -u)
          if [ -z "$CHANGED" ]; then
            echo "matrix=[]" >> $GITHUB_OUTPUT
            echo "any=false" >> $GITHUB_OUTPUT
          else
            MATRIX=$(echo "$CHANGED" | jq -R -s -c 'split("\n") | map(select(length > 0))')
            echo "matrix=$MATRIX" >> $GITHUB_OUTPUT
            echo "any=true" >> $GITHUB_OUTPUT
          fi

  lint:
    needs: detect-changes
    if: ${{ needs.detect-changes.outputs.any-changed == 'true' }}
    runs-on: ubuntu-latest
    timeout-minutes: 10
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: ./.github/actions/setup-node-with-cache
      - run: npm run lint --workspace=services/${{ matrix.service }}

  test:
    needs: detect-changes
    if: ${{ needs.detect-changes.outputs.any-changed == 'true' }}
    runs-on: ubuntu-latest
    timeout-minutes: 30
    strategy:
      fail-fast: false
      max-parallel: 4
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
        node: [20, 22]
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: ./.github/actions/setup-node-with-cache
        with:
          node-version: ${{ matrix.node }}
      - run: npm test --workspace=services/${{ matrix.service }}

  build:
    needs: [lint, test]
    if: ${{ needs.detect-changes.outputs.any-changed == 'true' && github.event_name == 'push' }}
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - id: meta
        run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
      - run: docker build -t app:${{ steps.meta.outputs.tag }} .

  notify:
    needs: [detect-changes, lint, test, build]
    if: ${{ !cancelled() && (failure() || needs.build.result == 'success') }}
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - run: |
          if [ "${{ needs.build.result }}" = "success" ]; then
            echo "CI passed for ${{ needs.build.outputs.tag }}"
          else
            echo "CI failed; check logs"
            exit 1
          fi
```

What this demonstrates:
- Conditional concurrency: `cancel-in-progress` only on PRs (where stale runs are wasted minutes), not on main pushes (where every commit deserves a build).
- Dynamic matrix from a setup job emitting JSON.
- `if: !cancelled() && (failure() || needs.build.result == 'success')` for a notify job that runs on success or failure but respects user cancel.
- Realistic timeout overrides on every job.
- Composite action call (`./.github/actions/setup-node-with-cache`) DRY-ing up the boilerplate.
- All actions SHA-pinned (`actions/checkout@11bd71...`).

### Example 2: Reusable workflow + caller using it across two environments via matrix

The reusable workflow:

```yaml
# .github/workflows/deploy-to-env.yml (CALLEE)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
      region:
        required: true
        type: string
    outputs:
      deploy-url:
        value: ${{ jobs.deploy.outputs.url }}
    secrets:
      AWS_ROLE_TO_ASSUME:
        required: true

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    environment: ${{ inputs.environment }}        # Triggers protection rules
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    concurrency:
      group: deploy-${{ inputs.environment }}-${{ inputs.region }}
      cancel-in-progress: false                    # NEVER cancel a deploy
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: aws-actions/configure-aws-credentials@b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ inputs.region }}
      - id: deploy
        run: |
          # Update the manifest in the config repo; ArgoCD will reconcile
          ./scripts/bump-tag.sh \
            --env=${{ inputs.environment }} \
            --region=${{ inputs.region }} \
            --tag=${{ inputs.image-tag }}
          URL=$(./scripts/deploy-url.sh ${{ inputs.environment }} ${{ inputs.region }})
          echo "url=$URL" >> $GITHUB_OUTPUT
      - run: echo "Deployed to ${{ steps.deploy.outputs.url }}"
```

The caller workflow:

```yaml
# .github/workflows/release.yml (CALLER)
name: Release
on:
  push:
    tags: ['v*']

jobs:
  build:
    uses: ./.github/workflows/build-and-push.yml
    with:
      image-name: my-app
    secrets: inherit

  deploy-staging:
    needs: build
    strategy:
      fail-fast: false                               # Don't cancel us-east just because eu-west failed
      matrix:
        region: [us-east-1, eu-west-1]
    uses: ./.github/workflows/deploy-to-env.yml      # Reusable workflow inside a matrix!
    with:
      environment: staging
      image-tag: ${{ needs.build.outputs.image-tag }}
      region: ${{ matrix.region }}
    secrets: inherit

  deploy-prod:
    needs: deploy-staging
    strategy:
      fail-fast: false
      matrix:
        region: [us-east-1, eu-west-1]
    uses: ./.github/workflows/deploy-to-env.yml
    with:
      environment: production
      image-tag: ${{ needs.build.outputs.image-tag }}
      region: ${{ matrix.region }}
    secrets: inherit
```

Worth noting:
- A reusable workflow can be called inside a matrix (each cell becomes its own workflow run).
- The `environment: production` on the callee triggers any protection rules configured in the GitHub UI -- required-reviewer gates, wait timer, deployment branches.
- Concurrency is on the **callee**, scoped per `(environment, region)` -- so us-east-1 and eu-west-1 deploys are independent, but two us-east-1 prod deploys queue.
- `cancel-in-progress: false` on the deploy callee.
- Image tag flows: `build` job output -> `needs.build.outputs.image-tag` -> caller's `with:` -> callee's `inputs:`.
- `permissions: id-token: write` on the callee enables OIDC -- the modern keyless-auth path covered in depth on Day 5 of this week. For now, treat it as "the alternative to storing long-lived AWS keys as repo secrets."

### Example 3: `workflow_dispatch` with typed inputs and an environment gate

```yaml
name: Manual Hotfix Deploy
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version tag (e.g., v1.2.3-hotfix.1)'
        required: true
        type: string
      environment:
        description: 'Target environment'
        required: true
        type: environment
      skip-canary:
        description: 'Skip the canary phase (emergency only)'
        required: false
        type: boolean
        default: false
      reason:
        description: 'Why this hotfix is bypassing the normal release flow'
        required: true
        type: string

permissions:
  id-token: write
  contents: read

concurrency:
  group: hotfix-${{ inputs.environment }}
  cancel-in-progress: false

jobs:
  audit:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - run: |
          echo "::notice::Hotfix triggered by ${{ github.actor }}"
          echo "::notice::Version: ${{ inputs.version }}"
          echo "::notice::Environment: ${{ inputs.environment }}"
          echo "::notice::Skip canary: ${{ inputs.skip-canary }}"
          echo "::notice::Reason: ${{ inputs.reason }}"

  canary:
    needs: audit
    if: ${{ ! inputs.skip-canary }}
    runs-on: ubuntu-latest
    timeout-minutes: 30
    environment: ${{ inputs.environment }}-canary    # Canary env has its own gate
    steps:
      - run: ./scripts/canary-deploy.sh --version=${{ inputs.version }}

  full-deploy:
    needs: [audit, canary]
    # Run if canary succeeded OR canary was skipped (but always require audit)
    if: ${{ !cancelled() && needs.audit.result == 'success' && (needs.canary.result == 'success' || needs.canary.result == 'skipped') }}
    runs-on: ubuntu-latest
    timeout-minutes: 30
    environment: ${{ inputs.environment }}            # Full env has the strictest gate
    steps:
      - run: ./scripts/full-deploy.sh --version=${{ inputs.version }}

  notify:
    needs: [audit, canary, full-deploy]
    if: ${{ !cancelled() }}
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - run: |
          STATUS="success"
          if [ "${{ needs.full-deploy.result }}" != "success" ]; then STATUS="FAILED"; fi
          ./scripts/post-to-slack.sh "Hotfix ${{ inputs.version }} -> ${{ inputs.environment }}: $STATUS (by ${{ github.actor }}, reason: ${{ inputs.reason }})"
```

Worth noting:
- `type: environment` on the input gives a dropdown of configured environments in the dispatch UI, eliminating typos.
- `type: boolean` for `skip-canary` -- but remember, the value is the string `"true"` or `"false"`. Here we use `if: ${{ ! inputs.skip-canary }}` which exploits the parser's coercion (treats string `"false"` as falsy in negation). The fully-explicit form is `if: ${{ inputs.skip-canary == false || inputs.skip-canary == 'false' }}`.
- The `full-deploy` `if:` uses `needs.canary.result == 'skipped'` to handle the skip-canary path -- without this, `success()` would treat skipped-canary as not-success and full-deploy would be skipped too.
- Two environments (`{env}-canary` and `{env}`) with separate protection rules: canary may auto-approve, full-deploy may require two reviewers.
- `concurrency: hotfix-${{ inputs.environment }}` ensures only one hotfix runs to a given environment at a time, but staging and prod hotfixes don't block each other.

---

## Part 12: Production Gotchas (the bug pile)

### 1. `pull_request_target` running on fork code with secrets

**Gotcha:** A workflow uses `on: pull_request_target` (gets secrets) AND does `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}` (checks out the PR's code). An attacker forks the repo, modifies a build script to `cat $AWS_SECRET_ACCESS_KEY`, and opens a PR. Your workflow runs the attacker's code with your prod credentials. Secrets are exfiltrated.
**Why this matters:** This is the canonical GitHub Actions security incident -- `pull_request_target` exists *because* `pull_request` doesn't have secrets on fork PRs, and people circumvent the safety to "just make CI work for forks."
**Fix:** Two-workflow split. A `pull_request` workflow runs untrusted code with a read-only `GITHUB_TOKEN`, no secrets exposed -- catches the common "does this code compile and pass tests?" signal safely. A separate `pull_request_target` workflow handles labeling/commenting only -- never checks out the PR head, only reads metadata (PR number, head SHA, author) or pulls artifacts from the unprivileged run. If you genuinely need privileged execution against PR code (canary publishing, integration tests against protected resources), use the **labeled-trigger pattern**: `on: pull_request_target` with `types: [labeled]` filtered to a maintainer-only label like `safe-to-test`. The label becomes the human-in-the-loop bridge between unprivileged and privileged contexts -- a maintainer reviews the diff and applies the label only when satisfied. Pair this with auto-stripping the label on `synchronize` events so a re-push forces re-review (otherwise "approve once, ship anything later" attacks become possible). Remember: the attack surface isn't just `run:` blocks -- `npm install`/`pip install`/`cargo build`/`go generate`/`make`/`bundle install`/Gradle/Maven all execute attacker-controlled config files (`postinstall`, `setup.py`, `build.rs`, `Makefile`, `Gemfile`, `build.gradle`). In a privileged context, every config-processing tool is an arbitrary-code-execution primitive; the rule is "secrets and untrusted code must never be in the same workspace at the same time."

### 2. `if: always()` makes a job uncancellable -- and the trade-off compounds in reusable workflows

**Gotcha:** A cleanup job has `if: always()` so it runs regardless of upstream success/failure. A user clicks "Cancel workflow" -- the cleanup job still runs, often for 30 minutes, often making the cancellation feel non-responsive. **The compounded gotcha**: when the cleanup logic is wrapped in a reusable workflow called from the parent, *each level evaluates `if:` independently*. People put `if: always()` on the caller's `uses:` job and assume it propagates inside. It doesn't -- the inner job inside the reusable workflow has its own default `if: success()` and silently skips on cancellation. Same trap with `timeout-minutes`: the caller's timeout doesn't propagate to the called workflow's jobs.
**Why this matters:** "Cancel" is supposed to mean cancel. `always()` treats cancel as just another reason to run -- and for legitimate uses (terminal cleanup that *must* happen, audit jobs, billing teardown) the workflow becomes uncancellable for the duration of that job.
**Fix:** When you actually need the run-on-cancel semantic (teardown of billed test infra, terminal audit, partial-deploy detection), pair `if: always()` with **mandatory** `timeout-minutes:` (15 is a reasonable default; never accept the 360-minute default) plus tool-level timeouts inside the script (`aws --cli-read-timeout`, `kubectl --request-timeout`) so a hung dependency can't eat the whole budget. The cleanup script itself must be **idempotent and tolerant of partial state** -- treat "tear down to nothing" as a converge-to-target-state operation (`--ignore-not-found`, catch 404s) rather than imperative kill commands. For most "run on success or failure but respect Cancel" cases, prefer `if: !cancelled()`. And if the cleanup logic is in a reusable workflow, put `if: always()` on **both** the caller's `uses:` job *and* the inner cleanup job inside the reusable workflow, with explicit `timeout-minutes` at both levels -- the inner job decides for itself.

### 3. `concurrency.cancel-in-progress: true` on a deploy corrupts state

**Gotcha:** A team copies the PR-CI concurrency block (`cancel-in-progress: true`) into the deploy workflow because it "works for CI." Someone re-runs deploy mid-rollout (perhaps to test the workflow change). The first deploy is cancelled with half its manifests pushed, half not. ArgoCD now sees an inconsistent set of manifests and the cluster sits between two image versions.
**Why this matters:** The Apr 23 Alertmanager doc and Apr 19 ArgoCD doc both stress that deploys are atomic units; partial state is the worst state. Concurrency `cancel-in-progress: true` is the easiest way to introduce partial state.
**Fix:** `cancel-in-progress: false` on every deploy / migration / external-state-mutation workflow, paired with a sane `timeout-minutes` (queue mode means a hung deploy blocks the lane until the timeout fires). The mental check: "does anything outside the workflow remember partial work?" -- if yes, never `true`. **Defensive bonus**: add an audit job with `if: ${{ always() }}` that fires on `cancelled()` or `failure()`, dumps `${{ toJSON(needs) }}` so you can see exactly which jobs completed, and posts to a deploy-audit channel. Cancellation in the runs UI looks bland and is easy to miss; a loud audit makes a cancelled-mid-deploy event impossible to ignore. This is one of the legitimate uses of `always()` -- the "avoid always()" guidance in Gotcha #2 is about *user-cleanup* jobs that block the Cancel button, not about deliberate audit jobs whose entire purpose is to fire on every terminal state including cancellation. (Note: with queue depth = 1 running + 1 pending, bursty pushes silently drop intermediate deploys -- commits 2-4 of a 5-commit burst get evicted from pending. Acceptable for most teams because the latest code ships and Git provides bisectability.)

### 4. Matrix contexts unavailable outside matrix jobs (silent empty)

**Gotcha:** `${{ matrix.region }}` referenced in a downstream non-matrix job returns the empty string. The deploy script becomes `./deploy.sh --region= --tag=abc`, which often defaults to a working but incorrect region.
**Why this matters:** No error. No warning. The deploy ran, the tests passed, and you're now serving us-east-1 traffic from us-west-2. You'll find out when latency dashboards spike at midnight.
**Fix:** Don't try to pull per-cell values out via `outputs.region: ${{ matrix.region }}` on the matrix job -- every matrix cell writes to the **same** `jobs.<id>.outputs.<name>` slot, so whichever cell finishes last wins (and with `fail-fast: true` you don't even know which cells ran to completion). The slot is one-write-wins regardless of `fail-fast`. Per-cell data must be written as **artifacts** named with the matrix dimension (`upload-artifact` with `name: result-${{ matrix.region }}`), then aggregated in a downstream non-matrix coordinator job that downloads all of them and emits a single proper output.

### 5. Expressions evaluating to empty silently when context is wrong scope

**Gotcha:** `${{ secrets.MY_SECRET }}` in a workflow-level `env:` block (before any job-level `env:`). Returns empty. The downstream job uses `env.MY_SECRET` which is empty. The deploy "succeeds" but doesn't authenticate. Two silent layers stack to make the failure invisible: (1) the Actions/YAML layer silently substitutes empty strings for missing context references, and (2) the shell layer accepts `--region=` as a syntactically-valid CLI argument whose deploy script then falls back to a hardcoded default.
**Why this matters:** Silent string-coercion to empty is the universal failure mode. There's no static checker that says "you referenced `secrets` in workflow-level `env:`, that doesn't work." And `set -euo pipefail` does **NOT** save you here -- by the time bash parses the line, Actions has already substituted the empty string, so the variable is *literally set to empty*, not unset. `set -u` triggers on unset variables, not empty ones.
**Fix:** Three layers of defense. (1) Read secrets and matrix/needs values at the job or step level via an `env:` block, never workflow level. (2) Use bash's "fail on empty or unset" parameter expansion `${VAR:?message}` as a defensive guard before the side-effecting command:
```yaml
env:
  REGION: ${{ matrix.region }}
  TAG: ${{ needs.test.outputs.tag }}
run: |
  set -euo pipefail
  : "${REGION:?REGION is empty -- check matrix.region}"
  : "${TAG:?TAG is empty -- check needs.test.outputs.tag}"
  ./deploy.sh --region="$REGION" --tag="$TAG"
```
(3) For diagnostic precision when an output IS empty, the smoking gun is whether `toJSON(needs)` shows `outputs: {}` (the upstream job didn't declare an `outputs:` mapping at all -- layer-2 problem) vs `outputs: {tag: ""}` (the mapping exists but the value is empty -- layer-1 step-side problem). Treat the availability matrix in Part 3 as a checklist before writing any `${{ }}` expression.

### 6. Secrets unavailable in `on:` filters and trigger-level `if:`

**Gotcha:** `on.push.if: ${{ secrets.DEPLOY_KEY != '' }}` -- this doesn't exist as a field. Or you write `if:` that references `secrets` at the workflow level and wonder why it always evaluates false.
**Why this matters:** Secrets aren't loaded into context until job-scheduling time, on purpose -- to limit secret exposure surface.
**Fix:** Move secret-presence checks to job-level `if:`, or read the secret into env first and check `env.X != ''`.

### 7. `secrets: inherit` is reliable only within a single org

**Gotcha:** Reusable workflow in `myorg/platform/.github/workflows/deploy.yml` is called from `partnerorg/app/.github/workflows/release.yml` with `secrets: inherit`. The reusable workflow runs but with no secrets. The deploy fails silently or produces broken output.
**Why this matters:** GitHub's docs say `secrets: inherit` works across orgs within the same enterprise, but community reports through 2025 still show org-level secrets often don't propagate cross-org even within an enterprise. The runs UI shows "completed successfully" even though authentication failed downstream because credentials were missing.
**Fix:** Treat the safe rule as **"same org or enumerate explicitly."** For any cross-org call, enumerate every secret: `secrets: { AWS_ROLE: ${{ secrets.PROD_AWS_ROLE }}, ECR: ${{ secrets.PROD_ECR }} }`. Better: keep reusable workflows in an org-internal repo and consume only from same-org repos.

### 8. Reusable workflow nesting and total-count caps

**Gotcha:** GitHub raised the reusable-workflow nesting limit from 4 to **10 levels** in 2024, and capped total unique reusable workflows reachable from a single top-level run at **50**. Old guides (and old internal docs) still say "4 levels." Refactoring against the wrong number leads to either prematurely flattening a clean abstraction or hitting the 50-workflow cap when you fan out a top-level workflow into a hundred per-service reusable-workflow calls.
**Why this matters:** Both errors are discovered late: the 4-level number leads to needless flattening; the 50-workflow cap surfaces when an ApplicationSet-style fan-out workflow tries to call the same reusable deploy workflow 60 times across services.
**Fix:** Plan for 10 levels max nesting / 50 unique workflows. Most production setups use 2-3 levels (caller -> reusable -> sometimes sub-reusable) and stay well under 50 unique workflows. If a fan-out exceeds 50, the right answer is usually a matrix on the leaf job rather than 50 separate reusable-workflow calls.

### 9. Default 360-minute timeout billing trap

**Gotcha:** A workflow has no `timeout-minutes:` override. A job hangs (network call, deadlock, infinite loop). The runner consumes 6 hours of billed time before GitHub kills it.
**Why this matters:** On a $7/hour large runner, that's $42 per stuck run. Across a busy CI pipeline, can be hundreds of dollars per week wasted.
**Fix:** `timeout-minutes:` on every job. 10 for lint, 30 for normal build, 60 for integration tests. Plus organizational-level alarms on workflow run duration p99 (set in your monitoring system, not in GitHub).

### 10. Legacy stdout-command form (`::set-output::`, `::set-env::`, `::add-path::`) is on the chopping block

**Gotcha:** A workflow mixes the old stdout-parsing forms -- `echo "::set-output name=tag::abc"`, `echo "::set-env name=FOO::bar"`, `echo "::add-path::/some/path"` -- with the modern env-file forms. Some lines work, some emit deprecation warnings, and when GitHub finally removes the legacy form (postponed multiple times since October 2022, but coming) every workflow that still uses it hard-fails with "set-output is no longer supported." Inherited workflows from 2020-2022 often still use the old form throughout.
**Why this matters:** Two things make this worse than a normal deprecation. First: the original stdout form was a **log-injection vulnerability** -- a malicious package emitting `::set-output name=admin::true` from npm install could set arbitrary outputs. Second: a scheduled removal will silently break dozens of workflows org-wide on the same day, with no PR-level notice.
**Fix:** Migrate everything to the env-file form: `echo "tag=abc" >> $GITHUB_OUTPUT`, `echo "FOO=bar" >> $GITHUB_ENV`, `echo "/some/path" >> $GITHUB_PATH`, `echo "key=val" >> $GITHUB_STATE`. Add a `grep -rE '::(set-output|set-env|add-path|save-state)' .github/` lint check to your repo CI -- exit nonzero if any matches.

### 11. `runs-on: ubuntu-latest` shifting underneath you

**Gotcha:** The team's CI passed for months on `ubuntu-latest`. GitHub flips `ubuntu-latest` from 22.04 to 24.04 over a weekend. Tests start failing because Python is now 3.12, not 3.10, and a deprecation breaks. Or because pre-installed `gh` is now newer and a CLI flag was renamed.
**Why this matters:** No PR. No commit. No notification beyond a vague changelog blog post. Failures appear seemingly out of nowhere.
**Fix:** Pin to a major version (`ubuntu-24.04`) for production. Migrate to a newer major on a planned schedule. For really critical workflows, pin tools (`actions/setup-python@... with: python-version: '3.10'`) explicitly to override what's pre-installed.

### 12. `${{ }}` in `if:` -- when you need it and when you don't

**Gotcha:** `if: always() && github.event_name == 'push'` -- the parser treats this as a literal string. The job runs regardless of any condition, because the string is always truthy.
**Why this matters:** Subtle parsing inconsistency between "bare expression" and "wrapped expression" forms. The error message is non-existent; the workflow runs but with the wrong gate.
**Fix:** When mixing status functions with other expressions, always wrap: `if: ${{ always() && github.event_name == 'push' }}`. Rule of thumb: when in doubt, wrap.

### 13. `needs.X.outputs` are always strings (numeric/JSON encoding pitfall)

**Gotcha:** Upstream emits `echo "count=42" >> $GITHUB_OUTPUT`. Downstream does `if: ${{ needs.upstream.outputs.count > 10 }}`. The comparison **does** work because GHA expressions auto-coerce strings that look like numbers. But `if: ${{ needs.upstream.outputs.config.key }}` does NOT work -- the value is a string `'{"key":"v"}'`, not a JSON object. You must `fromJson`.
**Why this matters:** People expect outputs to round-trip rich types. They don't. Two implicit conversions (write: serialize-to-string; read: best-effort-coerce-on-comparison) give a misleading impression of fidelity.
**Fix:** Treat outputs as opaque strings. For numbers, comparisons happen to work. For JSON, always `${{ fromJson(needs.X.outputs.foo) }}`. For booleans, prefer comparing strings: `outputs.flag == 'true'`.

### 14. `success()` returns false when an upstream is skipped -- and the right fix is often the DAG, not the condition

**Gotcha:** `cleanup` job has `if: success()` (or no `if:`, which is the same). Upstream `optional-test` was conditionally skipped (its `if:` was false). `cleanup` is then also skipped because `success()` says "any upstream was skipped, that's not all-success." Worse: skip cascades through the DAG. If `deploy` skips because `if: github.ref == 'refs/heads/main'` is false on a feature branch, every job that `needs: deploy` (and every job that needs *those*) defaults to `success()` and silently skips even though `build` succeeded.
**Why this matters:** Cleanup not running because something was conditionally skipped is non-intuitive -- nothing failed! And the symptom -- "Slack notify doesn't fire on feature branches" -- looks like a notify bug when it's actually a DAG-shape bug.
**Fix:** Two paths, in this order. **First, fix the DAG**: if `notify` only depends on `build` succeeding, narrow `needs: [build, deploy, integration-test]` to `needs: [build]`. The cleanest fix isn't a permissive `if:` -- it's matching the dependency graph to the actual semantic intent. With only `build` in `needs:`, `deploy` and `integration-test` become sibling branches that can't poison `notify`, and `notify` runs in parallel with them. The general lesson: **fix DAG-shape problems by changing the DAG, not by working around them with permissive `if:` conditions.** **Second, when notify genuinely needs to wait** for the rest of the pipeline before firing (e.g., final-status reporter), reach for `if: !failure()` (skipped is OK) or `if: !failure() && !cancelled()` (run unless something failed or user cancelled). The two-axis decision: *trigger semantic* (success / failure / always) × *tolerance for skipped or cancelled upstreams* picks the right form -- success-only path uses `success()`, real-failure-only path uses `failure()`, terminal cleanup uses `always()` + timeout, and notify-on-green-respect-cancel uses `!failure() && !cancelled()`.

### 15. Concurrency `group:` interpolated at scheduling time, not runtime

**Gotcha:** `concurrency.group: deploy-${{ github.head_ref }}`. On a `push` event, `github.head_ref` is empty, so the group is `deploy-`. ALL deploy runs across all branches now share this group and serialize against each other.
**Why this matters:** What looks like a per-branch group becomes a global group on push events. Deploys queue inexplicably.
**Fix:** Use `${{ github.ref }}` or `${{ github.ref_name }}` instead -- they're populated for both push and PR events. Or include the event name: `group: deploy-${{ github.event_name }}-${{ github.ref }}`.

### 16. `fromJson` errors on malformed input crash the workflow

**Gotcha:** Setup job emits `echo "matrix=$RAW" >> $GITHUB_OUTPUT` without quoting. `$RAW` contains a special character. The downstream `fromJson(needs.setup.outputs.matrix)` can't parse it. The workflow fails with a cryptic "expression evaluation failed."
**Why this matters:** Dynamic matrices are a fragile pipeline; one bad input value tears the whole run down at scheduling time, not at the matrix cell.
**Fix:** Always `jq -c -R` the JSON in the setup script (compact, raw input) before writing to `$GITHUB_OUTPUT`. And gate the downstream job: `if: ${{ needs.setup.outputs.matrix != '[]' && needs.setup.outputs.matrix != '' }}`.

### 17. Schedule cron drift on the hour

**Gotcha:** `cron: '0 * * * *'` -- "every hour on the hour." GitHub's docs explicitly note that scheduled workflows can be **delayed** during high load, especially at top-of-hour boundaries (when everyone schedules things). Your "hourly" workflow runs at :02, :07, :13, never :00.
**Why this matters:** Anything depending on precise cadence (data-pipeline ingestion windows, cost reports closing at midnight UTC) will drift.
**Fix:** Offset to a less-popular minute: `cron: '7 * * * *'` (every hour at :07). For sub-hour cadence, use a self-hosted runner with a dedicated scheduler or an external cron service triggering via `repository_dispatch`.

### 18. Matrix `runs-on:` label-match must be exact

**Gotcha:** A matrix uses `runs-on: ${{ matrix.os }}` with `os: [Ubuntu-Latest, macos-latest]`. The Ubuntu cell hangs indefinitely in queue with status "Waiting for a runner" because **`Ubuntu-Latest` is not a valid runner label** -- the canonical hosted alias is the lowercase `ubuntu-latest`. Self-hosted runners are even more unforgiving; a typo on a custom label like `gpu-runner` vs `gpu-runners` (singular vs plural) means the job sits in the queue forever.
**Why this matters:** No error, no warning. The job simply waits. After 30+ minutes a frustrated developer cancels and assumes "GitHub is slow today" -- when in fact the workflow will never get a runner. On a deploy workflow with `cancel-in-progress: false`, this also blocks the queue for any other deploy that arrives after.
**Fix:** Lowercase all hosted labels (`ubuntu-latest`, `ubuntu-24.04`, `macos-latest`, `windows-latest`). For self-hosted, normalize labels in your runner registration scripts and add a workflow-level lint that greps for known-bad casings. If a job sits in the queue more than 5 minutes, the very first thing to check is the runner label string -- not GitHub's status page.

---

## Part 13: Decision Frameworks

### 1. Reusable workflow vs composite action

| Use reusable workflow | Use composite action |
|---|---|
| Need separate `runs-on:` per call | Always runs on caller's runner |
| Need `secrets:` block with explicit declaration | No secrets needed (or accept input-passing trade-off) |
| Multi-job pipeline (fan-out, fan-in) | 3-7 steps glued together |
| Need `environment:` with protection rules | No env gating needed |
| Want each call to appear as its own workflow run | Want steps inline in caller's job log |
| Cross-repo platform sharing | Same-repo or tightly-coupled cluster |
| Need `permissions:` block per call | Inherits caller's permissions |

**Wrong choice in either direction:** reusable-when-it-should-be-composite fragments the run UI; composite-when-it-should-be-reusable forces secrets through plaintext inputs and forbids different runners.

### 2. `cancel-in-progress: true` vs `false`

| Use `true` | Use `false` |
|---|---|
| PR validation runs | Production deploys |
| Lint/test on every push to feature branch | Staging deploys |
| Documentation builds | Database migrations |
| Preview-environment teardown | Anything that mutates external state |
| Cost-sensitive non-critical CI | External API calls that aren't idempotent |

**The mental check:** does anything outside the workflow remember partial work? Yes -> `false`. No -> `true`.

### 3. `pull_request` vs `pull_request_target`

| Use `pull_request` | Use `pull_request_target` |
|---|---|
| Default for all CI | Auto-labeling, welcome bots |
| Anything that runs the PR's code | Security scans against the PR (read metadata only) |
| Anything that needs secrets only on same-repo PRs | Anything where you must have secrets even on fork PRs |
| Anything where you'd accept that fork PRs run with no secrets | Things that explicitly never check out the PR's content |

**The security warning:** `pull_request_target` + `actions/checkout` of the PR's head + any script execution = remote code execution with your secrets. If you're tempted, split into two workflows. The legitimate use cases for `pull_request_target` are narrow and never include "build the PR's code."

### 4. Matrix vs separate jobs vs reusable workflow called from a loop

| Approach | Use when |
|---|---|
| **Matrix** | Cells share most of their step list (>= ~50% identical steps); want shared `if:`, timeout, concurrency, fan-in |
| **Separate jobs** | Cells share less than ~50% of their steps; differ in runner, secrets, dependencies, or step order |
| **Reusable workflow loop** | Each call is heavy (10+ steps); want separate run UI; different `environment:` gates per call |

**Default:** matrix. Reach for the others only when matrix can't express the difference. The shared-step heuristic beats counting cells: a 12-cell matrix where every cell runs the same 6 steps is clean; a 4-cell matrix where each cell has a different step order is a tangle of `if:` conditions and is better as 4 separate jobs.

### 5. `if: success()` vs `if: !failure()` vs `if: always()` vs `if: !cancelled()`

| Want | Use |
|---|---|
| All upstream succeeded (default) | (no `if:`) or `if: success()` |
| Run unless something actually failed (skipped is OK) | `if: !failure()` |
| Run regardless, including on cancel (for true terminal cleanup that MUST happen) | `if: always()` |
| Run on success or failure but respect user cancel (most cleanup jobs) | `if: !cancelled()` |
| Run only when something failed (alerting/notify) | `if: failure()` |
| Run when cancelled (rollback / abort hook) | `if: cancelled()` |

**Pick the most restrictive form that captures intent.** `always()` should be the rare choice -- only when terminal cleanup must run *even if a user clicks Cancel.*

---

## Part 13.5: Debugging Workflows -- Tools You'll Reach For at 3 AM

When a workflow misbehaves, four tools cover ~90% of investigation:

### Re-run with debug logging

Set two repository secrets (Settings -> Secrets and variables -> Actions):
```
ACTIONS_RUNNER_DEBUG=true
ACTIONS_STEP_DEBUG=true
```
The first enables verbose runner-internal logs (job-scheduling, runner setup, network calls); the second enables debug logs from individual `run:` steps (every command echoed before execution, environment dumps, expression evaluations exposed). They're **per-repo** and **persistent** -- enable for a debug session, then revert. Don't leave them on; the log volume balloons and they expose more about your runner internals than you usually want in a public-repo log archive.

To re-run a single failed run with debug enabled, the UI's "Re-run failed jobs" -> "Enable debug logging" checkbox is the one-shot equivalent (Settings -> Actions -> "Enable debug logging" toggles the same secret automatically for that one re-run).

### `gh run view --log` and `gh run watch`

```bash
gh run list                        # 20 most recent runs in the current repo
gh run view 12345 --log            # Print full logs of a run
gh run view 12345 --log-failed     # Only the failed jobs/steps
gh run rerun 12345                 # Re-run failed jobs
gh run watch                       # Live-tail the most recent run
```

Beats clicking through the UI when investigating multiple runs. `--log-failed` is especially good for "show me only what broke" -- avoids scrolling past 800 lines of green checkmarks.

### `tmate` for SSH into a stuck runner

The `mxschmitt/action-tmate` community action drops a `tmate` shell into the runner mid-workflow, exposing it via SSH:

```yaml
- name: Setup tmate session (debug only)
  if: ${{ runner.debug == '1' && failure() }}
  uses: mxschmitt/action-tmate@<sha>
  with:
    limit-access-to-actor: true     # Only the user who triggered the run can SSH in
    detached: false                 # Block the workflow until the SSH session ends
```

Gated on `runner.debug == '1'` (which `ACTIONS_STEP_DEBUG=true` enables) and `failure()` so it only fires when something has already broken in debug mode. This is a third-party action -- treat it with the SHA-pin and audit the action source before using it on private-repo workflows. Some orgs forbid `tmate` entirely on security grounds; an alternative is to add a debug step that dumps the runner's filesystem state (`ls -laR`, `env`, `pwd`, `df -h`) to logs, then re-run.

### Local act -- `nektos/act`

```bash
brew install act
act push                            # Run the push event locally
act -j build                        # Run a single job
act -e event.json                   # Use a saved event payload
```

`act` runs your workflows in a local Docker container. It's not perfect -- some actions (anything that calls GitHub's API with `GITHUB_TOKEN`) can't fully simulate, and the Docker image used for `ubuntu-latest` is a community-maintained subset of what GitHub's actual runner has -- but for fast iteration on a workflow's YAML structure, expression evaluation, and basic `run:` step logic, it cuts the "push commit, wait for runner, watch logs" loop down from 5 minutes to 30 seconds.

---

## Part 14: 10 Commandments of GitHub Actions

1. **Thou shalt not assume jobs share state.** Every job is a fresh runner. Use artifacts for files, outputs for strings, external storage for everything else. The runner is sterile; treat it that way.
2. **Thou shalt override `timeout-minutes` on every job.** The 360-minute default is a billing trap. Set 10 for lint, 30 for build, 60 for integration tests. Stuck jobs cost real money.
3. **Thou shalt use `cancel-in-progress: true` on PR CI and `false` on deploys.** The mental check is "does anything outside the workflow remember partial work?" -- yes for deploys, no for CI. Get this backwards and you either burn minutes or corrupt state.
4. **Thou shalt prefer `if: !cancelled()` over `if: always()`.** `always()` makes jobs uncancellable. Cleanup jobs that don't terminate on Cancel make the Cancel button feel broken.
5. **Thou shalt never use `pull_request_target` to run a PR's code.** Two-workflow split: untrusted code runs on `pull_request` without secrets; labeling/commenting runs on `pull_request_target` without ever checking out the PR's source.
6. **Thou shalt SHA-pin every third-party `uses:`.** Third-party actions (`uses: someorg/some-action@v1`) are remote code execution waiting to happen if the tag is repointed; pin to a 40-char SHA, full stop. First-party `actions/*` (checkout, setup-node, upload-artifact, configure-aws-credentials, etc.) can use major-version tags (`@v4`) because GitHub controls those repos and treats tag-repointing as a security incident. Combine SHA-pinning with Dependabot for automatic, reviewable bumps. Tj-actions/changed-files is the canonical reason this matters.
7. **Thou shalt treat all outputs as strings.** Numbers and JSON round-trip through string serialization. Use `fromJson` for JSON, string comparisons (`'true'`, `'42'`) for everything else. Don't trust auto-coercion in non-trivial expressions.
8. **Thou shalt use `secrets: inherit` only same-org.** Cross-org calls silently pass no secrets. Enumerate explicitly when crossing org boundaries.
9. **Thou shalt write env-file syntax (`>> $GITHUB_OUTPUT`, `>> $GITHUB_ENV`), not the deprecated `::set-output::` form.** The legacy form is a log-injection vector and is on the removal list. Migrate now; lint repo-wide for stragglers.
10. **Thou shalt audit context availability before writing every expression.** Half of "my workflow does the wrong thing" bugs are wrong-context-wrong-scope -- `secrets` in `on.push.if:`, `matrix` in a non-matrix downstream, `needs.X.outputs` from an upstream that was skipped. Keep the availability matrix in Part 3 next to your editor.

---

## Self-Check -- The "Did I Internalize This?" Quiz

If you can answer all of these without re-opening the doc, the material has stuck. If any answer makes you hesitate, that's the section to re-read.

1. A `cleanup` job has no `if:` set. Three of its upstream `needs:` succeeded; one was conditionally skipped. Does cleanup run? Why or why not? What's the right `if:` for "run unless something actually failed"?
2. Your workflow has `concurrency: { group: deploy-prod, cancel-in-progress: true }`. A deploy is mid-rollout to production. A teammate clicks "Re-run all jobs" on a fresh commit. What happens to the in-flight deploy and what's the production state afterward? What `cancel-in-progress` value should this have been?
3. A reusable workflow is defined in `myorg/platform/.github/workflows/deploy.yml`. It's called from `partnerorg/app` with `secrets: inherit`. The deploy workflow runs to completion and reports success in the runs UI, but the actual AWS call inside it 403s. What's the most likely cause and the fix?
4. A matrix job declares `outputs.region: ${{ matrix.region }}` and runs across 5 region cells. What value does `needs.matrix-job.outputs.region` show in a downstream job? How should you actually pass per-cell data to a downstream consumer?
5. A workflow has `on: pull_request_target` and uses `actions/checkout@SHA` with `ref: ${{ github.event.pull_request.head.sha }}`, then runs `npm ci && npm test`. An attacker forks the repo and opens a PR. What's the attack surface? Why is it still dangerous even if the workflow doesn't have a literal `run:` block executing PR-controlled code?
6. The `permissions:` block at the workflow level is `permissions: { contents: read }`. A job in that workflow needs to comment on a PR and push a Docker image to GHCR. What permissions does that job need declared at the job level, and does the workflow-level `contents: read` carry through?
7. `${{ matrix.os }}` returns the empty string. List three different reasons why this could happen, in order of likelihood.
8. `if: always() && github.event_name == 'push'` -- does this gate the job to push events only, or does it ignore the push check entirely? What's the fix?
9. A workflow runs on `schedule: '0 * * * *'` and is supposed to run every hour at :00. The actual runs land at :04, :07, :13, varying. Why? What cron expression would be more reliable?
10. You see `id-token: write` in a workflow's `permissions:` block but no `secrets.AWS_ACCESS_KEY_ID`. How is the workflow authenticating to AWS? What's the security advantage over keys-in-secrets?

---

## Further Reading

- **Workflow syntax reference (official)** -- https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- **Contexts reference (official)** -- https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs
- **Events that trigger workflows (official)** -- https://docs.github.com/en/actions/reference/events-that-trigger-workflows
- **Reuse workflows (official)** -- https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows
- **Security hardening for GitHub Actions (official)** -- https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions
- **GitHub Actions Concurrency Patterns (DEV.to)** -- https://dev.to/kanta13jp1/github-actions-concurrency-patterns-cancel-in-progress-false-for-parallel-deployments-1bjm
- **Composite Actions vs Reusable Workflows (DEV.to)** -- https://dev.to/n3wt0n/composite-actions-vs-reusable-workflows-what-is-the-difference-github-actions-11kd
- **The Matrix Strategy in GitHub Actions (RunsOn Blog)** -- https://runs-on.com/github-actions/the-matrix-strategy/
- **Mastering `if` Conditions (DEV.to)** -- https://dev.to/cloud-sky-ops/mastering-github-actions-insights-and-pitfalls-of-if-conditions-1j87
- **GitHub Actions: Deprecating set-output (Changelog)** -- https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/
- Repo cross-references: `2026/april/kubernetes/2026-04-19-argocd-helm-production-patterns.md` (CI/CD separation, image tag promotion), `2026/april/kubernetes/2026-04-17-argocd-advanced-patterns.md` (multi-environment promotion), `2026/april/observability/2026-05-01-slos-error-budgets-production-observability.md` (deploy SLOs gating production releases).
