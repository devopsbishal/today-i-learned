# GitHub Actions Custom Actions & Reusable Patterns -- The Recipe Library and the Catering Service

> Yesterday's doc was about the *factory floor*: how a workflow spins up runners, how jobs flow through a DAG, how concurrency and matrices and contexts behave. That covered "how do I run YAML." This doc is about the deeper question of "how do I stop writing the same YAML." Every real engineering org hits the same wall after 6-8 workflow files: 30 lines of `checkout + setup-node + cache + install` repeat 14 times across the repo, the same 200-line build-and-push pipeline lives in every microservice's workflow file, and a security-team change to (say) "always SHA-pin the AWS credentials action" has to be made in 27 places. The `.github/workflows/` directory rots into copy-paste sprawl. **GitHub Actions answers this with two reuse mechanisms that solve different problems**, and a third tier of *built-in* reusable building blocks (`actions/cache`, `actions/upload-artifact`, `dorny/paths-filter`) that handle the cross-cutting concerns of caching, hand-off, and monorepo selectivity.
>
> **The core analogy for today: a custom action is a recipe card in a shared kitchen library, and a reusable workflow is a contracted catering service.** A *recipe card* (composite action) lives in the kitchen and tells your cook how to do a multi-step prep right at your station -- "checkout, setup-node, restore cache, run npm ci" -- using *your* knives, *your* counter, *your* mise-en-place. A *blender recipe* (JavaScript action) says "feed these ingredients into the blender on setting 3 for 20 seconds" -- it runs an opcode-level program (`node20 dist/index.js`) that the kitchen has no insight into. A *sealed-appliance recipe* (Docker action) says "use the bread machine that came with this exact firmware" -- you trade flexibility and cross-stove portability for a deterministic, hermetic environment that always behaves the same way. **In contrast, a catering service** (reusable workflow via `workflow_call`) is a separate kitchen with separate staff: you fill out an order form (`inputs:`), authorize their use of certain corporate purchasing cards (`secrets:`), and they ship plated dishes back (`outputs:`). They run on their own time, on their own equipment, with their own log book -- and crucially, **they bill separately**. Composite actions are tax-deductible kitchen tools you keep on-site; reusable workflows are external invoices. That's the pricing model and that's the foot-gun: pick the wrong one and you either fragment your CI run into 14 tiny separate jobs (caterer for a peanut-butter sandwich) or stuff a multi-stage deploy pipeline into a composite action that can't take secrets cleanly (home cook running a wedding banquet).
>
> Three corollaries this analogy carries that show up in every section below. **(1) The pantry persists, the prep does not.** `actions/cache` is the restaurant's mise-en-place pre-prep that survives between services -- "the chimichurri we made on Tuesday, still in the walk-in." Artifacts (`actions/upload-artifact`) are the plated meal handed off course-to-course in a single dinner -- consumed once, then gone with that night's service. Conflate them and you get the canonical bug: developer uses `actions/cache` to pass a `dist/` folder from build to deploy, the cache is evicted on a cold day, and prod ships the wrong build. **(2) Composite secrets are a YAML-coupling foot-gun, not a security one.** A composite action has no `secrets:` block. To pass a secret in, the caller writes a regular `with:` input -- e.g. `with: api-key: ${{ secrets.PROD_KEY }}`. The runtime *value* is still masked in logs (GitHub auto-masks any string sourced from `secrets.*` regardless of how it reached the action), so this is **not** a value-leak. The cost is YAML coupling and discoverability: every caller has to know the composite needs that secret, has to remember to pass it, and any change that adds a new required secret to the composite is a breaking change for every consumer. Reusable workflows skip this with `secrets: inherit` plus a proper `secrets:` declaration block -- which is why anything that meaningfully handles secrets should be a reusable workflow, not a composite. **(3) Path filtering hides skipped jobs from required-status-checks.** The "build" job in a monorepo skips when no service files changed -- but GitHub's *required status checks* are specified by job name, and a "skipped" job does not satisfy "required must pass." This is the silent merge-blocker that defeats half of monorepo CI implementations. The fix is a status-aggregator job that reports success-on-skip; we'll detail the pattern.

---

**Date**: 2026-05-06
**Topic Area**: cicd
**Difficulty**: Intermediate -> Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Custom action | A recipe card in the shared kitchen library | A reusable unit invoked via `uses:`; lives in `action.yml` |
| `action.yml` | The recipe card itself (ingredients, instructions, return) | The contract: `inputs`, `outputs`, `runs`, `branding` |
| JavaScript action | "Feed these ingredients into the blender on setting 3" | `runs.using: node20` + a pre-bundled `dist/index.js` |
| Docker container action | "Use this sealed bread machine" | `runs.using: docker`; Linux-only, slow cold start, hermetic |
| Composite action | "Do this 5-step prep at your station" | `runs.using: composite` + steps array; runs on the caller's runner |
| `dist/` bundling | The blender setting that's pre-recorded so the kitchen doesn't have to install plug-ins | `@vercel/ncc` produces single-file `dist/index.js`; commit it |
| `node_modules/` in Git | Shipping the blender along with the recipe | Anti-pattern -- bundle with ncc, gitignore `node_modules/` |
| `pre`/`post` hooks | The setup ritual before service and the cleanup after | `runs.pre`/`runs.post` for JS actions only |
| Reusable workflow | A contracted catering service (separate kitchen, separate billing) | `on.workflow_call` workflow; called as `uses:` at job level |
| `secrets: inherit` | "Use the parent factory's locked binders" | Caller passes ALL its secrets to the called workflow |
| 10-deep nesting limit | The corporate policy on stacked catering subcontracts | Reusable workflows nest up to 10 levels deep (top-level caller + 9 nested) |
| `actions/cache` | The walk-in cooler with labeled mise-en-place | Restores prep work across CI runs |
| Cache key immutability | "Once labeled 'tomato confit, 2026-05-06', that container is sealed forever" | A cache key written once cannot be overwritten -- bump a segment to refresh |
| `restore-keys:` | "If today's exact label is missing, accept yesterday's prep" | Partial-prefix fallback hierarchy |
| `actions/upload-artifact` | Plating a course and handing it to the next station | Pass whole files between jobs in one workflow run |
| Artifact immutability (v4) | "The plated dish, once sent out, can't be reopened and added to" | v4 artifacts are immutable -- a name can't be re-uploaded in the same run; downstream jobs can still download freely |
| `dorny/paths-filter` | The expediter calling only the pasta station for pasta orders | Per-job path filtering with boolean outputs |
| Status aggregator job | "All-stations green" light that fires even when stations are dark | Required-status-check sentinel that returns success-on-skip |

---

## The Whole Picture in One Diagram

```
THE RECIPE LIBRARY AND THE CATERING SERVICE
================================================================================

CALLER WORKFLOW (the customer)
+---------------------------------------------------+
| name: CI                                          |
| jobs:                                             |
|   test:                                           |
|     runs-on: ubuntu-latest                        |
|     steps:                                        |
|       - uses: ./.github/actions/setup-tools  <----+--- COMPOSITE ACTION
|         with: { node: '20' }                     |       (recipe card on-site)
|       - uses: actions/setup-node@<sha>  <--------+--- JS ACTION
|         with: { node-version: '20' }              |       (blender recipe)
|       - uses: docker://my/scanner:1.2  <---------+--- DOCKER ACTION
|         with: { target: . }                      |       (sealed appliance)
|       - run: npm test                             |
|                                                   |
|   build-and-push:                                 |
|     uses: org/platform/.github/workflows/  <-----+--- REUSABLE WORKFLOW
|           build-push.yml@<sha>                   |       (catering service:
|     with:                                         |       separate kitchen,
|       image-name: my-app                          |       separate billing)
|     secrets: inherit                              |
+---------------------------------------------------+

WHERE EACH RUNS
================================================================================
                            Runner            Logs           Billing
Composite action            Caller's          Inline         Caller's job
JavaScript action           Caller's          Inline         Caller's job
Docker action (Linux)       Caller's          Inline         Caller's job
Reusable workflow           SEPARATE          Separate job   Separate billing line

PERSISTENCE LAYERS
================================================================================
Within a job:               Filesystem persists step-to-step (same workbench)
Across jobs in one run:     Use ARTIFACTS (upload-artifact / download-artifact)
Across runs (cache):        Use CACHE (actions/cache or setup-* cache: param)
Forever (history):          External storage (S3, ECR, GHCR, registry)

CACHE vs ARTIFACT DECISION
================================================================================
Question                           Cache                   Artifact
--------                           -----                   --------
Can it be evicted?                 YES (LRU, 7d, 10GB)     NO (90d retention)
Survives across runs?              YES                     NO (scoped to one run)
Mutable?                           NO (immutable per key)  NO (immutable in v4)
Available immediately?             On cache restore step   On upload (v4) / end (v3)
Right tool for                     Dependency caches,      CI->CD handoff,
                                   Docker layers,          test reports,
                                   Terraform plugins       coverage, build outputs
```

---

## Part 1: Custom Action Anatomy -- The `action.yml` Contract

Every custom action is described by a single `action.yml` (or `action.yaml`) file. This is the **API contract** of the action: what it takes (`inputs:`), what it returns (`outputs:`), how it's executed (`runs:`), and how it's identified in the Marketplace UI (`name`, `description`, `branding`). Treat it like a `package.json` for a recipe.

```yaml
# .github/actions/my-action/action.yml
name: 'My custom action'                # Display name in Marketplace and UI
description: 'A one-line summary of what this action does'
author: 'devopsbishal'                  # Optional, Marketplace only

branding:                               # Optional, Marketplace only
  icon: 'package'                       # Feather icon name
  color: 'blue'                         # white, yellow, blue, green, orange, red, purple, gray-dark

inputs:
  api-key:                              # Input name (kebab-case by convention)
    description: 'API key for upstream service'
    required: true                      # If missing, action fails before main runs
  retries:
    description: 'Number of retries'
    required: false
    default: '3'                        # Defaults are STRINGS even for numbers
  legacy-flag:
    description: 'Old toggle, use --new-flag instead'
    deprecationMessage: 'Use --new-flag instead. Will be removed 2026-12-01.'
    required: false

outputs:
  status:
    description: 'Success or failure'
    # For JS actions, set via core.setOutput('status', value)
    # For composite, must explicitly map: value: ${{ steps.X.outputs.Y }}
  duration-ms:
    description: 'How long the action ran in ms'

runs:
  using: 'node20'                       # node20 | node22 | docker | composite (node24 in preview)
  main: 'dist/index.js'                 # JS only -- the bundled entrypoint
  pre: 'dist/setup.js'                  # JS only -- runs BEFORE main
  pre-if: runner.os == 'Linux'        # JS only -- conditional pre
  post: 'dist/cleanup.js'               # JS only -- runs AFTER main, even on failure
  post-if: 'success() || failure()'     # JS only -- skip on cancel by default
```

Five contract rules to internalize:

1. **All input values arrive as strings.** Even if you write `default: '3'` for retries, your action code receives the string `"3"`. Parse with `parseInt`, `JSON.parse`, etc. The `getBooleanInput` helper accepts `true/True/TRUE/false/False/FALSE` and rejects everything else with an error.
2. **`required: true` is a hard fail before your code runs** -- the runner refuses to invoke the action if a required input is missing. Use this sparingly; defaults are friendlier.
3. **`deprecationMessage` produces a warning annotation** in every workflow run that uses the input. This is the supported way to phase out parameters without breaking callers.
4. **Outputs declared in `action.yml` but never set return empty strings**, not undefined. Downstream `${{ steps.x.outputs.y }}` becomes `''`. No error, no warning -- the binder is empty.
5. **`branding:` is only consumed when the action is published to the Marketplace.** Local actions in your own repo ignore it. Don't agonize over icon choice for an internal-only action.

---

## Part 2: JavaScript Actions -- The Blender Recipe

JS actions are the most flexible and most performant of the three types. They run in the runner's Node.js process directly -- no container start-up, full access to GitHub's `@actions/toolkit` packages, cross-platform (Linux/macOS/Windows runners). The cost: you must **bundle** all your dependencies into a single JS file before you ship.

### The bundling requirement

Actions cannot run `npm install` at execution time -- the runner downloads the action's source and immediately invokes the entrypoint. So all `node_modules/` content has to be either committed to the repo (huge, anti-pattern) or **bundled into a single file with `@vercel/ncc`** (modern correct answer).

```bash
# One-time setup
npm init -y
npm install @actions/core @actions/github
npm install -D @vercel/ncc typescript

# Build (run before every commit / via husky pre-commit / via CI)
npx ncc build src/index.ts -o dist --license licenses.txt
```

```
# .gitignore
node_modules/
# do NOT add dist/ -- you MUST commit dist/index.js
```

The `dist/` directory ends up containing a single self-contained `index.js` (typically 200KB-2MB) plus a `licenses.txt` file. **This file is what `runs.main` points to**, and it's what the runner executes. The `node_modules/` of your dev environment is gitignored; the bundled output is committed.

The most common authoring error: **forgetting to rebuild `dist/` before committing.** You change `src/index.ts`, push, and the action runs the old code from `dist/index.js`. Two safeguards:
- A `.github/workflows/check-dist.yml` workflow that rebuilds `dist/` and fails if it differs from what's committed.
- A `husky` pre-commit hook that runs `npx ncc build` automatically.

### Wiring to `@actions/core` and `@actions/github`

```typescript
// src/index.ts
import * as core from '@actions/core';
import * as github from '@actions/github';

async function run(): Promise<void> {
  const start = Date.now();
  try {
    // --- Read inputs (all arrive as strings) ---
    const apiKey = core.getInput('api-key', { required: true });
    const dryRun = core.getBooleanInput('dry-run');                  // 'true'/'false' -> boolean
    const tags   = core.getMultilineInput('tags');                   // 'a\nb\nc' -> ['a','b','c']
    const retries = parseInt(core.getInput('retries') || '3', 10);

    // --- Mask the secret in logs (any future occurrence of apiKey gets '***') ---
    core.setSecret(apiKey);

    // --- Group log lines under a collapsible heading ---
    await core.group('Validating inputs', async () => {
      core.info(`Tags: ${tags.join(', ')}`);
      core.info(`Retries: ${retries}`);
    });

    // --- Modify the runner environment for subsequent steps ---
    core.exportVariable('MY_TOOL_HOME', '/opt/my-tool');             // sets env for next steps
    core.addPath('/opt/my-tool/bin');                                // prepends to PATH

    // --- Annotations (file/line-attached warnings/errors) ---
    core.warning('Deprecated config detected', {
      title: 'Migration needed',
      file: 'config.yml',
      startLine: 12
    });

    // --- Use Octokit for GitHub API calls ---
    const token = core.getInput('github-token');
    if (token && github.context.eventName === 'pull_request') {
      const octokit = github.getOctokit(token);
      await octokit.rest.issues.createComment({
        ...github.context.repo,
        issue_number: github.context.payload.pull_request!.number,
        body: `Action ran. Dry run: ${dryRun}`
      });
    }

    // --- Set outputs (must be declared in action.yml to be visible) ---
    core.setOutput('status', 'success');
    core.setOutput('duration-ms', String(Date.now() - start));

    // --- Append to the job summary (rich Markdown shown in the run UI) ---
    await core.summary
      .addHeading('My Action Results')
      .addTable([
        [{ data: 'Metric', header: true }, { data: 'Value', header: true }],
        ['Tags processed', String(tags.length)],
        ['Dry run', String(dryRun)]
      ])
      .write();

  } catch (err) {
    if (err instanceof Error) core.setFailed(err.message);
  }
}

run();
```

The `@actions/core` API is the entire surface area for JS actions; ten functions cover 95% of needs:

| Function | Purpose | Common gotcha |
|----------|---------|---------------|
| `getInput(name, opts)` | Read input | Returns `''` for missing optional inputs, NOT undefined |
| `getBooleanInput(name)` | Boolean coercion | Throws on values other than true/True/TRUE/false/False/FALSE |
| `getMultilineInput(name)` | Newline-split list | Returns `[]` if input is empty |
| `setOutput(name, value)` | Expose to downstream | Must be declared in `action.yml` to be consumable by callers |
| `setFailed(msg)` | Mark step as failed | Sets exit code 1 AND prints msg as error annotation |
| `info` / `warning` / `error` / `notice` | Logged messages | `warning`/`error`/`notice` create UI annotations |
| `setSecret(val)` | Mask in logs | Apply BEFORE first log line containing the value |
| `exportVariable(name, val)` | Set env for next steps | Persists across steps in the same job (uses `$GITHUB_ENV` under the hood) |
| `addPath(path)` | Prepend to PATH | Persists across steps (uses `$GITHUB_PATH`) |
| `summary.X(...).write()` | Append to job summary | Markdown rendered in run UI; NOT sent to logs |

### The `pre` and `post` lifecycle hooks (JS-only)

A JS action can declare three entrypoints, all bundled separately:

```yaml
runs:
  using: 'node20'
  pre: 'dist/setup.js'        # Runs BEFORE the step's main entrypoint
  pre-if: runner.os == 'Linux'   # Conditional pre
  main: 'dist/index.js'       # The step's actual work
  post: 'dist/cleanup.js'     # Runs AFTER the step, in reverse order
  post-if: 'success() || failure()'  # By default; skip post on cancel
```

**`pre` runs once at the start of the job, in the order the actions appear; `post` runs once at the end, in REVERSE order.** This is exactly how `actions/cache@v4` implements automatic save-on-success: its `main` restores the cache, its `post` saves it -- and because `post` runs after every other step in the job has finished, it captures the final state.

The mental model: `pre` and `post` are **outside the step's main run window**. If you have steps A B C D in a job and A is a JS action with `pre`/`post`, the actual execution is `A.pre, A.main, B, C, D, A.post`. This is why `actions/cache` saves the *final* state of `~/.npm` after `npm ci` ran in a later step.

---

## Part 3: Docker Container Actions -- The Sealed Appliance

Docker actions trade flexibility for **hermeticity**. The action ships a Dockerfile (or pre-built image reference); the runner builds (or pulls) the image and runs your entrypoint inside the container. Inputs become environment variables; outputs are written to `$GITHUB_OUTPUT` inside the container.

```yaml
# .github/actions/scanner/action.yml
name: 'Custom security scanner'
description: 'Runs a sealed scanner image against a target directory'

inputs:
  target:
    description: 'Path to scan'
    required: true
    default: '.'
  severity-threshold:
    description: 'Minimum severity to fail on'
    required: false
    default: 'high'

outputs:
  findings-count:
    description: 'Number of findings'

runs:
  using: 'docker'
  # Two options:
  image: 'Dockerfile'                    # Build from a Dockerfile in this action's dir
  # image: 'docker://ghcr.io/me/scanner:1.2.3'   # Or pull a pre-built image (much faster)

  args:                                  # Passed as $1, $2, ... to the entrypoint
    - '--target'
    - ${{ inputs.target }}
    - '--severity'
    - ${{ inputs.severity-threshold }}

  env:                                   # Or as env vars (preferred over args)
    INPUT_TARGET: ${{ inputs.target }}
    INPUT_SEVERITY: ${{ inputs.severity-threshold }}

  entrypoint: '/scan.sh'                 # Optional override; default is image's ENTRYPOINT
  pre-entrypoint: '/setup.sh'            # Optional; runs before entrypoint
  post-entrypoint: '/cleanup.sh'         # Optional; runs after entrypoint
```

```dockerfile
# .github/actions/scanner/Dockerfile
FROM alpine:3.19
RUN apk add --no-cache jq curl
COPY scan.sh /scan.sh
RUN chmod +x /scan.sh
ENTRYPOINT ["/scan.sh"]
```

```bash
#!/bin/sh
# .github/actions/scanner/scan.sh
set -e
TARGET="${INPUT_TARGET:-.}"
echo "Scanning $TARGET..."

# ... do scan ...
COUNT=42

# Outputs go to $GITHUB_OUTPUT (mounted from the host)
echo "findings-count=$COUNT" >> "$GITHUB_OUTPUT"
```

Five rules for Docker actions:

1. **Linux runners only.** No macOS, no Windows. If your matrix includes Windows, Docker actions silently break the matrix. This is the most cited reason teams convert Docker actions to JS.
2. **Cold start dominates.** Building the image from a `Dockerfile:` adds 30-90 seconds to every step invocation. Use **pre-built images** (`image: 'docker://ghcr.io/owner/image:1.2.3'`) in production -- the runner just pulls (5-10s) instead of builds.
3. **Inputs become `INPUT_<UPPER>` env vars** automatically when you use `runs.using: docker`. Inputs with hyphens become underscores: `severity-threshold` -> `INPUT_SEVERITY_THRESHOLD`.
4. **The action's working directory inside the container is `/github/workspace`**, which is the mounted repo checkout. `$GITHUB_OUTPUT`, `$GITHUB_ENV`, `$GITHUB_STEP_SUMMARY`, and `$GITHUB_PATH` are also mounted in.
5. **Reach for Docker actions when the toolchain is sealed and not Node**: a Rust binary scanner, a custom Python data tool with system-level deps, a CLI shipped only as a container. For everything else, prefer JS or composite -- Docker's cold-start cost compounds across many invocations in a workflow.

---

## Part 4: Composite Actions -- The Recipe Card on the Counter

Composite actions are the most under-used of the three types because they're the most *under-marketed*. They're also the right answer to "I have 5 lines of YAML duplicated in 14 jobs" -- the case where neither JS nor Docker makes sense.

```yaml
# .github/actions/setup-tools/action.yml
name: 'Setup tools'
description: 'Checkout, setup-node, restore cache, install deps'

inputs:
  node-version:
    description: 'Node version to install'
    required: false
    default: '20'
  install-args:
    description: 'Extra args to pass to npm ci'
    required: false
    default: ''

outputs:
  cache-hit:
    description: 'Whether the npm cache was a hit'
    value: ${{ steps.setup.outputs.cache-hit }}     # MUST be wired explicitly

runs:
  using: 'composite'
  steps:
    - uses: actions/checkout@a1b2c3d                # Composite can use other actions
      with:
        fetch-depth: 0

    - id: setup
      uses: actions/setup-node@b2c3d4e
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash                                   # MANDATORY on every run: step
      working-directory: ./services/api             # Optional per-step
      env:
        EXTRA_ARGS: ${{ inputs.install-args }}      # Optional per-step
      run: |
        echo "Installing on $(uname -s)..."
        npm ci $EXTRA_ARGS

    - name: Print version
      if: ${{ inputs.node-version == '20' }}        # Per-step if: works
      shell: bash
      run: node --version
```

Called inline:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: ./.github/actions/setup-tools
        id: tools
        with:
          node-version: '20'
      - run: echo "Cache hit: ${{ steps.tools.outputs.cache-hit }}"
      - run: npm test
```

### The seven things that bite composite-action authors

1. **`shell:` is mandatory on every `run:` step.** This is the #1 authoring bug. Workflow files inherit a default shell from the runner OS; **composite actions do not** -- there is no `defaults.run.shell` in `action.yml`. Forget `shell: bash` and you get the cryptic `Required property is missing: shell` error.
2. **No `secrets:` block.** A composite action cannot read `secrets.X` directly. To pass a secret in, the *caller* writes `with: api-key: ${{ secrets.PROD_KEY }}`. Inside the composite, you read it as `${{ inputs.api-key }}`. The runtime value IS masked in logs identically to a reusable workflow's `secrets:` block (GitHub auto-masks anything sourced from `secrets.*` regardless of how it reaches the action) -- this is **not** a value-leak. The cost is **YAML coupling and discoverability**: every caller must know which secrets the composite needs, must remember to pass them, and adding a new required secret is a breaking change for every consumer. **This is the single strongest reason to choose a reusable workflow over a composite when secrets are involved** -- reusable workflows skip the coupling with `secrets: inherit` plus a proper `secrets:` declaration block.
3. **Runs on the caller's runner.** No `runs-on:`, no separate billing, no separate runner spin-up. Steps appear inline (collapsed under the action name) in the caller's job log. This is the *killer feature* -- composites are essentially zero-cost YAML reuse.
4. **Outputs require explicit wiring.** Unlike JS actions where `core.setOutput` directly populates `outputs.x`, composite outputs must be mapped: `outputs.cache-hit.value: ${{ steps.setup.outputs.cache-hit }}`. Forget this and the output is empty -- no error.
5. **`if:` from the outside gates the WHOLE composite, not individual steps.** When the caller writes `- uses: ./my-composite\n  if: github.ref == 'refs/heads/main'`, the entire composite runs or doesn't. Per-step conditionality must be expressed *inside* the composite via per-step `if:`.
6. **Composite actions can `uses:` other actions** (including other composites, JS, Docker). This is how you build composite-of-composites for layered platform setup -- but: **respect the 10-deep nesting limit** for composite-action chains (per GitHub's documented limit).
7. **`continue-on-error:` per step works inside a composite**, but a failure in any step without `continue-on-error: true` aborts the whole composite and the caller sees the composite step as failed.

### Composite secret foot-gun: a worked example

```yaml
# WRONG -- this looks like it should work, but the composite has no access to secrets.*
# .github/actions/deploy/action.yml
runs:
  using: 'composite'
  steps:
    - shell: bash
      run: ./deploy.sh
      env:
        API_KEY: ${{ secrets.PROD_API_KEY }}     # ${{ secrets.* }} is empty here!
```

```yaml
# RIGHT -- pass it as an input from the caller
# .github/actions/deploy/action.yml
inputs:
  api-key:
    required: true
runs:
  using: 'composite'
  steps:
    - shell: bash
      run: ./deploy.sh
      env:
        API_KEY: ${{ inputs.api-key }}

# .github/workflows/cd.yml (caller)
- uses: ./.github/actions/deploy
  with:
    api-key: ${{ secrets.PROD_API_KEY }}         # caller resolves the secret
```

The masking is identical to what a reusable workflow's `secrets:` block would give you -- if `PROD_API_KEY` is "abc123", any future log line containing "abc123" will appear as `***`. The cost is purely **YAML coupling and the inability to use `secrets: inherit`**: every caller must know which secrets the composite needs and pass each one explicitly. Add a new required secret to the composite and you must edit every caller.

---

## Part 5: Reusable Workflows -- The Catering Service

Yesterday's doc covered the *decision* of "reusable workflow vs composite action" via the secrets+runner test. Today we go deeper on *authoring*: the precise shape of the `on.workflow_call` block, the two-layer outputs mapping, the calling syntax variants, and the production-grade patterns (matrix-fan-out, SHA pinning, permission propagation).

### The `on.workflow_call` block, dissected

```yaml
# .github/workflows/build-and-push.yml (CALLEE)
name: Build and Push

on:
  workflow_call:
    # ----- INPUTS -----
    inputs:
      image-name:
        description: 'ECR image name (without registry prefix)'
        type: string                             # MANDATORY: string | boolean | number
        required: true
      environment:
        description: 'staging | production'
        type: string
        required: true
      dry-run:
        description: 'Skip the push step'
        type: boolean
        required: false
        default: false
      retention-days:
        description: 'Image retention'
        type: number
        required: false
        default: 30

    # ----- SECRETS -----
    secrets:
      AWS_ROLE_TO_ASSUME:
        description: 'OIDC role ARN'
        required: true
      ECR_REGISTRY:
        description: 'ECR registry URL'
        required: true
      OPTIONAL_SLACK_WEBHOOK:
        required: false                          # Optional secret -- absent = empty string

    # ----- OUTPUTS (two-layer mapping) -----
    outputs:
      image-tag:
        description: 'Pushed image tag'
        value: ${{ jobs.build.outputs.tag }}     # Layer 1: workflow output <- job output
      image-digest:
        description: 'sha256 digest'
        value: ${{ jobs.build.outputs.digest }}

permissions:                                     # Reusable workflow can declare its own
  id-token: write                                # OIDC
  contents: read                                 # Checkout

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:                                     # Layer 2: job output <- step output
      tag:    ${{ steps.meta.outputs.tag }}
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@a1b2c3d
      - id: meta
        run: |
          TAG="${{ inputs.environment }}-$(git rev-parse --short HEAD)"
          echo "tag=$TAG" >> $GITHUB_OUTPUT
      - uses: aws-actions/configure-aws-credentials@b2c3d4e
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: us-east-1
      - id: push
        if: ${{ !inputs.dry-run }}
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/${{ inputs.image-name }}:${{ steps.meta.outputs.tag }} .
          docker push ${{ secrets.ECR_REGISTRY }}/${{ inputs.image-name }}:${{ steps.meta.outputs.tag }}
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${{ secrets.ECR_REGISTRY }}/${{ inputs.image-name }}:${{ steps.meta.outputs.tag }})
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT
```

Five things that distinguish reusable workflows from composite actions:

1. **`type:` is mandatory** on every input -- one of `string`, `boolean`, `number`. There is **no `choice` type** for `workflow_call` (unlike `workflow_dispatch` which supports `choice` and `environment`). If you need enum-like validation, do it inside the workflow with a step.
2. **Outputs are two-layer.** A step writes to `$GITHUB_OUTPUT`; the job declares `outputs.X: ${{ steps.id.outputs.x }}`; the workflow declares `outputs.X.value: ${{ jobs.id.outputs.x }}`. Skip either layer and the caller sees an empty string with no error.
3. **Secrets are declared per-secret.** Each secret gets its own `description` and `required` flag. Alternative: caller uses `secrets: inherit` to pass everything.
4. **The `permissions:` block is independent.** A reusable workflow can declare what GITHUB_TOKEN scopes it needs -- but **the caller cannot grant the called workflow more than the caller has**. This is the propagation rule: if the caller has `permissions: { contents: read }`, the called workflow cannot magically get `contents: write` even if it asks.
5. **It's a separate job (or set of jobs).** Each job runs on its own runner with its own log group, its own billing line, its own ~15-30s spin-up cost. This is the trade-off vs composite actions.

### Calling syntax variants

```yaml
jobs:
  # Same repo, branch ref (DEV ONLY)
  build-staging:
    uses: ./.github/workflows/build-and-push.yml
    with:
      image-name: my-app
      environment: staging
    secrets: inherit                                  # Pass all caller's secrets

  # Cross-repo, SHA pinned (PRODUCTION)
  build-prod:
    uses: org/platform/.github/workflows/build-and-push.yml@a1b2c3d4e5f6
    with:
      image-name: my-app
      environment: production
      retention-days: 90
    secrets:                                          # Or enumerate explicitly
      AWS_ROLE_TO_ASSUME: ${{ secrets.PROD_AWS_ROLE }}
      ECR_REGISTRY: ${{ secrets.PROD_ECR_REGISTRY }}
    permissions:                                      # Caller grants permissions to callee
      id-token: write
      contents: read

  # Cross-repo, version tag (acceptable for audited platform repos)
  build-staging-v2:
    uses: org/platform/.github/workflows/build-and-push.yml@v2.3.1

  # Dependent on the reusable workflow's output
  deploy:
    needs: build-prod
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build-prod.outputs.image-tag }}"
```

### `secrets: inherit` -- the silent foot-gun

`secrets: inherit` passes the **caller's entire secret namespace** into the called workflow. **Reliable within a single org.** Cross-org-within-an-enterprise is documented as supported, but in practice org-level secrets often don't propagate, so the safe rule is **"same org, or enumerate explicitly."** Cross-org `secrets: inherit` to a different organization passes nothing silently; the workflow runs with empty secrets, `secrets.X` is the empty string in the called workflow, and authentication fails downstream with no error message tying the failure back to inheritance.

The fix: **always enumerate** when calling cross-org workflows.

```yaml
# Cross-org -- enumerate explicitly
jobs:
  scan:
    uses: othergroup/security/.github/workflows/scan.yml@v1
    secrets:
      SCAN_API_KEY: ${{ secrets.SCAN_API_KEY }}      # Explicit map; no inheritance
```

### The 10-deep nesting limit

GitHub enforces a hard limit on call-chain depth: **a top-level workflow can nest reusable-workflow calls up to 10 levels deep -- that is, the caller plus up to 9 nested reusable workflows.** (This was raised from the original 4-deep limit in August 2024; older blog posts citing "4 levels" are stale.) Yesterday's doc cites the same 10-deep figure -- they agree. A community-circulated "~50 unique reusable workflows reachable per run" cap also gets quoted, but it's not in the current public docs as of May 2026 and may have been quietly relaxed; treat it as folklore. Most production setups use 2-3 levels (`caller -> reusable -> sometimes sub-reusable`), and the depth cap matters mainly when you discover it during refactoring. Always SHA-pin reusable workflows in production regardless of depth.

### Matrix on a calling job -- the multi-env deploy pattern

This is the canonical "deploy to all environments in parallel" pattern. The matrix is on the *calling job*; each cell invokes the same reusable workflow with different parameters.

```yaml
# .github/workflows/cd.yml (caller) -- PARALLEL fan-out, no ordering
jobs:
  deploy:
    strategy:
      fail-fast: false                                # Don't cancel siblings
      matrix:
        include:
          - environment: dev
            cluster: dev-eks
          - environment: staging
            cluster: stg-eks
          - environment: production
            cluster: prod-eks
    uses: org/platform/.github/workflows/deploy.yml@a1b2c3d4e5f6789012345678901234567890abcd
    with:
      environment: ${{ matrix.environment }}
      cluster: ${{ matrix.cluster }}
    secrets: inherit
```

Each matrix cell becomes a separate run of the reusable workflow, with its own logs, its own runner, and -- if the called workflow uses `environment:` -- its own protection-rule gates.

**Critical pitfall: `max-parallel: 1` does NOT give you serialized success-gating.** It's a *concurrency limiter*, not a *dependency gate*. With `fail-fast: false` and `max-parallel: 1`, cells start one at a time but a failed staging cell does NOT prevent the prod cell from running -- they're siblings with no `needs:` relationship. For **strict ordered rollout where each stage must be green before the next starts** (the `dev -> staging -> prod-us -> prod-eu` deploy chain), use `needs:`-chained jobs, NOT a matrix:

```yaml
# .github/workflows/cd.yml -- SERIALIZED with success gating
jobs:
  dev:
    uses: org/platform/.github/workflows/deploy.yml@a1b2c3d4e5f6789012345678901234567890abcd
    with: { environment: dev, cluster: dev-eks }
    secrets: inherit
  staging:
    needs: dev
    uses: org/platform/.github/workflows/deploy.yml@a1b2c3d4e5f6789012345678901234567890abcd
    with: { environment: staging, cluster: stg-eks }
    secrets: inherit
  prod_us:
    needs: staging
    uses: org/platform/.github/workflows/deploy.yml@a1b2c3d4e5f6789012345678901234567890abcd
    with: { environment: prod-us, cluster: prod-us-eks }
    secrets: inherit
  prod_eu:
    needs: prod_us
    uses: org/platform/.github/workflows/deploy.yml@a1b2c3d4e5f6789012345678901234567890abcd
    with: { environment: prod-eu, cluster: prod-eu-eks }
    secrets: inherit
```

`needs:` blocks the downstream job until upstream `result == 'success'`, so a staging failure cleanly short-circuits prod-us and prod-eu. Pair with `environment:` declared inside `deploy.yml` for required-reviewer gates per stage. **Matrix for parallel fan-out among siblings; `needs:` for ordered fan-out across stages; environments for human gates.** Don't ask one to do another's job. (Note: `workflow_call` inputs are typed string/boolean/number with no list type, so the matrix can't live inside the called workflow either -- ordering between environments is fundamentally a multi-job concern at the caller layer.)

### Why pin reusable workflows to a SHA in production

Same rule as third-party actions: pin to a SHA, not `@main` or a moving tag, so a bad commit (or supply-chain compromise) in the central platform repo doesn't instantly break every consumer. See yesterday's doc for the canonical CVE-2025-30066 / Tj-actions cautionary tale.

---

## Part 6: Composite Actions vs Reusable Workflows -- Authoring Comparison

Yesterday's doc gave the *decision framework* (secret + runner test). Today's angle is the **authoring trade-offs side-by-side**:

| Dimension | Composite action | Reusable workflow |
|-----------|------------------|-------------------|
| **Where it runs** | Caller's runner (free reuse) | Its own `runs-on:` (extra spin-up cost) |
| **Log integration** | Inline in caller's job log (collapsed group) | Separate job with separate log group |
| **Billing line** | Caller's job (one bill) | Separate billable job |
| **Secrets handling** | NO `secrets:` block; secrets must come in via `with:` inputs (caller-resolved) | First-class `secrets:` block + `secrets: inherit` |
| **Conditional execution** | `if:` per step inside; outside `if:` gates whole composite | Full `if:` per job; can fan out to many jobs |
| **Outputs** | `outputs.X.value: ${{ steps.X.outputs.Y }}` (one mapping layer) | Two-layer: step -> job -> workflow output |
| **Inputs typing** | Inputs are strings (no `type:` field) | Mandatory `type: string \| boolean \| number` |
| **`uses:` inside** | Yes -- can call other actions, including other composites (10-deep) | Yes -- can call other reusable workflows (10-deep) |
| **`runs-on:` control** | No -- inherits caller's | Yes -- own runner labels per called workflow |
| **`environment:` gates** | No (composite is just steps in the caller's job) | Yes -- protection rules, required reviewers, wait timer |
| **`permissions:` block** | No -- inherits caller's | Yes -- own permissions block (capped by caller's) |
| **Marketplace publishable** | Yes (it's a custom action) | No (workflows can't be Marketplace items) |
| **Reuse pattern fit** | "Checkout + setup-node + cache + install" 3-7 step glue | "Build-test-publish" multi-job pipeline |

The two-line summary: **composite for shared step glue; reusable workflow for shared multi-job pipelines.** If your unit of reuse fits inside a single job and doesn't need its own secrets, it's a composite. If it needs its own runner, its own `environment:` gate, its own multi-job DAG, or proper secrets handling, it's a reusable workflow.

### When each is the wrong choice

- **Reusable workflow for what should have been composite**: fragments your run UI into 6 tiny jobs, each with its own ~20s runner spin-up cost and its own log group, making "what happened in this CI run" 5x harder to read.
- **Composite for what should have been reusable**: forces secrets through `with:` plaintext (security/coupling risk), forbids different runners, can't use `environment:` gates, can't fan out across services.

---

## Part 7: `actions/cache` -- The Walk-In Cooler

Caching is what separates a 30-second CI run from a 12-minute one. Every CI dependency download (`npm ci`, `pip install`, `mvn dependency:go-offline`, `terraform init` provider downloads) is doing the same work as the previous run. Cache it.

### Cache key composition

```yaml
- uses: actions/cache@v4
  id: cache
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}-v1
    restore-keys: |
      ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}-
      ${{ runner.os }}-node-
```

Three rules for keys:

1. **Include `${{ runner.os }}`.** A Linux-saved cache will not restore on Windows even with the same key (binary deps differ). Cross-OS pitfall = inflate cache mostly without restoring.
2. **Include a `hashFiles(...)` of the lockfile.** This is what makes the cache invalidate when dependencies change. `hashFiles('**/package-lock.json')` produces a SHA256 of all matching files; the SHA changes when any lockfile changes.
3. **Add a manual version segment (`-v1`).** When you need to force-invalidate the cache (e.g., dependency layout changed, deps weren't being saved correctly), bump it to `-v2`. This is the only way to "delete" all caches with that key prefix without going to the UI.

### `restore-keys:` -- the partial-prefix fallback hierarchy

```yaml
key: linux-node-abc123-v1                           # exact match preferred
restore-keys: |
  linux-node-abc123-                                # any key matching this prefix
  linux-node-                                       # any key matching this prefix
  linux-                                            # broadest fallback
```

The runner tries the exact `key` first. If not found, it tries each `restore-keys:` prefix in order, picking the **most-recently-saved cache that matches that prefix**. On a hit-via-fallback, `cache-hit` is `false` (signaling that you should re-save under the exact key after the build runs).

The pattern: use `restore-keys:` to accept a stale cache that's *close enough* (yesterday's `package-lock.json`-based cache while you wait for today's to be saved), then re-save under today's exact key for tomorrow.

### The immutability rule -- THE caching gotcha

**A cache key written once cannot be overwritten.** Once `linux-node-abc123-v1` exists in the cache, no future run can update its contents -- only read it. To "refresh" a cache, you must write to a *new* key (which is why lockfile-hash-based keys work: changing the lockfile produces a new hash and a new key).

The walk-in-cooler analogy is exact: once you've written "tomato confit, 2026-05-06" on the prep label, that container is sealed forever. You cannot relabel it. You must throw it out (or wait for LRU to evict) and prep a new container with a new label.

This explains why the `cache hit` log message on a `restore-keys` fallback hit is *not* automatically saving back under the exact key -- you must ALSO have `actions/cache` configured to save (which `@v4` does by default in a `post:` hook unless `save-always` is false or you used `cache/restore`).

### Eviction and scope

- **10 GB per repo, LRU eviction.** When you exceed 10 GB, the oldest-accessed cache is evicted.
- **7 days of inactivity** evicts a cache regardless of total size.
- **Branch scoping for read access**: a workflow run's cache scope includes its **current branch + its base branch (and ancestry)**. Feature branches see main's caches; main never sees feature-branch caches; sibling feature branches don't see each other's caches. This is the cache-poisoning protection: a malicious PR cannot write a cache that gets read on main, and one feature branch can't poison another.

### `setup-*` actions with built-in `cache:` -- the easier path

For language toolchains, do not write `actions/cache` by hand. The `setup-*` actions wire it for you:

**Critical distinction: `setup-node` `cache: 'npm'` caches `~/.npm` (the npm download cache), NOT `node_modules/`.** This is what makes it safe and content-addressable. The naive instinct is to cache `node_modules/` directly -- but `node_modules/` is a *materialized* dependency tree tied to a specific `package-lock.json` resolution; if you restore a stale `node_modules/` against a fresh lockfile, `npm ci`'s strict mode fails with peer-dep mismatches that don't reproduce locally (where `npm install` is forgiving). By caching the *download cache* instead, `setup-node` lets `npm ci` always rebuild a correct `node_modules/` from the current lockfile while skipping the network round-trips. This is the load-bearing reason `setup-node` `cache: 'npm'` works where the obvious "just cache `node_modules/`" approach silently corrupts builds. Same idea for `setup-python` (caches `~/.cache/pip`, not `site-packages/`), `setup-go` (caches `~/go/pkg/mod`, not `vendor/`), `setup-java` (caches `~/.m2/repository` or Gradle cache, not target/). **Cache the download index, never the materialized tree.**

```yaml
# Node
- uses: actions/setup-node@a1b2c3d
  with:
    node-version: '20'
    cache: 'npm'                                    # 'npm' | 'yarn' | 'pnpm'
    cache-dependency-path: '**/package-lock.json'   # Optional override

# Python
- uses: actions/setup-python@b2c3d4e
  with:
    python-version: '3.12'
    cache: 'pip'                                    # 'pip' | 'pipenv' | 'poetry'

# Java
- uses: actions/setup-java@c3d4e5f
  with:
    distribution: 'temurin'
    java-version: '21'
    cache: 'maven'                                  # 'maven' | 'gradle'

# Go
- uses: actions/setup-go@d4e5f6g
  with:
    go-version: '1.22'
    cache: true                                     # boolean for Go
```

These read the lockfile, compute a key with the OS + lockfile hash, and call `actions/cache` for you with sensible paths. Default to this for language deps; reach for raw `actions/cache` only for non-language artifacts (Docker layers, Terraform plugins, custom build outputs, downloaded test fixtures).

### `save-always:` -- save even on failure

```yaml
- uses: actions/cache@v4
  with:
    path: ./test-results
    key: test-results-${{ github.sha }}
    save-always: true                               # Save even if subsequent steps fail
```

Useful for saving partial progress / failure artifacts -- but **can poison the cache with broken state** if used carelessly on dependency caches. Production rule: only use `save-always: true` for diagnostic caches, never for build/dep caches.

---

## Part 8: Artifacts (`upload-artifact`/`download-artifact` v4)

Artifacts are the **plated-meal-handed-off-course-to-course** primitive. Job A builds a jar; Job B downloads the jar and signs it; Job C downloads the signed jar and pushes it to a registry. They live for the workflow run (default 90-day retention) and disappear after.

### v4 vs v3 -- the hard break

`actions/upload-artifact@v4` is a **breaking change** from v3. Three differences that bite every upgrade:

1. **v4 artifacts are immutable.** A given artifact name can only be uploaded once per workflow run. **Multiple jobs uploading to the same name now fail** (in v3, they appended). Each job must use a unique artifact name.
2. **v4 artifacts are available immediately after upload.** No more end-of-workflow wait. A job downstream of an upload sees the artifact as soon as the upload step finishes.
3. **v3 and v4 artifacts cannot be mixed in the same workflow.** If one step uses `@v3` and another uses `@v4`, downloads fail with confusing errors. Pick one and use it everywhere.

Plus: **500 artifacts per job limit**, **1 GB free tier of total artifact storage** (paid plans go higher), default **`retention-days: 90`** (configurable per artifact, max 90 on free / 400 on paid).

### Basic upload/download

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a1b2c3d
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist                                # Unique within the run
          path: |
            dist/
            !dist/**/*.map                          # Globs supported, ! for exclusion
          retention-days: 7                         # Override default 90
          if-no-files-found: error                  # error | warn | ignore (default warn)
          compression-level: 6                      # 0-9, default 6

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: ./dist                              # Where to extract
      - run: ls -la ./dist
```

### The v3 -> v4 fan-out matrix breakage

The most common upgrade-pain pattern: a matrix where N jobs each uploaded to the *same* artifact name (which v3 quietly merged into one combined artifact).

```yaml
# v3-style -- BREAKS in v4
jobs:
  build:
    strategy:
      matrix:
        platform: [linux, darwin, windows]
    steps:
      - uses: actions/upload-artifact@v3
        with:
          name: binaries                            # Same name in every cell -- v3 merged, v4 conflicts
          path: dist/${{ matrix.platform }}/
```

```yaml
# v4-correct -- unique name per cell, then merge on download
jobs:
  build:
    strategy:
      matrix:
        platform: [linux, darwin, windows]
    steps:
      - uses: actions/upload-artifact@v4
        with:
          name: binaries-${{ matrix.platform }}     # Unique per cell
          path: dist/${{ matrix.platform }}/

  package:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: binaries-*                       # Glob across artifact names
          merge-multiple: true                      # Combine into one path
          path: ./all-binaries
      - run: ls -la ./all-binaries                  # Contains all 3 platforms
```

The `pattern:` + `merge-multiple: true` combination is the v4-idiomatic way to recover the v3 multi-uploader-one-name pattern. Without `merge-multiple`, each artifact lands in its own subdirectory under `path:`.

### `tar` before upload for executable bits, symlinks, case-sensitive paths

Artifacts are **zipped server-side**, which strips Unix permissions to 644/755, flattens symlinks, and on Windows runners normalizes case-sensitive path collisions (`Foo.txt` vs `foo.txt` → one survives). For static-asset payloads going to S3/CloudFront, this is fine. For payloads that include compiled binaries with execute bits, symlinks, or case-sensitive directory layouts (Node ESM dual-package directories, certain Java JAR resources), **`tar -cf` before upload and `tar -xf` after download** preserves the full filesystem metadata:

```yaml
# Build job
- run: tar -cf bundle.tar ./build
- uses: actions/upload-artifact@v4
  with: { name: bundle, path: bundle.tar }

# Downstream job
- uses: actions/download-artifact@v4
  with: { name: bundle, path: . }
- run: tar -xf bundle.tar
```

Cost: one extra step on each side. Benefit: byte-for-byte fidelity.

### Artifacts vs cache -- the decision

The single most common confusion:

| Question | Use cache | Use artifact |
|----------|-----------|--------------|
| Do I want to share *across* runs? | YES | NO |
| Do I want to pass between jobs *in one* run? | NO | YES |
| Can it be regenerated if lost? | YES (cache is best-effort) | NO (artifact is the source of truth for that run) |
| Does it have a lockfile that determines correctness? | YES | NO (it's a build product) |
| Examples | `~/.npm`, `~/.m2`, Docker layers, Terraform `.terraform/` | `dist/`, `target/*.jar`, test-results.xml, coverage.xml, container.tar |

The wrong-tool-for-the-job pattern: **using cache to pass build outputs between jobs.** The cache might be evicted (10 GB LRU, 7-day inactivity), so the deploy job sometimes gets a stale build. Use artifacts for build outputs; they're guaranteed to be there for the run's lifetime.

---

## Part 9: Monorepo Path Filtering

Monorepos break the assumption that every change should rebuild everything. A change to `services/api/` should not run the `services/web/` test suite. GitHub Actions has two layers of path filtering.

### Workflow-level `on.push.paths:` -- coarse-grained

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'services/api/**'
      - 'shared/**'
      - '.github/workflows/api-ci.yml'             # Self-reference: workflow change re-runs
    paths-ignore:
      - 'docs/**'
      - '**.md'
```

If none of `paths:` match the diff (or if everything is in `paths-ignore:`), the **entire workflow does not run**. This is binary: workflow runs or doesn't.

The trade-off: **`paths:` filters are evaluated against the push's diff** (or the PR's full diff), so a single PR touching `api/` AND `web/` would trigger a workflow that filters on `api/**` even though `web/**` was the actual change focus. For multi-service monorepos, you usually want **per-job** path filtering.

### Per-job filtering with `dorny/paths-filter`

`dorny/paths-filter` is the de-facto solution because GitHub's native `paths:` only filters at the workflow level. It runs as a step in an early "detect-changes" job, computes which filter groups matched the diff, and emits boolean outputs that downstream jobs gate on.

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
      shared: ${{ steps.filter.outputs.shared }}
      packages: ${{ steps.filter.outputs.changes }}    # JSON list of which filters fired
    steps:
      - uses: actions/checkout@a1b2c3d
        with:
          fetch-depth: 0                                # CRITICAL: default fetch-depth: 1 cannot compute diffs on push
      - id: filter
        uses: dorny/paths-filter@b2c3d4e
        with:
          filters: |
            api:
              - 'services/api/**'
              - 'shared/**'
            web:
              - 'services/web/**'
              - 'shared/**'
            shared:
              - 'shared/**'

  test-api:
    needs: detect
    if: needs.detect.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a1b2c3d
      - run: cd services/api && npm test

  test-web:
    needs: detect
    if: needs.detect.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a1b2c3d
      - run: cd services/web && npm test
```

Three rules for `dorny/paths-filter`:

1. **`fetch-depth: 0` is mandatory for push events.** The default `fetch-depth: 1` only fetches the latest commit; the action cannot compute a diff against the previous commit and falls back to "everything changed" or fails outright. For PRs, the default works because the action diffs against the PR base. For pushes, you need full history.
2. **Filter values support glob patterns AND negations** (`!docs/**`).
3. **The `changes` output is a JSON array of filter names that matched.** This is what you `fromJson` for dynamic matrices.

### Conditional dynamic matrix from changed paths

The pattern: detect-job emits a JSON list of changed packages; downstream matrix job consumes it via `fromJson` and builds/tests only those packages.

```yaml
jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@a1b2c3d
        with: { fetch-depth: 0 }
      - id: filter
        uses: dorny/paths-filter@b2c3d4e
        with:
          filters: |
            api: 'services/api/**'
            web: 'services/web/**'
            worker: 'services/worker/**'

  test:
    needs: detect
    if: needs.detect.outputs.packages != '[]'         # Skip if nothing changed
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        package: ${{ fromJson(needs.detect.outputs.packages) }}
    steps:
      - uses: actions/checkout@a1b2c3d
      - run: cd services/${{ matrix.package }} && npm test
```

This is the cleanest expression of "test only what changed" -- one matrix cell per changed service.

### The "always-green-required-check" problem

GitHub's branch protection rules can require specific job names to pass. **A skipped job does NOT satisfy "required must pass" -- it shows as missing.** So if you require `test-api` but `test-api` skips because the diff was web-only, your PR is permanently blocked from merging.

**Fix: a status-aggregator job that returns success-on-skip.**

```yaml
jobs:
  detect: ...
  test-api: ...
  test-web: ...

  ci-pass:                                            # The job in branch-protection's required list
    needs: [detect, test-api, test-web]
    if: ${{ !cancelled() }}                           # Run on success OR failure, but NOT on user-cancelled workflow
    runs-on: ubuntu-latest
    steps:
      - if: contains(needs.*.result, 'failure')
        run: exit 1
      - if: contains(needs.*.result, 'cancelled')
        run: exit 1
      - run: echo "All required CI passed (or skipped)"
```

This sentinel job runs with `if: ${{ !cancelled() }}` and explicitly fails only if any upstream actually FAILED -- skipped is treated as success. Configure branch protection to require **`ci-pass`** instead of the individual jobs, and the merge-block problem disappears.

**Why `!cancelled()` and not `always()`.** `always()` runs even when the user clicks "Cancel workflow," which means a cancelled run could produce a green `ci-pass` and accidentally green-light a merge. `!cancelled()` evaluates true on success or failure but false on cancellation -- so the gate is reachable on every legitimate run outcome but does NOT report success when the user explicitly cancelled. This is the difference between "the gate runs no matter what" (wrong, lets cancellations pass) and "the gate runs on every real outcome" (right).

---

## Part 10: Production Gotchas

15 things that bite custom-action and reuse authors:

1. **Forgetting `shell: bash` in composite steps.** The #1 composite authoring bug. The error message (`Required property is missing: shell`) is clear enough that you only do this once -- but you WILL do it once.
2. **Forgetting to rebuild `dist/` after editing JS action source.** Workflow runs the old code; you debug for an hour. Mitigate with a `check-dist.yml` workflow that fails the PR if `dist/` differs from what `ncc` would produce.
3. **Committing `node_modules/`** in a JS action repo. Bloats the action's distribution; slows every consumer's checkout. Always bundle with ncc.
4. **Composite action expecting `secrets.X` to work.** It doesn't -- composites have no secrets context. The fix is to take inputs and have the caller resolve `${{ secrets.X }}` in `with:`.
5. **Reusable workflow `secrets: inherit` cross-org silently passing nothing.** The called workflow runs with empty `secrets.*`. Same-org-or-enterprise it usually works; cross-org it doesn't. Enumerate explicitly when crossing org boundaries.
6. **`type:` missing on `workflow_call.inputs`.** Workflow fails to load with a YAML schema error. There is no default type for `workflow_call` (unlike `workflow_dispatch`).
7. **`outputs.X.value` not wired in composite/reusable.** The output is empty with no error. Always check the value chain: step output -> job output -> workflow/action output.
8. **Cache key without `${{ runner.os }}`.** A Linux-saved cache won't restore on Windows; cross-OS matrix tests inflate cache and don't hit. Always include OS.
9. **Cache used for build artifacts that go between jobs.** Cache can be evicted (10 GB LRU, 7-day inactivity); deploy job sometimes sees a stale build. Use artifacts for cross-job handoff.
10. **`actions/upload-artifact@v3` mixed with `@v4`.** Confusing errors at download time. Pin to one version across the workflow.
11. **v4 fan-out matrix uploading to the same artifact name.** v3 silently merged; v4 errors on the second upload. Make names unique per cell + use `pattern:` + `merge-multiple: true` on download.
12. **`dorny/paths-filter` on push without `fetch-depth: 0`.** Action can't compute a diff; falls back to "everything changed" or errors. Always set fetch-depth: 0 on the checkout for push events.
13. **Required-status-checks blocking PRs because the path-filtered job skipped.** Skipped != passed. Add a status-aggregator job that returns success-on-skip and require *that* in branch protection.
14. **Reusable workflow not SHA-pinned in production.** A bad commit to the platform repo's `main` instantly breaks every product team. Pin to SHA.
15. **Docker action in a multi-OS matrix.** Linux-only constraint silently breaks macOS/Windows cells. Convert to JS or skip those cells with a matrix `exclude:`.

---

## Part 11: Decision Frameworks

### Framework 1: Which custom action type?

| Question | JS | Docker | Composite |
|----------|----|----|---|
| Cross-OS (Linux + macOS + Windows)? | YES | NO (Linux only) | YES |
| Heavy native deps / non-Node toolchain? | NO | YES | depends on inline tools |
| Pure YAML glue (3-7 steps)? | NO | NO | YES |
| Want fastest cold start? | YES | NO (image build/pull adds 30-90s) | YES (no separate process) |
| Need `pre`/`post` lifecycle hooks? | YES | YES (pre/post-entrypoint) | NO |
| Want zero JS/Docker authoring? | NO | NO | YES |
| Need to call multiple existing actions? | NO (would re-implement) | NO | YES (uses: inside) |
| Marketplace publishable? | YES | YES | YES |

**Default: composite for glue, JS for logic, Docker only when sealed toolchain matters.**

### Framework 2: Composite vs reusable workflow

| If you need... | Use |
|---|---|
| Different `runs-on:` per call | Reusable workflow |
| Proper `secrets:` block / `secrets: inherit` | Reusable workflow |
| Multi-job pipeline (fan-out / fan-in) | Reusable workflow |
| `environment:` gates with protection rules | Reusable workflow |
| Inline log integration in caller's job | Composite |
| Zero extra runner spin-up cost | Composite |
| Step-level setup ritual (3-7 steps) | Composite |
| Caller's filesystem state preserved | Composite |

### Framework 3: Cache vs artifact

| Use cache when... | Use artifact when... |
|---|---|
| Sharing across runs | Sharing across jobs in one run |
| Lossy is OK (regenerable) | Lossy is NOT OK (build product) |
| Lockfile-based invalidation | Build-output-based handoff |
| Examples: `~/.npm`, Docker layers, `~/.terraform.d/plugin-cache` | Examples: `dist/`, `*.jar`, `coverage.xml` |

### Framework 4: Manual `actions/cache` vs `setup-*` built-in cache

| Manual `actions/cache` | `setup-*` `cache:` parameter |
|---|---|
| Non-language artifacts (Docker layers, Terraform plugins, custom outputs) | Language deps (npm/pip/maven/gradle/go) |
| Custom key composition (multi-dimensional hashes) | Standard lockfile-based key |
| Multi-path caches with custom retention | One canonical cache path per language |

**Default: `setup-*` cache for language deps; manual `actions/cache` for everything else.**

### Framework 5: Path filtering -- workflow-level vs per-job

| Workflow-level `on.paths:` | Per-job `dorny/paths-filter` |
|---|---|
| Whole workflow runs or doesn't | Some jobs run, others skip |
| Single-service repo with selective triggers | Monorepo with multi-service CI |
| Coarse-grained ("docs change skips CI") | Fine-grained ("API change runs api tests only") |
| No status-aggregator needed | Status aggregator needed for required checks |

---

## Part 12: Putting It All Together -- A Production Monorepo CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  # ----- Detect changed services -----
  detect:
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
    steps:
      - uses: actions/checkout@a1b2c3d
        with: { fetch-depth: 0 }
      - id: filter
        uses: dorny/paths-filter@b2c3d4e
        with:
          filters: |
            api: 'services/api/**'
            web: 'services/web/**'

  # ----- Test only changed services (dynamic matrix) -----
  test:
    needs: detect
    if: needs.detect.outputs.packages != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        package: ${{ fromJson(needs.detect.outputs.packages) }}
    steps:
      - uses: ./.github/actions/setup-tools                # Local composite action
        with:
          node-version: '20'
          working-dir: services/${{ matrix.package }}
      - run: cd services/${{ matrix.package }} && npm test
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-${{ matrix.package }}
          path: services/${{ matrix.package }}/coverage.xml
          retention-days: 14
          if-no-files-found: warn

  # ----- Build and push image (only on main; reusable workflow) -----
  build-and-push:
    needs: [detect, test]
    if: github.ref == 'refs/heads/main' && needs.detect.outputs.api == 'true'
    uses: org/platform/.github/workflows/build-and-push.yml@a1b2c3d4e5f
    with:
      image-name: api
      environment: staging
    secrets: inherit
    permissions:
      id-token: write
      contents: read

  # ----- Status aggregator: required check that survives skips -----
  ci-pass:
    needs: [detect, test, build-and-push]
    if: ${{ !cancelled() }}                           # NOT always() -- don't green-light cancelled runs
    runs-on: ubuntu-latest
    steps:
      - if: contains(needs.*.result, 'failure')
        run: exit 1
      - if: contains(needs.*.result, 'cancelled')
        run: exit 1
      - run: echo "All required CI passed"
```

This composes everything: composite action for setup glue, reusable workflow for build-and-push, dynamic matrix from path filter, artifacts for coverage handoff, status aggregator to satisfy required checks. Branch protection requires `ci-pass`.

---

## Part 13: 2026 Currency Notes

- **Node 20 is the default for JS actions.** Node 16 EOL'd in Sept 2023; the runner removed Node 16 support. New actions should `runs.using: node20` (or `node22`, also accepted). Node 24 is in preview but not yet the default in 2026.
- **Composite `shell:` mandatory.** Has been since composite actions launched; no change.
- **`actions/upload-artifact@v4` and `actions/download-artifact@v4`** are current. v3 is deprecated and being removed in scheduled brownouts; migrate now if you haven't.
- **`actions/cache@v4`** is current.
- **`dorny/paths-filter`** -- still the de-facto monorepo path-filter solution; no first-party GitHub equivalent in 2026.
- **SHA-pinning is the strong recommendation** post-Tj-actions/changed-files supply-chain incident (March 14-15, 2025; CVE-2025-30066). The Dependabot config can keep SHA-pinned actions auto-updating with PRs to bump.
- **`@actions/core` and `@actions/github`** continue to be the canonical toolkit packages; API has been stable since 2022.
- **`@vercel/ncc`** remains the standard JS-action bundler. Alternatives (`esbuild`, `rollup`) work but ncc is the documented path.

---

## The 10 Commandments of Custom Actions & Reusable Patterns

1. **Pick the right reuse primitive.** Composite for step glue; reusable workflow for multi-job pipelines; JS for logic; Docker only when the toolchain is sealed.
2. **Always declare `shell: bash` on every composite `run:` step.** No exceptions.
3. **Bundle JS actions with `@vercel/ncc`** and commit `dist/`; gitignore `node_modules/`. Verify `dist/` freshness in CI.
4. **Treat `action.yml` as an API contract** -- inputs are parameters, outputs are return values. Document each with `description:`. All inputs arrive as strings.
5. **Pass secrets as inputs in composites** (the caller resolves `${{ secrets.X }}`); use proper `secrets:` blocks in reusable workflows. Use `secrets: inherit` only within an org.
6. **Wire outputs explicitly** in composite (`outputs.X.value: ${{ steps.X.outputs.Y }}`) and reusable workflows (two-layer: step -> job -> workflow). Forgetting the wiring returns empty strings.
7. **SHA-pin reusable workflows AND third-party actions in production.** Tags are mutable; Tj-actions is the canonical lesson.
8. **Use the right persistence layer**: cache for cross-run dependency reuse (lossy OK), artifacts for cross-job handoff in one run (lossy NOT OK). Don't conflate.
9. **`fetch-depth: 0` on checkout** when using `dorny/paths-filter` on push events. Always.
10. **Add a status-aggregator job** with `if: ${{ !cancelled() }}` for monorepos with required checks. Skipped does not equal passed -- and `!cancelled()` (not `always()`) prevents cancelled runs from green-lighting merges.

---

## Cheat Sheet

```yaml
# ---------- COMPOSITE ACTION SKELETON ----------
# .github/actions/X/action.yml
name: 'X'
description: '...'
inputs:
  foo:
    description: '...'
    required: true
outputs:
  result:
    description: '...'
    value: ${{ steps.s.outputs.r }}
runs:
  using: composite
  steps:
    - id: s
      shell: bash                        # MANDATORY
      run: echo "r=ok" >> $GITHUB_OUTPUT

# ---------- JS ACTION SKELETON ----------
# action.yml
runs:
  using: node20
  main: dist/index.js
  pre: dist/setup.js                     # optional
  post: dist/cleanup.js                  # optional

# src/index.ts
import * as core from '@actions/core';
const x = core.getInput('foo', { required: true });
core.setSecret(x);
core.setOutput('result', 'ok');
core.setFailed('msg');                   // sets exit 1

# ---------- DOCKER ACTION SKELETON ----------
runs:
  using: docker
  image: 'docker://ghcr.io/me/img:1.2'   # or 'Dockerfile'
  args: ['--target', '${{ inputs.target }}']

# ---------- REUSABLE WORKFLOW SKELETON ----------
on:
  workflow_call:
    inputs:
      env:
        type: string                     # MANDATORY: string|boolean|number
        required: true
    secrets:
      KEY:
        required: true
    outputs:
      tag:
        value: ${{ jobs.b.outputs.t }}

# ---------- CALLING REUSABLE WORKFLOW ----------
jobs:
  b:
    uses: org/platform/.github/workflows/x.yml@<sha>
    with: { env: prod }
    secrets: inherit                     # or enumerate explicitly

# ---------- CACHE (manual) ----------
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}-v1
    restore-keys: |
      ${{ runner.os }}-node-

# ---------- CACHE (built-in) ----------
- uses: actions/setup-node@<sha>
  with: { node-version: '20', cache: 'npm' }

# ---------- ARTIFACT v4 UPLOAD/DOWNLOAD ----------
- uses: actions/upload-artifact@v4
  with: { name: dist-${{ matrix.os }}, path: dist/ }
- uses: actions/download-artifact@v4
  with: { pattern: dist-*, merge-multiple: true, path: ./all }

# ---------- PATH FILTER + DYNAMIC MATRIX ----------
- uses: actions/checkout@<sha>
  with: { fetch-depth: 0 }                 # MANDATORY for push
- id: f
  uses: dorny/paths-filter@<sha>
  with:
    filters: |
      api: 'services/api/**'
      web: 'services/web/**'
# downstream:
strategy:
  matrix:
    package: ${{ fromJson(needs.detect.outputs.packages) }}

# ---------- STATUS AGGREGATOR ----------
ci-pass:
  needs: [a, b, c]
  if: ${{ !cancelled() }}                # NOT always() -- cancelled runs must NOT green-light merge
  runs-on: ubuntu-latest
  steps:
    - if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
      run: exit 1
    - run: echo ok
```

---

## Cross-Links

- **Yesterday's GitHub Actions doc**: [`2026-05-05-github-actions-advanced-workflows.md`](./2026-05-05-github-actions-advanced-workflows.md) for the workflow/job/step model, contexts, conditionals, concurrency, matrices.
- **ArgoCD GitOps**: [`2026-04-16-argocd-gitops-fundamentals.md`](../../april/kubernetes/2026-04-16-argocd-gitops-fundamentals.md) for what consumes the images this CI builds.
- **ArgoCD + Helm production patterns**: [`2026-04-19-argocd-helm-production-patterns.md`](../../april/kubernetes/2026-04-19-argocd-helm-production-patterns.md) -- the image-tag-update bridge from CI to CD via Image Updater or PR-back-to-config-repo.
- **Helm Advanced Patterns**: [`2026-04-15-helm-advanced-patterns.md`](../../april/kubernetes/2026-04-15-helm-advanced-patterns.md) for OCI registries (which CI typically pushes to) and chart testing patterns.
- **Tomorrow (Day 3 of Week 6.8)**: Container CI -- multi-stage builds with `docker/build-push-action` + cache-to/cache-from, OIDC to AWS for keyless ECR push, Trivy scanning, cosign image signing.
